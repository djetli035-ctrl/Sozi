# Sozi Modernization Plan (`24.11.17` -> Modern LTS Electron & Build Toolchain)

> **Scope:** Baseline audit, dependency/issue/PR triage, risk register, architectural decisions, and phased modernization roadmap from upstream commit `2296052c2bd9a6f7f5361663fbe261931f9e4b2d` (`sozi-projects/Sozi`).
> **Execution Rule:** This run covers **Phase 0 -> Phase 1 -> Phase 2**, then **STOPS for user review** before Phase 3 and Phase 4. All pushes target `origin` (`https://github.com/djetli035-ctrl/Sozi.git`); `upstream` (`https://github.com/sozi-projects/Sozi.git`) is read-only (`push = no_push`).

---

## 1. Baseline Toolchain & Verification Findings (Node `v24.11.0`, Windows)

All checks below were executed read-only against unmodified upstream `2296052` on **Windows** (`node v24.11.0`, `npm 11.6.1`, `git 2.52.0.windows.1`, workspace `D:\02_Development & Coding\sozi`).

| Check | Command | Exit Code | Key Output / Baseline Finding |
|---|---|---|---|
| Clean install | `npm ci` | `1` | Fails in `node_modules\inferno` (`8.2.3`) running `opencollective-postinstall` via `cmd.exe /d /s /c`: unescaped `&` and spaces in `D:\02_Development & Coding\sozi` split the shell command (`'Coding\sozi\node_modules\.bin\' is not recognized...`, `Cannot find module 'D:\opencollective-postinstall\index.js'`). |
| Install without scripts | `npm ci --ignore-scripts` + `node node_modules/electron/install.js` | `0` / `0` | Installs `1,032` packages cleanly (`148` prod, `870` dev, `30` optional, `6` peer) and downloads Electron `33.2.0` binary without error. |
| Prod audit | `npm audit --omit=dev --json` | `1` | **2 vulnerabilities** (`0` critical, **2 high**, `0` moderate, `0` low): `electron` (`<=42.11.15`, 46 advisories) and `extract-zip` (`<=2.0.1`, 2 symlink traversal advisories). |
| Full audit | `npm audit --json` | `1` | **43 vulnerabilities** (**5 critical**, **22 high**, **10 moderate**, **6 low**). All 5 critical and 20 dev high CVEs stem from `jspot`, `gulp-jsdoc3`, and `electron-builder@25.1.8`. |
| Outdated check | `npm outdated` | `1` | **17 outdated direct dependencies** (`@babel/core`, `@babel/preset-env`, `@electron/remote`, `@fontsource-variable/inter`, `core-js`, `electron`, `electron-builder`, `eslint`, `eslint-plugin-jsdoc`, `globals`, `gulp`, `gulp-babel`, `gulp-nunjucks`, `gulp-rename`, `inferno`, `inferno-hyperscript`, `screenfull`). Note: `eslint-plugin-jsdoc` reports `63.3.3` on Node `24.11.0` because `65.5.6` requires `^22.22.2 \|\| >=24.15.0`. |
| npm script shims | `npm run eslint` / `npm run csslint` / `npm run build` | `1` / `1` / `1` | All three fail under `cmd.exe` on Windows when workspace path contains `&` (`node_modules\.bin\*.cmd` splits at `&`). |
| Direct ESLint | `node node_modules/eslint/bin/eslint.js --config config/eslintrc.mjs src/js/` | `1` | **95 problems (6 errors, 89 warnings)**: 6 `jsdoc/require-jsdoc` errors (`Player.js:397`, `PlayerController.js:232,236,240,244,821`), plus `no-unused-vars`, `no-prototype-builtins`, `jsdoc/no-undefined-types`, `jsdoc/check-types` warnings. |
| Direct CSSLint | `node node_modules/csslint/dist/cli.js --config=config/csslintrc.json src/css/` | `0` | 0 errors across 5 CSS files, with 8 core rules suppressed in `config/csslintrc.json`. |
| Direct Gulp build | `node node_modules/gulp/bin/gulp.js` | `1` | `jspotTask`, `msgmergeTask`, `po2jsonTask`, `transpileTask`, `browserifyTask`, `templateTask`, `copyTask`, `renameTask`, and `packageJsonTask` succeed (with Node 24 `[DEP0128]`/`[DEP0180]` warnings from `po2json`/`jspot`); **`installDepsTask` fails** inside `build/electron` when `npm install --only=prod` hits `inferno@8.2.3` `opencollective-postinstall` on the `&` path. Also `jspotTask` dirtied `locales/messages.pot` with Windows `\` paths (restored cleanly via `git checkout --`). |

---

## 2. Complete Dependency Inventory (38 Direct Packages) & `npm audit`

All **13 `dependencies`** and **25 `devDependencies`** in `package.json` (`optionalDependencies`: 0) are compatible with Sozi's **MPL-2.0** license (`po2json` is `LGPL-2.0-or-later` build-time only; replaced by MIT `gettext-parser` in Phase 1).

### 2.1 Production `dependencies` (13)

| # | Package | `package.json` | `package-lock.json` | Latest Stable | Status / Maintenance | License (MPL-2.0 Compat) | Electron / Node 24 Notes & Phase 1 Action |
|---|---|---|---|---|---|---|---|
| 1 | `@electron/remote` | `^2.0.8` | `2.1.2` | `2.1.4` | Maintained | `MIT` (Compat) | **Remove in Phase 1.** Forces `contextIsolation: false` / `sandbox: false`. Replace with Main IPC + `preload-editor.js`. |
| 2 | `@fontsource-variable/inter` | `^5.1.0` | `5.1.0` | `5.3.0` | Active | `OFL-1.1` (Compat font) | Static CSS/WOFF2 UI font; update to `^5.3.0` (or keep `5.x`). |
| 3 | `core-js` | `^3.16.0` | `3.39.0` | `3.51.0` | Active | `MIT` (Compat) | Only used by Babel `useBuiltIns: "usage"` (`gulpfile.js:213`). **Remove in Phase 1** when migrating to `esbuild` targeting modern Electron/evergreen browsers. |
| 4 | `electron-app-settings` | `^1.3.1` | `1.3.1` | `1.3.1` | Low churn (Sozi author, 0 deps) | `MPL-2.0` (Same) | Keep; move all access to Electron Main process behind IPC. |
| 5 | `fork-awesome` | `^1.2.0` | `1.2.0` | `1.2.0` | **DEPRECATED on npm** | `(OFL-1.1 AND MIT)` (Compat) | Static CSS/icon font; works offline now. **Defer replacement to Phase 4** (or keep until Phase 4 UX pass). |
| 6 | `inferno` | `^8.0.3` | `8.2.3` | `10.0.1` | Active (`8.x` works; `8.2.3` has `opencollective-postinstall` script) | `MIT` (Compat) | Keep `8.x` in Phase 1 to avoid breaking UI VDOM changes (`10.x` major optional later); avoid invoking `opencollective-postinstall` in build sub-installs (bundle renderer with `esbuild` so `build/electron` doesn't need `inferno` installed at runtime). |
| 7 | `inferno-hyperscript` | `^8.0.3` | `8.2.3` | `10.0.1` | Active | `MIT` (Compat) | Version-lock with `inferno`. Bundled into `editor.js` in Phase 1. |
| 8 | `jed` | `^1.1.0` | `1.1.1` | `1.1.1` | **Unmaintained (2016)** | `MIT` (Compat) | Pure JS, 0 deps, works cleanly in `esbuild` bundle. Keep for Phase 1–2; optional replacement later. |
| 9 | `nunjucks` | `^3.2.4` | `3.2.4` | `3.2.4` | Low churn | `BSD-2-Clause` (Compat) | Keep; verify template loading in bundled renderer (or precompile/load templates cleanly without Node `fs`/`path` in sandboxed renderer). |
| 10 | `officegen` | `^0.6.5` | `0.6.5` | `0.6.5` | **Unmaintained (2021)** | `MIT` (Compat) | Keep for PPTX export; move execution to Main process (requires Node `fs`/`stream`, incompatible with `nodeIntegration: false` renderer). |
| 11 | `pdf-lib` | `^1.16.0` | `1.17.1` | `1.17.1` | **Unmaintained (2021)** | `MIT` (Compat) | Pure JS, works in Node 24 Main process for PDF merge. Keep. |
| 12 | `screenfull` | `5.2` | `5.2.0` | `6.0.2` | Low churn | `MIT` (Compat) | `6.x` is pure ESM (fine with `esbuild`, incompatible with Browserify). Upgrade to `^6.0.2` or keep `5.2.0`. |
| 13 | `tmp` | `^0.2.7` | `0.2.7` | `0.2.7` | Maintained (`>=14.14`) | `MIT` (Compat) | Safe at `0.2.7`; move execution to Main process alongside exporters. |

### 2.2 Development `devDependencies` (25)

| # | Package | `package.json` | `package-lock.json` | Latest Stable | Status / Maintenance | License (MPL-2.0 Compat) | Phase 1 Action |
|---|---|---|---|---|---|---|---|
| 14 | `@babel/core` | `^7.29.6` | `7.29.6` | `8.1.3` | Active | `MIT` (Compat) | **Remove** (replaced by `esbuild`). |
| 15 | `@babel/preset-env` | `^7.14.8` | `7.26.0` | `8.0.7` | Active | `MIT` (Compat) | **Remove** (replaced by `esbuild`). |
| 16 | `browserify` | `^17.0.0` | `17.0.1` | `17.0.1` | Maintenance mode | `MIT` (Compat) | **Remove** (replaced by `esbuild`; drops vulnerable crypto polyfills). |
| 17 | `csslint` | `^1.0.5` | `1.0.5` | `1.0.5` | **Abandoned (2016)** | `MIT` (Compat) | **Replace with `stylelint`** (+ `stylelint-config-standard` or minimal rules). |
| 18 | `electron` | `^33.0.2` | `33.2.0` | `45.5.0` | **33.x EOL** | `MIT` (Compat) | **Upgrade to supported stable `>=43` (e.g., `^45.5.0`).** Fixes both prod HIGH CVEs. |
| 19 | `electron-builder` | `^25.1.8` | `25.1.8` | `26.18.0` | Active | `MIT` (Compat) | **Upgrade to `^26.18.0`.** Fixes CRITICAL `tar` & dev HIGH CVEs (`app-builder-lib`, etc.). |
| 20 | `envify` | `^4.1.0` | `4.1.0` | `4.1.0` | **Unmaintained (2017)** | `MIT` (Compat) | **Remove** (replaced by `esbuild` `define`). |
| 21 | `eslint` | `^9.14.0` | `9.15.0` | `10.15.0` | Active | `MIT` (Compat) | Upgrade to `^9.39.5` (or `^10.15.0`); declare `@eslint/js` explicitly and remove `FlatCompat`/`@eslint/eslintrc`. |
| 22 | `eslint-plugin-jsdoc` | `^50.4.3` | `50.5.0` | `65.5.6` (`63.3.3` for Node `24.11.0`) | Active | `BSD-3-Clause` (Compat) | Update within Node `24.11.0`-compatible range (`^50.8.0` or `^63.3.3`) and use native flat config. |
| 23 | `fancy-log` | `^2.0.0` | `2.0.0` | `2.0.0` | Minimal (unused) | `MIT` (Compat) | **Remove** (imported in `gulpfile.js:11`, never called). |
| 24 | `globals` | `^15.11.0` | `15.12.0` | `17.14.0` | Active | `MIT` (Compat) | Update to `^17.14.0` (or `^16.x`). |
| 25 | `gulp` | `^5.0.0` | `5.0.0` | `5.0.1` | Maintained | `MIT` (Compat) | **Remove** (replaced by `scripts/build.mjs` + `esbuild`). |
| 26 | `gulp-babel` | `^8.0.0` | `8.0.0` | `8.1.0` | Low churn | `MIT` (Compat) | **Remove** with Gulp. |
| 27 | `gulp-exec` | `^5.0.0` | `5.0.0` | `5.0.0` | **Unmaintained (2020)** | `MIT` (Compat) | **Remove** with Gulp. |
| 28 | `gulp-jsdoc3` | `^3.0.0` | `3.0.0` | `3.0.0` | **Unmaintained (2020)** | `Apache-2.0` (Compat) | **Remove** (eliminates unfixable `jsdoc`/`taffydb`/`sanitize-html`/`postcss`/`tmp` HIGH/MOD CVEs). |
| 29 | `gulp-newer` | `^1.4.0` | `1.4.0` | `1.4.0` | **Unmaintained (2018)** | `MIT` (Compat) | **Remove** with Gulp. |
| 30 | `gulp-nunjucks` | `^5.1.0` | `5.1.0` | `6.1.0` | Maintained | `MIT` (Compat) | **Remove** (call `nunjucks` directly in `scripts/build.mjs`). |
| 31 | `gulp-rename` | `^2.0.0` | `2.0.0` | `2.1.0` | Low churn | `MIT` (Compat) | **Remove** with Gulp. |
| 32 | `gulp-uglify` | `^3.0.2` | `3.0.2` | `3.0.2` | **Unmaintained (2019)** | `MIT` (Compat) | **Remove** (replaced by `esbuild` `minify: true`). |
| 33 | `jspot` | `^0.3.17` | `0.3.17` | `0.3.17` | **Unmaintained (2021)** | `BSD` (Compat) | **Remove** (hardcoded `ecmaVersion: 9`, pulls CRITICAL `underscore` via `nomnom`->`po2json@0.4.5` & HIGH `moment`). |
| 34 | `modclean` | `^3.0.0-beta.1` | `3.0.0-beta.1` | `3.0.0-beta.1` | **Abandoned 2018 beta** | `MIT` (Compat) | **Remove** (not needed when Main/Renderer are bundled). |
| 35 | `node-glob` | `^1.2.0` | `1.2.0` | `1.2.0` | **Dead (2016) & UNUSED** | `BSD-3-Clause` (Compat) | **Remove** (`gulpfile.js:10` requires `glob` and never calls it). |
| 36 | `po2json` | `github:mikeedwards/po2json#1.0.0-beta-3` | `1.0.0-beta-3` | `0.4.5` (npm) | Unmaintained 2016 git tag | **`LGPL-2.0-or-later`** (Compat build tool) | **Replace with `gettext-parser` (`MIT`)** in `scripts/build.mjs`. |
| 37 | `vinyl-buffer` | `^1.0.1` | `1.0.1` | `1.0.1` | Unmaintained (2017) | `MIT` (Compat) | **Remove** with Gulp/Browserify. |
| 38 | `vinyl-source-stream` | `^2.0.0` | `2.0.0` | `2.0.0` | Unmaintained (2017) | `MIT` (Compat) | **Remove** with Gulp/Browserify. |

