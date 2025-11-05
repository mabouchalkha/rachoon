# LaFacture/Rachoon - Complete Technical Specifications

## Table of Contents
1. [System Architecture](#system-architecture)
2. [Authentication & Authorization](#authentication--authorization)
3. [Multi-tenancy](#multi-tenancy)
4. [Core Business Entities](#core-business-entities)
5. [Document Management](#document-management)
6. [Client Management](#client-management)
7. [Template System](#template-system)
8. [Recurring Invoices](#recurring-invoices)
9. [PDF/Document Rendering](#pdfdocument-rendering)
10. [Validation Rules](#validation-rules)
11. [API Endpoints](#api-endpoints)
12. [Business Logic & Calculations](#business-logic--calculations)
13. [Database Schema](#database-schema)
14. [Numbering System](#numbering-system)
15. [ID Obfuscation](#id-obfuscation)
16. [Pagination, Search, Sort, Filter](#pagination-search-sort-filter)

---

## 1. System Architecture

### Technology Stack

**Backend:**
- Framework: AdonisJS 5.9 (Node.js TypeScript)
- Database: PostgreSQL with Lucid ORM
- Authentication: OAT (Opaque Access Tokens)
- PDF Generation: Gotenberg (external service via HTTP)
- Image Processing: Sharp
- PDF Parsing: PDFium
- Template Engine: Nunjucks
- Password Hashing: Argon2
- Date/Time: Luxon, date-fns
- ID Obfuscation: Sqids (HashIDs)

**Frontend:**
- Framework: Nuxt 3 + Vue 3
- State Management: Pinia
- Styling: Tailwind CSS + DaisyUI
- Rich Text: TipTap
- Build: Vite

**Architecture Pattern:**
- Monorepo structure (Turbo + pnpm)
- REST API backend
- SPA frontend
- Shared common package for types and business logic
- Multi-tenant design with organization-level data isolation

### Project Structure
```
rachoon/
├── apps/
│   ├── backend/          # AdonisJS API
│   │   ├── app/
│   │   │   ├── Controllers/Http/    # API endpoints
│   │   │   ├── Models/              # Database models
│   │   │   ├── Services/            # Business logic
│   │   │   ├── Validators/          # Request validation
│   │   │   ├── Middleware/          # HTTP middleware
│   │   │   └── Helpers/             # Utilities
│   │   ├── database/migrations/     # Schema migrations
│   │   ├── config/                  # Configuration
│   │   └── start/                   # Bootstrap (routes, kernel)
│   └── frontend/         # Nuxt 3 SPA
│       ├── pages/                   # Route pages
│       ├── components/              # Vue components
│       ├── composables/             # API client, state
│       └── models/                  # TypeScript models
└── packages/
    └── common/           # Shared code
        └── src/
            ├── Document.ts          # Document business logic
            ├── Client.ts            # Client model
            ├── Format.ts            # Formatting utilities
            └── Locale.ts            # Translations
```

---

## 2. Authentication & Authorization

### Registration Flow

**Endpoint:** `POST /api/register`

**Process:**
1. Validate organization slug (must be unique, cannot be 'app', 'rachoon', 'api', 'www')
2. Validate user email (must be unique globally)
3. Create organization record
4. Create first user with ADMIN role
5. Password is hashed with Argon2 on save
6. Auto-generate username from fullName if not provided

**Validation Rules:**
```typescript
{
  organization: {
    name: required string,
    slug: required string, unique, not in reserved words
  },
  user: {
    email: required, valid email, unique globally,
    password: required string,
    data: {
      fullName: required string
    }
  }
}
```

### Login Flow

**Endpoint:** `POST /api/auth`

**Process:**
1. Extract organization from request context (likely from subdomain or header)
2. Find user by email + organizationId combination
3. Verify password using Argon2
4. Return OAT token via `ctx.auth.use('api').attempt()`
5. Token is used as Bearer token in subsequent requests

**Important:** Users are scoped to organizations - same email can exist in different organizations

**Endpoint:** `DELETE /api/auth` (requires auth)
- Logs out user, invalidates token

### Authorization

**User Roles (enum):**
- `ADMIN` = highest privileges
- `USER` = standard user
- `EDITOR` = intermediate (between USER and ADMIN)

**Role Validation:**
- Role must be in range: EDITOR-1 to ADMIN+1
- Only admins can create/manage other users (implied by business logic)

**Multi-tenant Security:**
- ALL queries filtered by `organizationId` from authenticated user
- Organization extracted from context (subdomain, header, or token)
- No cross-organization data access possible

---

## 3. Multi-tenancy

### Implementation

**Organization Isolation:**
- Every major entity has `organization_id` foreign key
- All queries automatically scoped to user's organization
- Database indexes on `organization_id` for performance

**Organization Context:**
- Extracted via `OrganizationHelper.getFromContext(ctx)`
- Likely based on:
  - Subdomain (e.g., `acme.lafacture.com`)
  - Custom header
  - Token payload

**Organization Settings:**
- Stored as JSONB in `organizations.settings`
- Contains:
  - General settings (locale, currency)
  - Document numbering formats (invoices, offers, reminders, clients)
  - Document titles per type
  - Custom configurations

---

## 4. Core Business Entities

### Organization
```typescript
{
  id: number (PK),
  name: string,
  slug: string (unique),
  data: JSONB,          // Custom metadata
  settings: JSONB,      // App settings
  createdAt: DateTime,
  updatedAt: DateTime,
  deletedAt: DateTime?  // Soft delete
}
```

**Settings Structure:**
```typescript
{
  general: {
    locale: string,      // e.g., "en-US", "fr-FR"
    currency: string     // e.g., "USD", "EUR"
  },
  invoices: {
    title: string,
    number: {
      format: string,    // e.g., "INV-{number}-{date:yyyy}"
      padZeros: number   // e.g., 5 → "00001"
    }
  },
  offers: { ... },       // Same structure
  reminders: { ... },    // Same structure
  clients: {
    number: { ... }
  }
}
```

### User
```typescript
{
  id: number (PK),
  email: string (unique per org),
  password: string (Argon2 hashed),
  role: UserRole,
  data: JSONB {
    fullName: string,
    username: string,
    avatar?: string,
    rate?: number        // Hourly rate for time tracking
  },
  settings: JSONB,       // User preferences
  organizationId: number (FK),
  createdAt: DateTime,
  updatedAt: DateTime,
  deletedAt: DateTime?
}
```

**Computed Properties:**
- `isAdmin`: returns `role === UserRole.ADMIN`

**Auto-generation:**
- Username generated from fullName: `fullName.replace(' ', '').toLowerCase()`

### Client
```typescript
{
  id: number (PK),
  name: string,
  number: string (sequential, unique per org),
  organizationId: number (FK),
  data: JSONB {
    address: {
      street: string,
      zip: string,
      city: string,
      country: string
    },
    info: {
      vat?: string,
      addition?: string
    },
    contactPerson: {
      fullName?: string,
      email: string (required, validated)
    },
    conditions: {
      earlyPayment?: {
        days?: number,
        discount?: number
      },
      invoiceDueDays?: number,
      rate?: number,
      discount?: {
        value?: number,
        valueType: 'percent' | 'fixed'
      }
    }
  },
  createdAt: DateTime,
  updatedAt: DateTime,
  deletedAt: DateTime?
}
```

**Validation:**
- Name: required
- Email: required, valid email format
- Address fields: all required strings
- Conditions: all optional

**Computed Extras (from queries):**
- `totalInvoices`: count of invoices
- `pendingInvoices`: count of pending invoices past due date
- `totalOffers`: count of offers
- `pendingOffers`: count of pending offers
- `totalReminders`: count of reminders
- `invoicesTotal`: sum of invoice totals

### Document (Invoice, Offer, Reminder)
```typescript
{
  id: number (PK),
  type: DocumentType,           // 1=Invoice, 2=Offer, 3=Reminder
  number: string (sequential per type/org),
  status: DocumentStatus,       // 0=Draft, 1=Pending, 2=Accepted, 3=Paid, 4=Overdue
  clientId: number (FK),
  organizationId: number (FK),
  templateId?: number (FK),
  offerId?: number (FK),        // For invoices converted from offers
  invoiceId?: number (FK),      // For reminders linked to invoices
  recurringId?: number (FK),    // For docs created from recurring
  data: JSONB {                 // See DocumentData below
    title: string,
    positions: Position[],
    discountsCharges: DiscountCharge[],
    taxes: { [rate: string]: number },
    taxOption: TaxOption,
    date: Date,
    dueDate: Date,
    dueDays: number,
    headingText?: string,
    footerText?: string,
    total: number,              // Calculated
    net: number,                // Calculated
    netNoDiscount: number       // Calculated
  },
  createdAt: DateTime,
  updatedAt: DateTime,
  deletedAt: DateTime?
}
```

**Relationships:**
- `client`: BelongsTo Client
- `organization`: BelongsTo Organization
- `template`: HasOne Template
- `offer`: BelongsTo Document (self-reference via offerId)
- `overdueInvoice`: BelongsTo Document (via invoiceId)
- `invoices`: HasMany Document (for offers converted to invoices)
- `reminders`: HasMany Document (for invoice reminders)
- `recurringInvoice`: HasOne RecurringInvoice

**Computed Properties:**
- `overdue`: `isPast(data.dueDate)`
- `isFromRecurring`: `!!recurringId`
- `isRecurring`: `!!recurringInvoice`

**Lifecycle Hook:**
- `@beforeSave()`: Automatically recalculates all financial fields using `Document.rebuild()`

### Template
```typescript
{
  id: number (PK),
  title: string,
  html: string (Nunjucks template),
  thumbnail: string (base64 PNG image),
  default: boolean,
  premium: boolean,
  organizationId?: number (FK, null for global templates),
  data: JSONB,
  createdAt: DateTime,
  updatedAt: DateTime,
  deletedAt: DateTime?
}
```

**Template Types:**
- Global templates: `organizationId = null`, visible to all orgs
- Organization templates: `organizationId = X`, only visible to that org
- Default template: `default = true`, auto-selected for new docs
- Premium templates: `premium = true`, may require paid plan

**Business Rules:**
- Only ONE template can be default per organization
- Global templates cannot be edited/deleted by users
- When setting a template as default, all other org templates set to `default=false`
- Thumbnail auto-generated on create/update using example invoice data

### RecurringInvoice
```typescript
{
  id: number (PK),
  invoiceId: number (FK to documents),
  organizationId: number (FK),
  cron: string,                 // Cron expression
  startDate: Date,
  nextRun: Date,
  active: boolean,
  deletedAt?: Date
}
```

**Business Rules:**
- One recurring config per invoice
- Cron expression defines frequency
- When executed, creates new invoice from template invoice
- `nextRun` updated after each execution based on cron schedule

---

## 5. Document Management

### Document Types (enum)
```typescript
enum DocumentType {
  Invoice = 1,
  Offer = 2,
  Reminder = 3
}
```

### Document Status (enum)
```typescript
enum DocumentStatus {
  Draft = 0,
  Pending = 1,
  Accepted = 2,
  Paid = 3,
  Overdue = 4
}
```

### Document Data Structure

**Position (line item):**
```typescript
{
  id: number,
  title: string,
  text: string (description),
  quantity: number,
  unit: string,
  price: number,
  tax: number (percentage),
  taxPrice: number (calculated),
  discount: number (percentage),
  net: number (calculated),
  netNoDiscount: number (calculated),
  total: number (calculated),
  focused: boolean,
  totalPercentage: number (calculated - position's % of total)
}
```

**DiscountCharge:**
```typescript
{
  id?: string,
  title: string,
  value: number,
  type: 'discount' | 'charge',
  valueType: 'percent' | 'fixed',
  amount: number (calculated)
}
```

**TaxOption:**
```typescript
{
  title: string,
  applicable: boolean,        // Whether taxes apply
  default: boolean
}
```

### Document Calculation Logic

**Executed automatically before save via `@beforeSave()` hook**

**Step 1: Calculate Positions**
```typescript
calcPositions() {
  // 1. Calculate sum of all position prices (before position-level discounts)
  sumPositions = Σ(position.quantity * position.price)

  // 2. For each position:
  position.net = quantity * price
  position.netNoDiscount = quantity * price

  // 3. Apply position-level discount
  if (position.discount > 0) {
    position.net -= (position.net / 100) * position.discount
  }

  // 4. Calculate tax on position
  if (taxOption.applicable) {
    position.taxPrice = (position.net / 100) * position.tax
  } else {
    position.taxPrice = 0
  }

  // 5. Position total
  position.total = position.net + position.taxPrice

  // 6. Calculate position's percentage of total (for proportional distribution)
  position.totalPercentage = (100 / sumPositions) * position.net

  // 7. Calculate document-level discounts/charges
  discountsCharges.forEach(dc => {
    dc.amount = dc.valueType === 'percent'
      ? (sumPositions / 100) * dc.value
      : dc.value

    // Add to sum (negative for discounts, positive for charges)
    if (dc.type === 'discount') sumDiscountsCharges -= dc.amount
    else sumDiscountsCharges += dc.amount
  })

  // 8. Distribute document-level discounts/charges proportionally to positions
  positions.forEach(position => {
    position.net += (sumDiscountsCharges / 100) * position.totalPercentage

    // Recalculate tax and total after distribution
    if (taxOption.applicable) {
      position.taxPrice = (position.net / 100) * position.tax
    }
    position.total = position.net + position.taxPrice
  })
}
```

**Step 2: Calculate Taxes**
```typescript
calcTaxes() {
  taxes = {}

  if (taxOption.applicable) {
    // Group tax amounts by rate
    positions.forEach(position => {
      if (!taxes[position.tax]) taxes[position.tax] = 0
      taxes[position.tax] += position.taxPrice
    })
  }

  // Result: { "19": 152.50, "7": 35.00 }
}
```

**Step 3: Calculate Net**
```typescript
calcNet() {
  net = Σ(position.net)
}
```

**Step 4: Calculate Total**
```typescript
calcTotal() {
  total = net

  if (taxOption.applicable) {
    // Add all tax amounts
    total += Σ(Object.values(taxes))
  }

  // Round to 2 decimals
  total = Math.round(total * 100) / 100
}
```

### CRUD Operations

**Create Document:**
`POST /api/documents?type={type}`

1. Validate request body
2. Generate sequential number for document type
3. Create document with status Draft
4. If `recurringInvoice` data provided, create recurring config
5. Return document

**Update Document:**
`PUT /api/documents/:id`

1. Verify document belongs to user's organization
2. Validate request body
3. Merge changes
4. Recalculate on save (via hook)
5. Update recurring config if provided
6. Return updated document

**Delete Document:**
`DELETE /api/documents/:id`

- Soft delete (sets `deletedAt`)
- Document remains in database but excluded from queries

**Duplicate Document:**
`GET /api/documents/duplicate/:id`

1. Load original document
2. Create new document with same data
3. Generate new number
4. Set date to now
5. Set dueDate to now + dueDays
6. Clear id, createdAt, updatedAt
7. If recurringId provided, link to recurring
8. Save and return

**Update Status:**
`PATCH /api/documents/status`

- Body: `{ status: DocumentStatus }`
- Simple status change without validation

### Offer-to-Invoice Conversion

**Conversion Options:**

1. **Full Conversion** (`ConvertOption.Full`)
   - Copy all positions as-is
   - Copy all discounts/charges
   - Link invoice to offer via `offerId`

2. **Partial Conversion** (`ConvertOption.Partial`)
   - Adjust position prices by percentage or fixed amount
   - If percent: `newPrice = (oldPrice / 100) * value`
   - If fixed: `newPrice = ((totalValue / 100) * position.totalPercentage) / quantity`
   - Copy discounts/charges

3. **Final Conversion** (`ConvertOption.Final`)
   - Calculate remaining amount: `newNet = offer.net - Σ(previous_invoices.net)`
   - Distribute remaining proportionally to positions
   - `newPrice = ((newNet / 100) * position.totalPercentage) / quantity`
   - Copy discounts/charges

**Implementation:**
```typescript
calculateInvoiceToConvertPositions(offer, option, value, valueType) {
  this.data.taxOption = offer.data.taxOption
  this.removePositions()

  if (option === ConvertOption.Partial) {
    // Adjust prices by value
  }
  else if (option === ConvertOption.Full) {
    // Copy all positions
  }
  else if (option === ConvertOption.Final) {
    // Calculate remaining amount
  }

  // Copy discounts/charges
  offer.data.discountsCharges.forEach(dc => {
    this.addDiscountCharge({ ...dc })
  })
}
```

### List/Query Documents

**Endpoint:** `GET /api/documents?type={type}&page={p}&perPage={pp}&q={search}&sort[field]=asc&filter[field][operator]=value`

**Query Features:**

1. **Filtering:**
   - By organization (automatic)
   - By type (required)
   - By clientId (optional)
   - Custom filters via `filter` parameter

2. **Searching:**
   - Full-text search on: number, dueDate, net, total
   - Uses `q` parameter

3. **Sorting:**
   - Sortable fields: number, status, dueDate, net, total
   - Via `sort[field]=asc|desc`

4. **Pagination:**
   - `page`: page number (default 1)
   - `perPage`: records per page (default 20)
   - Response headers: `x-total`, `x-page`, `x-per-page`, `x-pages`

5. **Relationships preloaded:**
   - client
   - offer
   - invoices
   - overdueInvoice
   - recurringInvoice

6. **Count:**
   - `?count=true` returns total count including soft-deleted

---

## 6. Client Management

### CRUD Operations

**Create Client:**
`POST /api/clients`

1. Validate client data
2. Generate sequential client number
3. Create client in user's organization
4. Return client

**Update Client:**
`PUT /api/clients/:id`

1. Verify belongs to organization
2. Validate data
3. Update record
4. Return success

**Delete Client:**
`DELETE /api/clients/:id`

- Soft delete (sets `deletedAt`)
- Cascades to documents (database FK)

**Get Client:**
`GET /api/clients/:id`

- Returns client with all fields

**List Clients:**
`GET /api/clients?page={p}&perPage={pp}&q={search}`

**Query Features:**
- Pagination (default: 20 per page)
- Search on: name, number
- Sort on: name, number, totalInvoices, totalReminders, totalOffers
- Filter by custom criteria

**Aggregations:**
Each client includes computed fields:
- `totalInvoices`: count
- `pendingInvoices`: count of invoices past due with Pending status
- `totalOffers`: count
- `pendingOffers`: count of offers past due with Pending status
- `totalReminders`: count
- `pendingReminders`: count of reminders past due with Pending status
- `invoicesTotal`: sum of invoice totals

### Client Conditions

Clients can have default conditions applied to documents:

```typescript
conditions: {
  earlyPayment: {
    days: number,        // Early payment period
    discount: number     // Discount % for early payment
  },
  invoiceDueDays: number,     // Default due days for invoices
  rate: number,               // Default rate/price
  discount: {
    value: number,
    valueType: 'percent' | 'fixed'
  }
}
```

These can be applied when creating documents for the client.

---

## 7. Template System

### Template Structure

Templates use **Nunjucks** templating engine with custom context:

**Available Context Variables:**
```typescript
{
  document: Document,        // Full document object with calculated fields
  template: Template,        // Template metadata
  organization: Organization,// Org details & settings
  title: string,            // Document title from org settings
  user: User,               // Current user
  t: (key, ...args) => string,  // Translation function
  format: {
    currency: (value) => string,    // Format as currency
    date: (value) => string,        // Short date format
    longDate: (value) => string     // Long date format
  }
}
```

**Example Template Usage:**
```html
<h1>{{ title }}</h1>
<p>{{ t('document.number') }}: {{ document.number }}</p>
<p>{{ t('document.date') }}: {{ format.date(document.data.date) }}</p>

<table>
  {% for position in document.data.positions %}
  <tr>
    <td>{{ position.title }}</td>
    <td>{{ position.quantity }} {{ position.unit }}</td>
    <td>{{ format.currency(position.price) }}</td>
    <td>{{ format.currency(position.total) }}</td>
  </tr>
  {% endfor %}
</table>

<p>{{ t('document.total') }}: {{ format.currency(document.data.total) }}</p>
```

### Template Operations

**Create Template:**
`POST /api/templates`

1. Validate template data
2. Create template in organization
3. Generate thumbnail (render with example data, convert to PNG, scale down)
4. If `default=true`, unset default flag on other org templates
5. Return template

**Update Template:**
`PUT /api/templates/:id`

1. Verify template belongs to org (or is global)
2. Validate data
3. Regenerate thumbnail
4. If `default=true`, unset default on others
5. Save and return

**Delete Template:**
`DELETE /api/templates/:id`

- Cannot delete if `default=true`
- Cannot delete global templates
- Soft delete

**Get Default Template:**
`GET /api/templates/default`

- Returns organization's default template
- Falls back to global default if none set

**Duplicate Template:**
`GET /api/templates/duplicate/:id`

1. Load template (org or global)
2. Create new template with same HTML
3. Set title to "Original Title (Copy)"
4. Set `organizationId` to current user's org
5. Set `default=false` and `premium=false`
6. Save and return

### Thumbnail Generation

**Process:**
1. Prepare HTML with example invoice data
2. Send to Gotenberg to convert HTML → PDF
3. Use PDFium to render PDF → PNG image
4. Use Sharp to resize image (downscale by factor 10)
5. Convert to base64 data URL
6. Store in `template.thumbnail`

**Used for:** Template selection UI in frontend

---

## 8. Recurring Invoices

### Configuration

**Endpoint:** `POST /api/invoices/recurring`

**Request Body:**
```typescript
{
  active: boolean,
  invoiceId: number,        // Template invoice to duplicate
  cron: string,             // Cron expression: "0 9 1 * *" = 1st of month at 9am
  startDate: Date
}
```

**Validation:**
- `active`: required boolean
- `invoiceId`: required number
- `cron`: required string (cron format)
- `startDate`: required date

**Note:** There's a typo in validator: `actcive` instead of `active`

### Execution

**Endpoint:** `GET /api/run/recurring` (should be protected, currently not)

**Process:**
1. Find all recurring invoices where:
   - `startDate <= today`
   - `nextRun <= today`
   - `active = true`

2. For each recurring:
   - Parse cron expression
   - Calculate next run date
   - Check if invoice already created for this period
   - If not, duplicate template invoice with `recurringId` set
   - Update `nextRun` to next cron date
   - Save recurring config

**Duplicate Logic:**
- Uses `DocumentService.duplicate(invoiceId, organizationId, recurringId)`
- Sets new document number
- Sets date to now
- Sets dueDate to now + dueDays
- Links to recurring via `recurringId`

**Cron Scheduling:**
- External cron job calls this endpoint
- Idempotent: won't create duplicate invoices
- Updates `nextRun` after execution

---

## 9. PDF/Document Rendering

### Rendering Service

**Endpoint:** `POST /api/render?preview={true|false}`

**Request:**
```typescript
{
  templateId?: number,
  data: DocumentData     // Full document data
}
```

**Process:**

1. **Prepare HTML:**
   - Load template (or use org default)
   - Get organization settings (locale, currency)
   - Create translation function `t(key, ...args)`
   - Create format functions (currency, date, longDate)
   - Render Nunjucks template with context

2. **Convert to PDF:**
   - Send HTML to Gotenberg service
   - POST to `{GOTENBERG_URL}/forms/chromium/convert/html`
   - Include options:
     - `printBackground: true`
     - `marginTop/Bottom/Left/Right: 0`
     - `preferCssPageSize: true`
     - `generateDocumentOutline: true`
   - Receive PDF buffer

3. **Convert to Image (if preview):**
   - Use PDFium to load PDF
   - Render each page at scale=3
   - Use Sharp to resize (downscale)
   - Convert to PNG
   - Return as data URL: `data:image/png;base64,...`

4. **Return:**
   - If preview: array of PNG data URLs (one per page)
   - If not preview: array with single PDF data URL

**Gotenberg Service:**
- Runs in separate container
- HTML to PDF conversion via Chrome headless
- Environment variable: `GOTENBERG_URL` (default: `http://gotenberg`)

---

## 10. Validation Rules

### Document Validation

```typescript
{
  clientId: number (required),
  number: string (required),
  status: number (required),
  offerId?: number (optional),
  templateId?: number (optional),
  invoiceId?: number (optional),
  recurringInvoice?: object (optional),
  data: {
    positions: array (required),
    discountsCharges?: array (optional),
    taxes: object (required),
    taxOption: object (required),
    date: date (required),
    dueDate: date (required),
    headingText?: string (optional),
    footerText?: string (optional),
    total: number (required),
    net: number (required),
    netNoDiscount: number (required),
    dueDays: number (required)
  }
}
```

### Client Validation

```typescript
{
  name: string (required),
  number: string (required),
  data: {
    address: {
      street: string (required),
      zip: string (required),
      city: string (required),
      country: string (required)
    },
    info?: {
      vat?: string,
      addition?: string
    },
    contactPerson: {
      fullName?: string,
      email: string (required, valid email format)
    },
    conditions?: {
      earlyPayment?: {
        days?: number,
        discount?: number
      },
      invoiceDueDays?: number,
      discount?: {
        value?: number,
        valueType: 'percent' | 'fixed' (optional)
      },
      rate?: number
    }
  }
}
```

### User Validation

**Profile Update:**
```typescript
{
  email: string (required, valid email, unique within org excluding self),
  data: {
    fullName: string (required),
    username: string (required),
    avatar?: string
  }
}
```

**User Create/Update:**
```typescript
{
  email: string (required, valid email, unique within org),
  role: number (required, range: EDITOR-1 to ADMIN+1),
  password: string (required on create, optional on update),
  data: {
    fullName: string (required),
    username: string (required),
    avatar?: string,
    rate?: number
  }
}
```

### Registration Validation

```typescript
{
  organization: {
    name: string (required),
    slug: string (required, unique, not in ['app', 'rachoon', 'api', 'www'])
  },
  user: {
    email: string (required, valid email, unique globally),
    password: string (required),
    data: {
      fullName: string (required)
    }
  }
}
```

### Template Validation

```typescript
{
  title: string (required),
  html: string (required),
  default?: boolean,
  premium?: boolean,
  data?: object
}
```

### Recurring Invoice Validation

```typescript
{
  active: boolean (required),
  invoiceId: number (required),
  cron: string (required),
  startDate: date (required)
}
```

---

## 11. API Endpoints

### Public Endpoints

```
GET    /api                        - API root info
GET    /api/info                   - App info
GET    /api/password/generate      - Generate random password
POST   /api/register               - Register organization + admin user
POST   /api/auth                   - Login
```

### Protected Endpoints (require Bearer token)

**Authentication:**
```
DELETE /api/auth                   - Logout
```

**Documents:**
```
GET    /api/documents?type={type}&page={p}&perPage={pp}&q={q}&sort[field]=dir&filter[field][op]=val
POST   /api/documents?type={type}
GET    /api/documents/:id
PUT    /api/documents/:id
DELETE /api/documents/:id
GET    /api/documents/duplicate/:id
PATCH  /api/documents/status
```

**Clients:**
```
GET    /api/clients?page={p}&perPage={pp}&q={q}
POST   /api/clients
GET    /api/clients/:id
PUT    /api/clients/:id
DELETE /api/clients/:id
```

**Templates:**
```
GET    /api/templates?page={p}&perPage={pp}&q={q}
POST   /api/templates
GET    /api/templates/:id
PUT    /api/templates/:id
DELETE /api/templates/:id
GET    /api/templates/default
GET    /api/templates/duplicate/:id
```

**Recurring Invoices:**
```
GET    /api/invoices/recurring
POST   /api/invoices/recurring
PUT    /api/invoices/recurring/:id
```

**Users:**
```
GET    /api/users
POST   /api/users
PUT    /api/users/:id
DELETE /api/users/:id
```

**Profile:**
```
GET    /api/profile               - Get current user
POST   /api/profile               - Update profile/password
```

**API Tokens:**
```
GET    /api/tokens
POST   /api/tokens
DELETE /api/tokens/:id
```

**Other:**
```
GET    /api/dashboard             - Dashboard stats
GET    /api/number/:type          - Get next document number
POST   /api/render                - Render document to PDF/PNG
POST   /api/run/recurring         - Execute recurring invoices (cron job)
```

### Common Query Parameters

**Pagination:**
- `page`: page number (default: 1)
- `perPage`: records per page (default: 20)

**Search:**
- `q`: full-text search query

**Sort:**
- `sort[fieldName]=asc|desc`
- Example: `sort[name]=asc&sort[createdAt]=desc`

**Filter:**
- `filter[fieldName][operator]=value`
- Operators: `=`, `!=`, `<`, `<=`, `>`, `>=`, `like`, `in`
- Example: `filter[status][=]=1&filter[total][>=]=1000`

**Count:**
- `count=true`: return total count instead of records (includes soft-deleted)

### Response Format

**Single Resource:**
```json
{
  "id": "encoded_hash_id",
  "name": "Client Name",
  "number": "CLI-00001",
  ...
}
```

**Paginated List:**
```json
{
  "meta": {
    "total": 156,
    "per_page": 20,
    "current_page": 2,
    "last_page": 8,
    "first_page": 1
  },
  "data": [ ... ]
}
```

**Response Headers:**
- `x-total`: total records
- `x-page`: current page
- `x-per-page`: records per page
- `x-pages`: total pages

---

## 12. Business Logic & Calculations

### Document Calculations (recap)

**Executed in this order:**
1. `calcPositions()` - Position net, tax, discounts
2. `calcTaxes()` - Group taxes by rate
3. `calcNet()` - Sum position nets
4. `calcTotal()` - Net + taxes

**Called by:**
- `calculate()` method (manual)
- `rebuild()` method (also adds position if empty)
- `@beforeSave()` hook (automatic)

### Number Generation

**Format Syntax:**
```
"{number}" = sequential number with padding
"{date:format}" = formatted date
```

**Examples:**
- `"INV-{number}"` with padZeros=5 → `"INV-00001"`
- `"{date:yyyy}-{number}"` → `"2025-00001"`
- `"RE-{date:MM}-{number}"` → `"RE-01-00023"`

**Implementation:**
```typescript
Format.number(entity: { format: string, padZeros: number }, add: number = 0) {
  let number = String(1 + add).padStart(entity.padZeros, "0")
  number = entity.format.replace("{number}", number)

  // Find date pattern: {date:format}
  const datePattern = number.match(/\{date:[a-zA-Z_\-\.]+\}/)
  if (datePattern) {
    const format = datePattern[0].replace("{date:", "").replace("}", "")
    const date = DateTime.now().toFormat(format)
    number = number.replace(datePattern[0], date)
  }

  return number
}
```

**Count Calculation:**
- Count ALL documents of type (including soft-deleted)
- Use count as `add` parameter
- Ensures unique numbers even after deletions

### Dashboard Statistics

**Returns:**
```typescript
{
  invoices: {
    total: number,        // Sum of all PAID invoices
    net: number,          // Sum of net for PAID invoices
    pending: Document[]   // 5 most urgent PENDING invoices (by dueDate)
  },
  offers: {
    total: number,        // Sum of all ACCEPTED offers
    net: number,
    pending: Document[]   // 5 most urgent PENDING offers
  },
  reminders: {
    total: number,        // Sum of all PENDING reminders
    net: number,
    pending: Document[]   // All PENDING reminders (by dueDate)
  }
}
```

**Pending Documents:**
- Status: Pending
- Ordered by: `data->>'dueDate'` ASC
- Limit: 5 for invoices/offers, unlimited for reminders

### Currency & Date Formatting

**Currency:**
```typescript
Format.toCurrency(value, locale, currency)
// Uses Intl.NumberFormat
// Examples:
//   Format.toCurrency(1234.56, "en-US", "USD") → "$1,234.56"
//   Format.toCurrency(1234.56, "de-DE", "EUR") → "1.234,56 €"
```

**Date:**
```typescript
Format.date(value, locale)
// Short format: "1/15/25" or "15.01.25"

Format.longDate(value, locale)
// Full format: "January 15, 2025" or "15. Januar 2025"
```

---

## 13. Database Schema

### Full Schema Overview

```sql
-- Organizations (multi-tenant root)
CREATE TABLE organizations (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) NOT NULL UNIQUE,
  data JSONB DEFAULT '{}',
  settings JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL
);

-- Users
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  password VARCHAR(180) NOT NULL,
  role SMALLINT NOT NULL,
  data JSONB DEFAULT '{}',
  settings JSONB DEFAULT '{}',
  remember_me_token VARCHAR NULL,
  organization_id INT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL,
  INDEX idx_organization_id (organization_id)
);

-- API Tokens (OAT)
CREATE TABLE api_tokens (
  id SERIAL PRIMARY KEY,
  user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name VARCHAR NOT NULL,
  type VARCHAR NOT NULL,
  token VARCHAR(64) NOT NULL UNIQUE,
  expires_at TIMESTAMPTZ NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL
);

-- Clients
CREATE TABLE clients (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  number VARCHAR(100),
  data JSONB DEFAULT '{}',
  organization_id INT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL,
  INDEX idx_organization_id (organization_id),
  UNIQUE (organization_id, number)
);

-- Documents (invoices, offers, reminders)
CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  type SMALLINT NOT NULL,
  number VARCHAR(30),
  status SMALLINT NOT NULL,
  data JSONB DEFAULT '{}',
  client_id INT REFERENCES clients(id) ON DELETE CASCADE,
  organization_id INT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  template_id INT NULL,
  offer_id INT NULL REFERENCES documents(id) ON DELETE CASCADE,
  invoice_id INT NULL REFERENCES documents(id) ON DELETE CASCADE,
  recurring_id INT NULL REFERENCES recurring_invoices(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL,
  INDEX idx_client_id (client_id),
  INDEX idx_organization_id (organization_id),
  INDEX idx_recurring_id (recurring_id),
  INDEX idx_created_at (created_at),
  UNIQUE (organization_id, number)
);

-- Templates
CREATE TABLE templates (
  id SERIAL PRIMARY KEY,
  title VARCHAR NOT NULL,
  html TEXT NOT NULL,
  thumbnail TEXT,
  default BOOLEAN DEFAULT FALSE,
  premium BOOLEAN DEFAULT FALSE,
  data JSONB DEFAULT '{}',
  organization_id INT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMPTZ NULL
);

-- Recurring Invoices
CREATE TABLE recurring_invoices (
  id SERIAL PRIMARY KEY,
  invoice_id INT NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  cron VARCHAR(20) NOT NULL,
  organization_id INT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  deleted_at DATE NULL,
  start_date DATE NOT NULL,
  next_run DATE NOT NULL,
  active BOOLEAN NOT NULL DEFAULT FALSE,
  INDEX idx_start_date (start_date),
  INDEX idx_next_run (next_run),
  INDEX idx_active (active),
  UNIQUE (organization_id, invoice_id)
);
```

### Key Constraints

**Unique Constraints:**
- `organizations.slug` - globally unique
- `clients.number` - unique per organization
- `documents.number` - unique per organization
- `recurring_invoices.invoice_id` - unique per organization

**Foreign Key Cascades:**
- Delete organization → cascades to all related data
- Delete client → cascades to documents
- Delete document → cascades to child documents (offers→invoices, invoices→reminders)
- Delete recurring → sets documents.recurring_id to NULL

**Indexes:**
- All `organization_id` columns indexed (multi-tenant queries)
- `documents.created_at` indexed (recurring checks)
- `recurring_invoices.start_date`, `next_run`, `active` indexed (cron queries)

---

## 14. Numbering System

### Sequential Number Generation

**Service:** `NumberService`

**Document Numbers:**
```typescript
NumberService.document(organizationId: number, type: DocumentType): string
```

**Process:**
1. Count all documents of type (including soft-deleted with `withTrashed()`)
2. Load organization settings
3. Get number format for type:
   - `settings.invoices.number` for Invoice
   - `settings.offers.number` for Offer
   - `settings.reminders.number` for Reminder
4. Call `Format.number(format, count)`
5. Return generated number

**Client Numbers:**
```typescript
NumberService.client(organizationId: number): string
```

**Process:**
1. Count all clients (including soft-deleted)
2. Load organization
3. Get `settings.clients.number`
4. Call `Format.number(format, count)`
5. Return generated number

**Configuration:**
```typescript
settings: {
  invoices: {
    number: {
      format: "INV-{number}",
      padZeros: 5
    }
  }
}
```

**Examples:**
- Count 0, format `"INV-{number}"`, padZeros 5 → `"INV-00001"`
- Count 42, format `"{date:yyyy}-{number}"`, padZeros 3 → `"2025-043"`
- Count 999, format `"RE-{date:MM}-{number}"`, padZeros 4 → `"RE-01-1000"`

---

## 15. ID Obfuscation

### HashIDs Implementation

**Library:** Sqids (successor to HashIDs)

**Configuration:**
- Minimum length: 20 characters
- Alphabet: customizable via `ALPHABET` env var
- Default: `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789`

**Encoding:**
```typescript
HashIDs.encode(val: number | null): string | null
// Development mode: returns number as string
// Production: encodes using Sqids
// Example: 123 → "xT5rK9mP7wQ3fL2nH8jY"
```

**Decoding:**
```typescript
HashIDs.decode(val: string | null): number | null
// Development mode: parses string to number
// Production: decodes using Sqids
// Example: "xT5rK9mP7wQ3fL2nH8jY" → 123
```

**Usage:**
- All model IDs serialized with `HashIDs.encode()`
- `HashIdParser` middleware decodes IDs in:
  - URL parameters (`:id`, `*Id`, `*_id`)
  - Query strings (`?clientId=...`)
  - Request body (recursively traverses objects)
  - Filter parameters (`filter[clientId][=]=...`)

**Security:**
- Prevents ID enumeration attacks
- Makes internal IDs non-guessable
- Does NOT provide encryption (just obfuscation)

**Error Handling:**
- Invalid hash returns 400 with error message
- Validates before decoding

---

## 16. Pagination, Search, Sort, Filter

### BaseAppModel Scopes

All models extend `BaseAppModel` which provides:

**Search Scope:**
```typescript
static searchFields = ['field1', 'field2', 'data.nestedField']

// Query:
Model.query().withScopes(scopes => scopes.searchBy(ctx, Model))

// Implementation:
// - Reads ctx.request.qs()['q']
// - Searches all searchFields with ILIKE
// - Supports JSONB fields (e.g., 'data.dueDate')
```

**Sort Scope:**
```typescript
static sortFields = ['field1', 'field2', 'data.nestedField']

// Query:
Model.query().withScopes(scopes => scopes.sortBy(ctx, Model))

// Implementation:
// - Reads ctx.request.qs()['sort']
// - Format: sort[field]=asc|desc
// - Supports multiple sorts
// - Validates against sortFields
// - Supports JSONB fields with raw SQL
```

**Filter Scope:**
```typescript
static filterFields = ['field1', 'field2']

// Query:
Model.query().withScopes(scopes => scopes.filterBy(ctx, Model))

// Implementation:
// - Reads ctx.request.qs()['filter']
// - Format: filter[field][operator]=value
// - Operators: =, !=, <, <=, >, >=, like, in
// - Example: filter[status][=]=1&filter[total][>=]=1000
```

**Pagination:**
```typescript
Model.query().paginate(page, perPage)

// Returns:
{
  meta: {
    total, per_page, current_page, last_page, first_page
  },
  data: [ ... ]
}

// Also sets response headers:
// - x-total
// - x-page
// - x-per-page
// - x-pages
```

### Examples

**Search Documents:**
```
GET /api/documents?type=1&q=draft&page=1&perPage=20
```
Searches: number, dueDate, net, total

**Sort Documents:**
```
GET /api/documents?type=1&sort[number]=desc&sort[data.dueDate]=asc
```

**Filter Documents:**
```
GET /api/documents?type=1&filter[clientId][=]=abc123&filter[status][=]=1
```

**Combined:**
```
GET /api/documents?type=1&q=invoice&sort[data.total]=desc&filter[status][=]=1&page=2&perPage=50
```

### Searchable Fields by Model

**Document:**
- Search: number, data.dueDate, data.net, data.total
- Sort: number, status, data.dueDate, data.net, data.total
- Filter: clientId, offerId, invoiceId

**Client:**
- Search: name, number
- Sort: name, number, totalInvoices, totalReminders, totalOffers
- Filter: (custom)

**User:**
- Search: email, data.fullName, data.username
- Sort: email, role, data.fullName, data.username
- Filter: (custom)

---

## Additional Implementation Notes

### Soft Deletes
- All main entities support soft delete
- `deleted_at` column set to timestamp
- Queries automatically exclude soft-deleted (via `BaseAppModel`)
- Use `withTrashed()` to include soft-deleted
- Useful for:
  - Audit trails
  - Number continuity
  - Preventing data loss

### JSONB Usage
- Flexible schema for custom data
- Used for:
  - `organization.settings` - app configuration
  - `document.data` - document content
  - `client.data` - client details
  - `user.data` - user profile
- Queryable with PostgreSQL JSON operators
- Indexed where needed (e.g., `data->>'dueDate'`)

### Middleware Stack
1. **JsonError** - Formats errors as JSON
2. **Auth** - Verifies bearer token, loads user
3. **HashIdParser** - Decodes obfuscated IDs
4. **PaginationHeaders** - Adds pagination headers to response
5. **SilentAuth** - Optional auth (doesn't fail if missing)

### Error Handling
- Validation errors: 422 with field errors
- Not found: 404
- Unauthorized: 401
- Forbidden: 403
- Invalid hash IDs: 400
- Server errors: 500

### Security Considerations
1. **Multi-tenancy:** All queries scoped to organization
2. **Password hashing:** Argon2 (secure)
3. **ID obfuscation:** Prevents enumeration
4. **Soft deletes:** Audit trail
5. **Token expiry:** API tokens can expire
6. **Validation:** Strict input validation
7. **SQL injection:** Protected by ORM
8. **CSRF:** SPA design (token-based auth)

### Missing/TODO Items
- Email sending (for invoices, reminders)
- Payment gateway integration
- File attachments
- Invoice reminders automation
- Overdue status automation (status 4 not automatically set)
- Webhook system
- Export to formats (CSV, Excel)
- Audit logging
- Two-factor authentication
- Rate limiting
- API versioning

---

## Summary

This is a **complete multi-tenant invoicing application** with:

✅ **Organizations** - Multi-tenant with isolated data
✅ **Users** - Role-based access control
✅ **Clients** - Customer management with stats
✅ **Documents** - Invoices, Offers, Reminders with full lifecycle
✅ **Templates** - Customizable Nunjucks HTML templates
✅ **Recurring Invoices** - Cron-based automation
✅ **PDF Generation** - Via Gotenberg service
✅ **Sequential Numbering** - Configurable formats with date placeholders
✅ **Financial Calculations** - Complex tax, discount, charge logic
✅ **Offer-to-Invoice Conversion** - Full, partial, final options
✅ **Search/Sort/Filter** - Advanced querying
✅ **Soft Deletes** - Audit trail
✅ **ID Obfuscation** - Security through obscurity
✅ **Currency/Date Formatting** - Locale-aware
✅ **Dashboard** - Business metrics

**Use this document to:**
1. Compare with your implementation
2. Identify missing features
3. Review calculation logic
4. Validate business rules
5. Check API contracts
6. Understand data model
7. Review security patterns

**Key Business Rules to Remember:**
- All amounts calculated before save
- Document numbers sequential per org + type
- One default template per org
- Soft delete preserves number continuity
- Multi-tenant isolation at query level
- HashIDs in all external APIs
- Recurring checks: startDate ≤ today AND nextRun ≤ today AND active=true
