# Millplan — Specification & Project Status

Current versions: **App v11** · **Softwood: sw-3 (locked)** · **Hardwood: hw-6**

Millplan is a single self-contained HTML/JS file (works as an offline-capable Home Screen app on iPhone via GitHub Pages + a service worker) that turns log measurements into a sawing plan for either softwood dimensional lumber or hardwood boards.

---

## 1. Purpose & Deployment

- Single-file web app, no build step, no backend. Runs entirely client-side.
- Deployed via GitHub Pages at the user's own repo (`index.html` + `sw.js` at repo root, Pages enabled under Settings → Pages → Deploy from branch `main` / root).
- Added to iPhone Home Screen via Safari's "Add to Home Screen" — opens full-screen, works offline after the service worker's first successful cache.
- **Lesson learned:** iOS Quick Look (opening a file from the Files app) never runs JavaScript — this is a hard platform restriction, not fixable by choice of file source. The Dropbox app's own in-app previewer *does* run JS (real embedded web engine), which is why opening directly from within Dropbox worked even though exporting to Files did not.
- **Lesson learned:** Netlify Drop (anonymous, no-signup deploy) links are deleted within about an hour unless claimed with a free account — not suitable for anything beyond an immediate one-time test.
- **Lesson learned:** GitHub Pages serves from `https://<username>.github.io/<repo>/`, *not* the `github.com/<username>/<repo>` source page. Pages must be explicitly enabled in repo Settings before any URL will resolve.

## 2. Log Geometry Model (shared by both species)

- Inputs: large diameter, small diameter, length, bow (all inches).
- Taper: linear interpolation of diameter from large end to small end.
- Bow: parabolic sweep, zero deflection at both ends, maximum at midspan — matches how sweep is measured on a log deck.
- **Straight sawing only** — no sweep-following/dogleg cuts. The user pre-sorts logs by eye before milling; excessively bowed logs are set aside rather than handled algorithmically.
- **Fixed full-length boards only** — no variable-length recovery (v1 scope decision, still standing).
- The cross-section diagram is drawn at the log's true *tightest* cross-section along its length — this is not always the small end; when bow is significant, an interior point can be more restrictive. (Fixing this — from a naive "always small end" assumption — was a real bug found via user testing.)

## 3. Settings (shared)

- Per-board-type actual dimensions, editable. Defaults:
  - Softwood 1x: 0.75" thick. "2" nominal thickness: 1.6" (not the standard 1.5" — this is the user's own mill's target).
  - Widths: nominal − 0.375" (2x4/1x4 = 3.625", 2x6/1x6 = 5.625", 2x8/1x8 = 7.625", 2x10/1x10 = 9.625"). 4x4/6x6/8x8 match the corresponding width, squared.
  - Hardwood quarter thickness: 4/4=1", 5/4=1.25", 6/4=1.5", 8/4=2".
- Default kerf: **0.082"**.
- Settings auto-save to the browser's `localStorage` (works because this runs as a real page, not inside Claude's sandboxed artifact preview). A status line on the Settings screen shows whether save succeeded (fails silently in private/incognito browsing).
- All results-screen numbers round to whole numbers.

## 4. Softwood — LOCKED as of sw-3

**Do not modify softwood logic without an explicit request.** This version is approved and stable.

### Board catalog
1x4, 1x6, 1x8, 1x10, 2x4, 2x6, 2x8, 2x10, 4x4, 6x6, 8x8.

### Cant sizing (matches the user's real sawing process)
The user's actual method: figure out the biggest cant, decide which board widths it'll yield, size the cant to match the *combined* width of those sizes plus kerf between them, then square each side down (removing wing material) before resawing by thickness.

- Cant width = sum of the widths of all *distinct* selected board widths (grouped — e.g. 1x6 and 2x6 share one width group) + kerf between groups.
- Cant height = the maximum the log allows for that fixed width (searched over vertical offset for the best position).
- **"Max Yield" mode:** searches every combination (subset) of selected width-groups — not just single widths — and picks whichever combination yields the most total board footage. Mathematically guaranteed to never score below "Your Mix," since that exact combination is always one of the candidates tried.
- **"Your Mix" mode:** uses all selected groups combined; if the combination doesn't fit the log, drops the lowest-total-percentage group and retries.
- The cant **shrinks to its actually-used height** (not the theoretical maximum) before wing/pocket evaluation — otherwise leftover material gets trapped inside the cant's own boundary, invisible to recovery. This was a significant bug fix.
- Each width segment is resawn independently by thickness, using percentage-based priority mixing (proportional height budget by %, then fill remaining leftover down the priority list) for board types sharing that width.

### Beams
- Up to 2 per log, each defined by width × height (actual inches).
- Placement: **boxed heart** (centered on pith) or **free of heart center / FOHC** (offset so the pith falls outside the beam). No partial-offset option — it was explicitly rejected as strictly worse than either alternative.
- Wings computed around the beam the same way as the no-beam case.

### Wings / pockets (north/south/east/west convention)
- Each side is searched **independently** for its own best size and position — not forced to match the cant's own height/width. (Forcing this equality was the root cause of an earlier bug where side wings mathematically always came back zero.)
- Exactly one kerf gap enforced between the cant and each wing.
- Fill direction corrected so boards always build starting adjacent to the cant (not leaving a gap next to center) — required mirroring the fill order for north and west specifically, since their natural fill direction was backwards relative to south/east.
- Bonus/fallback fill: leftover pocket space tries non-selected catalog types, thinnest first, reported separately as "Additional recoverable."
- Rotation ("on edge"): used **only as a fallback** when the flat orientation doesn't fit — never as a general optimizer. (A general optimizer version was tried and rejected because it distorted the requested percentage mix on the main cant.)
- Diagonal corner recovery (small slivers beyond the four wing rectangles) was built once, found consistently too small for any catalog board (~2" max vs. 3.625" minimum board width), and was later removed entirely when wings were rebuilt as independent searches — the new wing model already claims most of that material.

