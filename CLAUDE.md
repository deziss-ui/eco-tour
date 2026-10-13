# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language

Always reply to the user in Hebrew. Code, identifiers and commit messages stay in English.

## What this is

Enviro Audit is a Hebrew (RTL) mobile PWA for environmental site inspections ("tours"). The user records findings with photos on the phone, then exports a Word (.docx) report. All data stays on the device; nothing is uploaded. It is hosted on GitHub Pages and installed from Chrome on Android (see README.md).

## Development

There is no build step, package manager, linter or test suite. The files are served as-is.

- Run locally: serve the folder with any static server (e.g. `npx serve .` or `python -m http.server`) and open it in a browser. The service worker registers only over `https:`, so offline caching won't kick in on plain `http://localhost`.
- Deploy: push or upload the files to the GitHub Pages repo. The README's update flow relies on `sw.js` being network-first.
- When you change the list of shipped files, update `FILES` in `sw.js`. Bump `CACHE` only when you need to force-evict old caches.

## Architecture

- `index.html` holds the whole app: CSS, markup and one inline `<script>`. The script is split into sections marked `/* ---------- name ---------- */`: report builder (`TEMPLATES`, `SUMMARY_TOPICS`, `buildReport`), storage, capabilities, state, photos, templates, UI helpers, renderers (home / setup / tour / editor / annotator), export, events and boot.
- UI is plain string-template rendering. A global state object `S` (`view`, `tab`, `sheet`, `tours`, `settings`) and the current tour `T` drive `render()`, which dispatches to `renderSetup` / `renderTour` / `renderEditor` and so on. Clicks go through one delegated handler that reads `data-act` and other `data-*` attributes.
- Report templates (`TEMPLATES`) define per-client report layouts (`depot`, `contractor` for Tavel; `dan` for Dan). `mode: 'findings'` produces a findings table. `mode: 'checklist'` produces fixed category items marked OK/not OK. `buildReport` turns a tour into a docx using the global `docx` from `docx.js`.
- `docx.js` is a vendored, minified build of the `docx` library. Don't edit it by hand.
- Storage is IndexedDB, database `eco-tour`, with stores `tours`, `photos` (image blobs keyed by id) and `kv` (`settings`). An in-memory `mem` mirror serves as a fallback if IndexedDB fails. Backup/restore JSON is tagged `app: 'eco-tour'`.
  - **Do not rename these `eco-tour` identifiers** (the DB name, the backup `app` tag, and also the `CACHE` name in `sw.js`). The app was renamed from "eco-tour", but changing them would orphan users' saved tours or break restoring older backups.
- The app runs in two environments, chosen by `IN_CLAUDE` (`window.claude.use` exists):
  - Inside a Claude artifact, it uses the `downloads` and `sample` runtime capabilities for saving files and wording findings with AI. The service worker is skipped there.
  - As a standalone PWA, the optional Anthropic API key from settings is used for direct browser calls to `api.anthropic.com/v1/messages` (`apiSampler`), which expect a JSON reply.
- `LOGO_B64` (a base64 PNG embedded in the report) is a single very long line near the top of the script. Avoid printing it.