### 2.3 Exact `npm audit` Findings Summary
- **`npm audit --omit=dev --json` (2 HIGH, 0 CRITICAL, 0 MODERATE, 0 LOW):**
  - `electron` (`33.2.0` <= `42.11.15`): 46 Chromium/Electron advisories (including HIGH `GHSA-532v-xpq5-8h95`, `GHSA-8337-3p73-46f4`, `GHSA-jjp3-mq3x-295m`, `GHSA-9wfr-w7mm-pc7f`, etc.). Fix: upgrade `electron` to `45.5.0` (`>=43`).
  - `extract-zip` (`<=2.0.1`, via `electron`): HIGH `GHSA-jmr9-qjv8-65gv`, `GHSA-7pqw-9j4j-h8q3`. Fix: resolved by upgrading `electron`.
- **`npm audit --json` (43 total: 5 CRITICAL, 22 HIGH, 10 MODERATE, 6 LOW):**
  - **5 CRITICAL (dev only):** `underscore <=1.13.7` (via `jspot` -> `nomnom` -> `po2json@0.4.5`, `fixAvailable: false`), `tar <=7.5.20` (via `electron-builder@25.1.8`, fixable via `electron-builder@^26.18.0`).
  - **22 HIGH (2 prod + 20 dev):** `electron` + `extract-zip` (prod); `electron-builder` / `app-builder-lib` / `builder-util*` / `dmg-builder` / `cacache` / `make-fetch-happen` / `node-gyp` / `glob` (fixable via `electron-builder@^26.18.0`); `jsdoc` / `taffydb` / `linkify-it` / `markdown-it` / `postcss` / `sanitize-html` / `tmp <=0.2.5` (via `gulp-jsdoc3`, `fixAvailable: false`); `moment <=2.30.1` (via `jspot`, `fixAvailable: false`).
  - **10 MODERATE & 6 LOW:** `@babel/runtime`, `@electron/universal`, `ajv`, `bn.js`, `gulp-jsdoc3`, `ink-docstrap`, `ip-address`, `minimatch`, `pbkdf2`, `qs`, `@eslint/plugin-kit`, `eslint`, `browserify-sign`, `create-ecdh`, `crypto-browserify`, `elliptic`.
