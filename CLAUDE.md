# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal fork (`mcruzdev/reveal.js`) of reveal.js, the HTML presentation framework. Besides the upstream code, it carries the user's own presentations: `quarkus-flow-workflow.html` at the root and exported PDFs in `pdfs/` (linked from the README "Slides" section).

## Commands

- `npm start` — Vite dev server on port 8000 (override with `npm start --port=8001`). Serves `index.html` / `demo.html` / presentation HTML files directly from the repo root; source is aliased so no build step is needed during development.
- `npm test` — runs all QUnit suites (`test/*.html`) headlessly via Puppeteer against a temporary Vite server on port 8009.
  - To run a single suite, the simplest path is `npm start` and open `test/test-<name>.html` in a browser; `scripts/test.js` globs all of `test/*.html` and has no filter flag.
- `npm run build` — TypeScript check (`tsc`), then Vite builds for core JS, styles/themes, and each plugin into `dist/`. `npm run build:core` skips the plugins.
- Node >= 20.19.0 is enforced via `devEngines`.

React wrapper (`react/`, separate npm package `@revealjs/react`):
- `npm run react:test` / `npm run react:build` / `npm run react:demo` (or `npm <cmd> --prefix react`). Tests use Vitest, colocated as `*.test.tsx`.
- **Read `react/AGENTS.md` before changing anything under `react/`** — it documents the Deck lifecycle, sync/configure policies, and which tests/docs must be updated alongside behavior changes.

## Architecture

### Core (`js/`)

Mixed JS/TS, mid-migration to TypeScript. The build entry is `js/index.ts`, which wraps the real implementation and exposes two APIs:

- **Multi-instance API**: `new Reveal(element, options)` — the actual implementation.
- **Legacy singleton API**: `Reveal.initialize(options)` — a backwards-compat shell in `js/index.ts` that queues pre-init calls (`on`, `configure`, etc.) and replays them once initialized.

`js/reveal.js` (~3000 lines) is the heart: a factory function that owns deck state (current slide indices, scale, overview/paused flags) and orchestrates everything else. Functionality is split into ~19 controllers in `js/controllers/` (keyboard, fragments, autoanimate, scrollview, printview, backgrounds, location/URL handling, plugins, etc.). Each controller is a class instantiated in `reveal.js` and handed the `Reveal` API object; `reveal.js` assembles the public API and dispatches both DOM events and instance events.

Config lives in `js/config.ts` (`defaultConfig` + the `RevealConfig` type). Ambient types are in `js/reveal.d.ts`; shared helpers in `js/utils/`.

### Plugins (`plugin/`)

Each built-in plugin (highlight, markdown, math, notes, search, zoom) is a directory with `plugin.js` (implementation), `index.ts` (entry), and its own `vite.config.ts`. The root `npm run build` builds each one separately into `dist/plugin/`. Plugins register via `Reveal.registerPlugin` / the `plugins` config array and are managed by `js/controllers/plugins.js`.

### Styles (`css/`)

SCSS. `css/reveal.scss` is the core stylesheet; themes live in `css/theme/*.scss` and build on `css/theme/template/`. `vite.config.styles.ts` auto-discovers every `css/theme/*.scss` file as a build entry, so adding a theme file is all that's needed for it to be built into `dist/theme/`.

### Tests (`test/`)

Each `test/test-*.html` file is a self-contained QUnit page that loads a real presentation and asserts against the live API. New tests for core behavior go in a new or existing HTML file there — there is no JS-only unit test setup for the core (the React wrapper is the exception, using Vitest).

### Path aliases

Vite aliases `reveal.js` → `/js`, `reveal.js/plugin` → `/plugin`, and `reveal.css` → `/css/reveal.scss` (see `vite.config.ts`), matching the published package's export map. HTML files in the repo import via these specifiers and work in dev without building.
