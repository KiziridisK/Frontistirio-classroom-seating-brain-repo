# Frontistirio — Classroom Seating Brain

> **Κατάσταση τεκμηρίωσης** (σφραγίδα 2026-10-24) · τελευταία γραφή **2026-08-20**.
> **Δεν φέρει επαλήθευση έναντι κώδικα** — κανόνας #4 του ευρετηρίου. Από εκείνη την ημερομηνία ο κώδικας
> έχει προχωρήσει **110 commits** στο frontend και **32** στο API (όλο το repo, όχι μόνο αυτό το feature).
> Σημερινά HEAD: frontend `0f466e0` / API `cabaa2c`.
> Τι είναι **γνωστό** ότι λείπει: §5 και §5.1 του ευρετηρίου (`Frontistirio-brain-repos-index`).

> **Full-stack reference** for the classroom seating feature built 2026-06-28: the per-classroom
> floor-plan (κάτοψη) editor, the per-test-day seating assignment (drag students onto seats), and the
> printable PDF export. Spans the `ClassRoom.layout` model, the `Test.seating_plan` model, the Angular
> components (fabric.js canvas + native HTML5 drag-drop), the backend endpoints, and the seating PDF.
>
> **Updated 2026-07-23:** round desks (§Desk shapes), the same-course-at-one-desk warnings
> (§Conflict warnings), and the touch/tap-to-place + responsive-zoom interaction that had shipped
> undocumented in commit `8317565`.
>
> Related brains: [test-cycles](../Frontistirio-test-cycles-brain-repo/README.md) (host of the seating
> UI), [pdf-generation](../Frontistirio-pdf-generation-brain-repo/README.md) (the seating PDF helper).

---

## 2026-11-20 — fabric 6.9 → 7.4 (ασφάλεια) — live στο 1.6.10 (frontend `9d92ce2`)

- `package.json`: `"fabric": "^6.9.0"` → `"^7.4.0"`. Ο λόγος είναι ασφάλεια: η 7 ρίχνει τα προαιρετικά node deps της 6
  (jsdom 20, `canvas`, και το **critical** `tar` του `npm audit`).
- Committed `9d92ce2`, live στο **1.6.10** (2026-11-22, tag `v1.6.10` → `eb99e99`).
- **Η αλλαγή συμπεριφοράς της 7:** το default `originX/originY` γίνεται `center`. Εδώ δεν πειράζει, γιατί κάθε shape στο
  `classroom-diagram` και στο `test-seating-modal` ορίζει ρητά το origin του, και η `Line` της 7 τοποθετείται από το κέντρο.
  **Κανόνας:** νέο shape → όρισε ρητά `originX`/`originY`.
- Επαληθεύτηκε στο e2e harness: προσθήκη θρανίου / στρογγυλού θρανίου / θέσης, η θέση δένεται και ακολουθεί το θρανίο,
  αποθήκευση + reload ξαναφέρνει τη διάταξη, το modal καθισμάτων τοποθετεί και αποθηκεύει.

## 2026-11-03 — fabric εκτός initial bundle (frontend `dc6cfff`, dev)

- `/classrooms/:classroomId` (`ClassroomDiagramComponent`) → `loadComponent`.
- Το modal καθισμάτων (`TestSeatingModalComponent`) φορτώνεται με `await import('../test-seating-modal/…')` μέσα στο
  `test-cycle-details` τη στιγμή που ανοίγει· δεν υπάρχει πια στατικό import του.
- Αποτέλεσμα: το `fabric` (~400 kB πριν το minify) δεν κατεβαίνει σε κάθε χρήστη. **Κανόνας:** όποιο νέο σημείο χρειάζεται
  `fabric` το φέρνει lazy (route ή `import()`), αλλιώς ξαναμπαίνει στο `main`.
- Σχετικό (API `14ecbc7`): τα PDF κύκλου/συμμετεχόντων δεν τυπώνουν διαγραμμένους μαθητές (bugs → TC-08)· η διαγραφή
  αίθουσας καθαρίζει μόνο `classroom_id/classroom_name` στις ώρες του προγράμματος και η μόνιμη αρνείται όταν ένα
  διαγώνισμα έβαλε μαθητές να κάτσουν σε αυτήν (course-class-management → «Κάδος ανακύκλωσης»).

