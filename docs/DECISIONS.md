# Decisions log

Annotated list of trade-offs made during the build. Each entry: the choice,
what was rejected, and why.

## Architectural

### Single polling source, fan-out via WebSocket

**Chose:** one `scada-server` process per site, polls everything, pushes
deltas to all desktop viewers.
**Rejected:** "every desktop polls independently" — cleaner code per app,
but multiplies bus load on a half-duplex line, and gives operators
inconsistent views during gateway latency spikes.

### Native desktop client (Tauri)

**Chose:** Tauri 2 (Rust + WebView).
**Rejected:** Electron (3–10× larger binary, heavier memory, Chromium
update treadmill); pure web app (operators don't want to babysit a
browser tab in a 24-hour shift); SwiftUI / WPF native (single-platform
each, doubles maintenance).

### LAN-only by default

**Chose:** WebSocket on a chosen LAN port, server URL configurable in the
desktop UI.
**Rejected:** anything cloud-hosted as default. Customer policy. Cloud
remains an explicit future option behind a consent flag.

### View-only (no register writes from operator UI)

**Chose:** read-only operator app; writes are off in product.
**Rejected:** an "edit anything" mode. Industrial change control matters;
on-device parameter changes happen through a separate authenticated path,
not through a generic operator screen. The write infrastructure is fully
implemented in the server (audit log included) and dormant — re-enabling
is a config flag away when the role/permissions story is built out.

## Server

### Per-gateway request serialization

**Chose:** a promise queue per gateway; reads/writes for that gateway run
strictly in order.
**Rejected:** a single global queue (kills throughput when multiple
gateways are reachable in parallel); fully concurrent requests (RS485 is
half-duplex; the bus would corrupt).

### Two cadences: fast & slow

**Chose:** poll the live-status range every ~1 s (fast), the settings
range every ~10 s (slow). Fast group drives dashboards; slow group keeps
the detail view current without flooding the bus.
**Rejected:** a single uniform cadence (either too slow for live values
or too noisy for settings); a per-register cadence map (over-engineered
for the current ~80-register footprint).

### SQLite for history, in-process

**Chose:** `better-sqlite3` with WAL mode, a single file on the server PC.
**Rejected:** an external Postgres / TimescaleDB (extra dependency to
deploy and operate at every site, no value at this scale); writing time-
series to flat files (no efficient range queries).

### Snapshot in memory, history on disk

**Chose:** snapshot store is purely in-memory; history is written through
by a recorder. Server restart → snapshot is empty until the next poll
fills it.
**Rejected:** persisting snapshot to disk on every change (write
amplification, no operational benefit; the next poll round refills it in
~1 s anyway).

### Throttled history sample rate

**Chose:** maximum 1 sample per metric per card per second written to
DB, regardless of bus rate.
**Rejected:** writing every poll value (a 100-card site at 1 s polling
yields ~6 inserts/s/card × 6 metrics → 3.6k/s; pointless for human-
readable trends; samples beyond ~Hz never get visualized).

## Wire protocol

### Versioned envelope (`v: 1`)

**Chose:** every server-pushed and client-sent message carries `v: 1`.
**Rejected:** unversioned shapes. The first time we change a field, we'll
be glad we did this.

### Two-phase: hello then deltas

**Chose:** on connect, server sends a single `hello` with the full card
list; thereafter only deltas.
**Rejected:** per-card "subscribe" requests from the client (cleaner in
theory but doubles the round-trip count and complicates the state machine
without buying anything for our scale).

### Request/response over the same socket (`reqId`)

**Chose:** client → server queries (`querySamples`, `queryEvents`,
`queryAudit`, `write`) carry a `reqId`; the server replies with a single
correlated message.
**Rejected:** a separate HTTP API for queries (extra port to firewall,
extra auth surface, no value when both endpoints serve the same client).

## Desktop UI

### CSS variables + semantic class names (no Tailwind)

