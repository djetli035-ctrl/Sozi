# Sozi Baseline Architecture (`24.11.17` / `2296052`)

> **Scope:** Read-only architectural baseline of `https://github.com/sozi-projects/Sozi.git` at commit `2296052c2bd9a6f7f5361663fbe261931f9e4b2d` (branch `main` / `phase-0/architecture-and-plan`, license `MPL-2.0`). No upstream source files were modified in Phase 0.

---

## 1. High-Level System Overview

Sozi is an SVG "zooming" presentation editor and standalone HTML presentation runtime. It builds two targets from a shared ES module source tree (`src/js/`, **44** `.js` files total) using Gulp 5, Babel 7, Browserify 17, UglifyJS, and Nunjucks:

1. **Desktop Editor (`build/electron/`)** — Electron application (`src/js/index-electron.js` Main process + `src/index-electron.html` Renderer + `src/js/editor.js` + `src/js/backend/Electron.js` + `src/js/exporter/index-electron.js`).
2. **Browser Editor & Standalone Player (`build/browser/`, `src/templates/`)** — Read-only browser demo (`src/index-browser.html` + `src/js/backend/FileReader.js` + `src/js/exporter/index-browser.js`) and self-contained presentation HTML/Presenter outputs (`player.html`, `presenter.html`) whose minified Browserify bundles (`player.js`, `presenter.js`) are inlined at build time.

```
+-------------------------------------------------------------------------+
| Electron Main Process (src/js/index-electron.js)                        |
| - @electron/remote/main.initialize() & enable(mainWindow.webContents)   |
| - electron-app-settings (enableColorCorrectRendering, HW accel)         |
| - Creates mainWindow (nodeIntegration:true, contextIsolation:false)     |
+------------------------------------+------------------------------------+
                                     | loads file://build/electron/src/index.html
                                     v
+-------------------------------------------------------------------------+
| Electron Renderer / Editor (src/index-electron.html -> src/js/editor.js)|
| - Direct Node require("./js/editor.js") & fs/path/process/@electron/remote|
| - Controller + Storage + Preferences + Selection + Presentation         |
| - Views: Preview (SVG DOM), Properties, Timeline, Toolbar (Inferno v8)  |
| - Backend: Electron.js (fs.watch, dialogs, process.env.PWD)             |
| - Exporter: index-electron.js (spawns remote.BrowserWindow per export)  |
+------------------------------------+------------------------------------+
                                     | spawns child BrowserWindow (file://*.sozi.html)
                                     v
+-------------------------------------------------------------------------+
| Export Child Window (src/js/exporter/exporter-preload.js)               |
| - contextIsolation: false                                               |
| - Uses removed ipcRenderer.sendTo(callerId, ...) -> hangs in Electron 28+|
| - PDF: printToPDF({ marginsType: 1 }) + pdf-lib                         |
| - PPTX: CDP Page.captureScreenshot + officegen                          |
| - Video/PNG: CDP Page.captureScreenshot + ffmpeg / resourcesPath        |
+-------------------------------------------------------------------------+
```

---

## 2. Main vs. Renderer Process Architecture & Security Gaps

### 2.1 Electron Main Process (`src/js/index-electron.js`)
- **`remoteMain.initialize()`** (`src/js/index-electron.js:3,6`): Initializes `@electron/remote/main`.
- **`createWindow()`** (`src/js/index-electron.js:12-48`): Creates an `800x600` `BrowserWindow` loading `file://${__dirname}/../index.html` (`line 35`), attaches `remoteMain.enable(mainWindow.webContents)` (`line 27`), hides the menu bar (`line 29`), and opens DevTools if `process.env.SOZI_DEVTOOLS` is set (`lines 31-33`).
- **Chromium/Electron Flags & Persistent Settings** (`lines 4, 52-70`): Appends `--disable-gpu-sandbox` (`line 52`, workaround for issue `#603`) and reads/initializes `enableColorCorrectRendering` (`lines 55-61`) and `enableHardwareAcceleration` (`lines 64-70`) via `electron-app-settings`.
- **Lifecycle Events** (`lines 75-92`): `app.on("ready", createWindow)`, `window-all-closed` (`app.quit()` except `darwin`), `activate` (re-creates window on macOS dock click).

