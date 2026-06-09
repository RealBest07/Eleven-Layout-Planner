# 7-Eleven Layout Planner — work.md
> อ่านไฟล์นี้ก่อนทำงานทุกครั้ง แทนการอ่าน source ทั้งหมด

## Project
Single-file app: `index.html` (~1300 lines)
Stack: vanilla HTML/CSS/JS + Canvas 2D
No build step, no dependencies

**Two modes** (toggle at top of panel, `setMode('rect'|'poly')`):
- **rect** — original rectangular plot + auto parking zones (`generate()` → `draw()`)
- **poly** — freeform polygon land, click-to-draw, multi-parcel, manual building placement (`polyRender()`)

---

## Layout Structure (HTML)

```
body
  .panel (left, 290px)
    .panel-header
    .panel-body (scrollable)
      section: ขนาดที่ดิน      → #plotW, #plotD
      section: แบบร้าน 7-11    → #storeSize (dropdown), #rotateStore, #customDims (#storeW, #storeDInput)
      section: ระยะแนวร่น      → #sbBack, #sbLeft, #sbRight
      section: ที่จอดรถ         → #carW, #carD, #rearClear, #storeClear
      buttons: Generate Layout, Save Image
    .legend
  .canvas-wrap
    canvas#canvas
  .stats-bar
    #sv-plot, #sv-store, #sv-park, #sv-cars, #sv-ratio, #sv-status
```

---

## Key Functions

| Function | Line | Description |
|---|---|---|
| `updateStoreDims()` | 354 | show/hide custom dims input |
| `getStoreDims()` | 359 | returns [storeW, storeD] with rotate applied |
| `calcParkingLayout(availW, availD, carW, carD, aisleW)` | 375 | returns `{rows, totalCars, usedDepth}` |
| `generate()` | 402 | reads inputs → computes layout → calls `draw()` |
| `draw(L)` | 492 | renders canvas from layout object `L` |
| `drawSetbackHatch(ctx,cx,cy,cs,L)` | 686 | dashed lines for sbBack/sbLeft/sbRight |
| `drawParkingSlots(ctx,cx,cy,cs,L)` | 732 | car rectangles + aisle labels |
| `drawEntrance(ctx,cx,cy,cs,L)` | 804 | arrow + "ทางเข้า" label |
| `drawRoad(ctx,roadX,roadY,roadW)` | 837 | road strip at bottom |
| `drawNorthArrow(ctx,x,y,size)` | 860 | N arrow top-right |
| `drawScaleBar(ctx,x,y,SC)` | 881 | scale reference bar |
| `drawDimensions(ctx,cx,cy,cs,L)` | 904 | dimension arrows/labels |
| `arrowHead(ctx,x,y,dir)` | 988 | helper for dimension arrows |
| `saveImage()` | 1016 | download canvas as PNG |

---

## Layout Object `L` (passed to draw)

```js
{
  plotW, plotD,             // ที่ดิน
  storeW, storeD,           // ร้าน
  storeX,                   // = sbLeft
  storeY_road,              // = parkDepth (ระยะจากถนนถึงหน้าร้าน)
  sbBack, sbLeft, sbRight,  // ระยะร่น (ไม่มี sbFront)
  parkX,                    // = sbLeft
  parkWidth,                // = plotW - sbLeft - sbRight
  parkDepth,                // = plotD - sbBack - storeD
  parkSlotDepth,            // = max(0, parkDepth - rearClear - storeClear - wheelStop)
  rearClear,                // ระยะถอย (exact = input)
  storeClear,               // ทางเดินหน้าร้าน
  wheelStop,                // หมอนจอด/curb ระหว่างช่องจอดกับทางเดิน (default 1)
  parking,                  // {rows, totalCars, usedDepth}
  freeSpace,                // พื้นที่เหลือ (ล่างสุด ติดถนน)
  z_freeTop,                // = freeSpace
  z_slotBot,                // = freeSpace + rearClear
  z_slotTop,                // = freeSpace + rearClear + parking.usedDepth
  z_wsTop,                  // = z_slotTop + wheelStop (bottom of walkway)
  carW, carD, aisleW,       // aisleW = rearClear (follows ระยะถอย)
  valid,                    // errors.length === 0
}
```

---

