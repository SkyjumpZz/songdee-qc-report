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
- **songdee-vehicle-lookup** (private) — dedicated Worker that reads
  the "MDVR Tracking" Google Sheet server-side (Google's CSV export
  sends no CORS headers, so a browser can't fetch it directly).
  Endpoints used by this app: `GET /vehicle` (equipment by ทะเบียน),
  `GET /plates` (full plate list, for autocomplete), `POST /update`
  (write-back — see "Vehicle sheet write-back" below). Also gained
  AI Box/ADAS/DMS device tracking + ISSUE Tracking endpoints from a
  separate session's work (see git history there:
  `6007f4f`/`4fd758c`) — not detailed here since that work lives in
  its own repo.
  - `EQUIPMENT_COLUMNS` (in that repo's `src/worker.js`, feeds this
    app's "รุ่นเก่าที่เปลี่ยน" dropdown + equipment banner) deliberately
    excludes `3G Module` and `AI System` — not useful for this list.
    `MCR`/`Motor` are relabeled for display only (`เครื่องรูดบัตร`/
    `มอเตอร์สั่น` — the raw sheet header used for write-back matching
    is untouched), and any `Yes` value displays as `มีอุปกรณ์`. This
    Worker has only one live URL that both `songdeetest` and
    `songdeetest-preview` share — there's no isolated preview instance
    to test against, so changes here were verified via `curl` against
    the one production endpoint before/after deploying (it's a
    read-only lookup, no write-path risk).

## ISSUE Tracking write-back — found and fixed a real bug via testing

Testing `pushIssueClose()` (closes a matching open ISSUES-All ticket
when a report is sent — feature built by the other session, see
`6007f4f` in songdee-vehicle-lookup) surfaced a real, live bug:
**closing a ticket silently failed to register from the app's own
point of view**, because ISSUES-All's FIXED DATE column has a
`DATE_IS_VALID` data validation rule that `writeCell()`'s default
`RAW` value-input mode doesn't satisfy — the write succeeds (200 OK)
but the cell fails that validation, and this app's own gviz-based reads
then still see the ticket as open. Full root-cause and fix are in
songdee-vehicle-lookup's own CHANGELOG.md (commit `1a145c5`); the
matching client-side half of the fix is here (commit `44cd80b`,
v1.13.2): `buildDeviceSwaps()`/`pushIssueClose()` now send the date as
ISO (`YYYY-MM-DD`) instead of reformatting to `DD/MM/YYYY`, which is
locale-ambiguous for Sheets' date parser.

**How this was tested without a disposable test account**: appending a
fake test row to ISSUES-All was tried first but blocked — the sheet
protects column A (Ticket No) for all rows, which blocks inserting any
new row at all via the service account (confirmed via a read-only
`protectedRanges` check, not just trial and error). Fell back to the
same "safe round-trip" technique used for the original MDVR write-back:
picked an already-closed historical ticket (`SDML/000001`, closed since
2020) and wrote its own existing values back through the real
`/issue-close` endpoint, verifying before/after via a fresh read. The
*first* attempt (via a shell `curl` command with Thai text inline)
corrupted the row — separately from the DATE_IS_VALID bug, Windows
shell argument encoding mangled the Thai `FEEDBACK DETAILS`/`FIXED
DETAILS` text. Both issues were caught immediately (this app's own
`/issue` endpoint started reporting the closed ticket as open again)
and fixed within the same session — restored via a direct Node script
(no shell argument involved) rather than shell commands with
non-ASCII text embedded.

**Lesson for next time**: prefer a Node/JS script over inline shell
commands whenever a test payload contains non-ASCII (Thai) text — do
not trust curl's `-d` with Thai characters as an argument on Windows.
Also: a `200 OK` from the Sheets API does not guarantee a column's own
data validation was satisfied — check for validation rules on a target
column (`spreadsheets.get` with `includeGridData`, read-only) before
assuming a write "took" in every sense a downstream reader cares about.

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
- **รุ่นเก่าที่เปลี่ยน lookup**: sourced *only* from a "จากประวัติ
  ทะเบียนนี้" optgroup pulling the vehicle's actual recorded equipment
  from songdee-vehicle-lookup — the generic PRODUCTS-catalog fallback
  group was deliberately removed (a tech shouldn't be able to log a
  removed device that isn't really what's on the vehicle). If nothing's
  on file yet for a plate, the only option is "ไม่มีการเปลี่ยนอุปกรณ์".
  PRODUCTS itself is unaffected — still used for the new-equipment
  select in every แก้ไข row.
- **Vehicle equipment reference banner** (`renderVehicleEquipmentBanner()`
  in form.html): as soon as ทะเบียน is typed (for อัปเกรด/ซ่อม — hidden
  for ติดตั้งใหม่, same as the old-model dropdown), shows the vehicle's
  full recorded equipment list right away, so a tech doesn't have to
  open each row's "รุ่นเก่าที่เปลี่ยน" dropdown just to browse what's on
  the vehicle. Purely read-only display; same `vehicleOldModels` data
  source the dropdown already uses, doesn't affect what gets submitted.
- **ทะเบียน autocomplete**: `<datalist>` populated from
  songdee-vehicle-lookup's `/plates` on login — suggests known plates
  while typing but still free-text (not a hard lock), since a vehicle
  not yet in the sheet must still be enterable.
- **Vehicle sheet write-back**: when a แก้ไข row's รุ่นเก่า came from the
  "จากประวัติทะเบียนนี้" group (so we know exactly which sheet column it
  maps to), a successful "ส่งเข้า LINE" also PUTs the new product name
  into that cell in the "MDVR Tracking" sheet via
  `songdee-vehicle-lookup`'s `POST /update`
  (`buildSheetUpdates()`/`pushSheetUpdates()` in form.html). Non-fatal
  on failure — the report still sends either way, failure is just
  appended to the toast text.
  - Auth: Google Sheets API v4 + a **Service Account** (JWT/RS256
    signed with Web Crypto `crypto.subtle`, exchanged for an OAuth2
    access token at `oauth2.googleapis.com/token`). The key lives in
    the Worker's `GOOGLE_SERVICE_ACCOUNT_JSON` secret; the account
    (`songdee-sheet-writer@...iam.gserviceaccount.com`) must be shared
    as **Editor** on the actual Google Sheet, same as sharing with a
    person.
  - **Why not Apps Script** (tried first, abandoned — code kept in
    `apps-script.gs` for reference only, not deployed): the
    `songdeegps.com` Workspace admin console blocks anonymous ("Anyone")
    Apps Script Web App access org-wide. Confirmed via `curl -sv`
    returning an immediate 403 with no redirect, despite deployment
    settings correctly showing "ทุกคน" / "Execute as: ฉัน". Fixing this
    requires Super Admin, which the account owner is not. A Service
    Account sidesteps the policy entirely since it's a real OAuth2
    identity, not an anonymous hit.
- **LINE card**: `sendToLine()` posts a Flex Message card (title/badge/
  fields/sections) via songdee-line-proxy instead of a plain-text wall.
  Sections = ปัญหา/แก้ไข/สติ๊กเกอร์/checklist, each with its own heading
  and separator so it doesn't read as one undifferentiated blob.
- **Report persistence**: every "ส่งเข้า LINE" click also saves/updates
  a doc in Firestore `qc_reports` (own collection, not shared with
  songdee-stock/songdee-admin's report collections). `currentReportId`
  tracks create-vs-update; reopening from Dashboard/history sets it.
- **"+ ทะเบียนถัดไป"** (`startNextVehicle()`): for a customer visit
  covering several vehicles, keeps วันที่/ลูกค้า/Fleet/ประเภทงาน but
  resets everything ทะเบียน-specific (plate, ปัญหา/แก้ไข/สติ๊กเกอร์/
  checklist, stock-lock, old-model lookup) so the next vehicle's report
  starts clean without retyping customer info. Each vehicle is still its
  own independent "ส่งเข้า LINE" + `qc_reports` doc — deliberately **not**
  grouped/batched, since testing happens one vehicle at a time and
  sometimes by a different technician entirely (decided against adding
  cross-report grouping on Dashboard/history — not worth it unless a
  real need for progress-tracking across techs shows up later).
- **Job-type-dependent sections** (`tplHasProblems()`/`tplHasOldModel()`
  in form.html): ปัญหา and รุ่นเก่าที่เปลี่ยน aren't shown for every
  ประเภทงาน — only where they make sense:
  - **ติดตั้งใหม่**: no ปัญหา, no รุ่นเก่าที่เปลี่ยน (nothing old to log).
  - **อัปเกรด**: no ปัญหา, keeps รุ่นเก่าที่เปลี่ยน.
  - **ซ่อม**: keeps both (รุ่นเก่าที่เปลี่ยน optional).
  Switching tabs clears the now-inapplicable fields so stale input
  doesn't linger in `state`. `fixLine()`, `buildSheetUpdates()`, and the
  ปัญหา section in `buildMessage()`/`buildLineCard()` all re-check the
  *current* tpl too (not just "is there data") — otherwise a report
  saved before this feature existed (or edited across a tpl switch)
  could have e.g. a ติดตั้งใหม่ report's stale รุ่นเก่า silently leak
  into the LINE message, or worse, trigger an unintended sheet
  write-back, even though the field is hidden in the UI.

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
- **`wrangler deploy` uploads whatever's on disk, tracked or not.** An
  untracked, leftover/incomplete scratch file
  (`apps-script-sheet-update.gs`) got deployed to production this way
  even though it was never `git add`ed. Always check the deploy's
  asset-upload list for anything unexpected before trusting a deploy.
- **`wrangler deploy` with no `--env` flag deploys straight to
  production** (`songdeetest`, from `wrangler.toml`'s top-level `name`),
  even from an unmerged feature branch — there's no implicit "preview by
  default" safety net. `wrangler deploy --env preview` is what targets
  `songdeetest-preview` (Wrangler appends `-<env>` to the worker name
  automatically; no `[env.preview]` section is needed in
  `wrangler.toml`, though it does print a harmless warning about that).
  **Always double-check the `--env preview` flag is actually present**
  before running deploy on a branch that hasn't been merged/approved yet.

## Workflow used throughout this project

Every change: feature branch → `wrangler deploy --env preview` (deploys
to `songdeetest-preview`, NOT production) → test → merge to `main` →
plain `wrangler deploy` (deploys to `songdeetest`, production). Never
deploy an untested/unmerged branch straight to production — double-check
the `--env preview` flag is on the command before running it.

## Status as of last update

- Production (`songdeetest`, `main` branch): **v1.13.2**.
- Includes, on top of the original v1.10.0 tpl-section-visibility split:
  a separate session's **AI Box/ADAS/DMS serial-number tracking +
  ISSUE Tracking auto-close** (v1.11.0, pushed to `main` directly from
  another machine while this session was mid-conversation — see git
  history around commit `379182a`/`1494cfd`/`fa0591d` for details); a
  **stale-data fix** (v1.11.1→1.12.1, superseded the abandoned
  `fix/tpl-stale-data-bug` branch — use `fix/tpl-stale-data-bug-v2`'s
  history instead) making `fixLine()`/`buildSheetUpdates()`/
  `buildDeviceSwaps()`/the ปัญหา section in `buildMessage()`/
  `buildLineCard()` all re-check the *current* tpl, not just whether
  data exists — same bug class as the pushIssueClose incident, just in
  more places; the **FIXES_CARD_LABEL** wording change (v1.12.0)
  labeling the "แก้ไข" card/button per job type instead of one word for
  all three; and the **vehicle equipment banner + no-generic-fallback**
  change (v1.13.0→1.13.1, see "Key features" above).
- **songdee-vehicle-lookup** (separate repo/Worker, only one live URL
  shared by both `songdeetest` and `songdeetest-preview`): equipment
  columns trimmed/relabeled (3G Module/AI System dropped, MCR→
  เครื่องรูดบัตร, Motor→มอเตอร์สั่น, Yes→มีอุปกรณ์) — deployed straight
  to its production URL and verified via `curl` before/after, since
  there's no isolated preview instance for it to test against and it's
  read-only. Also had the same cross-machine surprise as this repo —
  another session's AI Box/ADAS/DMS/ISSUE Tracking work
  (`6007f4f`/`4fd758c`) was on `origin/master` but not yet pulled
  locally; fetched and fast-forwarded before making any changes.
- `fix/tpl-stale-data-bug` (v1 — **stale/abandoned**, do not merge):
  built before the other session's v1.11.0 landed; superseded by
  `fix/tpl-stale-data-bug-v2`, which is the one actually in `main` now.
- Verified with a Node-based unit test (loads form.html's inline
  `<script>` into a `vm` context with a stubbed DOM/firebase, then calls
  `fixLine`/`buildSheetUpdates`/`buildDeviceSwaps`/`buildMessage`/
  `buildLineCard`/`renderTabs` directly against crafted `state`) rather
  than a live browser login, plus a live browser click-through once the
  user logged in themselves on preview and production — useful pattern
  for next time a fix needs proving without Claude ever touching
  credentials.
- **Reminder for whoever picks this up next (any machine)**: before
  making changes here or in songdee-vehicle-lookup/songdee-line-proxy,
  run `git fetch origin && git log --oneline <branch>..origin/<branch>`
  first — this project has been edited from multiple machines in the
  same stretch of time more than once already.
