# Simulated Client Records (SCR) — Architecture

## What this is

One generic EHR-simulation engine (`index.html`) that renders whatever a case's
JSON files describe. The engine has **no knowledge of any specific client,
discipline, or clinical content** — everything clinical lives in `/clients/`,
`/schema/modules.json`, and `/schema/master-sections.json`.

```
index.html                    ← the entire application (nav, rendering, export, storage)
schema/modules.json           ← registry of every section TYPE the platform can render
schema/master-sections.json   ← the full menu of sections/folders available platform-wide
clients/index.json            ← list of cases the roster should show
clients/<case-id>/case.json   ← one case's identity, metadata, and which sections are ON
clients/<case-id>/stages.json ← that case's longitudinal data, one entry per stage
docs/ARCHITECTURE.md          ← this file
```

## Core concepts

**Module** — a section *type* (Vital Signs, Braden Scale, MAR, Allied Health...).
Defined once in `schema/modules.json` with a `kind` (which generic renderer
draws it) and, for form-like kinds, a `fields` schema. Adding a module never
touches `index.html`.

**Kind** — the generic renderer a module uses. Current kinds: `summary`,
`admission`, `systemsChecklist`, `flowsheetTable`, `ioRecord`, `scoredForm`,
`structuredForm`, `notes`, `mar`, `ordersTable`, `docList`. A new module
almost always reuses an existing kind (e.g. a new lab panel is just another
`flowsheetTable`); a genuinely new *shape* of documentation needs one new
renderer function in `index.html`, reused by every future module of that
shape.

**Case** — one simulated client. `case.json` holds identity/demographics,
metadata tags, and a `sections` tree: the subset (and order, and folder
nesting) of modules this case turns on. `stages.json` holds the actual
clinical content, one object per time point.

**Stage** — a point in the client's timeline (`{id, label, date,
encounterType, data}`). There is no fixed count or fixed naming — a case can
have 1 stage or 12, labeled however makes sense ("Home Visit", "ED Triage",
"Admission Day 3"). `data` is keyed by module key; whatever a module needs is
stored there.

**Sections tree** — an ordered list of nodes, each either a leaf
(`{key, label}`, referencing a module) or a folder (`{key, label, folder:true,
children:[...]}`). Folders can nest to any depth — the nav renders however
many levels deep the tree goes. "Nursing Care" and "Medication" are just
folders in the current cases; nothing about folders is hardcoded to those two
names.

## How a case renders

1. `index.html` fetches `clients/index.json`, then each case's `case.json`
   to build the roster.
2. Opening a case fetches its `case.json` + `stages.json`.
3. The nav is built by walking `case.sections` (or, in **Preview: show all
   possible sections** mode, `master-sections.json` instead).
4. Clicking a leaf resolves to a module key. The engine looks up that key in
   `modules.json` for its `kind`, pulls `stage.data[key]` for instructor
   content, and dispatches to the matching renderer.

## Adding a new case

No code changes. Add a folder under `/clients/`, write `case.json` +
`stages.json` following the shape of the existing two cases, and add one line
to `clients/index.json`. Turn sections on by listing their keys in
`case.sections`; anything not listed simply doesn't appear for that case (but
still shows, greyed out with a description, in Preview mode — so an
instructor building a simple HCA case can still see what a fully-loaded
chart *would* include).

## Adding a new module (including a new discipline)

1. Add an entry to `schema/modules.json`: `label`, `kind`, and (for
   form-based kinds) `fields`.
2. Add it to `schema/master-sections.json` so it shows up in Preview mode
   and any future case-builder UI.
3. Reference its key in whichever case(s) should use it, with matching data
   in that case's `stages.json`.

Two discipline-specific modules (`rtAssessment` for Respiratory Therapy,
`hcaCare` for Health Care Assistant scope) are already in the registry as a
proof that this requires zero engine changes — they just aren't turned on for
either current case.

## Student data & privacy

- No accounts, no login, no student-identifying fields anywhere.
- All student-entered documentation (`localState`) lives in that browser's
  `localStorage`, under a key scoped to the case id. It never leaves the
  browser automatically.
- Export (per-section or whole-case) tags the download with a random session
  code (`sessionStorage`, regenerated per browser session) instead of any
  name — the student submits the file themselves through whatever channel
  the course already uses.
- No analytics, tracking, or telemetry of any kind.

## Hosting

Static files only — no build step, no server, no database. Works on GitHub
Pages, Cloudflare Pages, Netlify, or any static host. **Must be served over
http(s)**, not opened as a bare `file://` path, because the engine loads its
JSON via `fetch()` — browsers block `fetch()` against `file://` URLs. Locally,
run any static server, e.g. `python3 -m http.server` from this folder, then
open `http://localhost:8000/`.

## Known simplifications in this version (candidates for follow-up work)

- **Case Builder UI** — not built yet. The JSON schema above was designed so
  a future form-based builder can read/write it without a redesign, but the
  builder itself doesn't exist.
- **Diagnostics trending across many stages** — labs currently render one
  flat table per stage; a dedicated multi-stage trend view (same test,
  values over time, in one table or chart) would be a natural next step now
  that stage data is structured for it.
- **Deeper accessibility pass** — semantic labeling, keyboard navigation,
  and ARIA roles have not had a dedicated audit yet.
- **Longitudinal export across a whole cohort** — export is per-student,
  per-browser by design (see Privacy above); there is intentionally no
  aggregate view unless a backend is added later.