## Three phases (all done)

1. **κάτοψη editor** — design a classroom's layout (desks + seats), saved on the classroom.
2. **seating assignment** — per test day, drag participants onto seats in a modal, saved on the test.
3. **PDF export** — per test day, export the seating plan (one landscape page/room with names) or the
   participants list.

Tech: **fabric.js v6** for the canvas (Angular 18, Ionic). **No `@angular/cdk`** — drag-drop is native
HTML5 on desktop, **tap-to-place** on touch (see Phase 2). Greek `firstName`/`lastName` on the
`Student` model are **camelCase**.

---

## PHASE 1 — κάτοψη editor

### Data model

**Frontend** `src/app/models/classroom-layout.model.ts`:
```typescript
type DeskShape = 'rect' | 'circle';           // absent = 'rect' (2026-07-23)
interface SeatLayout { id: string; deskId: string | null; left: number; top: number; label?: string; }
interface DeskLayout { id: string; left: number; top: number; width: number; height: number; angle: number; shape?: DeskShape; label?: string; }
interface ClassroomLayout {
  version: number;                              // starts at 1
  canvasSize: { width: number; height: number }; // default 1200x800
  desks: DeskLayout[];
  seats: SeatLayout[];
  updatedAt?: string;
}
```
Added to `ClassRoom` (`models/classroom.model.ts`): `layout?: ClassroomLayout;`

**Backend** `models/classroom.js` — sub-schemas `deskSchema` / `seatSchema` / `layoutSchema` (all `_id: false`),
field `layout: { type: layoutSchema, default: null }`. `left/top` = **absolute canvas coords of the center**.
`deskSchema.shape: { type: String, enum: ["rect","circle"], default: "rect" }`.

### Desk shapes *(2026-07-23)*

A **round desk is an ellipse with `rx = width/2`, `ry = height/2`** — it reuses the existing
width/height instead of adding a radius field, so scaling, rotation, persistence and the PDF's
bounds math are identical for both shapes (and an uneven scale gives an oval table for free).
`shape` is **optional and absent means `rect`**, so every layout saved before this needed **no
migration** — the schema default and `normalizeLayout` backfill it on the next save.

- Editor: `createDeskGroup(id, label, w, h, shape)` builds a `fabric.Ellipse` or a `fabric.Rect`;
  `addDesk(shape)` behind two toolbar buttons (new circle default **120×120**, rect stays 140×70).
  i18n `classrooms.diagram.add_round_desk`, icon `disc-outline`.
- **`deskContainsPoint(desk, point)`** — fabric's `containsPoint` only tests the **bounding box**,
  which overshoots at the corners of a round desk, so a seat dropped in a corner would bind to a desk
  it doesn't visually sit on. Circle desks therefore get an extra ellipse test
  (`(x/rx)² + (y/ry)² ≤ 1`) in the desk's own unscaled coords via `invertTransform`.
- `deskToLayout` writes `shape`; `loadLayout` and paste pass it through; the test-seating modal's
  read-only `buildDesk` mirrors the same branch.
- Backend `normalizeLayout` whitelists it: `d.shape === "circle" ? "circle" : "rect"`.
- **Verified end-to-end**: the `save-layout` payload carried `shape:"circle"` for the new desk and
  `shape:"rect"` for the pre-existing ones; the seating PDF's content stream contained exactly
  **2 `re` (rect) operators** (the rect desk + a legacy shapeless one) and 24 bezier segments
  (4 seats + 2 ellipse desks × 4).

### Component `ClassroomDiagramComponent` (standalone)
- Path: `src/app/classrooms/classroom-diagram/`. Route **`classrooms/:classroomId`** (RoleGuard, `store-user`).
- Reached from the arrow button in `classroom-item` → `goToClassRoom()` navigates `['/classrooms', _id]`
  (was previously the **broken** `['/classroom', _id]`, no such route).
- fabric canvas 1200×800 + grid. **Desk** = `fabric.Group` (rect + label), move/rotate/scale.
  **Seat** = `fabric.Group` (circle + label), **manually dragged** onto a desk.
- Desk↔seat link: on seat drop, bind to the containing desk via `containsPoint`; the seat's offset is
  stored as a **local point** computed from the desk's transform matrix
  (`fabric.util.invertTransform` + `transformPoint`). When a desk moves, `repositionSeatsOfDesk`
  re-applies `desk.calcTransformMatrix()` so seats follow. Persisted = **absolute coords + deskId**
  (the local offset is recomputed on load).
