# Document Status State Machine - Business Flows

## Overview

This document describes the complete status lifecycle for all document types (Invoices, Offers/Quotes, Reminders) in the LaFacture/Rachoon system.

---

## Document Status Enum

```typescript
enum DocumentStatus {
  Draft = 0,      // ⚠️ DEFINED BUT NEVER USED
  Pending = 1,    // Initial status for all documents
  Accepted = 2,   // Final status for Offers
  Paid = 3,       // Final status for Invoices
  Overdue = 4     // ⚠️ NOT A STORED STATUS - Computed property
}
```

### Important Notes:

1. **Draft Status (0)** - Exists in the enum but is **NEVER used** in the application
   - All new documents start with `Pending` status
   - No UI or backend logic references Draft status

2. **Overdue Status (4)** - Exists in the enum but is **NOT stored in database**
   - It's a **computed property** calculated on the fly
   - Formula: `overdue = isPast(document.data.dueDate) && status === Pending`
   - Used only for display purposes (red highlighting, warnings)

---

## State Machine Overview

### Key Characteristic: **TOGGLE BEHAVIOR**

The status system is **NOT a linear workflow**. Instead, it uses a **toggle mechanism**:

- Click on status badge → toggles between states
- No multi-step approval workflow
- Simple binary transitions

---

## 1. Invoice State Flow

### States

```
┌─────────────┐
│  PENDING    │ ◄─────┐
│  (Initial)  │       │
└──────┬──────┘       │
       │              │
       │ User clicks  │ User clicks
       │ status badge │ status badge
       │              │
       ▼              │
┌─────────────┐       │
│    PAID     │───────┘
│   (Final)   │
└─────────────┘
```

### Transitions

| From | To | Trigger | Notes |
|------|-----|---------|-------|
| **Pending** | **Paid** | User clicks status badge | Invoice marked as paid |
| **Paid** | **Pending** | User clicks status badge | Undo payment (reopen invoice) |

### Display Logic

**Status Badge Colors:**
```typescript
Pending + NOT overdue → Default (no color)
Pending + overdue     → Red/Error ⚠️
Paid                  → Green/Success ✓
```

**Status Tooltips:**
```typescript
Pending + NOT overdue → "Pending"
Pending + overdue     → "Overdue" (red text)
Paid                  → "Paid"
```

**Status Icons:**
```typescript
Pending → Clock icon (fa-regular fa-clock)
Paid    → Check icon (fa-solid fa-check)
```

### Business Rules

✅ **Can change status:**
- Anytime between Pending ↔ Paid
- No restrictions on toggling

❌ **Cannot change status:**
- (No restrictions for invoices)

### Special Features

**Overdue Detection:**
```typescript
if (status === Pending && isPast(dueDate)) {
  // Display as overdue (red badge)
  // Show "Overdue" tooltip
  // Enable "Create Reminder" action
}
```

**Reminder Creation:**
- Only available when invoice is overdue
- Creates a new Reminder document linked to this invoice
- Reminder includes invoice total + reminder fees

---

## 2. Offer/Quote State Flow

### States

```
┌─────────────┐
│  PENDING    │ ◄─────┐
│  (Initial)  │       │
└──────┬──────┘       │
       │              │
       │ User clicks  │ User clicks
       │ status badge │ status badge
       │              │
       ▼              │
┌─────────────┐       │
│  ACCEPTED   │───────┘
│   (Final)   │
└─────────────┘
```

### Transitions

| From | To | Trigger | Notes |
|------|-----|---------|-------|
| **Pending** | **Accepted** | User clicks status badge | Offer accepted by client |
| **Accepted** | **Pending** | User clicks status badge | Undo acceptance (reopen offer) |

### Display Logic

**Status Badge Colors:**
```typescript
Pending + NOT overdue     → Default (no color)
Pending + overdue         → Red/Error ⚠️
Accepted                  → Blue/Info ✓
Has invoices (partial)    → Yellow/Warning 💰
Has invoices (full)       → Green/Success ✓✓
```

