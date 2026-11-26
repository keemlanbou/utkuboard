# Utku Board — Handoff for Claude Code

You're picking up an existing, shipped project. Read this fully before changing anything.

## What it is
Utku Board is an infinite-canvas whiteboard delivered as **one self-contained HTML file** (React 18 + Tailwind via CDN, JSX compiled in-browser by Babel). No build step, no server, no backend. Projects save as local `.utkuboard` JSON files.

- **Owner:** Utku Demir (GitHub: `keemlanbou`)
- **Repo:** https://github.com/keemlanbou/utkuboard
- **Live demo:** https://keemlanbou.github.io/utkuboard/ (GitHub Pages, `main` branch, root)
- **Demo video:** https://youtu.be/Ak81a9eJCTM
- **Donations:** https://paypal.me/utkuboard (also in `.github/FUNDING.yml`)
- **License:** MIT
- **Current version:** v3.5.4

## Repo layout
```
index.html               ← the app (latest version; served by GitHub Pages)
utkuboard_v3.5.4.html    ← identical copy, versioned, attached to GitHub Releases
README.md
LICENSE
assets/icon.svg          ← app icon source (vector)
assets/icon.png          ← 512 px render, used by README
assets/screenshot.png    ← README screenshot
.github/FUNDING.yml
```
The app icon is **embedded** in `index.html` (`<link rel="icon">` = icon.svg as base64, `<link rel="apple-touch-icon">` = 180 px PNG as base64). The dashboard logo reuses the favicon via `APP_ICON`. If the icon changes: edit `assets/icon.svg`, re-render the PNGs (headless Chrome screenshot), and replace both base64 strings.
`index.html` and the versioned file must always be byte-identical for a release.

## Versioning & release workflow
- Semver. Single source of truth: `const APP_VERSION = 'X.Y.Z';` near the top of the script. Also update the HTML comment `<!-- Utku Board vX.Y.Z — ... -->` in `<head>`.
- Patch = bug fix, minor = new feature.
- The version shows on the dashboard header, editor status bar and help panel, and is written into every saved project as `appVersion`. Saved filenames are `<ProjectName>_vX.Y.Z.utkuboard`.
- Release: bump version → copy to `index.html` and `utkuboard_vX.Y.Z.html` (remove the old versioned file) → commit → create a GitHub Release tagged `vX.Y.Z` with the versioned HTML attached.

