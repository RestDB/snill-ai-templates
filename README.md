# Snapps — snill.ai templates

**Snapps** (Snill apps) are ready-made, tested snill apps. Each one is a complete,
working application you can **use as-is**, or treat as a **starting point** for building
your own customized solution. This repo is the library of Snapps that powers
[snill.ai](https://snill.ai): when a user creates a new app, snill fetches the gallery
from here and lets them pick a Snapp to start from.

A snill app is fully defined by a single `datamodel.json` (the app shell, its
collections, pages and tours). The application code is identical across every app —
only the data model differs. So a Snapp (template) here **is** a curated `datamodel.json`.

## How snill uses this repo

snill's backend exposes a templates API that reads from this repo on GitHub:

1. It fetches [`templates.json`](./templates.json) — the gallery index — to render the
   template picker.
2. When the user selects a template, it fetches that template's
   `templates/<id>/datamodel.json` and uses it to seed the new app.

No build or publish step is involved — **merging a PR to this repo updates the gallery.**

## Using a Snapp directly

You don't have to start a new app from the picker. Because a Snapp is just a
`datamodel.json`, you can **copy the contents of any template's `datamodel.json` straight
into an existing Snill app** using the built-in schema editor — paste it in and the app
adopts the Snapp's collections, pages and tours. Handy for adding a tested Snapp to an
app you've already started, or for cherry-picking a Snapp as a starting point and
customizing from there.

## Repository layout

```
.
├── templates.json                 # gallery index (the manifest)
└── templates/
    └── <id>/
        ├── datamodel.json          # the app definition (required)
        └── screenshot.png          # preview image shown in the picker (optional)
```

## The manifest — `templates.json`

A single, hand-edited file listing every template:

```json
{
  "version": 1,
  "templates": [
    {
      "id": "consulting",
      "name": "Consulting",
      "description": "Manage clients, consultants, projects and billable time tracking, with a ready-made dashboard.",
      "category": "Professional Services",
      "tags": ["consulting", "time tracking", "billing", "projects"],
      "icon": "users",
      "screenshot": "templates/consulting/screenshot.png",
      "path": "templates/consulting",
      "collections": ["clients", "consultants", "projects", "time_entries"],
      "features": [
        "Billable time tracking with an approval workflow",
        "Ready-made dashboard: active projects, hours this week, pending approvals"
      ],
      "screenshots": ["templates/consulting/screenshot-2.png"]
    }
  ]
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `id` | yes | Slug for the template. **Must equal the folder name** under `templates/`. |
| `name` | yes | Display name in the picker. |
| `description` | yes | One-line gallery blurb (longer than the app's own subtitle). |
| `category` | yes | Grouping label in the picker (e.g. `Professional Services`). |
| `tags` | no | Keywords for search/filtering. |
| `icon` | no | Lucide icon name, usually matching the app's `app.icon`. |
| `screenshot` | no | Repo-relative path to the primary preview image (the picker thumbnail). |
| `screenshots` | no | Additional repo-relative preview image paths beyond `screenshot`, shown as a gallery alongside the picker thumbnail. |
| `path` | yes | Repo-relative path to the template folder. |
| `collections` | no | Collection keys in the template, shown as a quick summary. |
| `features` | no | Short list of headline capabilities (strings), authored with the template. Surfaced in the gallery and template detail views. |
| `author` | no | Display name of the template's creator (e.g. `snill`). |
| `authorUrl` | no | Profile or org link for the author. |
| `version` | no | Semver for the template itself (`1.0.0`). Bumped when the template's `datamodel.json` changes meaningfully. |
| `updatedAt` | no | ISO date (`YYYY-MM-DD`) of the last meaningful update. Drives staleness signals in the picker. |

Paths are repo-relative; the snill backend resolves them to raw GitHub URLs.

## Contributing a Snapp

**We welcome pull requests for new Snapps.** The Snapps library is meant to grow with
community contributions — if you've built a clean, reusable app, share it here. `main` is
protected, so all changes (ours included) go through a PR.

1. Create a folder `templates/<id>/`.
2. Add `datamodel.json` — a complete, valid snill data model (`app`, `collections`,
   and optionally `pages` and `tours`).
3. Optionally add `screenshot.png` showing the app — and list any extra preview
   images (e.g. `screenshot-2.png`) in the manifest's `screenshots`.
4. Add an entry to `templates.json` with `id` matching the folder name — optionally
   with `features` (a few headline capabilities) and `screenshots`.
5. Open a PR.

### Keep templates clean

Templates are public starting points, so strip development leftovers before adding one:

- No sidebar items pointing to collections that don't exist.
- No schema fields that aren't surfaced anywhere (form layout, list, search, etc.).
- No seed/sample rows — sample data is generated at app-creation time, not stored here.