### 2.2 Renderer Entry & Bootstrap (`src/index-electron.html`, `src/index-browser.html`)
- **`src/index-electron.html:11-22`**: Loads `fork-awesome`, `@fontsource-variable/inter`, and 5 editor CSS files, then directly executes Node in the renderer:
  ```js
  process.env.NODE_ENV = "production";
  require("./js/editor.js");
  ```
- **`src/index-browser.html:12-21`**: Loads `<script src="src/js/editor.min.js" defer>` (`line 21`), renamed to `build/browser/index.html` by `gulpfile.js:284`.

### 2.3 Security, Isolation & Modern Electron Compatibility Gaps
| Location | Current Pattern | Risk / Electron Breakage | Phase 1 Target |
|---|---|---|---|
| `src/js/index-electron.js:19-24` | `nodeIntegration: true`, `contextIsolation: false`, `sandbox: false` | Full Node/OS access exposed to renderer DOM & loaded SVG context; blocks modern Electron security posture | `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`, `preload: preload-editor.js` |
| `src/js/index-electron.js:3,6,27` & `src/js/backend/Electron.js:13,25,73,99,137` | `@electron/remote` (`getCurrentWindow()`, `remote.dialog`, `remote.process.argv`, `new remote.BrowserWindow`) | Deprecated remote IPC surface; bypasses context isolation; unmaintained pattern | Remove `@electron/remote`; use explicit Main `ipcMain.handle` + `contextBridge` (`window.soziElectron`) |
| `src/js/backend/Electron.js:8-14,34,101` | Direct `fs`, `path`, `process`, `electron-app-settings` imports in renderer; `cwd = process.env.PWD` (`line 34`) | Fails under `nodeIntegration: false` + `sandbox: true`; `process.env.PWD` is `undefined` on Windows, crashing `path.resolve(cwd, arg)` (`line 101`) when `argv.length > 1` (`#788`); `npm start` passes `build/electron` triggering spurious `"File not found"` (`line 106`) | Move FS, dialogs, watchers, `electron-app-settings`, and `process.cwd()` fallback to Main process |
| `src/js/Storage.js:12,108,413,425` | `import path from "path"` used for `templatePath`, `toRelativePath`, and `readCustomFiles` | Fails in sandboxed renderer without Node built-ins | Bundle templates / pure-JS path helpers or expose via `window.soziElectron` |
| `src/js/view/Toolbar.js:10,164` | `import pkg from "../../../package.json"` | Leaks build/package path assumptions if unbundled | Bundle with `esbuild` JSON loader or expose version via preload/define |
| `src/js/view/Properties.js:305` | `evt.target.files[0].path` | Electron `File.path` removed/restricted in modern sandboxed Electron (`webUtils.getPathForFile` or native open dialog) | Route custom CSS/JS file picking through `window.soziElectron` dialog/helper |
| `src/js/exporter/index-electron.js:198,288,387` | `new remote.BrowserWindow({ webPreferences: { contextIsolation: false } })` in renderer | Depends on `@electron/remote` + unsafe `contextIsolation: false` | Run export orchestration in Main process with `contextIsolation: true` |
| `src/js/exporter/exporter-preload.js:17,22,45,50` | `ipcRenderer.sendTo(callerId, ...)` | **`ipcRenderer.sendTo` was removed in Electron 28**, causing exports to hang indefinitely (`#722`) | Replace with `ipcRenderer.send(...)` handled by Main process |
| `src/js/exporter/index-electron.js:223` | `printToPDF({ ..., marginsType: 1 })` | Legacy `marginsType` integer option removed in modern Electron (`margins: { marginType: "none" }`) | Update to `margins: { marginType: "none" }` |

---

## 3. Presentation Data Model, Formats & Migration