**Status Tooltips:**
```typescript
Pending + NOT overdue          → "Pending"
Pending + overdue              → "Overdue"
Accepted                       → "Accepted"
Has invoices (partial)         → "{amount} invoiced" (e.g., "$500 invoiced")
Has invoices (fully invoiced)  → "Fully invoiced"
```

**Status Icons:**
```typescript
Pending  → Clock icon (fa-regular fa-clock)
Accepted → Check icon (fa-solid fa-check)
Has invoices → Check icon (fa-solid fa-check)
```

### Business Rules

✅ **Can change status:**
- When offer has NO linked invoices
- Toggle between Pending ↔ Accepted

❌ **Cannot change status when:**
- Offer has been converted to invoice(s)
- Shows toast: "Cannot change status - Offer is already invoiced"
- This prevents changing status after billing started

### Special Features

**Invoice Tracking:**
```typescript
// Calculate total invoiced amount
const invoicedAmount = offer.invoices.reduce((sum, inv) => sum + inv.data.net, 0)

// Display different badge based on progress
if (invoicedAmount >= offer.data.net) {
  // Fully invoiced → Green badge
} else if (invoicedAmount > 0) {
  // Partially invoiced → Yellow badge with amount
} else {
  // Not invoiced → Normal status badge
}
```

**Conversion to Invoice:**
- Available when offer is NOT fully invoiced
- Three conversion modes:
  1. **Full** - Convert entire offer
  2. **Partial** - Convert percentage/amount
  3. **Final** - Convert remaining amount
- Creates invoice linked via `offerId`

---

## 3. Reminder State Flow

### States

```
┌─────────────┐
│  PENDING    │ ◄─────┐
│  (Initial)  │       │
└──────┬──────┘       │
       │              │
       │ User clicks  │ User clicks
       │ status badge │ status badge
       │              │
       ▼              │
┌─────────────┐       │
│    PAID     │───────┘
│   (Final)   │
└─────────────┘
```

### Transitions

| From | To | Trigger | Notes |
|------|-----|---------|-------|
| **Pending** | **Paid** | User clicks status badge | Reminder paid (invoice settled) |
| **Paid** | **Pending** | User clicks status badge | Undo payment (reopen reminder) |

### Display Logic

**Status Badge Colors:**
```typescript
Pending + NOT overdue → Default
Pending + overdue     → Red/Error ⚠️
Paid                  → Green/Success ✓
```

**Status Tooltips:**
```typescript
Pending + NOT overdue → "Pending"
Pending + overdue     → "Overdue"
Paid                  → "Paid"
```

**Status Icons:**
```typescript
Pending → Clock icon (fa-regular fa-clock)
Paid    → Check icon (fa-solid fa-check)
```

### Business Rules

✅ **Can change status:**
- Anytime between Pending ↔ Paid
- Same as invoices

❌ **Cannot change status:**
- (No restrictions for reminders)

### Special Features

**Creation from Overdue Invoice:**
- Reminder can only be created when invoice is overdue
- Automatically includes:
  - Invoice amount as first position
  - Reminder fees (configurable, default: $5 fixed charge)
- Links to invoice via `invoiceId`

**Auto-populated Data:**
```typescript
reminder.positions[0] = {
  title: invoice.number,        // e.g., "INV-00123"
  quantity: 1,
  price: invoice.data.total,    // Full invoice amount
  tax: 0                        // No tax on reminder
}

reminder.discountsCharges.push({
  title: "Reminder fee",
  value: 5,
  type: "charge",
  valueType: "fixed"
})
```

---

## Status Change Implementation

### Frontend Code (useDocument.ts)

