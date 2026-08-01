# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Velorn is a GPL-3.0 Electron desktop app: an AI video workstation built around a local ComfyUI server. React 18 + Vite + Tailwind + Zustand in the renderer, plain Node in the Electron main process. **No TypeScript, no test suite, no linter** — the quality gate is `npm run build` plus a manual smoke test of the area you changed (see `CONTRIBUTING.md`).

## Commands

```bash
npm install
npm run electron:dev          # normal dev path: vite on 127.0.0.1:5173 + electron
npm run dev                   # browser-only; most features need desktop APIs and will not work
npm run build                 # vite build -> dist/  (the pre-PR check)
npm run electron:build:win    # / :mac / :linux -> release/
npm run electron:pack         # unpacked build, faster for packaging checks
npm run starter-pack:build    # regenerate docs/workflow-starter-pack/ from the workflow registry
```

There is no way to "run a single test" — there are no tests.

## Naming rule (from AGENTS.md)

The product is **Velorn**. Never say "ComfyStudio" in user-facing text, MCP metadata, docs, or chat. `comfystudio` survives only as internal compatibility identifiers: the `appId` (`com.comfystudio.app`), the project file name `project.comfystudio`, the ComfyUI bridge node package, legacy `COMFYSTUDIO_*` graph markers, and backward-compatible MCP aliases. Leave those alone unless a migration is explicitly requested.

## Process architecture

Three layers, and knowing which one you're in determines what APIs exist:

1. **Main** (`electron/main.js`, ~6.5k lines) — all filesystem, dialog, path, settings (`userData/settings.json`), ffmpeg/ffprobe export, playback/proxy transcoding, the ComfyUI process launcher (`electron/comfyLauncher.js`), the Velorn Bridge installer, and the MCP server. Organized by `// IPC Handlers - <area>` section banners; grep those to find a subsystem.
2. **Preload** (`electron/preload.js`) — the entire main↔renderer contract lives here as `window.electronAPI.*`. Any new main-process capability needs a matching entry here or the renderer cannot reach it.
3. **Renderer** (`src/`) — React app, `@` aliased to `src/`.

### MCP server (the agent control layer)

`electron/mcpServer.js` (the largest file in the repo) runs an HTTP MCP server on `127.0.0.1:19790/mcp` from the main process, started in `main.js` at app ready. It gets project state two ways:

- **Snapshot push**: `src/services/mcpSnapshot.js` serializes project/timeline/asset state and pushes it via `mcp:updateSnapshot`. Read-only tools answer from this cached snapshot.
- **Action round-trip**: a tool calls `performMcpRendererAction()` in main, which emits `mcp:action` to the renderer; `startMcpActionBridge()` in `src/services/mcpActions.js` (~8k lines) dispatches to `runMcpAction(action, payload)` and replies over `mcp:actionResult`. Requests time out if the renderer doesn't answer.

So: **write tools live in `src/services/mcpActions.js`, tool schemas/registration live in `electron/mcpServer.js`.** Both sides must be edited together. Bump `MCP_ACTION_BRIDGE_VERSION` when the contract changes. Most write tools default to `previewOnly` and apply through the normal undo stack — preserve that pattern; see `docs/MCP.md` for the full safety model. `.mcp.json` at the repo root points a local client at the running app.

## Renderer state

Zustand stores in `src/stores/`, no context providers:

- `timelineStore.js` (~5.4k lines) — the heart of the app: tracks, clips, transforms, keyframes, transitions, effects, markers, undo/redo, multi-timeline. Most editing features start here.
- `projectStore.js` — project open/save/recents, resolution & fps presets, autosave.
- `assetsStore.js` — asset library shared across all timelines in a project.
- `frameForAIStore.js`, `generationMonitorStore.js`, `workflowsStore.js`, `musicPopoverStore.js` — small, single-purpose.

Big feature surfaces are single large components (`GenerateWorkspace.jsx` ~850KB, `Timeline.jsx`, `InspectorPanel.jsx`) with helpers in `src/services/` and pure logic in `src/utils/`.