### 3.1 Core Model Classes (`src/js/model/`)
1. **`Presentation` (`src/js/model/Presentation.js:636-1095`)** — Extends `EventEmitter`:
   - Holds `document` (`SVGDocumentWrapper`), `layers` (`Layer[]`), `frames` (`Frame[]`), `initialCameraState` (`CameraState`), `title`, `aspectWidth` / `aspectHeight`, player interaction flags (`enableKeyboard*`, `enableMouse*`, `updateURLOnFrameChange`), export settings (`exportType`, `exportToPDF*`, `exportToPPTX*`, `exportToVideo*`), `elementsToHide`, and `customFiles`.
   - **`setSVGDocument(svgDocument)`** (`lines 859-893`): Creates one `Layer` (`lines 539-619`) per top-level `<g id="...">` element (using `doc.handler.getLabel(svgNode) || "#" + nodeId`), collects un-ID'd `<g>` nodes into a trailing `auto` layer (`__sozi_auto__` via `Layer.groupId` at `line 582`), extracts and removes `svg > title` (`lines 886-890`), and emits `"svgChange"` (`line 892`).
   - **`updateLinkedLayers()`** (`lines 1058-1085`): Propagates `cameraStates`, `referenceElementId`, and `outlineElementId` across consecutive frames where `layerProperties[layerIndex].link === true`.
2. **`Frame` (`src/js/model/Presentation.js:251-527`)**:
   - Properties: `frameId`, `title`, `titleLevel`, `timeoutMs`, `timeoutEnable`, `transitionDurationMs`, `showInFrameList`, `showFrameNumber`, `notes` (HTML), `layerProperties` (`LayerProperties[]`, parallel to `presentation.layers`), `cameraStates` (`CameraState[]`).
   - **`fromStorable(storable)`** (`lines 455-490`): Restores frame properties and per-layer `LayerProperties`/`CameraState`; if a `referenceElement` exists in the SVG, re-anchors `cameraState.setAtElement(...)` (`lines 478-486`) using stored offsets (`referenceElementAuto` / `deltaX`, `deltaY`, `widthFactor`, `heightFactor`, `deltaAngle`). Note: upstream issue `#688` tracks floating-point drift across repeated save/load cycles here.
3. **`LayerProperties` (`src/js/model/Presentation.js:39-248`)**:
   - Stores per-frame/per-layer `link`, `referenceElementId`, `outlineElementId`, `transitionTimingFunction`, `transitionRelativeZoom`, `transitionPathId`, plus element visibility helpers `outlineElementHide` (`lines 206-225`) and `transitionPathHide` (`lines 231-247`) backed by `presentation.elementsToHide`.
4. **`CameraState` (`src/js/model/CameraState.js:30-345`)**:
   - Stores `cx`, `cy`, `width` (min `1`, `lines 148-154`), `height` (min `1`, `lines 163-169`), `opacity`, `angle` (normalized `[-180, 180]`, `lines 178-190`), `clipped`, `clipXOffset`, `clipYOffset`, `clipWidthFactor`, `clipHeightFactor`.
   - **`setAtElement(...)`** (`lines 260-292`): Computes center, scale, and rotation in layer coordinates via `svgElement.getBBox()`, `svgRoot.createSVGPoint()`, and `layerGroup.getScreenCTM().inverse().multiply(svgElement.getScreenCTM())` (`lines 262-280`).
   - **`offsetFromElement` / `applyOffset`** (`lines 312-344`): Computes and applies relative delta offsets between a reference SVG element and the current camera state.
5. **`Selection` (`src/js/model/Selection.js:12-183`)**:
   - Tracks `selectedFrames` and `selectedLayers`; serializes `frameId` and `groupId` arrays (`lines 47-52`).
6. **`Preferences` (`src/js/model/Preferences.js:9-128`)**:
   - Persists editor UI settings (`language="auto"`, `fontSize=11`, `enableNotifications=true`, `animateTransitions=true`, `saveMode="onblur"`, `reloadMode="auto"`, `keys` shortcut map `lines 89-100`) to `localStorage`.