- **Expected Post-Phase-1 Audit State:** `npm audit --omit=dev` -> **0 vulnerabilities**; `npm audit` -> **0 critical / 0 high** (only optional `yarn.lock` removal/cleanup if unused).

---

## 3. Classification of Open Upstream GitHub Issues (90) & PRs (12)

Queried from `https://api.github.com/repos/sozi-projects/Sozi/issues?state=open&per_page=100` (102 total open items = **90 issues + 12 PRs**).

### 3.1 Critical Bugs (12 Issues)

| # | Title | Area & Root Cause / Impact | Target Phase |
|---|---|---|---|
| **#722** | Video/PPTX/PDF export is not working in source code compiling | `ipcRenderer.sendTo` removed in Electron 28+ (`exporter-preload.js:17,22,45,50`) causes all exports to hang indefinitely. | **Phase 1** |
| **#688** | Sozi json is not stable over load/save iterations induce presentations defect after several cycles | Floating-point drift during repeated `Frame.fromStorable` -> `setAtElement` -> `toStorable` cycles. | **Phase 2** (regression tests) / **Phase 4** (precision fix) |
| **#650** | Video export is not working | Export reports success or fails opaquely when `ffmpeg` execution produces no output. | **Phase 2** (safe missing-binary handling) / **Phase 4** |
| **#604** | Sozi can't export into video and fail silently | Missing/incompatible `ffmpeg` binary fails without clear GUI error message. | **Phase 2** (no crash) / **Phase 4** |
| **#637** | Export produces videos with black background when editor shows white background | Video capture renders transparent page over black video background. | **Phase 4** |
| **#716** | Sozi loses/disregards transparency on PNG image sequence export | PNG sequence export drops SVG alpha transparency. | **Phase 4** |
| **#753** | Sozi is having performance issues for longer presentations | Severe lag on 100+ frames / large SVGs (targeted by PR `#759`). | **Phase 4** |
| **#483** | Sozi is extremelly slow with a large number of layers | Severe slowdown with ~270 layers as frames increase (partially addressed by PR `#759`). | **Phase 4** |
| **#618** | relative zoom % animations do not work with button events | Clicking SVG hyperlink buttons jumps with fade instead of playing relative zoom % transitions. | **Phase 4** |
| **#491** | Can't move your frame when you use buttons thats big enough | Large clickable SVG `<a>` elements intercept drag/pan events (`SVGDocumentWrapper.js:124`). | **Phase 4** |
| **#365** | Presentation not consistent at each iteration when looping | Overlay layers disappear and text scale drifts after looping. | **Phase 4** |
| **#505** | Clipping is activated automatically on some layers | Unexpected automatic activation of layer clipping state. | **Phase 4** |