```typescript
setStatus = (d: Document) => {
  // Rule: Cannot change status if offer has invoices
  if (d.invoices.length > 0) {
    useToast("Cannot change status", "Offer is already invoiced.", "warning");
    return;
  }

  // Toggle logic
  const newStatus = d.status === DocumentStatus.Pending
    ? d.type === DocumentType.Invoice
      ? DocumentStatus.Paid      // Invoice: Pending → Paid
      : DocumentStatus.Accepted  // Offer: Pending → Accepted
    : DocumentStatus.Pending;    // Any other status → Pending

  // Update locally and sync to server
  d.setStatus(newStatus);
  useApi().documents(this.docType()).setStatus(d.id, newStatus);
};
```

### Backend Code (DocumentsStatusController.ts)

```typescript
public async update(ctx: HttpContextContract) {
  const body = await ctx.request.validate(StatusValidator);
  // body = { status: number }

  return await Document.query()
    .where({
      id: ctx.request.param('id'),
      organizationId: ctx.auth.user?.organization.id,
    })
    .update(body);
}
```

**No validation or business rules in backend!**
- Frontend enforces all rules
- Backend simply updates the status field
- This is a security consideration (client-side only validation)

---

## Complete State Diagrams

### Invoice Lifecycle

```
┌──────────────────────────────────────────────────────┐
│                    NEW INVOICE                       │
│                 status = Pending                     │
└───────────────────────┬──────────────────────────────┘
                        │
                        │ Default status on creation
                        │
                        ▼
        ┌───────────────────────────────┐
        │        PENDING                │
        │  • Initial state              │
        │  • Awaiting payment           │
        │  • Can be edited              │
        │  • Clock icon                 │
        └────┬──────────────────┬───────┘
             │                  │
             │                  │ dueDate in past
             │                  ▼
             │         ┌────────────────┐
             │         │  PENDING       │
             │         │  (Overdue)     │
             │         │  • Red badge   │
             │         │  • Can create  │
             │         │    reminder    │
             │         └────────────────┘
             │
             │ User clicks badge
             │
             ▼
    ┌────────────────────┐
    │       PAID         │◄───────────┐
    │  • Final state     │            │
    │  • Green badge     │            │
    │  • Check icon      │            │
    └────────────────────┘            │
             │                        │
             │                        │
             │ User clicks badge      │
             │ (undo payment)         │
             │                        │
             └────────────────────────┘
```

### Offer/Quote Lifecycle

```
┌──────────────────────────────────────────────────────┐
│                    NEW OFFER                         │
│                 status = Pending                     │
└───────────────────────┬──────────────────────────────┘
                        │
                        │ Default status on creation
                        │
                        ▼
        ┌───────────────────────────────┐
        │        PENDING                │
        │  • Initial state              │
        │  • Awaiting acceptance        │
        │  • Can be edited              │
        │  • Clock icon                 │
        └────┬──────────────────┬───────┘
             │                  │
             │                  │ dueDate in past
             │                  ▼
             │         ┌────────────────┐
             │         │  PENDING       │
             │         │  (Overdue)     │
             │         │  • Red badge   │
             │         └────────────────┘
             │
             │ User clicks badge
             │ (no invoices)
             │
             ▼
    ┌────────────────────┐
    │    ACCEPTED        │◄───────────┐
    │  • Blue badge      │            │
    │  • Check icon      │            │
    │  • Can invoice     │            │
    └────────────────────┘            │
             │                        │
             │                        │
             │ User clicks badge      │
             │ (no invoices)          │
             │                        │
             └────────────────────────┘
             │
             │ Convert to invoice
             │
             ▼
    ┌────────────────────┐
    │   HAS INVOICES     │
    │  • Status locked   │
    │  • Yellow badge    │
    │    (partial)       │
    │  • Green badge     │
    │    (full)          │
    │  • Cannot change   │
    │    status anymore  │
    └────────────────────┘
```

### Reminder Lifecycle