### 3.2 File Formats: `.sozi.json` vs. `toMinimalStorable()` vs. `.sozi.html`
- **`.sozi.json` (Full Editor Project File)**: Produced by `Storage.prototype.getJSONData()` (`src/js/Storage.js:361-370`), merging `presentation.toStorable()`, `selection.toStorable()`, and `controller.toStorable()` (`editableLayers`, `defaultLayers`) as 2-space-indented JSON.
- **`toMinimalStorable()` (`Presentation.js:951-964`)**: Strips editor-only metadata (`selection`, `editableLayers`, `exportTo*`, `aspectWidth`/`Height`, `link`, `referenceElementId`, `outlineElementId`) and embeds only runtime playback data into `window.soziPresentationData` inside `.sozi.html`.
- **`.sozi.html` & `-presenter.sozi.html`**: Rendered via `nunjucks.render("player.html", ...)` (`Storage.js:382-390`) and `nunjucks.render("presenter.html", ...)` (`Storage.js:399-404`), containing the sanitized SVG, `window.soziPresentationData`, custom CSS/JS, and build-time inlined `player.js` / `presenter.js`.

### 3.3 Legacy Presentation Migration (`src/js/upgrade.js`)
- **`upgradeFromSVG(pres, controller)` (`src/js/upgrade.js:113-205`)**: Migrates Sozi <=13 presentations stored as `<sozi:frame>` and `<sozi:layer>` XML elements (`SOZI_NS = "http://sozi.baierouge.fr"`, `line 17`) inside the SVG document. Maps legacy timing profiles (`convertTimingFunction`, `lines 36-62`: e.g., `"accelerate"` -> `"easeIn"`, `"immediate-beginning"` -> `"stepStart"`), parses boolean/numeric attributes, anchors `cameraState.setAtElement(refElt)` (`line 187`), and populates `controller.editableLayers` (`line 168`).
- **`upgradeFromStorable(storable)` (`src/js/upgrade.js:213-232`)**: Migrates pre-`17.02.05` `.sozi.json` objects by removing `referenceElementAuto`, renaming `referenceElementHide` -> `outlineElementHide`, and defaulting `outlineElementId` to `referenceElementId`.

---

## 4. Editor Architecture (`editor.js`, `Controller`, `Storage`, Views, Backends)

- **Bootstrap (`src/js/editor.js:21-196`)**: On `window` `"load"`, instantiates `Presentation`, `Selection`, `Viewport(presentation, true)`, `Player`, `PlayerController`, `Preferences`, `Controller`, `Preview`, `Properties`, `Toolbar`, `Timeline`, calls `controller.activate()` (`line 36`), wires splitter drag handlers (`lines 56-94`), and dispatches global keyboard shortcuts (`lines 96-195`).
- **`Controller` (`src/js/Controller.js:54-1880`)**: Central state machine (`EventEmitter`) managing undo/redo (`UNDO_STACK_LIMIT = 100` at `line 20`, `perform`/`undo`/`redo` at `lines 1822-1879`), notifications (`lines 223-267`), frame/layer CRUD (`lines 467-708`), camera updates (`lines 1450-1491`), outline element auto-selection (`lines 1149-1168`), export triggers (`lines 1764-1809`), and i18n initialization (`applyPreferences` at `lines 1744-1755`).
- **`Storage` (`src/js/Storage.js:25-430`)**: Configures `nunjucks` (`lines 108-113`), tracks dirty flags (`jsonNeedsSaving`, `htmlNeedsSaving` at `lines 115-121`), triggers `onblur` autosave (`lines 123-127`), loads SVG/JSON, resolves relative `<image>` URLs (`lines 218-232`), and writes `.sozi.json`, `.sozi.html`, and `-presenter.sozi.html`.
- **Views (`src/js/view/`)**:
  - `VirtualDOMView.js:20-84`: Base class wrapping `inferno` (`render()`) + DOM field synchronization via `this.state`.
  - `Preview.js:10-170`: Non-VDOM host embedding `presentation.document.root` directly (`line 79`); handles Alt+click outline selection (`lines 130-137`).
  - `Properties.js:37-801`: Right-hand inspector for Preferences, Presentation/Frame/Layer properties, rich-text notes (`document.execCommand` at `lines 638-659`, deprecated), custom CSS/JS files (`evt.target.files[0].path` at `line 305`), and Export forms (`lines 394-556`).
  - `Timeline.js:27-344`: 4-quadrant synchronized frame (column) x layer (row) matrix with link/dirty indicators.
  - `Toolbar.js:17-170`: Aspect ratio inputs (note pre-existing `!width.isNaN` / `!height.isNaN` bug at `lines 73, 88`), drag mode toggles (`translate`, `scale`, `rotate`, `clip`), undo/redo, fullscreen (`screenfull`), save/reload/export, preferences, and version info (`pkg.version` at `line 164`).
