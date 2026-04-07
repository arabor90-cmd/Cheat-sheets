# Attendee Transfer Flow — Options Analysis & Recommendation

## Context
Adding an attendee-initiated ticket transfer flow. The organizer reassign path already exists via `transferBooking()` in `backend/src/lib/transfer.ts` (cancel+create pattern). The attendee flow is entirely new — no frontend, no route, no schema support. Three implementation approaches are being evaluated before choosing one to build.

---

## Current Schema State
`Booking` model (`prisma/schema.prisma:96-122`) has **no transfer-related fields**:
- No `transferredFrom`, `originalBookingId`, `transferredAt`
- No `BookingTransfer` or `TransferHistory` model
- Status values (stored as string): `CONFIRMED`, `CANCELLED`, `CHECKED_IN`, `WAITLISTED`
- `ticketCode` and `qrCodeData` are both `@unique` — a transfer that preserves these fields means the old holder's QR remains usable until they present it first

---

## Option Comparison

### Option A — Cancel + Create (existing organizer pattern)
**How it works:** Cancel original booking (status→CANCELLED, refundAmount=0, cancelledAt=now), decrement+re-increment capacity, generate new ticketCode+QR, create fresh booking for recipient. Already implemented in `transfer.ts:24`.

| | |
|---|---|
| **Pro** | No schema migration needed — reuses `transferBooking()` directly |
| **Pro** | Old QR credentials are invalidated immediately (new ticketCode/QR issued) |
| **Pro** | Capacity counters stay correct and consistent with rest of system |
| **Pro** | Financial fields (`pricePaid`, `discountAmount`) preserved on new booking |
| **Con** | Original booking ID gone — any email confirmations, external references break |
| **Con** | Original holder sees a "CANCELLED" entry in their booking history — misleading, they didn't cancel |
| **Con** | No explicit link between the cancelled and created booking — forensically opaque without querying by timestamp/user pair |
| **Con** | New `createdAt` on the new booking loses the original purchase timestamp |
| **Con** | The CANCELLED record has `refundAmount=0` — indistinguishable from an organizer reassign without extra context |

---

### Option B — Simple Ownership Update
**How it works:** `UPDATE booking SET userId = recipientId` — one DB write, no new records.

| | |
|---|---|
| **Pro** | Minimal code — no schema changes, no new rows, ~5 lines of logic |
| **Pro** | Booking ID, ticketCode, QR, createdAt, pricePaid all preserved exactly |
| **Pro** | Clean booking history for recipient — shows original purchase date and price |
| **Con** | **No audit trail** — no record that a transfer happened, who initiated it, or when |
| **Con** | **Old QR not revoked** — original holder retains a working QR screenshot until someone presents it first at the door (first-scan-wins race) |
| **Con** | Original holder's booking just disappears from their account — no "transferred" status, no explanation |
| **Con** | Organizer has no visibility into transfer activity for their events |
| **Con** | Forensically invisible — a dispute ("I never transferred my ticket") is impossible to investigate |

---

### Option C — Hybrid: Ownership Update + Transfer History (recommended)
**How it works:** Update `userId` on the existing booking AND generate fresh `ticketCode`+`qrCodeData` on the same record, then write a new `BookingTransfer` record linking `fromUser → toUser` via the booking.

| | |
|---|---|
| **Pro** | Booking ID is preserved — external references, email links, booking history continuity |
| **Pro** | New credentials generated on the same record — old QR is immediately invalidated |
| **Pro** | `BookingTransfer` table provides explicit, queryable audit trail (who, to whom, when) |
| **Pro** | Original holder can see "TRANSFERRED" status instead of "CANCELLED" — accurate UX |
| **Pro** | `pricePaid`, `discountAmount`, `createdAt` all preserved on the booking |
| **Pro** | Organizer can query transfer history for their events |
| **Con** | Requires a schema migration — one new `BookingTransfer` model |
| **Con** | Slightly more implementation surface than A or B |
| **Con** | Need to decide: does the original holder retain a read-only "transferred" booking in their list, or does it disappear? (a `TRANSFERRED` status string solves this) |

---

## Recommendation: Option C — Hybrid

