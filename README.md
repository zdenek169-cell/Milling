# Millplan

Type in a log's measurements, get a cutting plan.

Millplan is a planning tool for small sawmills. Before opening a log, it helps to know what should come out of it: which board sizes, how many of each, and where the cuts go. Millplan works that out from the log's measurements and shows the result as a drawing, a cut list, and an estimated value.

It runs in any web browser, works without an internet connection once loaded, and can be added to a phone's home screen like a regular app — handy standing at the mill.

## Using it

The app has four tabs, used in order:

1. **Settings** – board dimensions, saw kerf, and rough prices per board foot. The defaults are one particular mill's numbers, so adjust them to match yours. Changes are remembered on your device.
2. **Log** – measure the log: diameter at each end, length, how much it bows, and where along its length the bow is worst. Then choose softwood or hardwood.
3. **Boards** – say what you want out of the log. For softwood: the sizes you want and the mix between them, plus up to two beams. Beam sizes are checked against the log as you type, and a "fit to log" button shrinks a beam to the biggest size that will work. For hardwood: the board width you're aiming for and the thicknesses to cut.
4. **Results** – the plan. A cross-section drawing with every board labeled, a cut list with counts and values, and a 3D picture of the log sitting on the mill.

Tips for reading the results:

- The cross-section drawing shows the log at its narrowest usable point. On a bowed log that's usually somewhere in the middle, not at the small end.
- The slider moves a cut plane along the log's length. The white circle in both pictures shows exactly how big the log is at that spot.
- The Roll buttons turn the log a quarter turn at a time, the same way you'd roll it on the mill bed. The yellow stripe marks the north face and the cyan stripe the east face, and the whole plan turns with the log. This makes it much easier to keep track of which face is which between cuts.
- The value figures come from the prices you enter in Settings. Put your own numbers in first, or treat them as a rough guide only.

## Putting it on your phone

Open this address in the phone's browser:

**https://zdenek169-cell.github.io/Sawmill-app/**

- **iPhone**: open it in Safari, tap the Share button, then "Add to Home Screen".
- **Android**: open it in Chrome and choose "Install" when offered, or use the browser menu and pick "Install app".

Either way it opens fullscreen with its own icon, and keeps working with no signal once it has loaded once.

Note that the address ends in a slash and is a `github.io` page, not the `github.com` page where the code lives — the code page won't run the app. 

## What's in this repository

- `index.html` – the entire app
- `sw.js` – makes it work offline
- `manifest.json` and the two icon files – let phones install it like an app
- `millplan-spec.md` – a written description of how the planning works and the reasoning behind it

## A note on updates

After an update is published, the app serves the copy it already has the first time you open it and picks up the new one on the next load. This is a side effect of how the offline support works — if something looks out of date, reload once.
