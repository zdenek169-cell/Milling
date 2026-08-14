# Millplan — Specification

This document states what Millplan is required to do. Future changes must keep these requirements true, or explicitly revise this document in the same change. Requirements are numbered for reference; "must" is binding, "should" is strong preference.

---

## 1. Product constraints

- **1.1** The app must be a single self-contained `index.html`: no build step, no backend, no JavaScript dependencies, and no runtime requests to external services (the Google Fonts stylesheet is the sole allowed exception, and the app must remain fully functional when it is unavailable).
- **1.2** The app must work offline after its first successful load, via the service worker (`sw.js`).
- **1.3** The app must be installable to the home screen on both iPhone (Safari, `apple-*` meta tags) and Android (Chrome, `manifest.json` + icon files). The manifest must be a real file — a runtime-generated blob manifest breaks Android installability.
- **1.4** Deployment target is GitHub Pages from the repository root. All asset paths must be relative.
- **1.5** The UI must remain usable one-handed on a phone (max content width ~460px, touch targets on all controls, `touch-action` handled on interactive canvases).
- **1.6** Screens present controls and data, not tuition. Explanatory prose must not sit between controls or between report sections; it belongs in a collapsed panel at the end of the screen. Text that stays visible must be either live data (a computed limit, a readout, a legend) or a warning that requires action. This applies to every screen.

## 2. Log model

- **2.1** A log is described by: large-end diameter, small-end diameter, length, bow, bow peak position, and species. All dimensions are inches.
- **2.2** Taper is linear from the large end to the small end.
- **2.3** Bow displaces the pith perpendicular to the log axis: zero at both ends, maximum (= the bow measurement) at the peak position, with a smooth curve (no slope discontinuity at the apex). The peak position is expressed as a percentage of length from the large end, default 50%, effective range clamped to 10–90%.
- **2.4** Bow direction is the log-frame north face. Rolling the log (§7.5) rotates the bow with it.
- **2.5** Sawing is straight only: every board and cant is a straight prism through the full length. No sweep-following. (Deliberate scope decision — logs too bowed to saw straight are culled by eye before milling.)
- **2.6** Boards are full log length by default. Shorter boards are produced only when partial-length recovery is switched on (§3.8).
- **2.7** Planner inputs must be clamped so that no user input (including cleared/zero/negative fields) can hang or crash the app: diameters ≥ 0.5", length ≥ 1", kerf ≥ 0, and board dimensions at or below 1/16" are treated as not a real board.

## 3. Shared planning rules

- **3.1** Exactly one kerf of waste separates every pair of adjacent sawn faces: between boards in a stack, between cant segments, between a cant and anything grown outside it, and between two beams.
- **3.1a** **Sawability.** The mill is a bandsaw: a single horizontal blade that cuts the full width of whatever is on the bed, and the log is re-presented by rolling it in quarter turns. A plan is only valid if some roll order frees every piece it claims. Two consequences are binding:
  - No two pieces may occupy the same wood. Any plan containing overlapping pieces is wrong — it overstates board feet, yield and value, and cannot be cut.
  - The north and south faces are opened first, across the log's full width at those heights; the flats they leave then bound everything cut afterwards. East and west courses must therefore fit **within the cant's height band**, and must never be sized against the round log's full height at their position. (Hardwood's side passes satisfy this by construction, since a wider rectangle is always shorter; softwood's do not, because the cant height is user-set with a beam and shrunk to the used height without one.)
