# Incident Reporting — Snapp

Report incidents and deviations with AI photo scanning, severity triage, a review
workflow, linked corrective actions and an HSE compliance dashboard.

> A Snapp is fully defined by its [`datamodel.json`](./datamodel.json). This README
> explains what that data model sets up and what an app owner needs to do after
> creating an app from this template. See the [repo README](../../README.md) for how
> templates are used.

---

## What it does

Anyone can report an **incident** (HSE, quality, security or environment) — snapping a
photo or uploading a PDF and letting AI draft the report. A manager or HSE officer
triages severity, runs it through a review workflow, and opens **corrective actions**
with owners and due dates. Scheduled reminders chase due and overdue items. The
**Overview** dashboard tracks open/critical/overdue counts, severity and category
charts, and urgent items.

## Structure

**Collections**

| Collection | Purpose | Key relationships |
|---|---|---|
| `incidents` | The report — category, severity, status, investigation | `reported_by`, `assigned_to` → users |
| `corrective_actions` | Follow-up tasks per incident | `incident` → `incidents` (write-once), `owner` → user |

`incident_no` and `action_no` are **auto-assigned** sequence numbers (read-only).
`days_open`, `overdue` and `action_count` on an incident are **calculated**.

**Pages**: `dashboard` ("Overview", start page).
**Tours**: none defined.

## Roles to set up

Three roles are declared. Assign them to users in the app's member settings.

| Role | Can do |
|---|---|
| `employee` | Report incidents, see and edit their own, work corrective actions. |
| `manager` | See all incidents, drive the review workflow, assign, delete. |
| `hse` | Same authority as manager — the HSE/safety officer. |

`manager` and `hse` are exempt from owner-only visibility and can delete records;
`employee` cannot delete incidents.

## Admin setup checklist

1. **Assign roles** — everyone gets at least `employee`; safety leads get `hse`,
   supervisors get `manager`.
2. **Confirm reviewers are members** — `assigned_to` and corrective-action `owner`
   point at users, so reviewers must be app members to be pickable and to receive
   notifications.
3. **Agree on due-date conventions** — the scheduled reminders (below) fire off
   `due_date`, so set due dates when triaging.
4. **Review severity meaning** — `low / medium / high / critical` drive badge colors
   and dashboard alerts; align them with your HSE policy.

No reference data needs seeding — start reporting immediately.

## Status workflow

**Incident `status`** — controlled transitions (moves allowed only for the listed
roles):

```
new ───────────▶ under_review   (manager, hse)
new ───────────▶ rejected       (manager, hse)
under_review ──▶ in_progress    (manager, hse)
under_review ──▶ rejected       (manager, hse)
in_progress ───▶ resolved       (manager, hse)
resolved ──────▶ closed          (manager, hse)
resolved ──────▶ in_progress     (manager, hse)   ← reopen
rejected ──────▶ new             (employee, manager, hse)   ← reinstate
```

**Corrective action `status`**: `open → in_progress → done`, with reopen paths.
`employee`, `manager` and `hse` can move actions forward; only `manager`/`hse` can
reopen a `done` action.

## Triggers & notifications

**On update** (event-driven):

| When | Notifies | Message |
|---|---|---|
| `assigned_to` changes to someone | the new assignee | "Incident assigned to you" |
| `status` becomes `closed` | the reporter (`$OWNER`) | "Your incident is closed" (incl. root cause) |

**On schedule** (date-driven, off `due_date`):

| When | Notifies | Message |
|---|---|---|
| 3 days before due (incident still open) | `assigned_to` | "Incident due in 3 days" |
| 1 day after due (still open) | `assigned_to` | "Incident OVERDUE" |
| Corrective action due today (not done) | action `owner` | "Corrective action due today" |

Bodies interpolate fields with `$(field)` (e.g. `$(incident_no)`, `$(severity)`,
`$(due_date)`).

## Constraints to be aware of

- **Required fields** — incident: `title`, `category`, `severity`, `description`;
  corrective action: `incident`, `action`.
- **Write-once** — `incident` on a corrective action can't be reassigned.
- **Auto / read-only** — `incident_no`, `action_no` (sequence numbers); `days_open`,
  `overdue`, `action_count` (calculated).
- **Role access** — read/modify open to all three roles; **delete** restricted to
  `manager`/`hse`. `ownerAccess: own` on incidents keeps employees to their own reports,
  with `manager`/`hse` exempt.
- **AI fill** accepts images/PDFs and drafts title, category, severity, location and
  description, attaching the photo — always review before saving.

## Customizing

Add categories or a new severity in the schema editor; extend the `status` enum plus
its `x-transitions` to add workflow stages; adjust reminder timing by changing the
`offsetDays` on the scheduled triggers.