### Metrics
- **Volumetric yield %** = total board feet ÷ whole log volume, using the *exact* tapered-cone (frustum) formula — not an average-diameter cylinder approximation (that earlier version was ~0.7% off).
- **Small-end area utilization %** = sum of board cross-sectional areas ÷ small-end circle area. Added because volumetric yield (denominator = whole tapered log) doesn't match what the eye perceives looking at a single cross-section — this metric isolates the part the user can actually influence at the saw from taper loss, which they can't.

### Visualization
- Individual board-level layout — every board drawn as its own labeled, color-coded rectangle, not aggregate blocks.
- Bonus/extra boards shown with a dashed border to distinguish from the requested mix.
- Dashed outline marks the maximum cant/pocket boundary.
- Large, bold labels (canvas renders at 800×620 internally and scales down ~2× on typical screens, so early small font sizes were nearly illegible).

## 5. Hardwood — actively evolving, currently hw-6

Rebuilt from scratch partway through development around the user's actual milling process (grain-driven decisions, not geometric optimization): open the log with a first cut, take 1–2 boards, rotate 90°, cut more (gets one straight edge from the prior cut), rotate again, cut for consistent width, resaw to desired thickness.

### Core model
- User specifies a single **desired board width** (not a sum of catalog sizes) — matches flooring width, jointer capacity, or whatever the actual goal is.
- Cant width = desired width (fixed). Height = max the log allows for that width (same search function reused from softwood).
- Resawn by thickness using selected quarter classes (4/4, 5/4, 6/4, 8/4) with percentage-based priority mixing — same underlying packing infrastructure as softwood.
- No "sawing method" toggle (through-and-through vs. grade sawing was removed — the rotate-and-square model unifies both).
- Cant shrinks to actual used height before evaluating the south pocket or the milling-preference passes (same fix pattern as softwood).

### South cant
- An independent search for the best (generally *narrower*) width below the shrunk main cant. Unlike softwood, hardwood board width is free/searchable rather than catalog-fixed, so this required its own search rather than reusing softwood's pocket-search function verbatim.
- Shown as its own "South cant" table since its width can differ from the main desired width.
- One kerf gap enforced between main cant and south cant.

### Milling-preference passes (max-width, not catalog-width)
Follows the user's described rotate sequence precisely:
- **Top:** exactly 2 passes (fixed count, not "as many as fit"), each independently trimmed to its own true maximum width at the reference thickness (= whichever thickness class is highest-priority in the current selection).
- **East:** pass count *derived from the cant's own height* (⌊cant height ÷ (thickness+kerf)⌋) after rotating 90° CCW — each pass its own true maximum height/width.
- **West:** mirrors east exactly (matches the described 180°-from-east rotation).
- **South is not included in this pass set** — it's covered separately by the South cant search above.
- Rendered as **outline-only** rectangles (no fill), distinct from the filled catalog-board colors — matches a reference image the user provided.
- Labeled with the matching catalog thickness name (e.g. "4/4"), not raw dimensions — reverse-matched from the pass's computed thickness back to the catalog. East/west labels are **rotated 90°** and shown in a larger font, since those passes are narrow-and-tall rather than wide-and-short.

### Known gaps (not yet built, not necessarily needed)
- No corner recovery for hardwood.
- No beam-equivalent concept (not requested, may not be applicable).

## 6. Shared Algorithm Building Blocks

These are reused across both species and are useful vocabulary for future requests:
- `cantMaxWidth(c, H, ...)` — max width for a rectangle of given height H centered at vertical offset c.
- `cantMaxHeightForWidth(c, W, ...)` — max height for a rectangle of given width W centered at vertical offset c.
- `findBestCantForWidth(W, ...)` — searches vertical offset to find the best (c, H) for a fixed width W.
- `priorityPack` / `fallbackFill` — proportional-then-fill-leftover packing by percentage priority, used for thickness/width mixing within a single zone.
- `searchVerticalPocket` / `searchSidePocket` — independent search for a wing/pocket's own best size and position (softwood).
- "Pockets" and "cants" always maintain exactly one kerf gap from whatever they're adjacent to; fill direction always builds starting from the edge nearest the reference boundary.

## 7. Version Tracking

- `APP_VERSION` — bumps on every deploy, shown in the header.
- `SOFTWOOD_VERSION` — frozen at `sw-3 (locked)`. Should not change without an explicit, unambiguous request to modify softwood specifically.
- `HARDWOOD_VERSION` — bumps whenever hardwood logic changes, shown on the Results screen subtitle.
- Rationale: softwood and hardwood are being developed on different timelines (softwood approved and stable, hardwood still active), so tracking them separately avoids the confusion that previously happened when a hardwood-intended change accidentally landed in softwood.

## 8. Open Items / Things to Revisit

- Corner/diagonal recovery for softwood pockets was removed as not worth the complexity — revisit only if a specific log shows meaningful recoverable material there.
- Variable board length (shorter-than-full-log boards for extra yield near the wide end) was explicitly deferred at v1 scope and has not been revisited.
- No native iPhone app has been built — current distribution is a Home Screen PWA. Native wrapping (Capacitor) or a full Swift rewrite would require a Mac + Xcode; discussed but not pursued.
