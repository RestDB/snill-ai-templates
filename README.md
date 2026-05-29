# snill.ai templates

Starter templates for [snill.ai](https://snill.ai). When a user creates a new app,
snill fetches the gallery from this repo and lets them pick a template to start from.

A snill app is fully defined by a single `datamodel.json` (the app shell, its
collections, pages and tours). The application code is identical across every app —
only the data model differs. So a template here **is** a curated `datamodel.json`.

## How snill uses this repo

snill's backend exposes a templates API that reads from this repo on GitHub:

1. It fetches [`templates.json`](./templates.json) — the gallery index — to render the
   template picker.
2. When the user selects a template, it fetches that template's
   `templates/<id>/datamodel.json` and uses it to seed the new app.

No build or publish step is involved — **pushing to this repo updates the gallery.**

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
      "collections": ["clients", "consultants", "projects", "time_entries"]
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
| `screenshot` | no | Repo-relative path to the preview image. |
| `path` | yes | Repo-relative path to the template folder. |
| `collections` | no | Collection keys in the template, shown as a quick summary. |

Paths are repo-relative; the snill backend resolves them to raw GitHub URLs.

## Adding a template

1. Create a folder `templates/<id>/`.
2. Add `datamodel.json` — a complete, valid snill data model (`app`, `collections`,
   and optionally `pages` and `tours`).
3. Optionally add `screenshot.png` showing the app.
4. Add an entry to `templates.json` with `id` matching the folder name.
5. Open a PR.

### Keep templates clean

Templates are public starting points, so strip development leftovers before adding one:

- No sidebar items pointing to collections that don't exist.
- No schema fields that aren't surfaced anywhere (form layout, list, search, etc.).
- No seed/sample rows — sample data is generated at app-creation time, not stored here.