### 3.2 Compatibility Issues — Electron, OS, Inkscape/SVG, Browsers, Docs (15 Issues)

| # | Title | Category | Notes & Target Phase |
|---|---|---|---|
| **#670** | [Windows] Opening .SVG file creates never-ending cascade of "Document was changed. Reloading." | Windows `fs.watch` | Debounce/mtime check needed for `fs.watch` in Main process (`Phase 1` / `Phase 4`). |
| **#661** | Sozi for Apple Silicon (M1, M2, etc) ARM64 processor based macOS | macOS ARM64 | Ensure CI/packaging supports `arm64`/`universal` macOS targets (`Phase 2`). |
| **#366** | ARM version | Linux ARM | Packaging target consideration (`Phase 4`). |
| **#718** | Layers not showing in Sozi? | Inkscape 1.4 / SVG | Investigate Inkscape 1.2–1.4 SVG incompatibilities (alongside `InkscapeHandler.js:54` `pagecolor` fix in `Phase 1`). |
| **#479** | Preview in Sozi different from SVG and HTML-export | SVG / Editor | Editor preview vs raw SVG/HTML rendering discrepancy (`Phase 4`). |
| **#398** | Problem with text elements | Inkscape `<text>` | Frames fitted to `<text>` elements drift on save/reload (`Phase 4`). |
| **#506** | Different color display in Sozi on Ubuntu | Linux Color | Related to `enableColorCorrectRendering` (`index-electron.js:55`). |
| **#336** | Sozi 15 ignores screen dpi, interface hard to read on hidpi | Linux HiDPI | Modern Electron improves fractional/X11 scaling (`Phase 1`). |
| **#608** | Minor display issue when choosing SVG | Windows Dialog | Repaint behind native file dialog (`Phase 1` modern Electron + Main dialog). |
| **#456** | Cross-browser embedded videos | Browsers | `<foreignObject>` `<video>` positioning in Chromium/WebKit vs Firefox (`Phase 4`). |
| **#499** | Scroll problem with sozi embedded | Browsers `<iframe>` | Wheel events swallowed inside `<iframe>` (`Phase 4`). |
| **#680** | Unable to control my embedded Sozi form outside the `<iframe>` | Browsers / Docs | Cross-origin `postMessage` vs docs mismatch (`Phase 4`). |
| **#474** | Add support for layers in documents created with Draw.io | SVG Tools | Draw.io group layer detection (`Phase 4`). |
| **#301** | Add support for layers in documents created with SVG-edit | SVG Tools | SVG-edit `<g><title>...</title></g>` handler (`Phase 4`). |
| **#731** | Documentation: Installing on Windows needs an update | Docs | Website/README Windows artifact names (`Phase 1` README / `Phase 4` extended docs). |