## Architecture essentials
- **State:** React `useState` for `elements`, `frames`, `connections`; undo/redo via `past`/`future` (50 deep). `takeSnapshot()` uses **reference equality** (state is immutable). Never reintroduce `JSON.stringify` comparison — it froze large (60 MB) projects.
- **Performance (critical):** pan AND element/frame drag use **requestAnimationFrame + direct DOM transforms**, syncing to React state only on pointer-up (`dragDeltaRef`, `dragRafId`, `transformRef`). During drag, `updateConnectionsDOM()` rewrites SVG path `d` attributes directly via `data-conn-id` / `data-conn-dot`. Do not move drag logic back into `setElements` per pointermove.
- **Size meter / autosave:** `approxSize` is debounced (1.5 s). Autosave runs every 30 s and skips projects over ~4.5 MB (localStorage quota).
- **Heights:** cards and other `height: 'auto'` elements use `getDOMHeight()` (reads `offsetHeight`) for alignment and ports; `getExactHeight()` is only an estimate/fallback.
- **Ports:** `top/bottom/left/right` on all elements. Card tasks also expose output ports named `out-<taskId>`, rendered by the top-level `CardTaskPorts` component on the **card frame** — never inside the draggable task row (HTML5 drag steals the pointer). Positions come from the `offsetTop` chain (zoom-safe). `generatePath` normalizes `out-*` → right. A `useEffect` re-homes task connections when a task moves to another card; deleting a task removes its connections.
- **Connections:** click = cycle style (curved/straight/electric), right-click = toggle flow animation, middle-click or Alt/Option+click = cycle color through `PASTEL_COLORS`. Duplicate connections (same port pair, either direction) are blocked.
- **Alt+drag a port:** `transferringConnections` moves *all* wires on that port; the real lines follow the cursor live (amber). Alt never creates a new connection.
- **Shift+drag on empty canvas:** connection knife (`cutLine`, red dashed polyline); on release it deletes every connection whose sampled path intersects (`segIntersect`).
- **Selection:** AutoCAD style — left→right = blue window (fully contained), right→left = green dashed crossing (touch).
- **Isolation (Alt+Q):** `isolatedIds`; everything else, including all connections, fades to 10%. Selection must stay intact.
- **Arrows:** endpoints auto-snap to the nearest element port within 60 px (drawing and editing).
- **Links:** right-click → Insert/Edit/Remove Link (`el.href`) on shapes, text and images; double-click opens via `openLinkSafely()` (http/https only, `noopener,noreferrer`). Linked elements stay clickable in PDF export.
- **Undo:** `takeSnapshot()`/`handleUndo()` read the live board from `liveRef` (never stale closures). Undo/redo skip snapshots identical to the current board (a plain click takes one). Ctrl+Z inside a text field is left to the browser. The keydown, copy and paste listeners are re-bound every render on purpose.
- **Clipboard:** board items are written to the system clipboard as `application/x-utkuboard` JSON (`CLIP_MIME`) in the native `copy`/`cut` event, so one paste does exactly one thing: board items (same board = +30 px cascade, other board/tab = centre of view, internal wires and groups remapped), else an external image, link or plain text (→ text box). The keydown handlers are only fallbacks for browsers that fire no clipboard event.
- **Reroute nodes:** element `type: 'reroute'` (36×24, dot in the centre). Double-click a wire → `insertReroute()` splits it. Ports `left`/`right` both resolve to the centre in `getPortCoords`; side handles branch new wires. Deleting a reroute heals wires via `pruneConnections()` (use `deleteSelection()` for every delete).
- **File handles:** `HandleStore` (IndexedDB `utkuboard_handles`) remembers the file each project was opened from / saved to (Chrome/Edge only). Ctrl+S writes straight to it (browser may ask once per session for write permission); Ctrl+Shift+S = Save As. Safari/Firefox have no such API and always download.
- **Storage:** dashboard list in `localStorage['utkuboard_projects']`; projects over ~4.5 MB are stored as stubs (`storedExternally: true`) and loaded from file via `loadProjectViaPicker()`. Manual save uses `showSaveFilePicker` (Chrome/Edge), falls back to download.

## Theme
Circuit Lab palette (warm, Claude-Code-like), applied through a `tailwind.config` override that remaps `indigo` → coral (`#e07a5f`) and `zinc` → warm browns. Backgrounds: dark `#1b1916`, panels `#262320`, light `#f4f1ea`. Font: Inter (Google Fonts). **The dot-grid canvas background must not change.** Connections saved in older files keep their stored indigo color — intentional, not a bug.

## Security posture (keep it)
- All CDN scripts are **version-pinned with SRI `integrity` hashes**: React/ReactDOM 18.3.1 production, @babel/standalone 7.29.7, html-to-image 1.11.13, jsPDF 2.5.2. If you change any version, recompute the sha384 from the npm tarball — a wrong hash silently breaks the app.
- The Tailwind Play CDN can't be hash-pinned (known, accepted).
- Intentional external requests the owner chose to keep: Open-Meteo weather, noembed + microlink link previews, YouTube thumbnails, Google favicons, Google Fonts. Don't remove them without asking.

## Hard browser limits — do NOT attempt
- Revealing absolute file paths (`C:\Users\...`) or opening Explorer/Finder. Already tried three ways and fully removed in v3.4.4. Only an Electron wrapper could do this.
- The iOS/iPad version was abandoned (Documents by Readdle's WebView hijacks touch gestures). Don't revive it unless asked.
- A separate "Sequence" node type was built and then removed by request; its multi-output idea now lives in card task ports.

## How the owner likes to work
- **Feasibility first:** if a request hits a platform or sandbox limit, say so *before* writing code and let him decide.
- **Never break existing features.** Change surgically; after edits, open the app in a browser and exercise the touched features plus drag, connections, save/load and export.
- Explain changes concisely in plain language. He's a designer and creative technologist (AutoCAD, 3ds Max, Unreal), not a full-time web developer.
- Bump the version on every delivered change.

## Open items
- Confirm the Sponsor button shows (Settings → General → Features → Sponsorships).
- Possible future work: an Electron desktop wrapper ("Show in Explorer", desktop icon, fully offline); bundling the CDN libraries into the file for true offline use.

## Kickoff prompt
> Read CLAUDE.md, then open index.html and confirm the app loads with no console errors. Summarize the current state in a few lines and wait for my next request.