**Why not A:** The cancel+create pattern was designed for organizer reassignment where a clean "cancellation + fresh ticket" audit trail fits. For attendee-initiated transfer, "CANCELLED" on the original holder's account is actively confusing — it implies a refund or a failed booking, not a deliberate handoff. The broken booking ID also means any ticket confirmation email the original buyer received is now a dead link.

**Why not B:** No audit trail and no QR revocation make it unsuitable for anything beyond a toy app. A user who screenshots their QR before "transferring" retains a working ticket. A dispute about an unauthorized transfer is forensically unresolvable.

**Why C:** It matches the semantics of what a transfer actually is — the booking is the same booking, it now belongs to someone else, and there's a record of the handoff. Generating new credentials on the same record gives security without losing continuity. The `BookingTransfer` table is small and purpose-built. This is the right trade-off.

---

## Schema Changes Required (Option C)

```prisma
// New model
model BookingTransfer {
  id          String   @id @default(cuid())
  bookingId   String
  fromUserId  String
  toUserId    String
  transferredAt DateTime @default(now())

  booking     Booking  @relation(fields: [bookingId], references: [id])
  fromUser    User     @relation("TransfersSent",     fields: [fromUserId], references: [id])
  toUser      User     @relation("TransfersReceived", fields: [toUserId],   references: [id])

  @@index([bookingId])
  @@index([fromUserId])
  @@index([toUserId])
}

// Booking model additions
model Booking {
  // ... existing fields ...
  transferredAt DateTime?           // set when status→TRANSFERRED
  transfers     BookingTransfer[]   // history of all transfers on this booking
}

// User model additions
model User {
  // ... existing fields ...
  transfersSent     BookingTransfer[] @relation("TransfersSent")
  transfersReceived BookingTransfer[] @relation("TransfersReceived")
}
```

Add `"TRANSFERRED"` to the Booking status comment (string field, not enum).

---

## New Backend Route (Option C)

`POST /api/bookings/:id/transfer` — attendee-only, inside `backend/src/routes/bookings.ts`

Input: `{ recipientEmail: string }` — validated via Zod (reuse pattern from `reassignBookingSchema` in `validations.ts:70`)

Transaction steps:
1. Fetch booking, assert `userId === req.user.userId` + status `CONFIRMED`
2. Fetch recipient user by email, assert exists + not self + no existing CONFIRMED booking for same event
3. `booking.update` — `userId=recipientId`, new `ticketCode`, new `qrCodeData`, status→`TRANSFERRED` (or keep `CONFIRMED` and rely on `BookingTransfer` for history)
4. `BookingTransfer.create` — `{ bookingId, fromUserId, toUserId }`
5. Return updated booking

**Key decision:** Whether the original holder sees a `TRANSFERRED` record in their list requires either:
- Status `TRANSFERRED` on the booking (booking moves to recipient's account, original holder has no record) — simplest
- OR a separate read-only shadow record — unnecessary complexity

Simplest correct approach: status stays `CONFIRMED`, `userId` is updated to recipient, `BookingTransfer` holds the history. Original holder's booking disappears from their list (expected: they gave it away). Transfer history is queryable by admins/organizers via `BookingTransfer`.

---

## Files to Create / Modify

| File | Change |
|---|---|
| `prisma/schema.prisma` | Add `BookingTransfer` model, `transfers` relation on `Booking`, `transfersSent/Received` on `User` |
| `backend/src/routes/bookings.ts` | Add `POST /:id/transfer` route |
| `backend/src/lib/validations.ts` | Add `transferBookingSchema` (recipientEmail) |
| `frontend/src/lib/api.ts` | Add `bookingsAPI.transfer(token, id, recipientEmail)` |
| `frontend/src/app/tickets/[id]/page.tsx` | Add "Transfer Ticket" button + modal with email input |

`backend/src/lib/transfer.ts` — **not reused** for this flow. The existing `transferBooking()` function does cancel+create (Option A). The new attendee flow does an in-place ownership update instead.

---

## Verification
1. Run `npx prisma migrate dev --name add-booking-transfer` in `/backend`
2. POST `/api/bookings/:id/transfer` with valid recipient email → assert 200, booking now has recipient's userId, new ticketCode/QR, `BookingTransfer` record created
3. POST again with same booking → assert 400 (booking no longer belongs to original user)
4. Original QR should fail check-in (qrCodeData changed on booking record)
5. Recipient sees ticket in `/bookings`; old holder does not

---