### 3.3 Feature Requests & UX Enhancements (63 Issues Grouped by Area)

- **CLI, Automation & Packaging (9):** `#792` (FlatHub), `#786`/`#519` (explicit HTML title), `#785`/`#313` (output directory), `#784`/`#355` (independent `.sozi.json` filename), `#783`/`#585` (CLI video/JSON export), `#513` (Linux repo), `#384` (browser local SVG save), `#318` (single-file generation).
- **Export & Media (11):** `#717` (zero-padded PNG sequence), `#713` (embed HTML), `#707` (`.odp` export), `#659` (fullscreen video button), `#622` (PPTX speaker notes), `#185` (YouTube embed), `#184` (edit video URL), `#128` (auto video/audio dimensions), `#127` (poster attribute), `#115` (autoplay), `#82` (video controls visibility).
- **Editor UX & Workflow (22):** `#709` (keyboard frame reorder), `#600` (auto-frame layer), `#592` (Inkscape multipage), `#557` (rich notes formatting), `#524` (unsaved quit warning), `#489` (corrupted JSON recovery), `#480` (opacity staircase frames), `#477` (`Ctrl+Z` in text fields), `#449` (grid/guides), `#444` (edit hidden elements list), `#431` (intermediate frames), `#406` (more preferences), `#400` (frame arrangement), `#362` (out-of-flow frames), `#340` (per-slide element hiding), `#339` (drag-and-drop frame reorder), `#305` (drag-and-drop SVG open), `#304` (linked images in editor), `#262` (timeline thumbnails).
- **Player, Presenter & Transitions (21):** `#748` (frame number font size), `#697` (disable animations on out-of-sequence jumps), `#668` (teleprompter speaker notes), `#662` (auto-stop after last frame), `#638` (curved path turn direction), `#470` (presenter fullscreen button), `#408` (keyboard nav permissions), `#375` (jump-to-frame transition), `#307` (touchscreen improvements), `#265` (disable F5 refresh), `#225` (parallax scrolling), `#196` (play/stop controls), `#192` (per-link transitions), `#183` (total page count), `#181` (overview mode), `#177` (per-step attribute changes), `#168` (presenter stopwatch), `#144` (skip frame / blank jump), `#125` (drawing mode), `#123` (permanent frame list), `#77` (fixed-point rotation).

### 3.4 Complete Open PRs Inventory (12 PRs)

