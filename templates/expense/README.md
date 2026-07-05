# Expense — Snapp

Employee expense tracking and approval, with AI receipt scanning, itemized line
items and a ready-made dashboard.

> A Snapp is fully defined by its [`datamodel.json`](./datamodel.json). This README
> explains what that data model sets up and what an app owner needs to do after
> creating an app from this template. See the [repo README](../../README.md) for how
> templates are used.

---

## What it does

Employees create an **expense**, attach one or more itemized **expense items** (each
with a receipt), and submit it. A manager reviews and approves, rejects, or later
marks it paid. Every state change emails the relevant person automatically. The
built-in **Dashboard** shows spend totals, pending approvals, expenses by status and
top categories.

The headline feature: on an expense item, **"Scan / upload to fill"** reads a photo or
PDF of a receipt with AI and auto-fills category, amount, date and description.

## Structure

**Collections**

| Collection | Purpose | Key relationships |
|---|---|---|
| `expenses` | The claim header — title, employee, status, totals | `employee` → user, `approved_by` → user |
| `expense_items` | Line items with receipts | `expense` → `expenses` (write-once) |

An expense's `amount` is **calculated** by rolling up its item amounts — you don't type
it. `days_since_created` / `created_display` are calculated helper fields.

**Pages**: `dashboard` (start page).
**Tours**: `intro`, `submit-expense` — in-app walkthroughs shown to new users.

## Roles to set up

The data model declares two roles. Assign them to users in the snill app's member
settings after creating the app.

| Role | Can do |
|---|---|
| `user` | Create and submit their own expenses; see only their own. |
| `manager` | See **all** expenses, approve/reject/mark paid, and is exempt from owner-only visibility. |

Everyone who submits expenses needs the `user` role; approvers need `manager`.

## Admin setup checklist

1. **Assign roles** — give each employee `user`, each approver `manager`.
2. **Confirm approvers exist** — `approved_by` is a **required** field pointing at a
   user; an employee must pick their manager when creating an expense. Make sure the
   managers are members of the app so they appear in the picker.
3. **(Optional) Set an app currency** — amounts use the currency configured on the
   money fields (`x-currency`); change it in the schema editor if you aren't in the
   default.
4. **Walk the tours** — run `intro` and `submit-expense` to see the intended flow.

There's no base reference data to seed — the app is ready once roles and approvers are
in place.

## Status workflow

`status` on an expense is a controlled transition field. Only these moves are allowed,
and only by the listed roles:

```
draft ─────────▶ submitted        (user, manager)
submitted ─────▶ approved         (manager)
submitted ─────▶ rejected         (manager)
submitted ─────▶ draft            (user, manager)   ← send back
approved ──────▶ paid             (manager)
rejected ──────▶ draft            (user)            ← fix & resubmit
```

You cannot skip states (e.g. `draft → approved`) or move without the required role.

## Triggers & notifications

All four are e-mail notifications fired when `status` changes (`on: update`, comparing
against the previous value):

| When | Notifies | Message |
|---|---|---|
| `submitted` (from draft) | the `approved_by` manager | "New expense awaiting approval" |
| `approved` (from submitted) | the `employee` | "Your expense has been approved" |
| `rejected` (from submitted) | the `employee` | "Your expense has been declined" |
| `paid` (from approved) | the `employee` | "Your expense has been paid" |

Notification bodies interpolate record fields with `$(field)` (e.g. `$(title)`,
`$(amount)`, `$(employee.name)`).

## Constraints to be aware of

- **Required fields** — expense: `title`, `employee`, `expense_date`, `approved_by`;
  expense item: `expense`, `category`, `amount`, `attachment` (a receipt is mandatory
  on every item).
- **Write-once** — `employee` on an expense and `expense` on an item can't be changed
  after creation.
- **Calculated / read-only** — `amount` (rolled up from items), `days_since_created`.
  Don't expect to edit these.
- **Ownership** — `ownerAccess: own` means employees see only their own expenses and
  items; `manager` is exempt and sees everything.
- **AI fill** accepts images/PDFs (`x-accept`) and populates item fields — always
  review before saving.

## Customizing

Edit categories, add fields, or change the currency in the schema editor. To add
another approval tier, extend the `status` enum and its `x-transitions`, and add a
matching notification trigger.