```
┌──────────────────────────────────────────────────────┐
│              NEW REMINDER                            │
│          (from overdue invoice)                      │
│             status = Pending                         │
└───────────────────────┬──────────────────────────────┘
                        │
                        │ Auto-includes invoice amount + fees
                        │
                        ▼
        ┌───────────────────────────────┐
        │        PENDING                │
        │  • Initial state              │
        │  • Awaiting payment           │
        │  • Clock icon                 │
        └────┬──────────────────┬───────┘
             │                  │
             │                  │ dueDate in past
             │                  ▼
             │         ┌────────────────┐
             │         │  PENDING       │
             │         │  (Overdue)     │
             │         │  • Red badge   │
             │         └────────────────┘
             │
             │ User clicks badge
             │
             ▼
    ┌────────────────────┐
    │       PAID         │◄───────────┐
    │  • Final state     │            │
    │  • Green badge     │            │
    │  • Check icon      │            │
    └────────────────────┘            │
             │                        │
             │                        │
             │ User clicks badge      │
             │ (undo payment)         │
             │                        │
             └────────────────────────┘
```

---

## Related Document Workflows

### Offer → Invoice Conversion

```
┌─────────────┐
│   OFFER     │
│ (Accepted)  │
└──────┬──────┘
       │
       │ User clicks "Create Invoice"
       │
       ├───────────────┬───────────────┬──────────────┐
       │               │               │              │
       ▼               ▼               ▼              ▼
  ┌─────────┐    ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  FULL   │    │ PARTIAL  │   │ PARTIAL  │   │  FINAL   │
  │ Invoice │    │ Invoice  │   │ Invoice  │   │ Invoice  │
  │  100%   │    │   30%    │   │   50%    │   │   20%    │
  └─────────┘    └──────────┘   └──────────┘   └──────────┘
       │               │               │              │
       │               └───────┬───────┘              │
       │                       │                      │
       └───────────────────────┴──────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ OFFER               │
                    │ • Status locked     │
                    │ • Shows invoiced    │
                    │   amount/percentage │
                    │ • Cannot change     │
                    │   status anymore    │
                    └─────────────────────┘
```

### Invoice → Reminder Flow

```
┌─────────────┐
│  INVOICE    │
│  Pending    │
└──────┬──────┘
       │
       │ dueDate passes
       │
       ▼
┌─────────────┐
│  INVOICE    │
│  Overdue    │ (computed)
└──────┬──────┘
       │
       │ User clicks "Create Reminder"
       │
       ▼
┌─────────────┐         ┌─────────────┐
│  REMINDER   │◄────────│  INVOICE    │
│  Pending    │ linked  │  Overdue    │
│             │  via    │             │
│ Amount:     │ invoiceId│             │
│ $1,000 +    │         │             │
│ $5 fee      │         │             │
└─────────────┘         └─────────────┘
```

---

## Status Validation Rules Summary

### Invoice

| Action | Allowed When | Blocked When |
|--------|-------------|--------------|
| Edit document | Always | Never |
| Change status | Always | Never |
| Delete | Always | Never |
| Duplicate | Always | Never |
| Create reminder | Status = Pending AND overdue | Not overdue |

### Offer

| Action | Allowed When | Blocked When |
|--------|-------------|--------------|
| Edit document | No invoices | Has invoices |
| Change status | No invoices | Has invoices (shows error toast) |
| Delete | Always | Never |
| Duplicate | Always | Never |
| Create invoice | Not fully invoiced | Fully invoiced |

### Reminder

| Action | Allowed When | Blocked When |
|--------|-------------|--------------|
| Edit document | Always | Never |
| Change status | Always | Never |
| Delete | Always | Never |
| Duplicate | Never (not in UI) | Always |

---

## Missing/Unused Features

### ⚠️ Draft Status
- **Defined** in enum as status 0
- **NEVER used** in the application
- All documents start with `Pending` status
- No UI for saving drafts
- Consider removing from enum or implementing draft functionality

### ⚠️ Overdue Status
- **Defined** in enum as status 4
- **NOT stored** in database
- Computed property: `isPast(dueDate) && status === Pending`
- Should be documented as computed-only

