# Consulting — Snapp

Manage clients, consultants, projects and billable time tracking, with a ready-made
dashboard and reports.

> A Snapp is fully defined by its [`datamodel.json`](./datamodel.json). This README
> explains what that data model sets up and what an app owner needs to do after
> creating an app from this template. See the [repo README](../../README.md) for how
> templates are used.

---

## What it does

Keep **clients**, **consultants** and **projects** in one place and log **time entries**
against them. Each project carries three rate tiers (junior / standard / senior); a time
entry's billable **amount** is calculated automatically from the consultant's rate level
and the hours worked. A **Dashboard** shows active projects, hours this week and totals,
and two report pages produce a **billing report** and per-consultant **CV report**.

## Structure

**Collections**

| Collection | Purpose | Key relationships |
|---|---|---|
| `clients` | Customer records | — |
| `consultants` | Your people, with rate level & CV info | `employee` → user |
| `projects` | Engagements with per-tier rates | `client` → `clients` (write-once) |
| `project_team` | Which consultants staff a project, at what tier | `project`, `consultant` → (write-once) |
| `time_entries` | Logged hours | `project` → `projects`, `consultant` → `consultants` |

**Calculated fields**: project `total_hours` and `total_billable` roll up from time
entries; time-entry `amount` is `hours × the project rate for the consultant's level`.

**Pages**: `dashboard` (start page), `billing_report`, `consultant_cv_report`.
**Tours**: `welcome`, `log-time-entry`.

## Roles to set up

The data model declares a single role, `consultant`. Assign it to everyone who should
use the app. The app owner (you) administers members and data. There is **no separate
approver role** in this template — see "Access model" below for how visibility works.

## Admin setup checklist

1. **Assign the `consultant` role** to your team members.
2. **Link consultants to users** — a consultant's `employee` field points at a user
   account; set it so the "My Entries" filter (`consultant._id = logged-on user`) works
   and CVs map to real people.
3. **Set project rates** — `junior_rate`, `standard_rate` and `senior_rate` are
   **required** on every project; the amount calculation depends on them.
4. **Set each consultant's rate level** — `hourly_rate` (Junior / Standard / Senior)
   decides which project rate applies to that person's time entries.
5. **Walk the tours** — `welcome` and `log-time-entry`.

## Base data to fill

Enter in this order so lookups resolve:

1. **Clients** — name, contact, status (`active / inactive / prospect`).
2. **Consultants** — name, rate level, and CV fields (introduction, education,
   experience, skills, picture) used by the CV report.
3. **Projects** — linked to a client, with status and the three rate tiers.
4. **Project team** — staff each project by adding consultants at a `rate_level`.
5. **Time entries** — logged as work happens.

## How billing is calculated

- Each **project** defines three hourly rates: junior / standard / senior.
- Each **consultant** has a rate level (`hourly_rate`).
- A **time entry**'s `amount` = `hours × the project rate matching the consultant's
  level`, in NOK by default. Toggling **billable** off sets the amount to 0
  (via an `x-rule`).
- The **overtime comment** field only shows when hours exceed 7.5 (another `x-rule`).

There is **no approval/submission workflow or e-mail notifications** in this data model —
time entries are recorded directly. The `status` fields on clients, consultants and
projects are simple lifecycle labels (e.g. active / on_hold / completed), not gated
transitions.

## Constraints to be aware of

- **Required fields** — client: `name`; consultant: `name`; project: `name`, `client`,
  and all three rates; time entry: `date`, `project`, `consultant`, `hours` (0–24);
  project team: `project`, `consultant`, `rate_level`.
- **Write-once** — `client` on a project, and `project`/`consultant` on a team row,
  lock after creation.
- **Calculated / read-only** — project `total_hours` & `total_billable`, time-entry
  `amount`.
- **Access model** — every collection uses `ownerAccess: own-modify`: any consultant can
  **see all** records, but can only **edit or delete the ones they created**. The
  built-in "My Entries" list filter narrows time entries to the logged-on user.
- **Currency** — amounts default to **NOK** (`x-currency`); change it in the schema
  editor if needed.

## Customizing

Add fields (e.g. a client region), extra rate tiers, or a fourth project status in the
schema editor. If you need a submit/approve workflow on time entries, add a `status`
enum with `x-transitions` and matching notification `triggers` — see the Expense or
Incident Snapps for a worked example.