- **3.2** The binding cross-section — the position along the length where the log is most restrictive for the planned cant — must be found by search, not assumed to be the small end.
- **3.3** Percentage mixing: sizes sharing a packing zone are allocated by their target percentages using deficit-style scheduling, so rounding leftovers spread across the mix rather than accumulating in the highest-percentage size. Selected sizes with no percentage act as gap-fillers after the targeted sizes. If nothing has a percentage, all selected sizes are weighted equally.
- **3.4** After primary packing, leftover space is filled with non-selected catalog sizes (thinnest first) and reported separately as "extra" — never blended into the requested mix.
- **3.5** Rotating a board (on edge) is allowed only as a fallback when its normal orientation does not fit. It must never be used as a general optimizer (it distorts the requested mix).
- **3.6** Cants shrink to their actually-used height before anything outside them is evaluated, so leftover space inside a theoretical cant boundary is available for recovery rather than trapped.
- **3.7** When nothing the user selected fits the log, the app must still show something actionable: the largest rectangle the log geometrically allows, drawn on the layout and stated in the warning.
- **3.8** **Partial-length recovery** is an option, off by default, with a user-set minimum run length as a percentage of log length (default 50%, range 10–95%). When on:
  - It runs only after full-length packing is complete and must not alter the full-length plan in any way. Turning the option on can only add material.
  - A board qualifies if it clears a *contiguous* run of the log at least the minimum long, measured between the first and last sampled station that clear it, so a run is never claimed longer than the sampling supports. The board is sized to the tightest section within its own run.
  - Only sizes the user selected are used; the fallback/extra catalog is not extended into partial lengths.
  - Each partial board carries its own length and start position, and must be reported at that length rather than the log's.
  - A log with no taper and no bow yields nothing here, by construction.

## 4. Softwood planner

- **4.1** Catalog: 1x4–1x10, 2x4–2x10, 4x4, 6x6, 8x8, with per-size editable actual dimensions and $/bf value.
- **4.2** Cant width = the sum of the distinct actual widths among selected sizes (sizes sharing a width share a segment) plus kerf between segments. Cant height = the maximum the log allows for that width, found by searching vertical position.
- **4.3** Each width segment is resawn independently by thickness using the §3.3 mixing rules.
- **4.4** There is one plan, built from the sizes and percentages the user set. The cant uses all selected width groups; if the combination is too wide for the log, the group with the smallest share of the requested mix is dropped and it retries.
- **4.5** There is no alternative "maximise value" mode. One existed, choosing between subsets of width groups by total dollar value, but it scored only the central cant and ignored the wings — which are usually most of the recovery — so in the majority of tested configurations a subset it had already rejected was worth more than the one it picked. Prices remain user data and still produce the value figures in §6.3; they must not drive plan selection unless a future search scores the whole plan, wings included.
- **4.6** Wings grow outward from the cant in all four directions, course by course: each course takes the widest selected size that still fits at that position, stepping down sizes as the log narrows, until nothing selected fits; then fallback sizes continue (§3.4). Wing courses must build starting adjacent to the cant.
- **4.7** Beams: up to two, each width × height × $/bf, placed boxed heart (centered on pith) or free of heart center (offset so the pith falls outside). With two beams, they sit side by side separated by one kerf and are placed as one combined envelope at the taller height. Wings grow around the beam envelope the same way as around a cant.
- **4.8** Beam sizing must not require a round-trip to the Results page: the Boards screen must show, live as the user types, whether the current beam set fits the current log, the limiting dimension when it doesn't, and must offer a one-tap "fit to log" that shrinks the beam to the largest ¼"-rounded size that fits. These checks must use the same geometry as the planner, so the two can never disagree.

## 5. Hardwood planner

- **5.1** The user specifies one desired board width (not catalog widths) and a mix of quarter thicknesses (4/4, 5/4, 6/4, 8/4) with editable actual thickness and $/bf.
- **5.2** Main cant: width = desired width; height = maximum the log allows, resawn by thickness per §3.3, then shrunk per §3.6.
- **5.3** South cant: an independent search below the main cant for the best width (usually narrower), packed with the same thickness mix, reported as its own table since its width differs.
- **5.4** Milling-preference passes reflect the physical open–rotate–square sequence: exactly two top passes, then east and west passes with count derived from the cant height; each pass is trimmed to its true geometric maximum, labeled with the matching catalog thickness, and drawn outline-only to distinguish it from stacked catalog boards.
- **5.5** There is no sawing-method toggle; the rotate-and-square model is the single hardwood model.

## 6. Metrics and cut list

