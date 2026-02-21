# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Frontend development
yarn dev           # Start Vite dev server on port 1420
yarn build         # TypeScript type-check + Vite bundle to dist/
yarn preview       # Preview production frontend build

# Desktop app (requires Rust toolchain)
yarn tauri dev     # Run full Tauri app with HMR
yarn tauri build   # Package into native desktop app
```

No test or lint commands are configured.

## Yarn PnP

This project uses **Yarn 4 with Plug'n'Play** — packages are not unpacked into `node_modules`. If VSCode can't resolve types after `yarn install`, run:

```bash
yarn dlx @yarnpkg/sdks vscode
```

Then select "Use Workspace Version" when VSCode prompts for the TypeScript version.

## Architecture

This is a **Tauri 2 + React 19 + TypeScript** desktop app. Two separate runtimes communicate via Tauri's IPC bridge:

**Frontend** (`src/`): React app built by Vite. Calls into Rust using `invoke()` from `@tauri-apps/api/core`.

**Backend** (`src-tauri/src/`): Rust library (`grth_lib`) that defines Tauri commands with `#[tauri::command]`, registers them in `lib.rs`, and runs the event loop. `main.rs` is a thin entry point that calls `grth_lib::run()`.

**IPC pattern**: React calls `invoke("command_name", { args })` → Tauri routes to the matching `#[tauri::command]` Rust function → serialized result returned as a Promise. New commands must be both defined in Rust and registered in the `.invoke_handler()` call in `lib.rs`.

**Build flow**: `yarn tauri dev` starts the Vite dev server first, then Tauri launches the WebView pointed at `http://localhost:1420`. `yarn tauri build` runs `yarn build` to produce `dist/`, then Rust compiles and packages everything into a native binary.

Key config files:
- `vite.config.ts` — dev server port (1420), HMR port (1421), excludes `src-tauri/` from watching
- `src-tauri/tauri.conf.json` — app ID (`com.kplawver.grth`), window size, bundle targets
- `src-tauri/capabilities/default.json` — Tauri permission grants for the main window
