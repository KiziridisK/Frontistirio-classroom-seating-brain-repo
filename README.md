# Frontistirio — Classroom Seating Brain

> **Full-stack reference** for the classroom seating feature built 2026-06-28: the per-classroom
> floor-plan (κάτοψη) editor, the per-test-day seating assignment (drag students onto seats), and the
> printable PDF export. Spans the `ClassRoom.layout` model, the `Test.seating_plan` model, the Angular
> components (fabric.js canvas + native HTML5 drag-drop), the backend endpoints, and the seating PDF.
>
> Related brains: [test-cycles](../Frontistirio-test-cycles-brain-repo/README.md) (host of the seating
> UI), [pdf-generation](../Frontistirio-pdf-generation-brain-repo/README.md) (the seating PDF helper).

---

## Three phases (all done)

1. **κάτοψη editor** — design a classroom's layout (desks + seats), saved on the classroom.
2. **seating assignment** — per test day, drag participants onto seats in a modal, saved on the test.
3. **PDF export** — per test day, export the seating plan (one landscape page/room with names) or the
   participants list.

Tech: **fabric.js v6** for the canvas (Angular 18, Ionic). **No `@angular/cdk`** — drag-drop is native
HTML5. Greek `firstName`/`lastName` on the `Student` model are **camelCase**.

---

## PHASE 1 — κάτοψη editor

### Data model

**Frontend** `src/app/models/classroom-layout.model.ts`:
```typescript
interface SeatLayout { id: string; deskId: string | null; left: number; top: number; label?: string; }
interface DeskLayout { id: string; left: number; top: number; width: number; height: number; angle: number; label?: string; }
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
- Save: `TestCycleService.saveTestSeating(testId, assignments)` → toast → dismiss with assignments.

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
- Desks = rotated rects (+ label). Seats = circles; occupied seats are green with the student's name on
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