- **Backends (`src/js/backend/`)**:
  - `AbstractBackend.js:22-224`: Base interface + `doAutosave()` / `saveOutdatedFiles()`.
  - `Electron.js:40-295`: Desktop FS backend (`fs.readFile`, `fs.writeFile`, `fs.watch` with 100ms debounce at `lines 186-196`, `remote.dialog`, `electron-app-settings`).
  - `FileReader.js:13-73`: Read-only browser `<input type="file">` backend.
  - `GoogleDrive.js:15-241`: Legacy `gapi` Drive v2 backend (currently commented out in `backend/index-browser.js:3`).
  - Build-time entry selection: `gulpfile.js:281,285` renames `backend/index-{electron,browser}.js` -> `backend/index.js`.

---

## 5. Player & Presenter Architecture (`src/js/player*`, `src/templates/`)

- **Player Entry (`src/js/player.js:1-239`)**: On `"load"`, wraps `<svg>` in `SVGDocumentWrapper`, initializes `Presentation`, `Viewport`, `Player`, `PlayerController`, and optional modules (`Media`, `FrameList`, `FrameNumber`, `FrameURL`, `TouchGestures`), exposes `window.sozi` (`lines 174-187`), and communicates with presenter/parent windows via `window.postMessage` (`lines 105-143`, `225-239`).
- **Presenter Entry (`src/js/presenter.js:1-165`)**: Opens the main presentation popup (`window.open` at `line 74`), coordinates 3 preview `<iframe>`s (current, previous, next frames), displays speaker notes (`lines 20-27`), and forwards keyboard/button navigation.
- **Core Runtime Modules (`src/js/player/`)**:
  - `Viewport.js:18-473`: Manages `Camera[]` (one per layer), pointer drag/clip handles, and `translate`/`zoom`/`rotate`/`clip` helpers.
  - `Camera.js:59-534`: Extends `CameraState`; renders `<mask>` (editor) or `<clipPath>` (player) (`lines 98-156`), computes SVG `transform` attributes, and performs `interpolate()` (`lines 460-533`) supporting linear, quadratic `relativeZoom` (`lines 468-477`), and SVG path following (`lines 490-498`).
  - `Timing.js:19-179`: Cubic-Bézier Newton-Raphson solver (`makeBezier`) and step functions (`makeSteps`, `stepMiddle`) with `.reverse` functions.
  - `Animator.js:14-219`: Shared `requestAnimationFrame` loop driving transitions.
  - `Player.js:51-532`: Playback state machine (`playFromFrame`, `pause`, `jumpToFrame`, `moveToFrame`, `waitTimeout`, `blankScreen`). Note: `onAnimatorStep` passes `transition.reverse` (`line 461`) while `setupTransition` sets `backwards` (`line 445`).
  - `PlayerController.js:164-883`: Shared mouse, wheel, and keyboard input handler used in both editor (`editMode: true`) and standalone player (`editMode: false`).
  - `Media.js:72-222`, `FrameList.js:20-259`, `FrameNumber.js:22-31`, `FrameURL.js:26-81`, `TouchGestures.js:129-651`: Media `<foreignObject>` embedding, animated TOC drawer, frame counter, URL hash sync, and multi-touch gestures.
- **Templates (`src/templates/`)**:
  - `player.html` (`153` lines): Injects `{{ customCSS }}`, `{{ svg }}`, `soziPresentationData`, `{{ js|safe }}` (minified `player.js`), and `{{ customJS|safe }}`. Line `131` has hardcoded English `title="Click to show the frame list"`.
  - `presenter.html` (`105` lines): Injects 3 preview `<iframe>`s and `{{ js|safe }}` (minified `presenter.js`). Lines `89` and `91` have hardcoded English `title="Previous"` and `title="Next"`.

