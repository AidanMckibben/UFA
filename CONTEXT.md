# Project context: Ultimate Frisbee Field Dashboard

Read this file first in a new session to get oriented before making changes.

## What this is

A single-file, no-build-tools interactive web dashboard for planning ultimate
frisbee offensive/defensive formations. Everything — HTML, CSS, JS — lives in
one file: `index.html`. There is no bundler, no package.json, no dependencies.
Open it directly in a browser or serve the folder with any static file server.

## Repo

- GitHub: https://github.com/AidanMckibben/UFA
- Local working copy: `C:\Users\amckibben\OneDrive - Glotman Simpson Group of Companies\Desktop\Claude work\UFA`
  (This moved here from `G:\Claude work\UFA` mid-project when the `G:` network
  drive became inaccessible to the assistant's file tools. If `G:` is
  reachable again in a future session, check whether the two copies have
  diverged before treating either as canonical.)
- Branches: `master` (current/merged), plus feature branches kept around
  intentionally per user request rather than deleted after merging:
  `snap-grouping`, `positioning`, `selecting`. When starting new feature work,
  branch off `master`.

### Git quirk on this machine

If working from the `G:` drive copy again: `G:` is a network-mapped drive, and
plain `git` commands there fail with errors like `fatal: error reading
'//gs-storage/Users/.git'` because git's upward directory discovery walks into
a broken parent path. Fix: set `GIT_CEILING_DIRECTORIES` to the repo's parent
before running git, e.g.
`GIT_CEILING_DIRECTORIES="G:/Claude work" git -C "G:/Claude work/UFA" status`.
Also needed a one-time `git config --global --add safe.directory` entry for
the UNC-style path. The Desktop copy (on `C:`) does not have this problem —
plain git commands just work there.

## How to preview/test it

No dev server is required for a user to just open the file, but for the
assistant's Browser-pane tooling, serve the folder over HTTP (the pane can't
reliably load `file://` URLs) and navigate there, e.g.:
`python -m http.server 8936 --directory "<path to this folder>"`, then
navigate the Browser pane to `http://localhost:8936/index.html`.

Testing approach used throughout this project: simulate real user input via
`PointerEvent`/`KeyboardEvent` dispatch in `javascript_tool`, then assert on
`el.style.left/top` and computed geometry — not on `getBoundingClientRect()`
read immediately after a style change, which reflects the mid-CSS-transition
rendered position, not the logical target (a repeated gotcha in this repo;
read `.style.left`/`.style.top` directly instead). Also: this environment's
Browser pane tab often runs `document.hidden === true`
(backgrounded/unfocused), which throttles or fully pauses
`requestAnimationFrame` and clamps `setTimeout` — relevant if testing anything
frame-loop-based (see the arrow-key disc movement below). Override
`window.requestAnimationFrame` with a `setTimeout`-based shim for that kind of
test, or verify the per-tick math directly instead of relying on sustained
real-time animation.

## Domain model / on-field entities

- **Field**: an SVG (`viewBox`) inside `#fieldWrap`, drawn at a 40yd (wide) x
  70yd (tall) scale, oriented vertically, endzones at top and bottom.
  `preserveAspectRatio="none"` lets the viewBox be resized independently of
  the container's own aspect ratio.
- **Players** (`.player`, blue circles) — the 7 offensive players.
- **Defenders** (`.defender`, red rectangles, thin/tall) — the 7 defenders,
  1:1 paired with players by array order in most formation logic.
- **Disc** (`#disc`, white circle) — snaps to whichever offensive player holds
  it.

All three are absolutely-positioned divs sized as a percentage of a
**reference width/height** (`--player-ref-width`/`--player-ref-height`, CSS
custom properties set in JS), *not* a percentage of the field's current
rendered size — this is what keeps markers a constant size when Endzone mode
zooms the field in. Positions themselves (`left`/`top`) are plain percentages
of the field container's current size, and do scale/reposition correctly
across zoom modes since percentages are relative either way.

## Endzone (zoom) modes

`setFieldMode('full' | 'endzone')` swaps the SVG `viewBox` and the
`--field-ar`/`--field-ratio-wh` CSS custom properties that drive
`.field-wrap`'s width/aspect-ratio formula. Endzone mode shows the **top
third** of the field (not half — was half, changed to a third partway
through). A `currentEndzoneFormation` variable (`'vertical' | 'sideline' |
null`) tracks which *specific* zoomed layout is showing, which is what lets
the "Endzone Middle" button correctly toggle between "switch to this zoomed
layout" and "exit to full size" instead of just checking zoomed-vs-not (a real
bug that was fixed: switching from Sideline to Endzone Middle used to bounce
all the way out to full size).

