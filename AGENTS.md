# AGENTS.md

## Project

Min — an Electron (v41.2.0) desktop browser focused on privacy. Two-process architecture:

- **Main process** (`main/`): window management, filtering, downloads, menus, IPC hub
- **Renderer process** (`js/`): browser UI loaded from `index.html`. Uses `nodeIntegration: true`, `contextIsolation: false`
- **Preload scripts** (`js/preload/`): injected into web content views with `contextIsolation: true`
- **Extensions** (`ext/`): bundled third-party code (filter lists, readability, public suffixes)

## Prerequisites

- Node 15.7.0 (`.nvmrc`) for local dev; CI uses Node 25
- macOS: Xcode + command-line tools; may need `SDKROOT` set
- Windows: Visual Studio; may need `npm config set msvs_version 2019`

## Build

**Build is required before running.** The app entrypoint is the built artifact `main.build.js`, not a source file.

```
npm run build          # buildMain → buildBrowser → buildBrowserStyles → buildPreload
```

Each build step concatenates source files (plus browserify for the renderer bundle):

- **buildMain**: concatenates localization + `main/*` + `js/util/settingsMain.js` + `js/util/proxy.js` into `main.build.js`
- **buildBrowser**: concatenates localization + `js/default.js`, then runs browserify with `electron-renderify` transform. Excludes `chokidar` and `write-file-atomic`. Output: `dist/bundle.js`
- **buildBrowserStyles**: concatenates all `.css` files into `dist/bundle.css`
- **buildPreload**: concatenates `js/preload/*` into `dist/preload.js`
- **buildLocalization** (called by buildMain and buildBrowser): compiles `localization/languages/*.json` into `dist/localization.build.js`

All build outputs go to `dist/`, which is gitignored.

## Dev workflow

```
npm install              # runs setupDevEnv.js (electron fuses + macOS ARM codesigning)
npm run start            # build + watch + launch electron in development mode
```

During dev, file watchers do incremental rebuilds per area (main → js → preload → css). After changes, press `alt+ctrl+r` (Mac: `opt+cmd+r`) to reload the UI.

## Lint & format

```
npm test                 # standard --verbose (JS linter only — no test framework exists)
npm run lint             # prettier --write (CSS/MD/HTML/JSON) + standard --fix
```

- **Standard JS** (feross/standard) for JavaScript. Globals declared in `package.json` `standard.globals`: `l`, `tabs`, `tasks`, `globalArgs`, `platformType`, `throttle`, `debounce`, `empty`, plus browser globals
- **Prettier** for CSS, Markdown, HTML, JSON. `ext/` is excluded from prettier.

## No test framework

There are no unit tests in this repo. `npm test` only runs the JS linter. The `standard` globals config prevents false positives for globals set on `window` at runtime.

## Key conventions

- **`console.debug`** for logging, never `console.log`
- Build concatenates files in order; imports rely on browserify `require()` for the renderer bundle and simple script-order concatenation for main/preload
- Localization strings use the `l(stringId)` helper function. The `l` global is defined in `localization/localizationHelpers.js`
- `settings` is a global available in both main and renderer (separate implementations: `js/util/settings/settings.js` for renderer, `js/util/settings/settingsMain.js` for main)
- Native module rebuild is disabled (`npmRebuild: false`) due to Electron 32+ incompatibility with nan


## CI

Manual dispatch only (`workflow_dispatch` in `.github/workflows/build-packages.yml`). No automatic PR/commit CI.