| PR | Title | Author | Mergeable | Files Touched | Adoption Verdict & MPL-2.0 Compatibility |
|---|---|---|---|---|---|
| **#788** | Electron: fall back to the main process cwd when PWD is not set | `tebbiworld` | `true` (`clean`) | `src/js/backend/Electron.js (+3/-3)` | **ADOPT (Adapt in Phase 1).** Fixes Windows `PWD` crash (`process.cwd()` fallback in Main IPC, not `@electron/remote`). MPL-2.0 compatible. |
| **#787** | Editor: add a toolbar button to open another SVG file | `tebbiworld` | `true` (`clean`) | `Controller.js`, `AbstractBackend.js`, `Electron.js`, `Toolbar.js`, `messages.pot` | **ADOPT IN PHASE 4 (UX).** Clean toolbar Open button + save prompt + HTML-escaped error notification; adapt to `window.soziElectron` + add `id.po` strings. MPL-2.0 compatible. |
| **#759** | Substantial performance boost upgrades over 6 files | `Foly93` | `false` (`dirty`) | `Controller.js`, `Presentation.js`, `Selection.js`, `Camera.js`, `Timeline.js`, `VirtualDOMView.js`, `messages.pot`, `yarn.lock` | **ADOPT CAREFULLY IN PHASE 4.** Addresses `#753`/`#483` (`Set.has`, Camera dirty tracking, Timeline VDOM keys/RAF). Cherry-pick code only (drop `yarn.lock`/`messages.pot`) and verify with Phase 2 tests. MPL-2.0 compatible. |
| **#789** | Editor: open LibreOffice Draw documents (.odg) directly | `tebbiworld` | `true` (`clean`) | `OdgImport.js (+561)`, `Electron.js`, `Controller.js`, `Toolbar.js`, `package.json` (`jszip`) | **DEFER / OPTIONAL (Phase 4+).** Large feature requiring external headless LibreOffice + `jszip` (`MIT / GPLv3` dual-licensed — MIT option is MPL-2.0 compatible). |
| **#641** | Elapsed time | `Ichigo85` | `false` (`dirty`) | `Controller.js`, `Properties.js`, `it.po` | **OPTIONAL (Phase 4 UX).** Re-implement cleanly if desired; drop stale `it.po` diff. MPL-2.0 compatible. |
| **#536** | Sync HTML with an audio track, narrating the presentation | `momeni` | `true` (`clean`) | `gulpfile.js`, `Storage.js`, `Presentation.js`, `narrated.js`, `Properties.js`, `Toolbar.js`, `narrated.html` | **REJECT / SKIP.** Coupled to removed `gulpfile.js` and hardcoded `narrative.flac` flow. |
| **#364** | Hidden frame | `s-leroux` | `false` (`dirty`) | `js/player/Player.js` | **REJECT / SKIP.** Obsolete 2016 PoC against legacy `js/` layout. |
| **#782**, **#770**, **#769**, **#767**, **#738** | Dependabot bumps (`electron` 44.7.0, `electron-builder` / `tar` / `app-builder-lib` / etc.) | `dependabot[bot]` | `true`/`false` | `package.json`, `package-lock.json`, `yarn.lock` | **SUPERSEDED by Phase 1.** Phase 1 upgrades `electron@^45.5.0` and `electron-builder@^26.18.0` alongside required API/security fixes. |

---

## 4. Risk Register & Mitigation Strategies

| ID | Risk | Likelihood / Impact | Mitigation |
|---|---|---|---|
| **R1** | Removing `nodeIntegration` / `@electron/remote` breaks `Storage.js` (`path`, `nunjucks` template loading), `Toolbar.js` (`package.json`), `Properties.js` (`File.path`), or `Electron.js` (`fs.watch`, dialogs). | High / High | Audit every Node built-in in `src/js/`; bundle `editor.js` via `esbuild`; pass template strings or precompiled templates cleanly; replace `File.path` with `webUtils.getPathForFile` or Main IPC dialog; run Playwright Electron smoke tests across save/reload/export flows. |
| **R2** | Electron 45+ breaks `exporter-preload.js` (`sendTo`), `printToPDF` (`marginsType`), or CDP `Page.captureScreenshot` window visibility. | High / High | Move export orchestration into Main process, replace `sendTo` with `ipcRenderer.send`, use `margins: { marginType: "none" }`, and verify PDF generation + non-crash missing-`ffmpeg` behavior in Phase 2 E2E tests. |
| **R3** | Windows `cmd.exe` breaks `.bin\*.cmd` shims and `postinstall` scripts on directory paths containing `&` or spaces (`D:\02_Development & Coding\sozi`). | High / Med | Eliminate nested `npm install --only=prod` inside `build/electron` by bundling renderer/main scripts with `esbuild`; invoke `node scripts/build.mjs` / Node tooling reliably in npm scripts and CI. |
| **R4** | `InkscapeHandler.js:54` crashes on Inkscape 1.2+ `#RRGGBBAA` `pagecolor` attributes, blocking modern SVG imports. | High / High | Update regex/color parser in `InkscapeHandler.js` to support `#RRGGBB`, `#RRGGBBAA`, `#RGB`, and safe fallback, and cover with unit + E2E tests using a real Inkscape 1.2+ SVG fixture. |
| **R5** | `.sozi.json` schema drift or legacy Sozi <=13 SVG/JSON migration regressions (`upgrade.js`, `Presentation.js`). | Med / High | Preserve exact `.sozi.json` schema and `toMinimalStorable()` output; lock behavior with unit + E2E tests using legacy Sozi 13 SVG and pre-17.02.05 `.sozi.json` fixtures before refactoring. |
| **R6** | Cross-platform packaging/CI differences (Linux AppImage/FUSE, Windows NSIS, macOS code-signing/zip). | Med / Med | Run `.github/workflows/ci.yml` on `ubuntu-latest`, `windows-latest`, and `macos-latest` on fork `djetli035-ctrl/Sozi` with Xvfb/headless Electron support and unsigned CI packaging flags where needed. |
| **R7** | MPL-2.0 compliance violation from unattributed headers or incompatible dependencies. | Low / High | Add missing MPL-2.0 headers to the 5 `src/js/` files in Phase 1; replace `LGPL-2.0-or-later` `po2json` with MIT `gettext-parser`; document any adapted external code/MIT sources. |

