# CHANGELOG / Session Notes

Running log of what was built into songdee-qc-report and why, plus the
related repos it depends on. Kept here so context survives across
machines/sessions (not just in chat history).

## Related repos

- **songdee-qc-report** (this repo, public) — the QC report app itself.
- **songdee-stock** (private) — Stock System. Owns the shared Firebase
  project `songdee-stock` (Firestore + Auth) that this app also uses.
  Collections we read: `roles` (uid → techName/role), `sd` (ใบเบิก
  withdrawal docs, keyed `sd:doc:*`), `assignments` (Admin's job list,
  read-only here).
- **songdee-admin** (private) — Admin web (มอบหมายงาน / รายงานซ่อม-ติดตั้ง).
  Has its own `technicians` collection (own login, NOT Firebase Auth) —
  its `assignedToName` values do **not** reliably match this app's
  `roles.techName`. See `ADMIN_NAME_MAP` in form.html.
- **songdee-line-proxy** (private) — shared Cloudflare Worker used by
  stock/admin/qc-report to push LINE Flex Message cards. We added a
  backward-compatible `sections` field to its `buildFlexCard()` (see
  below) — existing `items`-based callers are unaffected.
- **songdee-vehicle-lookup** (private, new) — dedicated Worker that
  reads the "MDVR Tracking" Google Sheet server-side (Google's CSV
  export sends no CORS headers, so a browser can't fetch it directly)
  and returns a vehicle's recorded equipment by ทะเบียน.

## App structure (3 pages, one Cloudflare Worker `songdeetest`)

- `index.html` — **Dashboard**, the landing page. Owns the login form.
  Shows company-wide stats (total/today report counts) and the latest
  20 `qc_reports` across all technicians, linking to `form.html?report=<id>`.
- `form.html` — the actual QC report form (this was the original
  `index.html` before the Dashboard was added). Gates on an existing
  Firebase session and bounces to `index.html` if there isn't one.
- `history.html` — a technician's own report history, same idea as the
  Dashboard but scoped to `techUid == currentUser`.

All three share a Dashboard/ฟอร์ม/ประวัติ tab bar (`.page-tabs`).

## Key features

- **Login**: Firebase Auth (email/password), same accounts as
  songdee-stock. After login, reads `roles/{uid}` for `techName`.
- **Stock lock**: entering a ทะเบียน looks up the most recent matching
  ใบเบิก in `sd` (matched by plate only — techName matching was tried
  first and dropped, see History below) and restricts the "แก้ไข"
  product picker to only what was actually withdrawn, capped at qty.
- **Job assignment dropdown**: reads songdee-admin's `assignments`
  (own techName mapped via `ADMIN_NAME_MAP`), lets a tech pick their
  own assigned job to auto-fill ลูกค้า/ทะเบียน/ปัญหา instead of typing.
- **ปัญหา / แก้ไข**: ปัญหา is free-text symptom only. แก้ไข composes
  "ทำการเปลี่ยน {รุ่นเก่า} เป็น {รุ่นใหม่}" (or ทำการติดตั้ง/ทำการถอด for
  install-only/remove-only rows) automatically from the product/oldModel
  dropdowns — no manual sentence typing needed.
- **รุ่นเก่าที่เปลี่ยน lookup**: sourced from two places — the generic
  PRODUCTS catalog, and (new) a "จากประวัติทะเบียนนี้" optgroup pulling the
  vehicle's actual recorded equipment from songdee-vehicle-lookup.
- **LINE card**: `sendToLine()` posts a Flex Message card (title/badge/
  fields/sections) via songdee-line-proxy instead of a plain-text wall.
  Sections = ปัญหา/แก้ไข/สติ๊กเกอร์/checklist, each with its own heading
  and separator so it doesn't read as one undifferentiated blob.
- **Report persistence**: every "ส่งเข้า LINE" click also saves/updates
  a doc in Firestore `qc_reports` (own collection, not shared with
  songdee-stock/songdee-admin's report collections). `currentReportId`
  tracks create-vs-update; reopening from Dashboard/history sets it.

## Notable gotchas hit during development (so they don't get re-litigated)

- **`.assetsignore` is required.** Without it, `wrangler deploy` uploads
  the whole repo directory as public static assets — including `.git/*`,
  which leaked the entire git history (and the exposed LINE_API_KEY in
  it) on the live URL. Keep `.assetsignore` excluding `.git` and `.wrangler`.
- **Firestore compat SDK here has no `.count()` aggregate query** —
  don't use it; fetch a page and derive counts client-side instead.
- **`where()` + `orderBy()` on different fields needs a composite
  index** we don't have set up — filter server-side, sort client-side,
  to avoid "query requires an index" errors.
- **LINE_API_KEY is a real bearer secret hardcoded in client JS** (pre-
  existing, not something we introduced). Flagged early; rotating it is
  outside this app's control (belongs to songdee-line-proxy's owner).
- **Admin's technician names ≠ this app's techName.** Two separate
  identity systems; `ADMIN_NAME_MAP` in form.html bridges them by hand
  per confirmed person — add new entries there as more techs come online.
- **Firestore rules for `qc_reports`** had to be added manually in the
  Firebase Console (`allow read, write: if request.auth != null;`) —
  this repo/code can't manage security rules itself.

## Workflow used throughout this project

Every change: feature branch → deploy to a separate `songdeetest-preview`
Worker → test → merge to `main` → deploy to the real `songdeetest`
Worker. Never deploy an untested branch straight to production.