---

## 6. Exporters (`src/js/exporter/`)

- **`index-browser.js` (`lines 13-38`)**: Stub functions resolving immediately for browser builds.
- **`index-electron.js` (`lines 1-531`)**:
  - **PDF (`exportToPDF`, `lines 185-266`)**: Opens a visible frameless `remote.BrowserWindow`, steps through selected frames via IPC (`initializeExporter` / `jumpToFrame` -> `jumpToFrame.done`), captures each frame via `w.webContents.printToPDF({ pageSize, landscape, marginsType: 1 })` (`lines 220-224`), and merges pages with `pdf-lib`.
  - **PPTX (`exportToPPTX`, `lines 274-373`)**: Attaches Chrome DevTools Protocol (`w.webContents.debugger.attach("1.2")`), captures PNGs with `Page.captureScreenshot`, and writes slides with `officegen`.
  - **Video / PNG Sequence (`exportToVideo`, `lines 381-531`)**: Steps animation frames (`moveToNext` / `moveToNext.step` / `moveToNext.more`), captures PNGs (`img0.png`, `img1.png`, ... — unpadded, `#717`), and invokes `spawnSync("ffmpeg", ...)` then `process.resourcesPath/ffmpeg(.exe)` (`lines 451-458`).
  - **Known Bugs & Electron Breakages**:
    1. `exporter-preload.js:17,22,45,50` calls `ipcRenderer.sendTo(callerId, ...)`, removed in Electron 28 (`#722`).
    2. `printToPDF` passes removed `marginsType: 1` (`line 223`).
    3. `onDone()` at `line 484` (`if (index === 0 && imgIndex > 0) { onDone(); }`) lacks `return;`, continuing into the screenshot loop after export finishes.
    4. Missing `ffmpeg` binary triggers `reject()` -> `"Failed to write video file."` (`Controller.js:1805`), though silent/opaque errors are reported in `#604`/`#650` and `destDir` is only created in non-PNG mode (`line 409`).

---

## 7. SVG Handlers (`src/js/svg/`)

- **`svg/index.js:2-4`**: Registers handlers in priority order: `AiHandler`, `InkscapeHandler`, `ImpressHandler`.
- **`SVGDocumentWrapper.js:47-276`**:
  - Defines `DefaultSVGHandler` (`lines 47-94` — not a separate file) and `SVGDocumentWrapper` (`lines 97-276`).
  - `fromString(data)` (`lines 163-233`): Parses XML (`DOMParser`), selects first matching handler (`lines 167-173`), runs `doc.handler.transform(svgRoot)`, `removeViewbox()` (`lines 240-243`), `removeScripts()` (`lines 253-259`), `disableHyperlinks()` (`lines 268-275`), always runs `AiHandler.transform(svgRoot)` (`lines 191-194`), wraps top-level loose drawable elements into `<g>` groups (`lines 197-227`), and serializes `doc.asText` (`line 230`).
- **`InkscapeHandler.js:29-74`**:
  - Matches `xmlns:inkscape` (`line 33`), recognizes `inkscape:groupmode="layer"` (`line 65`), reads `inkscape:label` (`line 70`).
  - **Inkscape 1.2+ Bug (`line 54`)**:
    ```js
    const [, red, green, blue] = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(pageColor);
    ```
    Inkscape 1.2+ writes 8-digit `#RRGGBBAA` `pagecolor` values (e.g., `#ffffff00` or `#505050ff`), causing `.exec(pageColor)` to return `null` and throw `TypeError: Cannot read properties of null` during destructuring.
- **`AiHandler.js:13-54`** & **`ImpressHandler.js:16-42`**: Unwrap Illustrator `<switch>`/`<foreignObject>` blocks and LibreOffice Impress slide wrapper groups (`g.DummySlide`, `g.Master_Slide`, `g.SlideGroup > g.Slide`).

