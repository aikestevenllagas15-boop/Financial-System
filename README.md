# Finance System — Capstone Prototype

Tuition assessment, payment processing & transaction ledger. Standalone
Node.js/Express + SQLite implementation of the Finance System module from
the Distributed Enterprise Integration Architecture proposal.

## Run it

```bash
npm install
npm start
```

Then open **http://localhost:4000**

Demo login: `cashier` / `csfj2026`

## What's real vs. what's simulated

- The backend, database, validation, duplicate-O.R. rejection, JWT auth, and
  every API response are real — the frontend never fakes data locally.
- The Registrar System doesn't exist yet in this five-system architecture,
  so `GET /api/students/:query` reads Finance's own synchronized copy of
  student records instead of proxying a live Registrar API. Swap the query
  inside `src/routes/students.js` for an outbound `fetch()` to the real
  Registrar service once it exists — nothing else needs to change.

## Discounts — scholarships & employee dependents

Each seeded student carries a `discount_eligibility` field (as Finance would
receive it synced from Registrar/HR). When you look up a student who's
eligible, the form auto-fills the matching discount type and percentage and
shows a verification hint — the cashier still has to confirm supporting
documents before committing.

Built-in discount types (`src/routes/transactions.js` → `DISCOUNT_TYPES`):

| Type | Typical % |
|---|---|
| Academic Scholarship — Full | 100% |
| Academic Scholarship — Partial | 50% (editable) |
| Working Scholar | 50% (editable) |
| Employee Dependent (Parent/Guardian CSFJ Staff) | 50% (editable) |
| Sibling Discount | 10% (editable) |
| Other / Manual | 0% (cashier sets it) |

The percentage field is always editable — presets are a starting point, not
a hard rule. The **server** recomputes `discountAmount` and the net `amount`
from `grossAmount × discountPercent` on every commit (never trusts the
client's math), and rejects anything outside 0–100% or a non-zero percent
paired with discount type `None`. The ledger, stats, and `/api/sync/summary`
all total the **net** amount actually collected, with `gross_amount` and
`discount_amount` kept alongside every transaction for audit purposes.

To add a new discount category, add it to the `DISCOUNT_TYPES` array in
`src/routes/transactions.js` and to the `<select id="f-discountType">`
options and `DEFAULT_DISCOUNT_PERCENT` map in `public/index.html` /
`public/app.js`.

## Printing receipts

After committing a transaction, the form shows **🖨 Print Receipt** instead
of resetting immediately — click it to open a print-ready receipt in a new
window (auto-triggers the browser's print dialog; pop-ups must be allowed).
Every past transaction can also be reprinted from the Ledger — each row has
a 🖨 icon next to the void button. The printable receipt is built entirely
client-side in `printReceipt()` in `public/app.js` from the transaction
object the API returns, so it always reflects the exact committed record
(gross amount, discount, net amount, both sign-off names).

## API surface

| Method | Route | Auth | Purpose |
|---|---|---|---|
| POST | `/api/auth/login` | — | Get a JWT for the Accounting workstation |
| GET | `/api/students/:query` | — | Verify identity/program/status before payment |
| GET | `/api/transactions` | — | Ledger list, `?search=` and `?status=` filters |
| GET | `/api/transactions/:id` | — | One transaction |
| GET | `/api/transactions/balance/:studentNo` | — | Payment status Enrollment/Registrar check |
| POST | `/api/transactions` | Bearer | Validate → commit a new payment |
| PATCH | `/api/transactions/:id/void` | Bearer | Void an active transaction |
| GET | `/api/students` | — | List all students (Manage Students screen) |
| POST | `/api/students` | Bearer | Add a student record |
| DELETE | `/api/students/:studentNo` | Bearer | Remove a student record (transaction history is untouched) |
| GET | `/api/sync/summary` | — | Approved summary synced to the Central Server |

## Swapping SQLite for MySQL

Every route only calls the functions exported from `src/db.js`. To move to
MySQL (as named in the proposal's tech stack), replace that file's
`node:sqlite` calls with a `mysql2`/Prisma client exposing the same
`db.prepare(...).get/all/run` shape, or refactor the two route files to use
your ORM directly — the schema (`students`, `transactions`) carries over
unchanged.

## Transaction IDs

Every committed transaction gets a sequential, human-readable ID —
`COL000001`, `COL000002`, and so on — instead of a random string, so it's
easy to write on a receipt book or reference by hand. The counter is stored
in a `counters` table (not just `COUNT(*)` on transactions), so numbering
stays gapless and never collides even after transactions are voided or the
server restarts. If you're upgrading a database that already has old
random-style IDs, new transactions simply continue with `COL000001`
onward — old IDs aren't renumbered.

To change the prefix or digit count, edit `ID_PREFIX` / `ID_DIGITS` in
`src/db.js`.

## Project structure

```
finance-system/
  server.js              Express app, route wiring
  src/
    db.js                 SQLite schema + seed data
    middleware/auth.js     JWT login + guard
    routes/
      students.js          Registrar-style lookup
      transactions.js       Validate/commit/void/balance
      sync.js               Central Server summary
  public/                 Static frontend (HTML/CSS/JS, navy & gold theme)
    index.html
    styles.css
    app.js
  data/finance.db          Created on first run (gitignored)
```