## Project format on disk

Chosen by the user as a "projects folder"; each project is a directory:

```
MyProject/
├── project.comfystudio    # JSON: timelines, clips, assets, settings (legacy: project.storyflow)
├── assets/{video,audio,images}/
├── cache/                 # playback transcodes + render cache
├── renders/
└── autosave/
```

`src/services/fileSystem.js` owns read/write and the legacy-filename fallback. Assets are shared project-wide; tracks/clips/resolution/fps are per-timeline.

## ComfyUI integration

- `src/services/comfyui.js` — the `comfyui` singleton (queue/history/websocket progress/upload/download against `http://127.0.0.1:8188` by default, port configurable in Settings, loopback only) **plus ~40 `modifyXxxWorkflow(workflow, options)` functions**. Each built-in workflow gets one modifier that injects prompt/seed/image/audio/dimensions into that specific graph's node ids. Adding a workflow usually means adding a modifier here.
- `public/workflows/*.json` — the shipped ComfyUI API graphs. Vite copies them into `dist/` for the renderer (`getBundledWorkflowPath`); electron-builder also copies them to `resources/workflows` for the main process (`resolveBundledWorkflowDir` in `main.js`, which falls back to scanning when `src/config` is absent in packaged builds).
- `src/config/` — registries that describe those graphs to the UI: `workflowRegistry.js` (built-in list), `generateWorkflowCatalog.js` (Generate browser cards/routes), `workflowDependencyPacks.js` + `workflowInstallCatalog.js` (node/model preflight), `importedWorkflowRegistry.js` (community imports).

**Adding or changing a built-in workflow** (per `CONTRIBUTING.md`): update `workflowRegistry.js`, update `workflowDependencyPacks.js`, update dependent Generate UI, run `npm run starter-pack:build`, review the regenerated `docs/workflow-starter-pack/`.

### Custom user graphs

Users' own ComfyUI graphs are wired by **endpoint marker node titles** — `VELORN_INPUT_IMAGE`, `VELORN_PROMPT`, `VELORN_SEED`, `VELORN_WIDTH`, `VELORN_HEIGHT`, `VELORN_FPS`, `VELORN_DURATION`, `VELORN_AUDIO`, `VELORN_OUTPUT_IMAGE`, `VELORN_OUTPUT_VIDEO`. Scanning/validation is in `comfyui.js` (`scanUiWorkflowForCustomEndpoints`, `validateCustomKeyframeWorkflow`, `validateCustomVideoWorkflow`). Readable titles ("Velorn input image") and legacy `COMFYSTUDIO_*` titles must keep working. Missing endpoint = the graph controls that value itself.

The **Velorn Bridge** (`electron/comfyui-injected/comfystudio_bridge/`) is a ComfyUI custom node installed into the user's ComfyUI so a graph open there can be sent back into the right Velorn panel.

## Product guardrails

Keep these intact unless maintainers say otherwise: ComfyUI is local/loopback-only, LM Studio integration is local-only, Generate dependency preflight stays enabled, and the Starter Pack docs remain the supported bridge for advanced ComfyUI users.

## Releases

Releases go through GitHub Actions (`.github/workflows/release.yml`), never manual local packaging. Bump `package.json` version, write the release-notes doc, push to `main`, then push a `vX.Y.Z` tag; edit and publish the draft release. Read `docs/AI_RELEASE_HANDOFF.md` and `docs/RELEASE_PROCESS.md` before any release, packaging, tagging, or signing work; secret names are in `docs/CI_SECRETS.md`. Never commit certificates, `.p12` files, passwords, or API keys. Use throwaway tags to validate the automation.

## Other docs worth knowing

`docs/MCP.md` (agent tool guide), `PROJECT_SUMMARY.md` (long running feature/UX log — useful for "how is X supposed to behave", but dated and partly stale), `RELEASE_CHECKLIST.md`, `ROADMAP.md`, `docs/BACKLOG.md`, `docs/FEATURE_TRACKER.md`.