## Canvas Coordinate System

```
y=plotD (back/top of plot)  ← canvas TOP
y=0     (road/front)        ← canvas BOTTOM

cx(x_real) = OX + x * SC
cy(y_real) = OY + (plotD - y_real) * SC    // flip Y
cs(meters) = meters * SC
```

Plot is clipped (`ctx.save/clip/restore`) before zone drawing — `ctx.restore()` happens before Plot boundary stroke.

---

## Zone Drawing Order (bottom → top)

| Zone | Color | y range (real) | width |
|---|---|---|---|
| Free Space | gray | 0 → freeSpace (z_freeTop) | plotW |
| ระยะถอย | amber | z_freeTop → z_slotBot (=freeSpace+rearClear) | plotW |
| ช่องจอด | blue | z_slotBot → z_slotTop | parkX..parkX+parkWidth |
| หมอนจอด | dark curb | z_slotTop → z_wsTop (=+wheelStop) | parkWidth |
| ทางเดิน | green | z_wsTop → z_wsTop+storeClear | parkWidth |
| ร้าน | red | parkDepth → parkDepth+storeD | storeW |

**Poly mode** zone order เหมือนกัน. Boundaries: `z_road → z_freeEnd → z_rcEnd → z_slotEnd → z_wsEnd → z_scEnd`. **Slot block (free/rearClear/slots/wheelStop) ลากได้ด้วย offX/offY; walkway (ทางเดิน/ฟุตบาท) ยึดติดหน้าร้าน ไม่เลื่อนตาม drag** (anchored to live store face).

---

## Store Templates (shapes, replaces old #storeSize)

Store footprint = **polygon template**. Picker = custom dropdown `#rectTplPicker`/`#rectTplDropdown` (thumbnails rendered via `tplThumbDataURL`).
- `STD_TEMPLATES`: std-s/m/l (rect) + std-lshape; `loadCustomTpls()`/`saveCustomTpls()` ↔ `localStorage["storeTemplates_v1"]`; `getAllTemplates()`, `getTpl(id)`, global `selectedTplId`.
- Template `{id, name, w, d, std?, poly:[{x,y}…]}` — poly in local coords [0..w]×[0..d], y-up, origin bottom-left.
- `getStoreDims()` / `getStorePoly()` read selected tpl (+ rotate 90° → `(x,y)→(y, w-x)`).
- **Editor modal** `#tplModal` (`te` state): canvas `#teCanvas`, tools วาด/ปิดรูป/ลบจุด/undo/rect/clear, set W×D + snap (default 0.25, 0=free), drag vertices; `saveTpl()` → localStorage. Editing a std tpl saves a copy. While drawing: crosshair guide (`te.mouse`) + rubber-band line from last vertex + tooltip `(x,y) • dist m • angle°`.
- **Normalize on save**: `saveTpl` shifts footprint flush to (0,0) and sets w,d = drawn extent (no reserved margin) so bbox == shape → setback/walkway/parking hug the store (no gap). `migrateTpls()` on load normalizes pre-existing custom templates.
- Rendering: rect `draw()` builds `storePath()` from `L.storePoly`; poly building gets `b.poly`, `drawBld` uses `bldPolyWorld(b)` (rotation applied). **`bldRect`/`bldFits`/parking still use bbox (w×d).**
- Rotate 90° swaps [w, d]. Store overflow → dim fill + "⚠ ร้านเกินขนาดที่ดิน".
- **Plan image**: template optional `img` (downscaled ≤700px JPEG dataURL). Editor `#teImgFile` → `teLoadImg` → `te.img`. When set, store renders the image **clipped to footprint** (rect `draw` + poly `drawBld`) instead of red fill, and the big centered label is hidden. `getImg(dataURL,onload)` async cache (redraws on load). Thumbnail = the image. `getStoreImg()` → `L.storeImg`; building gets `b.img`.

---

## Fixed Values (not exposed as inputs)
- `aisleW = 6m` — aisle between parking rows (hardcoded in `generate()`)
- ถนน always at bottom (south)
- ร้านหันหน้าสู่ถนนเสมอ — no storeFront direction input

---