- Copy/paste: **Ctrl+C / Ctrl+V** (`@HostListener('window:keydown')`) + toolbar buttons. Copying a desk
  brings its seats; paste assigns **new ids** to desks & seats and remaps `seatId → deskId`. Pasted seats
  get **fresh sequential numbers** (so copies don't duplicate the source numbering). Guards against
  hijacking copy/paste while typing in the rename dialog.
- Save: `ClassroomService.saveClassroomLayout(classroomId, layout)` → store dispatch
  `updateClassRoomFields({ id, changes: { layout } })`.

### Backend
- `routes/classrooms.js`: `POST /classrooms/save-layout` (auth + isStoreUser).
- `controllers/classroom.js` `saveStoreClassRoomLayout` (session/transaction, 404 if not found).
- `controllers/handlers/classroom.js` `saveStoreClassRoomLayout` + `normalizeLayout` — coerces numbers,
  drops desks/seats without `id`, nulls a seat's `deskId` if it points to a desk not in the payload.
  `Test`-style IDOR scoping: `updateOne({ _id, store_id }, { $set: { layout, updatedBy } })`.
- The bootstrap (`controllers/bootstrap.js` → `fetchStoreClassRooms` plain `find()`) returns `layout`
  automatically, so the editor loads it from the NgRx store on page open.

### Removed (cleanup)
The old orphan `AddClassroomDiagramComponent` (route `classrooms/add`, localStorage-only prototype,
broken spec, duplicate `app-add-classroom` selector) was deleted — folder + `app.module.ts`
declaration/import + the unused import in `classrooms.component.ts`.

---

## PHASE 2 — seating assignment (per test day)

### Data model
**Backend** `models/test.js` — sub-schemas `seatingAssignmentSchema` / `seatingPlanSchema` (`_id:false`):
```javascript
seating_plan: {
  assignments: [{ classroom_id: ObjectId→ClassRoom, seat_id: String, student_id: ObjectId→Student }],
  updatedAt: String,
}   // default null
```
`seat_id` matches a `SeatLayout.id` in the classroom's `layout`.
**Frontend** `models/test.model.ts`: `SeatingAssignment`, `SeatingPlan`, and `Test.seating_plan?`.

### UI — host `TestCycleDetailsComponent`
- Each **saved** test day (a day with a Test `_id`) gets a **"Πλάνο"** button (disabled if `!day._id`
  or no classroom has a layout). It opens `TestSeatingModalComponent` as a 95%×95% modal
  (`cssClass: 'custom-width-iframe-modal2'`), passing
  `{ testId, participants, grades, courses, period, classrooms (with-layout), courseIds, assignments }`.
- The component subscribes to `selectClassRooms` → `classroomsWithLayout = filter(!isDeleted && layout?.seats?.length)`.

### `TestSeatingModalComponent` (standalone, `src/app/test-cycles/test-seating-modal/`)
- **Left ~70%:** `ion-segment` tabs (one per classroom with a layout) + a **read-only** fabric canvas
  rendering the active room's layout (reuses the desk/seat build logic; everything `selectable:false`).
- **Right ~30%:** participants grouped grade→course (same logic as
  `getParticipantsGroupedByGradeAndCourse`), each student a **native HTML5 draggable** chip; assigned
  students are removed from the list.
- **Drag chip → seat** = bind (nearest free seat within ~30px, converting drop coords with
  `getBoundingClientRect`). **Click an occupied seat** = unbind (returns student to list). Rules: **1
  seat per student, 1 student per seat**.
- **Tap-to-place (touch)** *(shipped in `8317565`, documented 2026-07-23)* — HTML5 drag-and-drop does
  not exist on touch, so the primary mobile interaction is **tap a chip to "arm" it
  (`armedStudentId`), then tap a seat**. Handled in fabric's `mouse:down` (fabric maps touch → mouse,
  so no drag-drop polyfill). Tapping an occupied seat with nothing armed still unbinds.
- **Responsive fit** *(same commit)* — `applyFit()` + a `ResizeObserver` scale the natural-size layout
  down to the pane width (`zoom`, never > 1). Drop coordinates are divided by `this.zoom` to get back
  to scene coords.