---

## 8. Internationalization (i18n) System (`locales/`, `src/js/i18n.js`)

- **Runtime (`src/js/i18n.js:10-51`)**: Uses `jed` (`new Jed(localeData)`) and build-time generated `src/js/locales.js` (compiled from `locales/*.po` by `po2json` in `gulpfile.js:107-128`). Normalizes BCP-47 tags (`en-US` -> `en-us`, fallback to base `en`, `lines 38-48`).
- **Language Registry (`src/js/view/languages.js:15-56`)**: Lists **38** locales matching the **38** `.po` files in `locales/` (including `id: "Bahasa Indonesia"` at `line 35`).
- **Template Catalog (`locales/messages.pot`)**: `738` lines, `180` active `msgid` strings extracted from `_("...")` calls via `jspot` (`gulpfile.js:84-97`).
- **`locales/id.po` Current State (`793` lines)**:
  - **180/180 active entries translated (100%)**, 0 untranslated active entries.
  - Contains **6 `#~` obsolete entries** (`lines 773-793`, 4 marked `#, fuzzy`) and minor typos/terminology inconsistencies (`lines 93, 129, 288, 292, 333, 511`) plus 5 hardcoded English strings in `player.html:131`, `presenter.html:89,91`, `player.js:194`, `presenter.js:87` — all cataloged as **Phase 4 Targets** in `MODERNIZATION_PLAN.md`.

---

## 9. Complete Traced End-to-End Flow

1. **Main Process Startup**: `build/electron/package.json` (`main: "src/js/index-electron.js"`) launches `src/js/index-electron.js` -> `remoteMain.initialize()` (`line 6`) -> applies `--disable-gpu-sandbox` (`line 52`) and `electron-app-settings` (`lines 55-70`) -> `app.on("ready", createWindow)` (`line 75`) creates `mainWindow` (`lines 16-25`), calls `remoteMain.enable(mainWindow.webContents)` (`line 27`), and loads `file://${__dirname}/../index.html` (`line 35`).
2. **Renderer Bootstrap**: `src/index-electron.html:20-21` runs `require("./js/editor.js")` -> `window` `"load"` (`editor.js:21-36`) constructs `Presentation`, `Selection`, `Viewport`, `Player`, `PlayerController`, `Preferences`, `Controller`, `Preview`, `Properties`, `Toolbar`, `Timeline` -> `controller.activate()` (`Controller.js:172-176`) calls `preferences.load()` (`Preferences.js:111`), `applyPreferences()` (`Controller.js:1744` -> `i18n.init` at `i18n.js:30`), and `storage.activate()` (`Storage.js:135-142`), which instantiates `Electron` (`backend/index-electron.js:2`).
3. **Opening an SVG**: `Electron.constructor()` (`Electron.js:47-114`) resolves CLI `remote.process.argv` (`lines 99-110`) or calls `openFileChooser()` (`lines 112, 134-146` -> `remote.dialog.showOpenDialogSync`), then calls `controller.storage.setSVGFile(fileName, this)` (`lines 103, 144`) -> `Storage.setSVGFile()` (`Storage.js:176-181`) -> `Electron.load()` (`Electron.js:174-207`, reads UTF-8 file & starts 100ms-debounced `fs.watch`) -> `Storage.loadSVGData()` (`Storage.js:191-205`) -> `SVGDocumentWrapper.fromString(data)` (`SVGDocumentWrapper.js:163-233`) -> `resolveRelativeURLs(location)` (`Storage.js:218-232`) -> `presentation.setSVGDocument(this.document)` (`Presentation.js:859-893`, emits `"svgChange"` -> `Preview.onLoad()` at `Preview.js:71-87` mounts SVG & initializes `Viewport`/`PlayerController`).
4. **Loading / Upgrading `.sozi.json`**: `Storage.loadSVGData()` awaits `openJSONFile(replaceFileExtWith(name, ".sozi.json"), location)` (`Storage.js:200, 242-281`). If found: `loadJSONData(data)` (`lines 326-332`) runs `JSON.parse`, `upgradeFromStorable(storable)` (`upgrade.js:213-232`), `presentation.fromStorable` (`Presentation.js:970-1008`, re-running `CameraState.setAtElement` at `lines 478-486`), `controller.fromStorable`, and `selection.fromStorable`. If missing: `upgradeFromSVG(presentation, controller)` (`upgrade.js:113-205`) imports Sozi <=13 XML and `backend.create(...)` (`Storage.js:265`) writes initial `.sozi.json`. Then `controller.onLoad()` (`Storage.js:273` -> `Controller.js:318-347`), `createHTMLFile()` (`Storage.js:279`), and `createPresenterHTMLFile()` (`Storage.js:280`) run.
5. **Editing Frames & Dirty Tracking**: Actions from `Toolbar.js` (`addFrame`), `Timeline.js` (`addFrame`/`deleteFrames`/`moveFrames`/selection), `Properties.js` (field setters), or `Preview.js` (`localViewportChange` -> `updateCameraStates`, Alt+click -> `setOutlineElement`) execute via `Controller.perform()` (`Controller.js:1822-1837`), pushing onto `undoStack` (limit `100`), calling `presentation.updateLinkedLayers()` (`Presentation.js:1058`), and emitting `"presentationChange"`, `"editorStateChange"`, and `"repaint"`. `Storage` listeners (`Storage.js:115-121`) mark `jsonNeedsSaving = htmlNeedsSaving = true` (or `jsonNeedsSaving = true` on selection/editor-state changes).
6. **Saving `.sozi.json` & Generating HTML**: Triggered by `Ctrl+S` (`editor.js:136`), `Toolbar.js:141`, window `blur` (`Storage.js:123-127` when `saveMode === "onblur"`), or `beforeunload` (`Electron.js:59-85` -> `quit` at `120-131`). `Storage.save()` -> `backend.doAutosave()` (`AbstractBackend.js:188`) -> `saveOutdatedFiles()` (`lines 173-179`) writes `getJSONData()` (`.sozi.json`, `Storage.js:361-370`) and `exportHTML()` (`.sozi.html`, `Storage.js:382-390` via `nunjucks.render("player.html", ...)` with `toMinimalStorable()` and inlined minified `player.js`). `-presenter.sozi.html` is generated on load via `exportPresenterHTML()` (`Storage.js:399-404`).