---

## 5. Prioritized Execution Roadmap

1. **Phase 0 — Architecture & Plan (Current Milestone M0 — Complete):**
   - Configure `origin` (`djetli035-ctrl/Sozi`) and push-disabled `upstream` (`sozi-projects/Sozi`), capture read-only baseline audit/lint/build results on Node `v24.11.0`, author `ARCHITECTURE.md` & `MODERNIZATION_PLAN.md` on `phase-0/architecture-and-plan`.
2. **Phase 1 — Toolchain, Electron Security, Build Migration & Core Fixes (Milestone M1, branch `phase-1/toolchain-and-security`):**
   - Upgrade `electron` (`^45.5.0`) and `electron-builder` (`^26.18.0`); eliminate `npm audit --omit=dev` HIGH/CRITICAL CVEs.
   - Replace Gulp 5 / Browserify / Babel / Uglify / `jspot` / `po2json` / `csslint` / `gulp-jsdoc3` / `modclean` / `node-glob` / `core-js` with `esbuild` + `scripts/build.mjs` + `gettext-parser` + `stylelint` + explicit `@eslint/js` flat config.
   - Harden Electron (`nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`, `preload-editor.js` `contextBridge` `window.soziElectron`, remove `@electron/remote`).
   - Move exporters to Main process, replace `ipcRenderer.sendTo` (`#722`) and `marginsType: 1`, fix `exportToVideo` loop `return;` bug (`index-electron.js:484`), ensure missing `ffmpeg` returns a clean error without crashing.
   - Fix Windows `process.env.PWD` (`#788`) & `npm start` `argv` handling, fix `InkscapeHandler.js:54` `#RRGGBBAA` `pagecolor` regex, fix the 6 `eslint` `jsdoc/require-jsdoc` errors, add MPL-2.0 headers to the 5 missing `src/js/` files, and update `README.md` install/build/run instructions.
3. **Phase 2 — Automated Testing, PDF/ffmpeg Verification & Cross-Platform CI (Milestone M2 + E2E Track, branch `phase-2/tests-pdf-ci`):**
   - Add unit tests (`node:test`) covering `Presentation`, `Frame`, `LayerProperties`, `CameraState` SVG CTM math, `Timing`, `.sozi.json` serialization, `upgradeFromSVG` / `upgradeFromStorable`, and `InkscapeHandler` `pagecolor`.
   - Add Playwright Electron + Chromium E2E smoke tests covering repo fixtures, a custom modern Inkscape 1.2+ SVG (3+ frames -> save `.sozi.json`/`.sozi.html` -> HTML zoom playback), a legacy Sozi 13 SVG/JSON presentation, valid multi-page PDF export, and missing-`ffmpeg` crash safety.
   - Configure `.github/workflows/ci.yml` on `ubuntu-latest`, `windows-latest`, and `macos-latest` running lint, unit tests, E2E tests, and `electron-builder` packaging on `djetli035-ctrl/Sozi`.
4. **MANDATORY STOP FOR USER REVIEW:** Stop after Phase 2 is verified green locally and on GitHub Actions (`djetli035-ctrl/Sozi`) before beginning Phase 3 or Phase 4.
5. **Phase 3 — Post-Review Hardening & Release Readiness (Deferred):**
   - Address user review feedback, remaining cross-OS packaging polish (macOS ARM64 `#661`, Windows `fs.watch` cascade `#670`, `.sozi.json` float drift `#688`), and dependency cleanup.
6. **Phase 4 — Feature, i18n, UX & Video Export Polish (Deferred — see Section 7):**
   - Complete Indonesian localization (`locales/id.po` + templates), extended documentation, editor UX PRs/fixes, and full video export / `ffmpeg` automation.

---

## 6. Architectural Decisions & Rationale