**Chose:** design-token CSS custom properties (`--bg-1`, `--tx-1`,
`--accent`, `--st-ok`, etc.) and small reusable class selectors (`.btn`,
`.chip`, `.kpi`, `.live-cell`). Theme switching = swap a single class on
the root.
**Rejected:** Tailwind (great for prototyping; here it would have buried
the design-system semantics in markup); CSS-in-JS (extra runtime cost for
no functional gain on a static design system).

### Hand-rolled `<canvas>` sparklines

**Chose:** ~120 LOC custom canvas sparkline component, DPR-aware, reads
theme tokens at draw time.
**Rejected:** Recharts / Chart.js / D3 (200+ KB extra in the bundle for
6 simple line charts; theme-aware integration would have been more code
than just drawing it).

### Zustand for global client state

**Chose:** a single `scadaStore` for live data + actions, plus tiny
helper stores (`prefsStore`, `cardNamesStore`).
**Rejected:** Redux (boilerplate-heavy, overkill); React Context-only
(re-renders too aggressively for a high-frequency snapshot); Jotai/Recoil
(fine, but Zustand is simpler for this shape).

### IBM Plex Sans + Plex Mono

**Chose:** Plex Sans for body, Plex Mono for tabular numerics.
**Rejected:** system-ui only (renders well but loses the brand-adjacent
typographic personality); Inter alone (no matching mono); a chunkier
display face for headlines (felt off-tone for a control room).

### Click-to-select, not click-to-navigate

**Chose:** clicking a card in the overview selects it (preview in the
right rail); a separate "Open detail →" button navigates to the full
detail view.
**Rejected:** clicking immediately navigates (the original implementation;
operators got "lost" in detail pages while trying to scan multiple cards).

### Conditional right rail

**Chose:** the overview's right rail only renders when a card is
selected; otherwise the grid spans the full width.
**Rejected:** a permanent rail with a "Plant pulse" summary when no card
is selected. It looked busier than it added value, and the same Plant
Pulse data lives in the System tab where it belongs.

### Status comes only from the firmware

**Chose:** card status (ok/warn/err/offline) is derived from `online`
plus the firmware's own error/warning bit-fields, full stop.
**Rejected:** "warn if motor current > X" / "warn if temp > Y" UI-side
heuristics. They generate false positives during normal cycles and
override the firmware's own classification — which is a domain it knows
better than the UI.

### Optimistic write feedback (when writes are enabled)

**Chose:** on a successful write ACK, server applies the new value to
its snapshot immediately, fan-outs the delta to all viewers — write
feels instant across the room.
**Rejected:** wait-for-next-poll-round (write feels laggy); fully client-
side optimistic + later reconcile (race conditions if device rejects).

## Internationalization

### Inline dictionary, no library

**Chose:** a single `strings.ts` with `en` + `tr` columns, a `useT()`
hook reading from a `prefsStore`.
**Rejected:** i18next / react-intl (overkill for a two-language UI with
~250 keys; brings runtime cost and bundle weight; does not buy us
anything for a single-team-managed translation set).

### Locale stays simple

**Chose:** card names are user-driven (operators name them in their own
language); UI chrome is the only thing translated.
**Rejected:** mirroring the operator's locale to numeric formatting (we
deliberately keep numerics monospaced and locale-agnostic so a 22.5 A
reading reads the same in TR and EN — operator literacy doesn't depend
on the comma).

## Theming

### Light is the default

**Chose:** light is on first launch (warm cream + navy, matched to the
customer's product line).
**Rejected:** dark by default. Most operator desks are in lit offices
during the day; dark is preferred at night and on lights-out walls.
Both are first-class; the toggle persists in `localStorage`.

### Status colors stay constant across themes

**Chose:** green / amber / red / gray have shifted slightly between
themes for legibility but their meaning never changes.
**Rejected:** different palettes per theme (would teach the operator
two different languages of color).