---

## 10. Baseline Build/Lint Layout & MPL-2.0 License Header Audit

- **Gitignored Generated Paths (`.gitignore:2-9`)**: `node_modules`, `build`, `dist`, `deb`, `cache`, `/js/backend/GoogleDrive.config.js`, `npm-debug.log`, `*.mo`.
- **MPL-2.0 Header Audit (`src/js/`, 44 files)**:
  - **39 of 44** `.js` files contain the standard 3-line MPL-2.0 header (`/* This Source Code Form is subject to the terms of the Mozilla Public / License, v. 2.0. ... */`).
  - **5 files lack the MPL-2.0 header** (slated for Phase 1 remediation):
    1. `src/js/index-electron.js`
    2. `src/js/presenter.js`
    3. `src/js/backend/index-browser.js`
    4. `src/js/backend/index-electron.js`
    5. `src/js/svg/index.js`
- **Build & Lint Baseline Summary**:
  - `config/eslintrc.mjs`: Uses `@eslint/js` and `@eslint/eslintrc` (`FlatCompat`, neither declared in `package.json`), `ecmaVersion: 8`, mixed `browser` + `node` globals; direct `eslint` fails with **6 errors** (`jsdoc/require-jsdoc` in `Player.js:397`, `PlayerController.js:232,236,240,244,821`) and **89 warnings**.
  - `config/csslintrc.json`: Targets abandoned `csslint@1.0.5` (2016) with 8 disabled rules.
  - `gulpfile.js`: Invokes `jspot` (writes `locales/messages.pot` with Windows `\` paths on Windows and fails on ES2020+ syntax due to `ecmaVersion: 9`), `po2json`, `babel`, `browserify`, `uglify`, `modclean`, and `npm install --only=prod` inside `build/electron`.