| Decision | Choice | Rejected Alternative | Rationale |
|---|---|---|---|
| **Build & Bundler** | `esbuild` + `scripts/build.mjs` + `gettext-parser` (MIT) + `nunjucks` | Keep Gulp 5 + Browserify + Babel + Uglify + `jspot` + `po2json` | Eliminates 15 unmaintained/vulnerable dev packages (including all 5 CRITICAL & 20 dev HIGH CVEs and `LGPL-2.0-or-later` `po2json`), removes Windows `opencollective-postinstall` inside `build/electron`, and compiles `editor.js`, `player.js`, `presenter.js`, and `locales.js` in <1s with modern ESM/ES2022+ support. |
| **CSS & JS Linting** | Native ESLint 9 flat config (`@eslint/js` + `eslint-plugin-jsdoc`) + `stylelint` | `FlatCompat` + `csslint@1.0.5` | Removes undeclared `@eslint/eslintrc` hoisting reliance, supports modern JS syntax (`ecmaVersion: "latest"`), and replaces abandoned 2016 `csslint` (which fails on modern CSS variables/grid). |
| **Electron IPC & Security** | `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`, `preload-editor.js` (`contextBridge`), Main `ipcMain.handle` | Keep `@electron/remote` or `nodeIntegration: true` | Aligns with modern Electron security guidelines, removes unmaintained `@electron/remote` IPC surface, and keeps loaded third-party SVG content isolated from Node/FS APIs. |
| **Exporter Orchestration** | Main-process `exportToPDF`/`PPTX`/`Video` + `exporter-preload.js` using `ipcRenderer.send` + `margins: { marginType: "none" }` | Renderer-driven `remote.BrowserWindow` + `ipcRenderer.sendTo` | Fixes Electron 28+ `sendTo` removal (`#722`), removes deprecated `marginsType: 1`, keeps `pdf-lib`/`officegen`/`tmp`/`child_process` strictly in Main, and avoids shipping Node integration in export windows. |
| **Test Stack** | `node:test` (unit) + Playwright (`_electron` + Chromium) (E2E) | Jest/Mocha + Spectron (archived) | Zero extra heavy unit-test dependencies (`node:test` is built into Node 24 LTS); Playwright natively supports Electron launch + standalone HTML playback verification in Chromium across Linux/Windows/macOS. |

---

## 7. Explicit Phase 4 Targets (Deferred Until After Phase 2 User Review)

The following items are **explicitly out of scope for Phases 0–2** and recorded here for Phase 4 execution after user review:

1. **Complete Bahasa Indonesia Localization (`locales/id.po` & Templates):**
   - Clean up the **6 `#~` obsolete entries** in `locales/id.po:773-793`.
   - Fix existing typos and inconsistent terminology in `locales/id.po`:
     - `line 93`: `"Berkas tidak ditemukan: '%s."` -> remove stray `'`.
     - `line 129`: `"Setel ulang layer geometri"` -> natural phrasing (`"Setel ulang geometri layer"`).
     - `line 288`: `"Ctrl+>t: Entri daftar indentasi"` -> fix `Ctrl+>t` typo (`Ctrl+>`).
     - `line 292`: `"Ctrl+<: Entri daftar Outdent"` -> translate `"Outdent"` consistently.
     - `line 333`: `"Aktifkan tengat waktu"` -> fix `"tengat"` -> `"tenggat"` / align with `line 329` (`"Batas waktu"`).
     - `line 511`: `"Pilih frame 2,5 dan 12: \"2, 5, 12\""` -> add missing space (`2, 5,`).
     - Standardize register (`Anda` vs `anda` at `lines 81, 116, 727, 742, 747`) and technical glossary (`zum`/`Zoom`, `frame`, `layer`, `Rendering`, `Ease*`, `path`, `klip`).
   - Add Indonesian translations for all new UI/error strings introduced in Phases 1–4.
   - Internationalize the **5 hardcoded English strings** outside `locales/`:
     - `src/templates/player.html:131` (`title="Click to show the frame list"`)
     - `src/templates/presenter.html:89` (`title="Previous"`) & `line 91` (`title="Next"`)
     - `src/js/player.js:194` (`" (Paused)"`)
     - `src/js/presenter.js:87` (`alert("Could not open presentation window...")`)
2. **Extended Documentation:**
   - Developer architecture/contribution guide, presenter/player embedding docs (`#680`, `#638`), JSDoc generation replacement for `gulp-jsdoc3`, and user manual updates beyond Phase 1's `README.md` install/build/run refresh.
3. **Editor UX & Performance Improvements:**
   - Adopt PR `#787` (toolbar button to open another SVG file) adapted to `window.soziElectron` IPC.
   - Cherry-pick PR `#759` (`#753`/`#483` large-presentation/layer performance optimizations) with unit/E2E verification.
   - Evaluate PR `#641` (elapsed presentation time) and PR `#789` (`.odg` import).
   - Replace deprecated `document.execCommand(...)` in `src/js/view/Properties.js:638-659` (`#557`) and npm-deprecated `fork-awesome@1.2.0` (`package.json:23`) with a maintained icon set / inline SVG icons, plus `#524` (unsaved quit prompt) and `#748` (configurable frame number size).
4. **Full Video Export & `ffmpeg` Automation:**
   - Fix video/PNG background transparency bugs (`#637`, `#716`), zero-pad exported PNG frame sequence filenames (`#717`, `index-electron.js:492`), improve CLI/UI `ffmpeg` error reporting (`#604`, `#650`), and automate fetching/bundling cross-platform `ffmpeg` binaries (`linux-x64`, `linux-arm64`, `win32-x64`, `darwin-x64`, `darwin-arm64`) in packaging scripts/CI.
