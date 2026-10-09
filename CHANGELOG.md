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
  below) — existing `items`-based callers are unaffected. Also gained
  optional `imageUrls` field for the photo-attachment feature below —
  every photo renders as its own inline image at the bottom of the
  card's body (not a Flex "hero" block, which is limited to one image
  pinned above the header). Backward compatible: a singular `imageUrl`
  still works as a 1-photo shortcut, existing `items`-based callers
  are unaffected. Also gained an optional `groupId` field on the push
  endpoint (see "Per-technician LINE groups" below) and a `/กลุ่ม`
  webhook command that replies with the current chat's Group ID.
- **songdee-drive-proxy** (private) — shared Cloudflare Worker
  (previously songdee-admin-only) that uploads a base64 image to Google
  Drive via a Service Account and returns a public `viewUrl`. This
  app's origin (`songdeetest`/`songdeetest-preview`) was added to its
  `ALLOWED_ORIGINS`, and it gained an optional `category` field (used
  here as `"qc-report"`) to keep this app's uploads in their own Drive
  subfolder instead of mixing with Admin's.
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

## Admin change-history + delete (history.html v1.1.0, form.html v1.37.0)

User asked whether the ประวัติ (history) page could show everything that's
changed across the system — edits, deletes, additions — not just each
tester's own current list of reports. Two things didn't exist yet to
support that: history.html only ever queried `qc_reports.where('techUid',
'==', user.uid)` (your own reports only), and there was no delete feature
or change-tracking of any kind anywhere in the app — every edit just
overwrote the same doc via `.set(payload, {merge:true})`, so nothing
recorded what it used to say.

Added a new `qc_report_logs` Firestore collection, written by form.html's
`saveReport()` on every create/edit and by history.html's new admin delete
action:
- **Create**: one `{action:'create', changes:[]}` entry — the report's own
  existence in the "สร้างใหม่" feed entry is the record.
- **Update**: `diffReportFields(originalReportDoc, payload)` (new) compares
  the doc as loaded (`originalReportDoc`, captured in `loadReportIntoForm()`
  and refreshed after every save) against the new payload field-by-field —
  scalar fields (ทะเบียน/ลูกค้า/Fleet/ประเภทงาน/เครือข่าย/IMEI/Device
  ID/Server) by plain comparison, `checklist`/`stickers` key-by-key (with
  Thai labels pulled from `TEMPLATES[tpl].checklist`/`STICKERS`), `problems`/
  `fixes` index-by-index (fix rows summarized via a diff-only
  `fixRowSummaryForDiff()`, kept separate from `fixLine()` since that one
  depends on live `state.tpl`), and `photos` by added/removed filename. If
  nothing actually changed (e.g. a plain resend), no log entry is written —
  confirmed via a direct no-op save in testing.
- **Delete**: there was no delete feature at all, so one was added —
  admin-only, from history.html's new "ทั้งหมด (แอดมิน)" scope. Writes a
  `{action:'delete', ...}` log entry (including `ownerTechName`/`ownerUid`
  so the feed shows whose report it was) *before* calling
  `qc_reports.doc(id).delete()`, so the deleted report's identity survives
  in the log even though the doc itself is gone. Confirmed via
  `confirm()`, same pattern as the existing duplicate-submission dialog in
  form.html.

history.html itself gained, admin-only (`role.role === 'admin'`, same gate
`index.html` already uses for its Dashboard cards):
- A "รายงานของฉัน / ทั้งหมด (แอดมิน)" scope toggle. "ทั้งหมด" drops the
  `techUid` filter (`qc_reports.limit(300)`, sorted client-side like
  before) and shows every tester's reports with an owner tag and a 🗑️
  delete button per row.
- A "ประวัติการเปลี่ยนแปลง" card: `qc_report_logs.orderBy('at',
  'desc').limit(200)`, rendered as actor + action badge (เพิ่มใหม่/
  แก้ไข/ลบ) + timestamp + subject line, with the update entries' field
  diffs shown inline (`จาก → เป็น`, strikethrough/green).