- Save: `TestCycleService.saveTestSeating(testId, assignments)` → toast → dismiss with assignments.

### Conflict warnings — same course at one desk *(2026-07-23)*

Two students sitting **the same exam** at **the same desk** is flagged. It was first built as a hard
block and then **deliberately relaxed to a warning**: the plan is the user's call and there may be a
reason for the seating, so the placement always goes through.

- **`dayCoursesByStudent`** (`indexDayCourses()` in `ngOnInit`) — `studentId → the day's courses the
  student sits`, i.e. `student.period_courses[period].courses ∩ courseIds`. A student can sit **more
  than one** of the day's courses, so this is a **list** and the check is a **set intersection**, not
  an equality test. `rebuildGroups` reads the same map — one source of truth.
- **`findDeskConflict(studentId, seatId)`** — scans the other seats sharing the target seat's
  `deskId` in the active room. Seats with `deskId: null` stand alone and never conflict.
- **`assignStudentToSeat`** assigns regardless, then raises a **warning toast**
  (`color: 'warning'`, `warning-outline`, 4s, `messages.seating_same_course_desk` with
  `{{student}}`/`{{course}}`). `presentToast` was widened to take `{ params, color, icon, duration }`.
- **Standing list** — `conflicts: SeatingConflict[]` is **derived state recomputed inside
  `syncIndexes()`**, so it (a) clears itself the moment the plan is fixed and (b) flags plans saved
  *before* this check existed, the moment the modal opens. One entry per **(desk, course)** carrying
  every student in it, so 3+ students at one desk collapse into a single warning.
- **UI** — a pinned panel under the participants list: `.list-scroll` takes the overflow so
  `.conflicts` stays in view. Clicking a warning calls `focusConflict()`, switching the classroom tab.
  i18n `test_cycles.seating.conflicts_title` / `.desk`.
- Conflicts are **frontend-only** — nothing is persisted or validated server-side.

### Performance gotcha (important)
The hot path is rendering, not data. Fixes that mattered:
- **In-place seat update** (`refreshSeat(seatId)` via a `seatObjById` map) instead of full `canvas.clear()`
  + rebuild on every drop.
- **Run fabric + the continuous `dragover`/`drop`/`mouse:down` listeners OUTSIDE Angular's zone**
  (`ngZone.runOutsideAngular`, listeners attached imperatively). Without this, every `dragover` tick
  triggered a full change-detection pass over the heavy Ionic accordion list. Only the list rebuild is
  wrapped back in `ngZone.run(...)`.
- **`trackBy`** on all participant `*ngFor`s so Angular removes only the one chip that changed.

### Backend
- `routes/tests.js`: `POST /test-cycles/save-test-seating` (auth + isStoreUser).
- `controllers/tests.js` `saveTestSeating` (session/transaction, 404).
- `controllers/handlers/tests.js` `saveTestSeating` + `normalizeSeatingAssignments` — ObjectId coercion,
  **dedupes by seat and by student**, store-scoped `Test.findOneAndUpdate({ _id, store_id, isDeleted:false })`.
- `getStoreTestCycleTests` (plain `find()`) returns `seating_plan` automatically.

---

## PHASE 3 — PDF export (per test day)

Two-role **"Εξαγωγή"** button next to "Πλάνο" on each day → `ActionSheetController` with two options:
- **Export seating plan** → `TestCycleService.exportTestSeatingPlan(test_id)` → `POST /test-cycles/export-store-test-seating`.
- **Export participants** → `TestCycleService.exportTestParticipants(test_id)` → `POST /test-cycles/export-store-test-participants`.

Both return `{ success, downloadUrl, filename }` and download via a shared `triggerDownload` (anchor with
`download` attr). The pre-existing **global** export (`exportReport`, next to the breadcrumbs) is unchanged;
the per-day participants export is the same PDF scoped to one test.

### Backend (`controllers/tests.js`)
- `exportTestParticipants` — fetches one Test (store-scoped, same populate as the global report), reuses
  `buildTestCycleStructure([test])` + `createTestCyclePdf`.
- `exportTestSeating` — reads `test.seating_plan.assignments`, fetches the referenced ClassRooms (with
  `layout`) and Students, builds `rooms: [{ name, layout, bySeat: { [seatId]: { last, first } } }]`, calls
  the new `helpers/createSeatingPlanPdf.js`.