- **6.1** Volumetric yield % = total board feet ÷ exact frustum log volume (not a cylinder approximation).
- **6.2** Small-end area used % = sum of board cross-section areas ÷ small-end circle area. This exists because volumetric yield mixes in taper loss the sawyer cannot influence. Partial-length boards are excluded from this figure — they do not reach the small end — while still counting toward board feet, yield and value.
- **6.3** Estimated value = Σ board feet × the size's $/bf, shown as a headline stat and per-line in the cut list with per-table totals. Prices are user-owned data in Settings; shipped values are seeds, not truth. Value is reported, not optimised against (§4.5).
- **6.4** Headline board feet and value round to whole numbers; dimensions display to the precision needed to cut them.
- **6.5** The cut list must separate: requested mix, extra recoverable sizes, partial-length boards (grouped by size and length, each with its own length), beams, south cant (hardwood), milling passes (hardwood), and cant/wing footprints (softwood).

## 7. Results visualization

- **7.1** The board layout view is a cross-section at the binding section (§3.2), stated as such in its caption, viewed from the small end, with every board drawn and labeled individually and extras dash-bordered.
- **7.2** Three reference sections are always shown, each with a fixed color and dash style used identically in every view where it appears: tightest (solid blue), small end (dashed green), large end (dotted pink), with a key showing their diameters.
- **7.3** A compass (N/E/S/W) labels the log's faces. N and E use dedicated stripe colors (yellow, cyan) reserved app-wide for orientation — nothing else may use those hues.
- **7.4** The slice slider selects a position along the length; both the layout view and the 3D view must show the log's true cross-section there (a high-contrast ring, and a cut plane in 3D) simultaneously, with a readout of distance from the large end and diameter. The slider sits between the two views.
- **7.5** Roll: the user can turn the log in quarter turns, matching how a log is rolled on the bed. Rolling rotates the log and everything cut into it — boards, cant, stripes, bow, compass — consistently in both views, with a readout of which face is up. Screen-up always means up on the mill. Rolling never re-plans; it is a view of the same plan.
- **7.6** The 3D view shows the log as a solid translucent body (taper, bow) resting on a bed grid, with the cant prism visible inside it, in perspective projection. It supports orbit (drag), zoom (pinch/scroll, clamped, double-tap reset), and persists its viewpoint, slice, and roll across re-renders within a session.
- **7.7** Canvas drawings must remain legible at phone scale: canvases render at 2× logical size and labels are sized for ~50% downscale.
- **7.8** The Results page reads as a report: plan data (drawings, figures, cut list) runs uninterrupted, per §1.6. Each view carries only a compact legend identifying its colors; instructions and method notes belong in the single collapsed panel at the end of the page.
- **7.9** Partial-length boards must be visually distinguishable from full-length ones and must not be drawn as if present at the binding section: they appear outlined-only in the board layout, labelled with their length, and as prisms spanning their actual run in the 3D view.

## 8. Persistence

- **8.1** Settings (dimensions, kerf, prices) and the last log's measurements persist in the browser via `localStorage`, loaded on startup. The Settings screen must state whether saving works (it fails silently in private browsing).
- **8.2** Kerf overrides are per-log: clearing the override field must restore the default kerf (empty ≠ zero).
- **8.3** No data leaves the device. There are no accounts and no network storage.
- **8.4** The app offers dark and light themes via a header toggle; the choice persists on the device. Every canvas element must stay legible in both themes: labels use a background-colored halo, and the section-ring and orientation-stripe inks have per-theme variants (§7.2, §7.3 apply per theme).

## 9. Versioning

- **9.1** Version history lives in git; the app displays no version numbers.
- **9.2** A commit that can change a planner's output must say so in its message and name the affected engine (softwood, hardwood, or both), so any plan can be traced to the math that produced it through the deploy history.
- **9.3** Softwood and hardwood evolve independently; a change intended for one must not alter the other's output. A change to shared geometry (taper, bow, kerf handling, packing rules) affects both and must be flagged as such. When in doubt, verify with before/after runs on a fixed set of test logs.

## 10. Offline / update behavior

- **10.1** The service worker precaches the page, manifest, and icons; the cached copy is served first with a background refresh.
- **10.2** Known accepted limitation: after a deploy, the first load shows the previous version; the new version appears on the next load. If this becomes unacceptable, the fix is network-first navigation with cache fallback — not removing offline support.
- **10.3** The service worker cache name must be bumped whenever the set of precached files changes.