## Login (hard-coded, client-side)
- `#loginScreen` overlay (z-index 2000) shown on load. `AUTH_USERS = { admin: '7eleven@2024' }` (id:pass).
- `doLogin()` checks creds → `sessionStorage['sevene_auth']='1'` + hide overlay. `logout()` clears + reload. On load, if flag set → skip.
- ⚠ NOT real security (creds visible in source). Default cred: **admin / 7eleven@2024**.

## Removed Features
- `sbFront` — removed (ไม่ใช้)
- `storeFront` direction dropdown — removed (always faces road)

---

## Polygon Mode (appMode === 'poly')

HTML: `#polyPanel` (hidden when rect). `#rectPanel` wraps all rect sections.

### State: global `poly` object
```js
poly = {
  plots: [ [{x,y},...], ... ],  // parcels, world meters, y UP (north up)
  draft: [{x,y},...],           // polygon being click-drawn
  drawing: bool,
  buildings: [ {id,x,y,w,d,rot,label,type,color}, ... ],
  selected: id, view: {...}, drag: {...}, nextId,
}
```
- Building (x,y) = bottom-left corner (world). `rot` 0|1 (1 = swap w/d, i.e. 90°).
- `type`: 'store'(red) | 'neighbor'(gray) | 'shophouse'(tan). Defaults in `BLD_DEFAULT`.

### Coordinate system (DIFFERENT from rect mode!)
- World y is UP (north up), opposite of rect mode. Auto-fit bounding box.
- `computeView(cw,ch)` → `{scale,minX,minY,pad,ecx,ecy,ch}` stored in `poly.view`.
- `vcx(v,x)`, `vcy(v,y)` map world→screen (vcy flips Y). `worldFromScreen(v,sx,sy)` inverts (for clicks).
- While `drawing` or empty: view LOCKED to fixed box (0..viewMeters) so click mapping stays stable. When idle: auto-fits to content.

### Key functions
| Function | Role |
|---|---|
| `setMode(m)` | toggle rect/poly, show/hide panels, restore stat labels |
| `polyArea(pts)` | shoelace area |
| `pointInPoly(px,py,pts)` / `pointInAnyPlot` | ray-cast hit test |
| `bldRect(b)` / `bldCorners(b)` | rect with rotation applied |
| `bldFits(b)` | all 4 corners + center inside union of plots |
| `toggleDrawPlot/closeDraftPlot/undoVertex/deletePlot` | plot editing |
| `addBuilding(type)/addShophouse()/deleteBuilding/updateBld` | building CRUD |
| `renderPlotList/renderBldList` | left-panel lists |
| `polyRender()` | master canvas render (poly mode) |
| `drawPlotPoly/drawDraft/drawBld` | canvas draw helpers |
| mousedown/move/up (IIFE `attachPolyEvents`) | click-add-vertex, drag-building, select |

### Behaviors
- Click-draw: click adds vertex; click near first vertex (≤12px) OR "ปิดรูป" closes. Trailing dup-of-first vertex deduped (<0.6m).
- Edge length labels drawn at each edge midpoint (skips zero-length edges).
- Overflow building → dim fill (`hexA`) + "⚠ ล้ำขอบที่ดิน".
- Snap to `#snapGrid` (m). `#viewMeters` sets fixed view extent while drawing.
- Stats relabeled in poly: sv-park→จำนวนอาคาร, sv-cars→จำนวนแปลง.

### Crosshair guide (drawing)
- `poly.mouse={sx,sy}` tracked in mousemove while drawing. `drawCrosshair()` draws full-canvas dashed cross + tooltip `(x,y) • dist-from-last-vertex m`.
- `updateCanvasCursor()` sets `canvas.style.cursor='crosshair'` while drawing, `'move'` over a building, else `'default'`. Call it after any drawing-state change.

### Plot geometry editor (`#plotEditor`)
- Click a plot row → `selectPlot(i)` toggles `poly.selPlot`; selected plot drawn blue.
- `renderPlotEditor()` lists, per edge: ระยะ length(m) + มุม bearing(°, 0°=+x/east, CCW). Per vertex: x,y + delete.
- `setEdgeLength(pi,i,len)`: moves end vertex along current edge direction.
- `setEdgeBearing(pi,i,deg)`: rotates edge around start vertex (keeps length).
- `setVertex(pi,i,field,val)` / `deleteVertex(pi,i)` (min 3 verts).
- `edgeBearing(a,b)` = atan2 normalized 0–360.