Also fixed in passing: history.html had its own separate, smaller copy of
`TEMPLATE_LABELS`/`TEMPLATE_ICONS` that was never updated when the
`remove` ("ถอดย้ายอุปกรณ์") template was added to form.html's `TEMPLATES`
back in v1.33.0 — a `remove`-type report showed the generic 📄 icon and
raw `"remove"` text in the history list. Added the missing entry (same
📤 icon form.html's `TPL_ICON` uses).

**Needs a manual Firestore Console step** before the log actually writes,
same as `qc_reports` originally needed (see "Firestore rules for
qc_reports" below) — add `allow read, write: if request.auth != null;` for
the new `qc_report_logs` collection too, or every create/edit/delete will
silently fail to log (the report itself still saves/deletes fine either
way — `logReportChange()`/the delete path swallow their own errors to
`console.error` rather than blocking the user-facing save).

Verified with Playwright: a unit test drives form.html's actual
`saveReport()`/`state`/`currentReportId` globals directly (create → edit →
no-op resave) and checks the exact `qc_report_logs` payloads written,
including the human-readable checklist diff labels; a separate test mocks
both an admin and non-admin login against history.html, checking the scope
toggle/change-log card are hidden for non-admins, that the admin "ทั้งหมด"
view shows all testers' reports with owner tags and delete buttons, and
that clicking delete writes the log entry, removes the Firestore doc, and
refreshes both lists.

## Edit ชื่อช่าง inline from the tester roster (index.html v1.16.0)

Follow-up to the songdee-stock techName/role fix (v5.94): a user set up
an account with role="แอดมิน" in Stock's user manager, and because that
UI used to hide/clear the techName field for any role other than
"ช่าง"/"กำหนดเอง", several other accounts (รายชื่อผู้ทดสอบ card showed
them) were stuck with no techName — and since this app's own login
check only cares about `role.techName`, not `role.role`, those accounts
couldn't log in at all. The admin asked whether this could be fixed
directly from this card instead of going back to songdee-stock each
time.

`loadTesterRoster()` (the admin-only "รายชื่อผู้ทดสอบ" card on the
Dashboard) was read-only — it just rendered each `roles/{uid}` doc's
email/techName/role as plain text, with a red "ยังไม่ผูกชื่อช่าง" label
for anyone missing one. Replaced the static techName text with an
editable `<input>` per row, writing straight back to that same
`roles/{uid}.techName` field (same Firestore project/collection this
app already reads from for login — no new backend needed) on blur,
skipping the write entirely if the value didn't change. Enter key also
blurs/saves. Status line ("กำลังบันทึก.../บันทึกแล้ว ✓/บันทึกไม่สำเร็จ")
reuses the same `.name-map-status` pattern the "จัดการชื่อช่าง" card
right below it already uses, for consistency.

Verified via Playwright: card renders one editable input per user with
the current techName (or the red placeholder when empty); editing and
blurring a previously-empty field writes exactly the expected
`{techName: "..."}` to the correct uid and shows the success status;
blurring an unchanged field triggers no extra write.

## Fix: dropdown clipped/hidden by its own card (v1.36.1)

Follow-up bug report on v1.36.0's fix: "dropdown โดนบังมองไม่เห็น" — the
race-condition fix above wasn't the whole story. `.card` sets
`overflow:hidden` (so its rounded corners and the colored left-edge bar
don't poke out past the border-radius), and `.suggest-dropdown` was
`position:absolute` relative to `.suggest-field` *inside* that card — so
whenever the dropdown's height pushed past the card's bottom edge
(routine on a short card or a small screen, and exactly what the
ทะเบียน field hits since it's near the bottom of the ข้อมูลงาน card),
the overflowing part was clipped and invisible, even though it still
occupied layout space.

`getBoundingClientRect()` doesn't reveal this — an ancestor's
`overflow:hidden` doesn't shrink a descendant's own box, it just stops
painting the part outside the clip region. Confirmed the real bug with
`document.elementFromPoint()`: a point inside the dropdown's reported
rect but past the card's bottom edge hit-tested to the *next* card's
`.stock-lock-banner` instead of the dropdown item — proof the item was
there in the DOM but not actually visible or clickable.

Fixed by computing the dropdown's position in JS from
`input.getBoundingClientRect()` and rendering it `position:fixed`
instead of `position:absolute`, so it's positioned relative to the
viewport and escapes any ancestor's `overflow:hidden` entirely (safe
here since nothing animates a `transform` on an ancestor by the time a
user actually opens the dropdown — the card's one-time entrance
animation finishes within the first second of page load).
`wireSuggestInput()` now also repositions on scroll/resize while a
dropdown is open, since `position:fixed` no longer tracks the input
automatically the way `position:absolute` did.

Re-ran the same `elementFromPoint()` hit-test after the fix: the same
point now correctly resolves back to the dropdown's own `.item` div.

## Fix ทะเบียน/ชื่อลูกค้า/Fleet dropdown race + move 3G/4G into checklist (v1.36.0)

Two reports from real usage, sent as screenshots.

**Dropdown bug**: "กดช่องทะเบียนแล้วรายการไม่ขึ้น" — `wireSuggestInput()`
used to only get called *after* its data finished loading
(`loadPlateSuggestions()`'s `/plates` fetch, `loadCustomerFleetSuggestions()`'s
Firestore reads — all real network round trips). A technician tapping into
ทะเบียน/ชื่อลูกค้า/Fleet quickly after page load — very plausible, these are
some of the first fields filled — got no event listeners yet, so the tap
did nothing; once the data *did* arrive, nothing re-opened a field that
was already sitting focused, so the dropdown stayed stuck closed until
they clicked away and back in. Reproduced with Playwright using a
deliberately slow (800ms) `/plates` response: focusing the field at 150ms
correctly found it empty, but it never auto-populated once the data
landed at ~950ms.

Fixed by splitting "wire the input" from "load the data": `wireSuggestInput()`
now returns its internal `render()`, and is called immediately for all
three fields (its `getList` closure reads the live suggestions array, so
it's safe to wire before that array has anything in it). Each loader
calls the returned `render()` once its data arrives, but only if that
field still has focus — so a field opened before data was ready now pops
open the moment it's available, instead of requiring a second click.
Verified the same test passes after the fix (and fails again against the
pre-fix code, confirming it reproduces the real bug).

**3G/4G relocated**: moved from a selector under every individual
equipment row (v1.33.0) into the "รายการตรวจสอบ" (checklist) card as one
`state.network` field for the whole report, rendered as an extra row
alongside the per-template checklist items. Removed the per-row
`network` field from fix rows entirely (all three default-object
locations, `renderFixRows()`'s selector, `fixLine()`'s tag) — still
optional, still not in `getMissingFields()`. `buildMessage()` and
`buildLineCard()` both include "เครือข่ายอุปกรณ์: 3G/4G" under the
checklist section when set. Verified via Playwright: no per-row selects
remain anywhere, the checklist card shows exactly one network select,
picking a value updates both message builders' output, survives
switching template tabs (state-level, not per-template), and fix rows no
longer carry a `network` key at all.

## Warn on cross-device duplicate submission (v1.35.0)

Follow-up to v1.34.1. That fix only closes the gap *within one browser
tab* — `sendBtn.disabled`/`currentReportId` are in-memory JS variables,
invisible to a second computer. A technician opening the same ทะเบียน
+ วันที่ + ประเภทงาน on two machines (e.g. started on a phone, forgot,
opened it again on a laptop) and submitting both isn't caught by that
fix at all: confirmed via Playwright with two separate browser
contexts acting as "two computers" — before this change, both created
a Firestore doc and both pushed a LINE card (2 of each) with nothing
stopping it.

Also checked a related, different scenario first: two *different*
technicians pushing *unrelated* reports into the *same* LINE group at
the same time. That one needed no fix — ran the real
songdee-line-proxy worker module directly in Node with two concurrent
requests; it has no shared mutable state across requests (everything
is request-scoped), so concurrent pushes to the same group never
cross-contaminate, confirmed down to verifying each push's card only
ever contains its own ทะเบียน/ช่าง, never the other request's, and that
one request failing (simulated a 429) doesn't affect the other.

For the real gap (same person/job, different computer), added
`findRecentDuplicateReport()`: before saving a *new* report (skipped
entirely when `currentReportId` is already set — i.e. editing/resending
from history, which is an intentional repeat, not a slip-up), queries
`qc_reports` by `where('plate','==', state.plate)` only — deliberately
a single equality filter with no `orderBy` mixed in, so it needs no new
Firestore composite index — then filters by date/tpl and a 15-minute
recency window in JS. A match triggers a native `confirm()`: "ส่งไปแล้ว
X นาทีที่แล้ว (โดย {techName}) — ส่งซ้ำอีกหรือไม่?" — cancelling aborts
the send (no save, no push); confirming proceeds as a normal second
report. This warns, it doesn't hard-block, since a tech legitimately
returning to the same vehicle same day for the same job type is a real
(if rare) case.

Verified via Playwright across three scenarios: Computer B warned and
declined → only Computer A's doc/push exist (1 of each, not 2);
Computer B warned and confirmed anyway → both go through normally (2
of each, correctly, since that's now an intentional duplicate); and
resending/editing an existing report (`currentReportId` set) never
triggers the check at all, even when a matching "duplicate" exists.

## Fix: double-tap on "ส่งเข้า LINE" sent duplicate reports (v1.34.1)

Asked to test for a bug around multiple submits happening at the same
time. Found and reproduced a real one: `sendToLine()` only set
`sendBtn.disabled = true` *inside* the `if(pendingPhotoCount)` branch —
a report with no pending photos (the common case: no photos attached,
or already uploaded from a previous attempt) left the button clickable
for the entire save+push duration. Two fast taps (easy to do on a slow
connection, waiting to see if the first tap "took") raced two full
`sendToLine()` calls: two `qc_reports` docs saved and two duplicate
LINE Flex cards pushed into the group for the same report.

Reproduced with Playwright: dispatched two `sendBtn.click()` calls in
the same tick against a mocked Firestore `.add()`/LINE-push endpoint —
before the fix, both went through (`addCallCount: 2`,
`LINE_PUSH_COUNT: 2`); native `.click()` on an already-disabled button
doesn't dispatch a click event at all, so once `sendBtn.disabled` is
set unconditionally and *synchronously* at the very top of
`sendToLine()` (before any `await`), the second same-tick click is
swallowed by the browser before the handler ever runs again — verified
after the fix: `addCallCount: 1`, `LINE_PUSH_COUNT: 1`. Still safe for
a real double-tap specifically because JS is single-threaded: the
first click's synchronous disable always completes before the event
loop can process a second input event, no matter how fast the two taps
land.

This only guards one browser tab/session against itself — two
different technicians each submitting their own separate report from
their own device at the same time is normal, expected concurrency
(separate docs, separate cards) and isn't affected.

## ชื่อลูกค้า/Fleet/ทะเบียน เป็น dropdown ค้นหาได้ (v1.34.0)

Asked to make ชื่อลูกค้า, Fleet, and ทะเบียน dropdowns so they're easier
to search. ชื่อลูกค้า/Fleet already had a custom filtered dropdown
(`wireSuggestInput()`), but it only showed matches after typing at
least one character — clicking into an empty field showed nothing.
ทะเบียน used a native `<input list="plateList">` datalist instead,
which has inconsistent/poor UX on mobile (no reliable "show full list"
affordance, can't be styled to match).

Fixed both: `wireSuggestInput()` now also renders on `focus`, showing
the first 20 options unfiltered when the field is empty, so clicking in
opens a full browsable dropdown immediately — typing still narrows it
down as before. Converted ทะเบียน from the native datalist to the same
`wireSuggestInput()` widget as ชื่อลูกค้า/Fleet (new `plateSuggestions`
array + `jobPlateSuggestions` suggest-dropdown div, picking a plate
still sets `state.plate` and re-triggers `scheduleStockLockCheck()`
same as typing it manually) — all three fields now behave identically.
Still free text everywhere; the dropdown only suggests values seen
before, nothing is locked to the list.

Verified via Playwright: focusing each of the three fields with no
value shows its full suggestion list; typing filters it; clicking an
item fills the field and updates state; the old `<datalist>` element
is gone from the page.

## ถอดย้ายอุปกรณ์ template + 3G/4G network selector (v1.33.0)

Two asks: a 4th job-type template for equipment removal/relocation jobs
(previously there was no tab for a job that's just "ถอด" with nothing
installed in its place — `repair`/`upgrade`/`new_install` all assume
something ends up installed), and a way to record whether a device is
3G or 4G.

Added `remove` to `TEMPLATES` ("ถอดย้ายอุปกรณ์") with its own checklist
(ถอดครบตามรายการ / เก็บสาย-ปิดช่อง / ไม่มีความเสียหาย) — no ปัญหา section
(`tplHasProblems` stays false for it, same as `new_install`), and
`tplHasOldModel` is already `true` for any non-`new_install` template so
the "รุ่นเก่าที่เปลี่ยน" picker works for it with zero extra code — a fix
row with only `oldModel` set and no `product` already renders as
"- ทำการถอด {oldName}" via the existing `fixLine()` branching. Added
matching entries to `CHECK_PASS_LABEL`, `FIXES_CARD_LABEL`,
`FIXES_ADD_BTN_LABEL`, `TPL_ICON` (📤). Also added `remove: "ถอดย้ายอุปกรณ์"`
to songdee-line-proxy's `QC_TEMPLATE_LABELS` so `/ล่าสุด` shows the right
Thai label instead of falling back to the raw key.

Item wording/order for `remove`'s checklist (and the pre-existing
`new_install`/`repair` ones) is still a starting draft, not yet checked
against a real reference report the way `upgrade`'s was.

3G/4G: added a `network` field (`""`/`"3G"`/`"4G"`) to every fix row,
with a `<select>` rendered under every row in `renderFixRows()` —
universal across all 4 templates and every row, not scoped to specific
device types, since there's no reliable product-name signal for which
rows are network-capable devices. Left optional on purpose (not added to
`getMissingFields()`) — most equipment isn't a SIM-based device, so
making it required would block submission on rows where it doesn't
apply. When set, `fixLine()` appends it as a `[3G]`/`[4G]` tag after the
SN tag so it shows up in both the on-screen preview and the LINE
message text.

## Show ใบเบิก AND Spare stock together, grouped (v1.32.0)

Follow-up on v1.31.1 (which made switching technician override/clear
the ใบเบิก lock). Asked for something different instead: show BOTH
sources together — this ทะเบียน's ใบเบิก and the selected technician's
own Spare stock — split into separate groups, rather than either one
replacing the other.

`renderFixRows()`'s equipment `<select>` now renders two `<optgroup>`
sections when both sources have something ("จากใบเบิก {noStock}" and
"ของ Spare ({techStockOwnerName})"), falling back to a single group
(no change in behavior) when only one source applies, or the free-pick
catalog when neither does. `remainingFor()`'s per-row accounting now
sums both sources into one combined pool per product name first — so a
product appearing in both groups (e.g. both the ใบเบิก and this tech's
Spare stock happen to include "เสาอากาศ GPS") tracks one shared
remaining count across every row and every occurrence of that name,
instead of each source tracking its own separate count.

Reverted the v1.31.1 "clear stockLock on technician switch" change —
no longer needed/correct now that both sources show together rather
than compete; switching technician only needs to recompute the Spare
side, the ใบเบิก side is untouched (it's tied to ทะเบียน, not
technician). `updateProductBanner()` now describes both sources
together when both are active, in addition to the existing
single-source and neither-source messages.

Verified via Playwright: both sources active renders two correctly
labeled optgroups with correct per-item remaining counts including a
deliberately overlapping product name; only-lock and only-Spare each
still render their one group correctly; neither source falls back to
the flat, ungrouped catalog; claiming an overlapping-name item in one
row correctly reduces the remaining count shown in every other row's
occurrence of that name in BOTH groups (combined-pool accounting), while
the claiming row's own dropdown still excludes its own claim per the
existing remainingFor() design; all three banner-text variants render
correctly.

## Switching technician didn't change the equipment picker (v1.31.1)

Reported as: "เวลาเปลี่ยนชื่อช่างแล้วของที่สามารถเลือกได้ไม่เปลี่ยนตาม"
(switching technician doesn't change the selectable items). Root cause:
`renderFixRows()`'s `lockSource` always prefers an active ใบเบิก lock
(`stockLock`, matched by ทะเบียน) over a technician's own Spare stock
(`techStock`) — by design, since a ใบเบิก is tied to the job/plate. But
nothing re-checked or cleared `stockLock` when the "ช่างที่กำลังทำ
รายงานนี้" select changed, so whenever a ทะเบียน already had a matched
ใบเบิก, switching technician silently did nothing — the picker stayed
stuck on that ใบเบิก's items no matter who got selected. Confirmed via
Playwright before fixing: switching between two technicians with
different Spare stock showed the same ใบเบิก items both times.

Asked which behavior was wanted — switching technician immediately
override to that person's own Spare stock, or keep the ใบเบิก priority
and just explain it better — and the answer was override. The
select's `onchange` now clears `stockLock = null` before applying the
new technician's stock, so the switch takes effect right away;
re-typing/blurring ทะเบียน re-triggers `checkStockLock()` and can
re-lock it if that's wanted again.

Verified via Playwright: with an active ใบเบิก lock, switching
technician now clears it and shows each technician's own distinct
Spare stock; the no-lock case (nothing changed here) still works as
before.

## Require at least one ปัญหา/แก้ไข row (v1.31.0)

`getMissingFields()` already blocked "ส่งเข้า LINE" (disabled button +
"ยังไม่ครบ" badge) until วันที่/ทะเบียน/stickers/checklist were filled —
but ปัญหา and แก้ไข had no completeness check at all. A repair report
could send with zero ปัญหา logged, and any report could send with zero
รายการแก้ไข/อุปกรณ์ที่ติดตั้ง, and still pass.

Now reuses `problemLine()`/`fixLine()` — the same functions
`buildMessage()`/`buildLineCard()` already use to decide whether a row
produces real text — to require: at least 1 ปัญหา for repair (the only
template `tplHasProblems()` is true for), and at least 1 แก้ไข/
อุปกรณ์ที่ติดตั้ง row for every template, labeled per-template via the
existing `FIXES_CARD_LABEL` map.

Deliberately did NOT add the new MDVR IMEI field (v1.30.0) to this list
yet — songdee-vehicle-lookup's `/mdvr-available` isn't deployed (no CI
there, needs a manual `wrangler deploy`), so the IMEI dropdown has
nothing to select from on live right now; requiring it would hard-block
every ติดตั้งใหม่ submission until that deploy happens. Left a comment
at the spot to add it once that backend is confirmed live.

Verified via Playwright: repair with empty ปัญหา+แก้ไข blocks on both;
filling ปัญหา only still blocks on แก้ไข; filling both clears the list;
upgrade doesn't require ปัญหา (not in `tplHasProblems`) but does require
แก้ไข, labeled "ถอด/ติดตั้งอย่างน้อย 1 รายการ" via `FIXES_CARD_LABEL`;
new_install with an empty MDVR IMEI is correctly NOT blocked.

## Capture IMEI/Device ID/Server for ติดตั้งใหม่ (v1.30.0)

ติดตั้งใหม่ previously had no way to record which physical MDVR unit was
installed — Device ID and Server had no field anywhere, and the
equipment picker's SN field only exists for AI Box/ADAS/DMS. Confirmed
architecture: each MDVR unit is already a row in the "MDVR Tracking"
sheet (added at stock intake, matched by IMEI, `Vehicle No.` still
empty) — not a separate spreadsheet, and not something this app creates
rows for.

- **songdee-vehicle-lookup**: new `GET /mdvr-available` (unassigned IMEI
  rows — empty `Vehicle No.`) and `POST /mdvr-install` (`installMdvrDevice()`
  — finds the row by IMEI, 404s as `"imei not found"` if it doesn't exist
  rather than creating one, and writes `Vehicle No.`/`Device ID`/`Server`
  plus the same `Customer`/`Installed Date`/`Technician1` columns
  `swapDevice()` already writes, each skipped individually if the sheet
  doesn't have that column). `Server` is validated server-side against a
  fixed list (`60`/`63`/`73`/`75`/`90`, the sheet's own dropdown values) —
  rejected before it reaches the sheet instead of silently failing that
  column's validation after a successful write.
- **form.html**: new "ข้อมูลเครื่อง MDVR" card, shown only for ติดตั้งใหม่
  — IMEI dropdown (populated from `/mdvr-available`, loaded once at
  startup independent of ทะเบียน), Device ID (free text — unlike IMEI,
  there's no pre-existing value to pick from), Server dropdown (the 5
  fixed options). `pushMdvrInstall()` fires in `sendToLine()`, non-fatal
  like the other sheet write-backs, only when an IMEI was actually
  picked. Persisted to `qc_reports` and restored by `loadReportIntoForm()`
  like every other field, so reopening a ติดตั้งใหม่ report to correct it
  keeps its IMEI/Device ID/Server.

Verified via a Node test against the Worker with a mocked gviz CSV +
Sheets API (a throwaway RSA keypair stands in for the service account
key, since `getGoogleAccessToken()` needs a syntactically valid PKCS8
key to sign with — the real token exchange is mocked, so the key's
actual validity doesn't matter): `/mdvr-available` lists only the
empty-plate row; a valid install writes exactly the 6 expected cells; an
unknown IMEI and an invalid Server value both reject with 500; a
non-allowed origin gets 403. Playwright on the form side: the card
shows/hides correctly per tab, `pushMdvrInstall()` no-ops with no IMEI
picked or on any other job type, the real POST body matches what was
entered through the actual `<select>`/`<input>` elements, and
`loadReportIntoForm()`/`startNewReport()` restore/clear the three fields
correctly.

## Color the resend card amber too (v1.29.1)

Follow-up on the resend badge above — asked for the resent card to stand
out by color, not just by its badge text. `buildLineCard()`'s `color`
(the Flex header background) is now `#ffb020` for a resend instead of
the normal `#3ddc97` green — the same amber already used for the app's
own stock-lock/issue "needs a look" banners, so it reads consistently
with the rest of the UI rather than introducing a new color meaning.

## Mark a resent report as an edit in LINE (v1.29.0)

LINE's push API can't edit or replace a message already sent, so
reopening a past report from history (`form.html?report=ID`) and sending
it again — the only way to correct a report, since the form only
requires วันที่/ทะเบียน and lets everything else go in blank — posts a
brand-new card into the group with no link back to the original. Readers
in the LINE group had no way to tell two cards for the same job apart,
or know which one was current.

Added `isResendOfExistingReport`, set true by `loadReportIntoForm()`
(only called when opening a previously-saved report) and cleared by
`startNewReport()`/`startNextVehicle()`. When true, `buildMessage()`
prefixes the plain-text message with "📝 แก้ไขจากรายงานเดิม (ส่งซ้ำ)" and
`buildLineCard()`'s badge line becomes "📝 แก้ไขจากรายงานเดิม (ส่งซ้ำ) ·
{date}" instead of the plain date badge. Doesn't link back to the
original card (LINE has no way to reference an older message from a
push) — just flags the resend so it isn't mistaken for an unrelated
second report.

## Surface Device Tracking SNs that weren't found (v1.28.1)

Read songdee-vehicle-lookup's `swapDevice()` in full while answering a
question about what exactly changes in the AI Box/ADAS/DMS Tracking sheet
on submit: it returns `{ok:true, updated, notFound}` — an SN with no
matching row in the tracking sheet is silently skipped, not an error. But
`pushDeviceSwaps()`'s caller in `sendToLine()` only ever read `swapRes.length`
for the "✓ อัปเดต" toast text, never `notFound` — so a mistyped or
unregistered SN reported the same success toast as one that actually wrote.

`sendToLine()` now flattens `notFound` across every swap result and, if
any SN wasn't found, appends it to the toast: "(หา SN ไม่เจอในชีต
ไม่ได้อัปเดต: ...)". No change to what gets written — this only makes an
already-silent partial failure visible instead of indistinguishable from
full success.

## Per-technician LINE groups (v1.15.0 / v1.28.0)

Previously every report went to one single shared LINE group
(`env.LINE_GROUP_ID` in songdee-line-proxy, hardcoded, no per-request
override). Requested so different technicians'/teams' reports can land in
their own LINE group instead of one shared feed.

- **songdee-line-proxy**: push endpoint now accepts an optional `groupId`
  field in the request body. If present and non-empty, the message is
  pushed there instead of `env.LINE_GROUP_ID`. If absent (Stock/Admin's
  existing calls, and any qc-report tester with no group configured),
  behavior is unchanged — pushes to the default shared group. Also added
  a `/กลุ่ม` webhook command: sent inside any LINE group chat, the bot
  replies with that group's ID, so setting up a new group needs no log-
  digging — add the bot, type `/กลุ่ม` in the group, copy the ID it
  replies with.
- **qc-report admin panel** (index.html, admin-only): new "ตั้งกลุ่มไลน์
  ตามช่าง" card, same pattern/UI as the existing ชื่อช่าง↔Admin name-map
  editor — maps `techName -> LINE Group ID`, stored in the same generic
  `sd` blob store (`sd:lineGroupMap`, doc id `sd_lineGroupMap`). Multiple
  technicians can share the same Group ID (routes them to one team's
  group); a technician with no row here still goes to the default shared
  group, so nothing breaks while the mapping is being filled in.
- **form.html**: `sendToLine()` looks up the logged-in tester's own
  `techName` in this map. If mapped, the push includes `groupId` and the
  report goes **only** to that group (not also to the shared one, per
  what was asked). If unmapped, `groupId` is omitted and the proxy falls
  back to the shared group as before.

## Photo attachments (v1.14.0)

Added an optional "แนบรูปภาพ" card (up to 6 photos/report) so a tech can
attach evidence photos (equipment close-ups, before/after, serial
plates, etc.) alongside the existing text fields.

- Each picked photo is downscaled + re-encoded as JPEG client-side
  (canvas, capped at 1440px on the long edge, quality 0.8) before
  upload — a phone camera photo straight off the sensor is routinely
  3-8MB, way more than needed for a reference photo and too big to
  ship comfortably through the Drive proxy's JSON body.
- On "ส่งเข้า LINE", any not-yet-uploaded photos are POSTed 2 at a time
  (`PHOTO_UPLOAD_CONCURRENCY` — enough to meaningfully cut wall-clock
  time for a multi-photo report without bursting the Drive API) to
  **songdee-drive-proxy**'s `/upload` (`date`/`plate`/`category:
  "qc-report"` passed through for Drive folder organization). Only the
  returned `viewUrl` is ever kept — the base64 bytes are dropped from
  memory right after a successful upload, and never written to
  Firestore.
  - Non-fatal like the existing sheet-update/device-swap/issue-close
    calls in `sendToLine()`: a failed photo upload just drops that one
    photo from the message/card/save and shows a count in the toast —
    it doesn't block the rest of the report from sending.
- Every successfully uploaded photo becomes its own inline image at
  the bottom of the LINE Flex card's body (`buildLineCard()`'s new
  `imageUrls` array) — see the songdee-line-proxy note above.
  `buildMessage()`'s plain-text preview (and clipboard-copy fallback)
  also lists every uploaded photo's `viewUrl` as a plain link, so the
  photos stay reachable even if the Flex card doesn't render for some
  reason.
- Persisted to `qc_reports` as `photos: [{name, driveUrl}, ...]` —
  uploaded ones only, never local/base64 state — so reopening a report
  from Dashboard/history restores the same viewUrls (re-attaching a
  removed/re-picked photo just uploads fresh).
- **Setup required before this actually works end-to-end**: the
  `DRIVE_API_KEY` constant hardcoded in form.html (same
  hardcoded-client-secret pattern `LINE_API_KEY` already uses — flagged
  as a pre-existing issue below, not something new) must match
  songdee-drive-proxy's `UPLOAD_API_KEY_QC` GitHub Actions secret — a
  **new, separate** secret from admin's existing `UPLOAD_API_KEY`
  (songdee-drive-proxy's worker.js now accepts either), so setting this
  up can never break songdee-admin's own uploads. A fresh key was
  generated for this feature; whoever deploys next needs to add it as
  that repo's `UPLOAD_API_KEY_QC` secret (Settings → Secrets and
  variables → Actions) before merging/deploying, or photo uploads will
  fail with 401 (report sending itself still works — see "non-fatal"
  above).

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

While testing, also confirmed (directly with the project owner, not
assumed from data — every sample ISSUES-All row, open or closed, showed
`COMPLIANCE=YES`, so it couldn't be inferred from the sheet alone) that
`COMPLIANCE` means "was this problem fixed" (YES) vs "still has a
problem" (NO/Unknown/Wait approve). `closeIssue()` (songdee-vehicle-lookup,
commit `58ad610`) now also writes `COMPLIANCE=YES` when closing a
ticket — no change needed on this side, purely a server-side addition.

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
  and separator so it doesn't read as one undifferentiated blob. Now
  also carries every attached photo as its own inline image at the
  bottom of the card — see "Photo attachments" above.
- **Photo attachments**: optional, up to 6 per report, uploaded to
  Google Drive via songdee-drive-proxy on send. See "Photo attachments"
  section above for the full writeup.
- **Report persistence**: every "ส่งเข้า LINE" click also saves/updates
  a doc in Firestore `qc_reports` (own collection, not shared with
  songdee-stock/songdee-admin's report collections). `currentReportId`
  tracks create-vs-update; reopening from Dashboard/history sets it.
- **"+ ทะเบียนถัดไป"** (`startNextVehicle()`): for a customer visit
  covering several vehicles, keeps วันที่/ลูกค้า/Fleet/ประเภทงาน but
  resets everything ทะเบียน-specific (plate, ปัญหา/แก้ไข/สติ๊กเกอร์/
  checklist, stock-lock, old-model lookup, attached photos) so the next
  vehicle's report starts clean without retyping customer info. Each
  vehicle is still its own independent "ส่งเข้า LINE" + `qc_reports`
  doc — deliberately **not** grouped/batched, since testing happens one
  vehicle at a time and sometimes by a different technician entirely
  (decided against adding cross-report grouping on Dashboard/history —
  not worth it unless a real need for progress-tracking across techs
  shows up later).
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
  `DRIVE_API_KEY` (added for photo attachments) follows the same
  existing pattern — same caveat applies, and it also needs a matching
  value set on songdee-drive-proxy's side (see "Photo attachments"
  above) before it actually works.
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

**Now automated via GitHub Actions** (`.github/workflows/deploy.yml`,
added since the owner doesn't run `wrangler` locally): merging to `main`
auto-deploys to `songdeetest-preview` — no manual step needed to get a
fresh preview. Production (`songdeetest`) still never deploys on its
own; it only happens when someone manually runs the same workflow from
the repo's Actions tab and picks "production" from the dropdown, after
checking preview themselves. This preserves the "never deploy
untested to production" rule above while removing the need for a local
wrangler setup. Requires a `CLOUDFLARE_API_TOKEN` repo secret (same
"Edit Cloudflare Workers" token type used by songdee-drive-proxy/
songdee-line-proxy) — also added an explicit `account_id` to
`wrangler.toml` since a non-interactive CI run can't prompt to pick an
account the way a local `wrangler deploy` can.

## Status as of last update

- Production (`songdeetest`, `main` branch): **v1.13.2** (this session
  adds **v1.14.0**, photo attachments, on a feature branch — not yet
  merged/deployed; see "Photo attachments" above for the
  `UPLOAD_API_KEY_QC` setup step that has to happen before it works
  end-to-end).
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
  all three; the **vehicle equipment banner + no-generic-fallback**
  change (v1.13.0→1.13.1, see "Key features" above); and now **photo
  attachments** (v1.14.0, see above).
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
- **songdee-drive-proxy**: previously songdee-admin-only. Gained this
  app's origins in `ALLOWED_ORIGINS` and an optional `category`
  subfolder param for the v1.14.0 photo-attachments feature — see
  "Photo attachments" above.
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
  credentials. The photo-attachment functions (`compressPhotoFile`,
  `uploadPhotoToDrive`, `buildLineCard`'s new `imageUrl`/`links`) were
  syntax/logic-checked the same way (no live browser/camera available
  in this session) — **still needs an actual live-browser pass** (pick
  a real photo, confirm the compressed upload lands in Drive, confirm
  the LINE card actually renders the hero image) before trusting it in
  production.
- **Reminder for whoever picks this up next (any machine)**: before
  making changes here or in songdee-vehicle-lookup/songdee-line-proxy/
  songdee-drive-proxy, run `git fetch origin && git log --oneline
  <branch>..origin/<branch>` first — this project has been edited from
  multiple machines in the same stretch of time more than once already.