Real-yardage math (angles, speeds, gaps) must account for which mode is
active, since the field is always 40yd wide but its visible height differs
(70yd full, 70/3yd in Endzone) — see `currentFieldHeightYd()`, and the
pattern used throughout (e.g. the Vertical formation's 45deg reset position,
the Endzone Sideline formation's 60deg stack, arrow-key disc movement) of
computing in real yards first, then converting to percentages, rather than
mixing percentages directly (equal x%/y% is NOT equal real distance on a
non-square field — a mistake made and caught more than once in this project).

## Physics / grouping system

- **Collision**: circle-circle (offense-offense), circle-rect (offense-
  defense, standard closest-point method), rect-rect (defense-defense,
  least-penetration-axis separation). Defender collision size is computed
  from a *fixed nominal* width/height, not `getBoundingClientRect()`, because
  defenders visually rotate to face the nearest offender and a rotated
  element's bounding box is its rotated AABB, not its true footprint.
- **Attachment/grouping** (`attachments` Map): a defender or the disc can be
  "attached" to a player, meaning it rigidly follows that player's every move
  (fixed offset from the moment of attachment). Attached items are excluded
  from collision resolution (they move as a rigid unit instead). Only one
  defender may be attached to a given player at a time. Grabbing an attached
  defender/disc directly always detaches it; it re-attaches automatically
  when it ends up touching a player (defenders) or is dropped near one (disc).
- **Defender facing**: each defender rotates (CSS `transform: rotate()`) to
  face whichever offensive player is nearest (or its attached target, if
  attached), skipping any player already covered by a *different* defender's
  attachment. Rotation is unwrapped against the previous angle to avoid a
  360deg spin glitch when the bearing crosses the +/-180deg seam.

## Selection system (multi-select)

- Shift+click toggles a player or defender's membership in `selectedItems`
  (shared Set for both types) without starting a drag.
- Right-click-drag draws a marquee. Standard CAD/Bluebeam convention:
  left-to-right = window select (fully-contained items only), right-to-left =
  crossing select (anything touched). This was initially implemented
  backwards from spec and later corrected — if a "which direction selects
  what" bug ever resurfaces, check `containMode` in the marquee `pointerup`
  handler.
- Left-click on empty field (not on a player/defender/disc) clears the
  selection. Escape also clears it.
- Dragging any selected item carries the rest of the selection along by the
  same per-frame delta (`applyGroupDragDelta`). Starting a group drag detaches
  any *other* selected defenders first, so the rigid-follow attachment system
  doesn't fight the manual delta-translation.

## Set-up formations & Defense presets

`FORMATIONS` object holds `horizontal`, `vertical`, `sideline` presets, each
with a `positions` array (index-matched to the 7 players) and a
`centerHandlerIndex` (who the disc snaps to). `applyFormation()` positions
players, assigns each defender to its matching player via
`snapDefenderToPlayer()` (top-right by default), and snaps the disc.

`snapDefenderToPlayer(d, target, angle)` computes the snap offset using the
defender's `halfW` (narrow/chest dimension), *not* the larger of
`halfW`/`halfH` — a bug fixed partway through this project that was leaving
too large a gap, since the defender rotates to face its target using its
narrow axis, not its long axis.

**Reset Defense** (Defense section) snaps every defender to its matching
player at one of 4 diagonal corners, chosen by two independent toggle groups:
Downfield (Fronting=bottom / Backing=top) and Mark (Force Backhand=left /
Force Flick=right — the *mark* defender, guarding whoever holds the disc,
always gets the *opposite* side and always the *top* corner). Strongside is a
**separate, independent** toggle (not part of Mark) that overrides the
downfield defenders' side to whichever side the disc is currently on relative
to their own player — it does not affect the mark defender's side. There's
also a universal override: any downfield defender at/behind the disc's level
(or up to 5% ahead) always takes the top corner regardless of Downfield.

## Arrow-key disc control

Holding arrow keys (up to 2 at once, 8-directional) moves the disc; releasing
all of them snaps it to the nearest offensive player. Speed is defined in
real yards/second and converted through the current field scale so movement
is a true undistorted compass in both zoom modes, with diagonal speed
normalized (divided by the direction vector's magnitude) so diagonals aren't
√2x faster than cardinals. A `window.blur` handler guards against a
stuck-drifting disc if the matching keyup is missed (e.g. alt-tab mid-hold).

## Known open items / things to watch

- The directional marquee convention was flipped once already at the user's
  request — if asked to "swap" it again, re-read the current `containMode`
  logic carefully rather than assuming which way is "standard."
- If the `G:` drive becomes reachable again, reconcile it against this
  Desktop copy before assuming either is authoritative.
- No automated test suite exists; verification throughout this project has
  been manual, simulated-event testing via the assistant's Browser tooling as
  described above.