- Shared `uploadReportPdf(pdfBuffer, group_id, store_id, filename)` — PDFKit buffer → S3 (bucket
  **`logeion-test-cycle-reports`**, path `test-cycle-reports/{group_id}/{store_id}/`) → 60s signed URL.

### `helpers/createSeatingPlanPdf.js`
- **Landscape A4, one page per classroom** (only rooms that have assignments).
- **Zoom-to-content:** fits the bounding box of the desks+seats (rotation-safe via half-diagonal, padded)
  into the drawable area instead of the whole 1200×800 canvas, so everything renders larger.
  The half-diagonal **over-estimates** an ellipse's extent, which is harmless (extra padding), so
  `computeBounds` needed no change for round desks.
- Desks = rotated rects, or **`doc.ellipse(cx, cy, w/2, h/2)` when `shape === "circle"`**
  *(2026-07-23)* (+ label). Seats = circles; occupied seats are green with the student's name on
  **two lines — surname (bold) on top, first name below** — auto-shrunk per line via `widthOfString`
  (font 8–14 / 7–12) so long names **don't get cut off**. Free seats show their number.

---

## Quick file map

| Layer | Phase 1 (κάτοψη) | Phase 2 (assign) | Phase 3 (PDF) |
|---|---|---|---|
| FE model | `models/classroom-layout.model.ts`, `classroom.model.ts` | `models/test.model.ts` | — |
| FE component | `classrooms/classroom-diagram/` | `test-cycles/test-seating-modal/` + `test-cycle-details` | `test-cycle-details` (buttons) |
| FE service | `classroom.service.ts` `saveClassroomLayout` | `test-cycle.service.ts` `saveTestSeating` | `test-cycle.service.ts` `exportTest{Participants,SeatingPlan}` |
| BE model | `models/classroom.js` `layout` | `models/test.js` `seating_plan` | — |
| BE route/ctrl | `/classrooms/save-layout` | `/test-cycles/save-test-seating` | `/test-cycles/export-store-test-{participants,seating}` |
| BE helper | — | — | `helpers/createSeatingPlanPdf.js` |

### Touch points of the 2026-07-23 pass

| Change | Files |
|---|---|
| Round desks | FE `classroom-layout.model.ts` (`DeskShape`), `classroom-diagram.component.{ts,html}` (`addDesk(shape)`, `createDeskGroup`, `deskContainsPoint`), `test-seating-modal.component.ts` (`buildDesk`), `i18n el/en` (`classrooms.diagram.add_round_desk`) · BE `models/classroom.js`, `controllers/handlers/classroom.js` (`normalizeLayout`), `helpers/createSeatingPlanPdf.js` |
| Conflict warnings | FE `test-seating-modal.component.{ts,html,scss}` (`SeatingConflict`, `indexDayCourses`, `findDeskConflict`, `recomputeConflicts`, `focusConflict`) · `i18n el/en` (`test_cycles.seating.conflicts_title`/`.desk`, `messages.seating_same_course_desk`) |
| Button sizing | FE `classroom-diagram.component.{html,scss}`, `test-cycle-details.component.{html,scss}` — see the CSS note below |

### Gotcha: the breadcrumb-bar buttons *(2026-07-23)*

Both this feature's pages had a **full-width, oversized** action button (the κάτοψη `Save` measured
1054px; the test-cycle-details `Add test day`/`Export report` ~505px each). Two **global** rules cause
it, so **every page with a `.breadcrumb-bar` is affected**:

1. `global.scss` `.breadcrumb-actions ion-button { flex: 1 1 auto }` — `flex-grow: 1` inflates the
   buttons to fill the bar.
2. `.page-background { display: flow-root }` is declared **after** `.breadcrumb-bar { display: flex }`
   and wins on source order, so the actions drop onto a row of their own instead of sitting at the
   right of the breadcrumbs.

Fixed **scoped to these two pages** (`display: flex` on `.breadcrumb-bar.page-background`,
`flex: 0 0 auto` on the buttons, stretch restored under 768px). A global fix is still open — it would
move buttons on ~20 pages at once. `flow-root` is there to keep a child's bottom margin from escaping;
flex establishes a formatting context too, so overriding it on this element is safe.
