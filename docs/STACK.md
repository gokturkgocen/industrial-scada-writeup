# Stack

Every dependency, why it was chosen, what was rejected. Prefixes:
`(server)` runs in the Node.js process; `(desktop)` runs in the Tauri /
browser bundle.

## Runtime

| Library | Where | Why |
|---|---|---|
| Node.js 24 LTS | server | LTS; pure-JS deps run unchanged on macOS / Windows / Linux |
| Rust (stable) | desktop build | Tauri toolchain; not shipped in the binary, only required at build time |
| TypeScript 5 (strict) | both | No `any`; `noUncheckedIndexedAccess`; `noImplicitOverride`; `noFallthroughCasesInSwitch` |

## Server

| Library | Why | Rejected |
|---|---|---|
| `modbus-serial` | Mature Modbus TCP/RTU client; supports the gateway-bridged TCP framing we need | `node-modbus` (less maintained); building on raw sockets (reinvents framing) |
| `ws` | Battle-tested WebSocket server | `socket.io` (we don't need rooms / fallbacks / namespaces) |
| `better-sqlite3` | Synchronous, fast, zero-config; perfect fit for in-process, on-disk time-series with WAL mode | `sqlite3` (callback API, slower); Postgres / Timescale (extra deployment surface for no value at this scale) |
| `pino` | Low-overhead JSON logger; structured fields integrate cleanly with audit log lines | `winston` (heavier); `console.log` (no structure) |
| `tsx` | TypeScript dev runner with watch mode; lets us run `src/*.ts` directly without a separate build step in dev | `ts-node` (slower startup); compile-then-run (slower iteration) |

## Desktop

| Library | Why | Rejected |
|---|---|---|
| Tauri 2 | ~10–20 MB binary, host WebView (no bundled Chromium), good packaging story for macOS + Windows | Electron (100+ MB, heavier memory); Wails (less mature for our needs); pure web app (operators want a Dock icon) |
| React 18 | Default for this kind of UI; large component pool; predictable mental model | Vue / Svelte (same reasons but smaller ecosystem); SolidJS (faster but less idiomatic for the team) |
| Vite 5 | Fast dev server, native ESM, zero-config TS | Webpack (slower, configs are an industry); Parcel (less ergonomic for plugins) |
| Zustand | Small, hooks-only state; perfect fit for "one big store + slices" | Redux (boilerplate); Jotai / Recoil (overkill for a single-store app); Context (re-render thrash on high-frequency updates) |
| IBM Plex Sans / Mono | Tabular numerics that don't jitter; brand-adjacent typographic personality | Inter alone (no matching mono); JetBrains Mono (great for code, less ideal for ambient UI numerics) |

### Things deliberately *not* in the stack

- **No CSS framework (Tailwind, Bootstrap, etc.)** — semantic CSS with
  custom-property tokens fits a multi-component design system better than
  utility classes.
- **No chart library** — ~120 LOC of canvas drawing is cheaper than
  bundling Recharts / Chart.js for six sparklines.
- **No i18n library** — a flat dictionary + a 12-line `useT()` hook
  handles two languages without runtime overhead.
- **No router** — the app has 4 top-level views; a 6-line state machine
  in `App.tsx` is enough.
- **No data-fetching library (SWR/React Query)** — there's no REST API
  to fetch from; everything flows through the WebSocket store.
- **No form library** — the UI is read-only; the dormant write path uses
  per-field draft state and a manual save bar.

## Build & ship

| Tool | Where | Why |
|---|---|---|
| `npm` | both | Default; lockfile committed |
| `tsc --noEmit` | both | Type-check only; emit handled by Vite (desktop) and `tsx` (server dev) / `tsc -p` (server prod) |
| `tauri build` | desktop | Produces `.dmg` (macOS Apple Silicon) and `.msi` (Windows) — the latter when run from a Windows host |
| `tauri icon` | desktop | Generates the platform-specific icon set from a single SVG |

## Deployment shape (intended)

| Component | Where it runs | How it starts |
|---|---|---|
| `scada-server` | one factory PC per site | systemd unit (Linux) / launchd (macOS) / nssm or `node-windows` (Windows); auto-start on boot |
| `scada-desktop` | each operator's PC | Installer (`.dmg` / `.msi`); pinned to dock / start menu |
| Modbus gateway | DIN-rail enclosure on the bus | Off-the-shelf RS485↔Ethernet bridge in Modbus TCP server mode |

The server PC's `localhost:WS_PORT` is what the desktop hits during single-
machine dev; in production each desktop is configured (per-machine, in
`localStorage`) to hit the site server's static IP.
