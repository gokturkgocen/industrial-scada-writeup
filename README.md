# Robodor SCADA — desktop viewer for multi-card factory monitoring

> Portfolio writeup. The implementation is closed-source; this repository is
> a self-contained description of what was built, why, and how. No source
> code, register map, database, customer data or deployment configuration is
> included.

A native desktop SCADA application for [Robodor](https://www.robodor.com)
industrial door controllers (RF15 / RF30 family), monitoring **up to ~100
cards** across one or more factory bays over a Modbus TCP / RS485 network.

Built end-to-end as a two-process system: a headless Node.js polling
service and a Tauri-based native client (macOS & Windows), both speaking a
versioned WebSocket protocol.

I designed and built the architecture, polling/service layer and operator UI
as a focused internal product prototype, then shaped it toward production
review with simulator-based validation and redaction-safe documentation.

---

## Public Evidence

This repository is not a source mirror. It documents the parts a reviewer can
evaluate safely:

- **Architecture:** single polling source, WebSocket fan-out, snapshot diffing,
  SQLite history and audit trail.
- **Industrial constraints:** RS485 half-duplex serialization, LAN-only
  deployment, multi-viewer operator screens, 24/7 control-room ergonomics.
- **Frontend decisions:** Tauri over Electron, CSS tokens over a framework,
  Zustand store, hand-rolled canvas sparklines, light/dark themes and TR/EN
  localization.
- **Disclosure boundary:** no Modbus register addresses, bit meanings, packed
  register encoders, customer/facility names, IPs, operator logs or source code.

---

## Problem

Industrial facilities (distribution centers, cold storage warehouses,
factory floors) operate dozens to a hundred Robodor high-speed doors,
each driven by an RF-family controller card with an RS485 interface and a
documented Modbus register map. Today an operator has to walk to a card
to read its state or adjust a parameter; if a door faults, no one knows
until production stops.

**Goal:** put a live view of every card on every operator's desktop, so the
control room sees the whole facility at a glance, can drill into any card
in a click, and audits the historical behavior of any unit.

Constraints (real and self-imposed):

- **LAN-only.** Customer policy: no internet exposure.
- **Multi-viewer, single source of polling.** RS485 is half-duplex; you don't
  want every operator's app polling the bus independently.
- **Operator first.** Calm, scannable, predictable. Not a designer's playground.
- **24/7 left-on screens.** Subtle live-tick animation; no jitter; no jarring
  modals.
- **Deployable on a vanilla factory PC.** No exotic dependencies.

---

## Architecture

```
┌──────────────────┐  ┌──────────────────┐
│ operator desktop │  │ operator desktop │  ... up to ~10 viewers
│  (native Tauri)  │  │  (native Tauri)  │
└────────┬─────────┘  └────────┬─────────┘
         │ WebSocket (LAN)     │
         └──────────┬──────────┘
                    │
            ┌───────▼────────┐
            │ scada-server   │   headless Node.js service
            │ (one per site) │   • single polling source
            └───────┬────────┘   • snapshot cache
                    │            • SQLite history
                    │ Modbus TCP • WebSocket fan-out
            ┌───────▼────────┐
            │ Modbus gateway │   off-the-shelf RS485↔Ethernet bridge
            └───────┬────────┘
                    │ Modbus RTU over RS485 (9600 8N1)
       ┌────────────┼────────────┐
   ┌───▼──┐     ┌───▼──┐    ┌───▼──┐
   │ card │     │ card │    │ card │  ... up to ~100 cards per site
   └──────┘     └──────┘    └──────┘
```

### Why two processes

A single Tauri app could embed the polling, but then every operator desktop
hits the bus on its own — multiplied trips, contention on a shared half-duplex
line, and inconsistent views between operators. The split (`scada-server` as
sole poller, `scada-desktop` as viewer) is the classic SCADA pattern, and it
also gives us:

- One place to persist history (SQLite on the server PC)
- One place to write audit logs of any operator action
- A clean upgrade story: viewer can be redeployed without disturbing polling

### Why native desktop instead of a web app

Operators run the app full-screen on a control-room PC. Asking them to manage
"the URL" and a browser tab in a 24-hour shift is friction. A native app:

- Gets a Dock/Start-menu icon and a window of its own
- Survives accidental browser closes and tab refreshes
- Looks and behaves like a tool, not a website
- Can ship a small `.dmg` / `.msi` installer

### Why Tauri (not Electron)

Tauri ships ~10–20 MB binaries vs. Electron's 100–150 MB, uses the host
WebView (no Chromium bundled), and handles the cross-platform packaging well
enough on its own. The trade-off — a Rust toolchain in the build pipeline —
is worth the memory and disk savings for a long-running operator app.

### Why not a battle-tested SCADA platform (Ignition, FactoryTalk, TIA Portal)

Each of those is a fine product but expensive, vendor-locked, and visually
stuck in 2008. The customer's hardware, register map, and use case are simple
enough that a focused 1-vendor solution beats a generic platform.

---

## Stack

| Layer | Choice | Notes |
|---|---|---|
| **Server runtime** | Node.js (TypeScript, ESM, strict) | Pure JS deps; cross-platform; readily packaged as a Windows service |
| **Modbus** | `modbus-serial` over TCP | Connects to the gateway; FC03 read, FC06 write (write currently disabled by product decision) |
| **Polling engine** | Custom async loop, per-gateway request serialization | RS485 is half-duplex — requests must be queued per gateway, parallel across gateways |
| **In-memory cache** | Custom snapshot store | Emits delta events to subscribers; is the source of truth between polls |
| **WebSocket fan-out** | `ws` | Versioned protocol (`v: 1`); `hello` → `delta` / `online` / `offline` / `queryResult` |
| **Persistent history** | SQLite (`better-sqlite3`) with WAL mode | Time-series samples (1s cadence per metric per card), state-transition events, audit log |
| **Desktop shell** | Tauri 2 (Rust + WebView) | Mac (Apple Silicon) and Windows targets |
| **UI** | React 18 + TypeScript + Vite | No CSS framework — design-tokens via CSS custom properties; semantic class names |
| **State** | Zustand | Tiny, no boilerplate; good fit for the "single store + slice selectors" pattern |
| **Charts** | Hand-rolled `<canvas>` sparklines | Brought no chart-library dep just for thin time-series; ~120 LOC, DPR-aware, tabular numerics |
| **Theming & i18n** | CSS variable themes + a dictionary `t()` | Light/dark switchable at runtime; English/Turkish |

Total client bundle: ~210 KB JS + 28 KB CSS minified, 65 KB gzipped.

---

## Features

### 1. Plant overview — at-a-glance facility view

- Up to ~100 cards in a responsive grid; three density modes (cards / tiles /
  list) chosen at runtime based on facility size and operator preference
- Per-card status (healthy / warning / error / offline) derived purely from the
  controller's own error/warning bit-fields — the UI never overrides the
  firmware's classification
- Top-bar filter pills (All / Online / Warnings / Errors / Offline) with
  live counts; instant client-side filtering; search by name or technical key
- A right-hand rail appears only when a card is selected, showing a mini live
  preview (door visualization, KPIs, last error, recent events) so operators
  can scan multiple cards rapidly without leaving the overview

### 2. Card detail — deep view of one unit

- Hero stat: large open-percentage number, animated door visualization, mode
  + door state in plain language
- Side panel: gateway, slave address, last read time, last error
- Live values grid: 6 metric cells with animated change-flash
- Bit-field decoder: 32-bit error / warning fields rendered as an 8-column
  grid; only shown when bits are actually set (no clutter for healthy cards)

### 3. History — per-card time-series & event log

- Range selector: 5 min, 1 hr, 6 hrs, 24 hrs
- Sparklines for each metric (motor current, temperature, open %, bus voltage)
- Auto-refresh interval scales with selected range (5 s → 60 s) to keep the
  data fresh without overwhelming the UI
- Discrete-event timeline: door state transitions, online/offline edges,
  error- and warning-bit transitions, all timestamped
- Per-metric stats (min / avg / max) computed client-side from the returned
  samples

### 4. Alarms

- Derived in real time from the live snapshot: per-card error count, warning
  count, offline cards
- Sorted by severity then recency; click an alarm to jump into the card's
  detail page

### 5. System

- WebSocket health, last server-tick age, online card count, gateway count
- Per-gateway breakdown: card count, offline count, last-tick age
- Plant pulse: total motor draw across the facility, average temperature,
  per-state door distribution (closed / opening / open / closing / stopped /
  fault)

### 6. Operator preferences

- Server URL is editable at runtime (modal on the System page); the change
  triggers a clean reconnect; setting persists in `localStorage`
- Theme (light / dark) and language (English / Turkish) toggles live in
  System; changes apply instantly across the whole UI

### 7. Connection lifecycle (UX)

- Initial empty state shows a "connecting…" screen with the target URL
- WebSocket reconnect uses exponential backoff (1 s → 30 s, 2× multiplier)
- A non-blocking amber banner surfaces during reconnect; the grid keeps
  showing last-known values, visually dimmed
- A live-pulse animation on the topbar confirms the data is moving — without
  it, a 24-hour-on screen can read as frozen

---

## Design system

I lifted the brand palette directly from [robodor.com](https://www.robodor.com)
(an HSL token sweep over their stylesheet), so the desktop tool reads as a
first-party extension of the product line — same navy `#161873`, same
warm-cream `#eceae3` surfaces, same amber `#f5a210` accent. The status
palette (green / amber / red / gray) is custom and chosen for legibility
on both the cream light theme and a deep navy-tinted dark theme.

Typography is IBM Plex Sans for body and IBM Plex Mono for tabular numerics
— monospaced numbers matter when you are scanning a column of motor
currents and temperatures and a digit jitter at the wrong moment looks like
a problem.

Tokens are defined as CSS custom properties on the theme class, and the
entire UI (~25 components) flips between light and dark in a single
class-name change.

---

## Engineering decisions worth noting

- **Status comes from the controller, not heuristics.** The first
  implementation marked any card with motor current above a threshold as
  "warn"; this triggered false positives for cards that were briefly in a
  high-load opening cycle. Removed: the firmware already publishes its own
  warning bits; the UI surfaces those and stays out of clinical decisions.

- **Optimistic write feedback** (architecturally present, currently disabled
  per product decision). When writes are re-enabled, the server applies the
  new value to the snapshot immediately on a successful FC06 ACK, which fans
  out a delta to all viewers within a single round-trip — no spinner needed.

- **Per-field draft state with a sticky save bar.** Editing one field doesn't
  send a write per keystroke; the operator commits a batch ("Save 3 changes")
  with one click. Multi-operator edit conflicts are handled by surfacing the
  newer remote value next to the operator's draft.

- **"Packed byte" register pairs.** Several controller registers contain two
  semantically different fields in their high and low bytes (e.g. open- and
  close-direction speeds). The settings layer renders these as paired UI
  cards with a single backing register, and the encoder splices the edited
  half into the current register value before sending — preserving the
  unedited half even when only one side is changed.

- **History is on the server.** Operators don't carry around a database; the
  server PC does. A single SQLite file with WAL-mode persistence handles a
  100-card site indefinitely with hourly pruning of samples beyond a
  retention window (events and audit log are kept).

- **No real-time without a real-time architecture.** Every register
  difference flows through the same path: poll → snapshot diff → emit delta
  → fan-out to N viewers → apply locally. There's no polling on the client,
  no "refresh" button, no stale data dance.

---

## What's left out of this writeup

This is a documentation-only repository. No source code is included.
The specific Modbus register map (addresses, bit-field semantics, encode
rules for packed-byte registers) is also withheld — those are firmware
contract details kept inside the codebase.

The architecture, tech choices, UX patterns, and engineering trade-offs
are all in scope and described below.

Screenshots are not committed by default because the operator UI can expose
facility names, gateway addresses, slave IDs, audit rows and live production
states. [`screenshots/README.md`](screenshots/README.md) documents the redaction
checklist for a safe release-approved screenshot set.

---

## Project status

In production-readiness review at the time of this writeup. Core read paths
(polling → snapshot → WebSocket → UI) are uçtan uca working and tested
against an in-process Modbus simulator at 100-card scale. Real-hardware
on-bus validation is the next milestone; the gateway is off-the-shelf and
the polling engine has been verified against the same Modbus TCP framing
that production gateways speak.

Tauri release builds for macOS (Apple Silicon) are produced locally; a
GitHub Actions matrix to also produce signed Windows MSIs is the planned
next packaging step.

---

## See also

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — deeper dive into the
  polling engine, snapshot store, and WebSocket protocol
- [`docs/DECISIONS.md`](docs/DECISIONS.md) — annotated list of the
  trade-offs made during the build
- [`docs/STACK.md`](docs/STACK.md) — every dependency, why it was chosen,
  what was rejected
- [`screenshots/`](screenshots/) — UI snapshots (light & dark themes,
  English & Turkish, key views)

---

## License

This document is shared under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).
The implementation is proprietary; no source code is included in this
repository.
