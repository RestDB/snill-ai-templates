# Contracts — Snapp

Track contracts, vendors and renewals with AI document scanning, notice-period alerts,
a status workflow and a compliance dashboard.

> A Snapp is fully defined by its [`datamodel.json`](./datamodel.json). This README
> explains what that data model sets up and what an app owner needs to do after
> creating an app from this template. See the [repo README](../../README.md) for how
> templates are used.

---

## What it does

Register **contracts** against **vendors**, capturing dates, renewal terms, notice
periods and financials. Drop in a PDF or image and let AI fill the terms. The app
computes the **notice deadline** and days-to-notice/expiry, then emails the owner as
those dates approach so nothing auto-renews by accident. A **status workflow** tracks
each contract's lifecycle, an **event timeline** records notes and amendments, and the
**Dashboard** surfaces annual obligations, active contracts, spend by category and
expiry alerts.

## Structure

**Collections**

| Collection | Purpose | Key relationships |
|---|---|---|
| `vendors` | Counterparties | — |
| `contracts` | The agreement — terms, dates, financials, status | `vendor` → `vendors` (write-once), `owner` → user |
| `contract_events` | Timeline entries per contract | `contract` → `contracts` (write-once) |

`notice_deadline`, `days_to_notice`, `days_to_expiry` and `action_required` on a
contract are **calculated** from its dates and notice period.

**Pages**: `dashboard` (start page).
**Tours**: `intro`.

## Roles to set up

Three roles are declared. Assign them in the app's member settings.

| Role | Typical holder | Can do |
|---|---|---|
| `admin` | App owner / procurement lead | Activate drafts, mark contracts expired, full access. |
| `legal` | Legal team | Drive most workflow transitions (renew, terminate, archive). |
| `finance` | Finance team | Same workflow authority as legal. |

All three roles are exempt from owner-only visibility on contracts, so the whole team
sees every contract regardless of who created it.

## Admin setup checklist

1. **Assign roles** — give the right people `admin`, `legal` and/or `finance`.
2. **Add vendors first** — a contract requires a `vendor`, and the field is write-once,
   so seed your vendor list before entering contracts.
3. **Set contract owners** — `owner` defaults to whoever creates the contract; reassign
   if someone else should receive the renewal alerts.
4. **Pick the currency** — each contract has a `currency` (USD/EUR/GBP/NOK/SEK/JPY,
   default USD); set your organization's default in the schema editor if needed.
5. **Fill notice periods** — the alert engine depends on `end_date`, `renewal_type` and
   `notice_period_days`; without them the deadline reminders can't fire.

## Base data to fill

1. **Vendors** — name, org number, contact, category
   (`service_provider / software / insurance / facility / other`).
2. **Contracts** — one per agreement, linked to a vendor, with dates, renewal type and
   financials. Use **"Scan / upload to fill"** to let AI populate from the PDF.
3. **Events** — optional, added over time as the relationship evolves.

## Status workflow

`status` is a controlled transition field:

```
draft ─────────▶ active            (admin, legal, finance)
active ────────▶ renewed           (legal, finance)
active ────────▶ terminated        (legal, finance)
active ────────▶ expired           (admin)
renewed ───────▶ active            (legal, finance)
renewed ───────▶ archived          (legal, finance)
terminated ────▶ archived          (legal, finance)
expired ───────▶ archived          (legal, finance)
```

Only these moves are permitted, and only for the listed roles.

## Triggers & notifications

All are **scheduled** e-mails to the contract `owner`, computed from date fields:

| Fires | Condition | Message |
|---|---|---|
| 30 days before `notice_deadline` | active, has renewal | "Notice window opens" |
| 7 days before `notice_deadline` | active, has renewal | "Act now — notice deadline" |
| On `notice_deadline` | active, has renewal | "Last day to give notice" |
| 30 days before `end_date` | active | "Contract expiring soon" |
| 7 days after `end_date` | still active | "Expired contract needs updating" |

Bodies interpolate fields with `$(field)` (e.g. `$(title)`, `$(vendor.name)`,
`$(end_date)`).

## Constraints to be aware of

- **Required fields** — contract: `title`, `vendor`, `category`, `owner`, `status`;
  vendor: `name`; event: `contract`, `type`.
- **Write-once** — on a contract: `vendor`, `category`, `owner`, `start_date`,
  `end_date`. Correct these before you rely on them; they lock after creation.
- **Calculated / read-only** — `notice_deadline`, `days_to_notice`, `days_to_expiry`,
  `action_required`.
- **Alerts need complete data** — reminders only fire when `renewal_type !== 'none'`
  (for notice alerts) and the relevant date fields are set.

## Customizing

Add vendor or contract categories, extra financial fields, or new event types in the
schema editor. Change reminder timing via the `offsetDays` on the scheduled triggers,
and extend the `status` enum plus its `x-transitions` to add lifecycle stages.