---

## Changelog

| Date | Change |
|---|---|
| 2026-05-29 | initial build |
| 2026-05-29 | remove sbFront; parkWidth respects sbLeft/sbRight |
| 2026-05-29 | freeSpace zone at bottom; ระยะถอย = exact input |
| 2026-05-29 | plot clip (ctx.save/clip/restore) to prevent store overflow |
| 2026-05-29 | store overflow = dim fill + ⚠ label |
| 2026-05-29 | remove storeFront dropdown (dead input) |
| 2026-05-29 | add saveImage() button → PNG download |
| 2026-05-30 | add Polygon mode: click-draw plots, multi-parcel, manual building placement (store/neighbor/shophouse) with drag + edge-overflow check; mode toggle keeps rect mode intact |
| 2026-05-30 | poly: crosshair guide while drawing (full-canvas cross + coord/distance tooltip); plot geometry editor (edit edge length + bearing° + vertex x,y, delete vertex); selectable/highlighted plot |
| 2026-06-05 | warning label in amber zone when no parking fits (rect+poly mode); status "⚠ ที่จอดไม่พอ" when totalCars=0 |
| 2026-06-05 | zone reorder: rearClear now at road, freeSpace moved near store face (rect+poly mode) |
| 2026-06-05 | poly parking: roadY derived from actual plot boundary (plotMinY/MaxY/X) not hardcoded 0 — fixes zero parking when plot coords are negative |
| 2026-06-05 | freeSpace moved back to road (bottom); rearClear above it (z_freeTop/z_slotBot) |
| 2026-06-05 | AISLE relabeled "ระยะถอย" + amber hatch; aisleW = rearClear (follows input) |
| 2026-06-05 | poly panel reorder: ที่ดิน→อาคาร→ระยะร่น→ที่จอด→ฟุตบาท→ล้าง |
| 2026-06-05 | Phase 1: wheel stop (หมอนจอด, default 1m) zone between slots & walkway (rect+poly); poly slot block draggable independent of walkway (walkway anchored to store face) |
| 2026-06-05 | poly parking: availW = store frontage (r.w/r.h, extent parallel to road) — block must NOT exceed store extent; parkW=carsPerRow×carW, parkX0=store edge; offX drag clamped to [0, availW−usedW] slack |
| 2026-06-05 | poly: bg zones (freeSpace/ระยะถอย/หมอนจอด/aisle) = full store frontage (blkX/fullW), fixed; ONLY car-slot block (slotX/slotW) drags horizontally via offX within slack (ชิดซ้าย/ขวา); hit-test grabs full frontage |
| 2026-06-05 | Phase 2: store template system — polygon footprint, std + custom templates, editor modal (drawing tools), thumbnail dropdown picker, localStorage persist; store renders as polygon (rect+poly), parking/fit use bbox |
| 2026-06-05 | template picker + สร้างแบบใหม่/แก้ added to poly panel too; renderTplPicker/toggleTplDropdown/buildTplDropdown generalized for rect+poly (ids rectTpl*/polyTpl*) |
| 2026-06-05 | tpl dropdown scroll fix: call expandAll() after toggle so section-body (overflow:hidden) isn't clipped |
| 2026-06-05 | store plan image: upload PNG/JPG in template editor (downscaled dataURL in localStorage); store renders image clipped to footprint instead of red fill (rect+poly); img cache getImg() |
| 2026-06-05 | render at devicePixelRatio (draw + polyRender: canvas.width=cssW*dpr, ctx.setTransform(dpr…), imageSmoothing high) → crisp/smooth clip edges; canvasToScreen divides by dpr to keep poly clicks in logical px. Image crops to drawn (non-rect) footprint polygon. |
| 2026-06-05 | tpl editor: crosshair guide + rubber-band + dist/angle° tooltip while drawing; snap default 0.25 (finer), 0=free |
| 2026-06-05 | normalize footprint on save (flush to 0,0; w,d=drawn extent) + migrateTpls on load → setback/walkway/parking hug store, no reserved gap around shape |
| 2026-06-05 | hard-coded login screen (sessionStorage gate) + logout btn; cred admin/7eleven@2024 (client-side, not real security) |
