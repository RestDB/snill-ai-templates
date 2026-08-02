# Consulting PSA — Snapp

Run a consultancy or agency's whole core business on one data model: **CRM &
sales pipeline → projects & delivery → time & expenses → invoicing & AR →
profitability**. A Professional Services Automation (PSA) app that consolidates
what agencies usually pay several SaaS tools for.

> A Snapp is fully defined by its [`datamodel.json`](./datamodel.json). This README
> explains what that data model sets up and what an app owner needs to do after
> creating an app from this template. See the [repo README](../../README.md) for how
> templates are used.

> **Relationship to the [`consulting`](../consulting) Snapp:** this is the bigger
> sibling. `consulting` is a focused time‑tracking + billing/CV app. `consulting-psa`
> keeps all of that and adds the CRM pipeline, invoicing/AR, project delivery
> (tasks/milestones), expenses, and profitability — the full services business loop.

---

## What it does

The connected loop, left to right:

1. **Sell** — track **contacts** at each client, run a **sales pipeline** of
   `opportunities` through a stage workflow (lead → qualified → proposal →
   negotiation → won / lost), and log **activities** (calls, meetings, follow‑ups).
2. **Win → deliver** — a won opportunity links to its **project**; break the project
   into **milestones** and **tasks** (assignee, status, priority) — tasks have a
   **Kanban board** (drag between status columns).
3. **Track** — log **time entries** (billable amount auto‑calculated from the
   consultant's rate level and the project's tier rates) and **expenses** (scan a
   receipt to auto‑fill it).
4. **Bill** — raise **invoices** with line items (pulled from logged time or billable
   expenses), VAT, and a draft → sent → paid/overdue status workflow.
5. **Measure** — every project rolls up **cost** (labour + expenses), **margin**, and
   **margin %**; dashboards show the pipeline and accounts receivable.

## Structure

**Collections (13)**

| Collection | Purpose | Key relationships |
|---|---|---|
| `clients` | Customer companies | — |
| `contacts` | People at a client | `client` → clients |
| `opportunities` | Sales pipeline (stage workflow, value, weighted value) | `client`, `primary_contact`, `won_project` |
| `activities` | Calls / meetings / notes / follow‑up tasks | `client`, `opportunity` |
| `consultants` | Your people — rate level, **cost rate**, CV | `employee` → user |
| `projects` | Engagements — tier rates, rollups, **margin** | `client` → clients |
| `project_team` | Who staffs a project, at what tier | `project`, `consultant` |
| `milestones` | Project milestones | `project` |
| `tasks` | Project tasks — **Kanban board** by status | `project`, `milestone`, `assignee` |
| `time_entries` | Logged hours (billable amount + labour cost) | `project`, `consultant` |
| `expenses` | Billable / reimbursable costs, with **receipt scanning**; a consultant sees their own via the **My Expenses** filter and their profile | `project`, `consultant` |
| `invoices` | Accounts receivable — VAT, status workflow | `client`, `project`, `billing_contact` |
| `invoice_lines` | Invoice line items | `invoice`, `time_entry`, `expense` |

**Calculated fields**
- `time_entries.amount` = hours × the project rate for the consultant's level; `cost` = hours × the consultant's `cost_rate`.
- `projects.total_hours` / `total_billable` / `total_cost` / `total_expenses` roll up from children; `margin` = billable − cost − expenses; `margin_pct` is margin as a % of billable.
- `opportunities.weighted_value` = value × probability.
- `invoices.subtotal` = Σ line amounts; `tax_amount` = subtotal × VAT %; `total` = subtotal + tax.
- `invoice_lines.amount` = quantity × unit price.

**Workflows** (`x-transitions` — enforced on save, shown as action buttons)
- **Opportunity `stage`**: lead → qualified → proposal → negotiation → won / lost.
- **Invoice `status`**: draft → sent → paid / overdue / cancelled.
- **Time entry `status`**: draft → submitted → approved.

**Pages**: `dashboard` (start), `sales_pipeline`, `accounts_receivable`,
`profitability`, `billing_report`, `consultant_cv_report`.
**Tours**: `welcome`, `log-time-entry`.

## Receipt scanning (`documentImport`)

The **expenses** collection has document import enabled: use **Scan / upload to fill**
to drop a PDF or photo of a receipt and let the AI extract the vendor, date, amount and
category into a draft you confirm before saving.

## Notifications

Two internal reminders keep work on track — both notify the **record owner** (the person
who created it), by email, via snill's built‑in notifications:

- **Follow‑up due** — an activity with a follow‑up date that isn't done yet.
- **Invoice overdue** — an invoice still `sent` past its due date.

> These are **internal** notifications to your own team. Sending messages **to clients**
> (e.g. emailing an invoice to the billing contact) is not part of this data model. The
> `invoices.billing_contact` field is already in place so that capability can be added
> later without restructuring.

## Roles to set up

The data model declares a single role, `consultant`. Assign it to everyone who should
use the app; the app owner (you) administers members and data. For a larger org you can
add `sales`, `finance` or `manager` roles in the schema editor and scope the pipeline,
invoicing and approval workflows to them.

## Admin setup checklist

1. **Assign the `consultant` role** to your team members.
2. **Link consultants to users** — a consultant's `employee` field points at a user
   account; set it so "My" filters and CVs map to real people.
3. **Set project rates** — `junior_rate`, `standard_rate`, `senior_rate` are required on
   every project; the billable amount depends on them.
4. **Set each consultant's rate level and `cost_rate`** — rate level picks which project
   rate applies to their time; `cost_rate` (internal cost/hour) drives project margin.
5. **Check the VAT rate** — `invoices.tax_rate` defaults to 25 (Norway); change per invoice.
6. **Walk the tours** — `welcome` and `log-time-entry`.

## Base data to fill (in this order so lookups resolve)

1. **Clients**, then **Contacts** at each client.
2. **Consultants** — rate level, cost rate, CV fields.
3. **Opportunities** — your pipeline; a won one links to a project.
4. **Projects** — linked to a client, with the three tier rates; add **milestones**,
   **tasks**, and staff the **project team**.
5. **Time entries** and **expenses** as work happens.
6. **Invoices** — add lines from logged time / billable expenses; move through the status
   workflow as you send and get paid.

## Constraints to be aware of

- **Access model** — every collection uses `ownerAccess: own-modify`: anyone can **see
  all** records but only **edit/delete the ones they created**.
- **Write‑once** — `client` on a project, `project`/`consultant` on a team row, `project`
  on a milestone, `invoice` on a line — lock after creation.
- **Calculated / read‑only** — all rollups and totals above (project margins, invoice
  totals, time amount/cost, weighted value, line amount).
- **Currency** — amounts default to **NOK** (`x-currency`); change it in the schema editor.

## What it deliberately does **not** do

Position these as integrations/exports, not gaps:

- **Statutory accounting** (VAT returns, general ledger, bank reconciliation) — track
  invoices and AR here; keep your accounting system (Xero/QuickBooks) for the books and export.
- **E‑signature** — track proposal/contract status; sign in your e‑sign tool.
- **Outbound marketing automation** (campaigns, sequences, segment sends) — out of scope
  by design; snill sends **internal** notifications, not client‑facing marketing.

## Customizing

Add fields, extra rate tiers, more pipeline stages, or a richer role model in the schema
editor. To gate time entries so only **approved** hours are invoiced, filter your billing
views on `time_entries.status = approved`.