### ⚠️ Automatic Status Updates
- No automatic transitions (e.g., Pending → Overdue)
- All status changes are manual (user clicks)
- No scheduled jobs to update statuses
- No email notifications on status changes

### ⚠️ Status Change Validation
- Only validated in frontend (security issue)
- Backend accepts any status value
- No audit trail of status changes
- No timestamps for status changes

---

## Recommendations for Improvements

### 1. Backend Validation
Add business rules validation in backend:
```typescript
// Before allowing status change
if (document.type === DocumentType.Offer && document.invoices.length > 0) {
  throw new Error('Cannot change status of invoiced offer')
}
```

### 2. Status Change Audit Trail
Log all status changes:
```typescript
status_changes table:
  - document_id
  - old_status
  - new_status
  - changed_by (user_id)
  - changed_at (timestamp)
  - reason (optional text)
```

### 3. Automatic Overdue Handling
Consider scheduled job:
```typescript
// Daily cron job
UPDATE documents
SET status = DocumentStatus.Overdue
WHERE status = DocumentStatus.Pending
  AND data->>'dueDate' < NOW()
  AND type IN (Invoice, Reminder)
```

### 4. Email Notifications
Trigger emails on:
- Invoice created → send to client
- Invoice overdue → send reminder
- Payment received → send receipt
- Offer accepted → notify team

### 5. Payment Integration
Instead of manual "Paid" status:
- Integrate with payment gateway
- Automatically mark as paid when payment received
- Record payment method and transaction ID
- Support partial payments

### 6. Draft Functionality
Either remove Draft status or implement:
- Save drafts without generating number
- Drafts not visible in main list
- "Publish" action to convert Draft → Pending
- Auto-save drafts

---

## Business Flow Examples

### Example 1: Simple Invoice Flow

1. **Create Invoice** → Status: `Pending`
2. Send to client
3. Client pays
4. **Click status badge** → Status: `Paid` ✓

### Example 2: Overdue Invoice with Reminder

1. **Create Invoice** → Status: `Pending`, Due: Jan 15
2. Send to client
3. Jan 16: Invoice becomes **overdue** (computed)
4. **Create Reminder** → New document, Status: `Pending`
5. Send reminder to client
6. Client pays
7. **Click invoice status** → Status: `Paid` ✓
8. **Click reminder status** → Status: `Paid` ✓

### Example 3: Offer with Partial Invoicing

1. **Create Offer** → Status: `Pending`, Total: $10,000
2. Send to client
3. Client accepts → **Click status badge** → Status: `Accepted` ✓
4. **Create Invoice (30%)** → New invoice for $3,000
   - Offer now shows: "$3,000 invoiced" (yellow badge)
   - Status still `Accepted` but **locked** (cannot toggle)
5. **Create Invoice (50%)** → New invoice for $5,000
   - Offer shows: "$8,000 invoiced" (yellow badge)
6. **Create Invoice (Final)** → New invoice for $2,000
   - Offer shows: "Fully invoiced" (green badge)
7. Invoices can be marked as paid independently

### Example 4: Undo Payment

1. Invoice is `Paid`
2. Realize payment was for wrong invoice
3. **Click status badge** → Status: `Pending`
4. Invoice reopened, can be edited

---

## Summary

**Key Takeaways:**

1. ✅ **Toggle Behavior** - All status changes are reversible toggles
2. ✅ **Simple Flow** - No complex workflows or approval chains
3. ⚠️ **Draft Unused** - Status 0 exists but never used
4. ⚠️ **Overdue Computed** - Not a real status, calculated on the fly
5. ⚠️ **Frontend Validation Only** - Backend accepts any status
6. ✅ **Offer Locking** - Status locked once invoiced (smart!)
7. ⚠️ **No Automation** - All transitions are manual

**Default Flow:**
```
Create → Pending → (Click badge) → Paid/Accepted
                ↑                        ↓
                └────────────────────────┘
                    (Click badge again)
```
