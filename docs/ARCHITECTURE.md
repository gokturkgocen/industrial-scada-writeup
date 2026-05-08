# Architecture deep-dive

A closer look at the moving parts of the Robodor SCADA system. No source
code; specific register addresses and bit semantics are also withheld.

## Two processes, one protocol

The server and desktop are independently deployable, talk over a versioned
WebSocket protocol, and share no code. Either side can be redeployed without
touching the other.

```
                 v1 wire protocol
                 ─────────────────
                       hello
        ◀──────────────────────────────  initial snapshot of all cards
                       delta
        ◀──────────────────────────────  per-card register changes
                       online
        ◀──────────────────────────────  card came online
                       offline
        ◀──────────────────────────────  card went offline + reason
                       writeAck
        ◀──────────────────────────────  ack/nack for an operator write
                       queryResult
        ◀──────────────────────────────  history / audit query response

                       write
        ──────────────────────────────▶  operator-issued register write
                       querySamples
        ──────────────────────────────▶  per-card metric time-series
                       queryEvents
        ──────────────────────────────▶  state-transition events
                       queryAudit
        ──────────────────────────────▶  audit log
```

The protocol is intentionally tiny. There's no resource-fetching pattern,
no polling endpoints; the server pushes everything that mutates and the
client pulls only on explicit user-driven queries (history, audit).

## Server: layered & event-driven

```
┌─────────────────────────────────────────────────────────────┐
│ index.ts                                                    │
│   wires everything; owns process lifecycle                  │
└──────────┬──────────────────────────────────┬───────────────┘
           │                                  │
   ┌───────▼─────────┐                ┌───────▼──────────┐
   │ PollingEngine   │                │ ScadaWsServer    │
   │  • round-robin  │                │  • on connect:   │
   │  • per-gateway  │                │      send hello  │
   │    serialized   │                │  • subscribe to  │
   │  • fast + slow  │                │    snapshot      │
   │    cadences     │                │    events → fan- │
   └───────┬─────────┘                │    out as deltas │
           │                          │  • handle client │
           │ writes via               │    queries       │
           │ ModbusGateway            └─────┬─────────┬──┘
   ┌───────▼─────────┐                      │         │
   │ ModbusGateway   │                      │         │
   │ • single TCP    │              ┌───────▼──┐  ┌───▼─────┐
   │   per gateway   │              │Snapshot  │  │History  │
   │ • req queue     │              │Store     │  │Store    │
   │   (RS485        │   ┌──────────▶ • cards  │  │ • SQLite│
   │   half-duplex)  │   │ writes   │ • diff   │  │ • events│
   │ • FC03/FC06     │   │ go here  │ • emit   │  │ • audit │
   └───────┬─────────┘   │          └─────┬────┘  └─────────┘
           │             │                │
           └─────────────┘                │
                                          │  HistoryRecorder
                                          │  subscribes to
                                          ▼  snapshot deltas
                                     samples + events → DB
```

### `PollingEngine`

Round-robin per gateway. For each `(gateway, slaveId)` it issues an FC03
"read holding registers" against the **fast group** (the live-status range,
~15 registers). On a slower cadence (default 10× the fast period) it also
reads the **slow group** (settings registers, ~60 more) so the desktop's
detail and history views see fresh values without paying for them every
poll.

Per-gateway requests are serialized through a promise queue: RS485 is
half-duplex and only one request can be in flight to any unit at a time.
Different gateways are polled in parallel, each owning its own TCP socket.

If a read fails, the engine doesn't abort — it logs, marks the card
offline in the snapshot, and continues with the next slave. A subsequent
successful read flips the card back online and emits the appropriate event.

### `SnapshotStore`

The in-memory truth between polls. `applyRead(key, ts, registers)` performs
a register-level diff against the previous snapshot for that card and
emits a `delta` event with only the changed addresses. Online/offline
transitions are emitted as separate events for cleaner client-side
handling.

The store doesn't know about WebSockets, the polling cadence, or persistence —
it's a pure in-memory cache with a pub/sub interface. Tests run against
it with no I/O.

### `ScadaWsServer`

A thin shell over `ws`. On every new connection it sends a `hello` frame
with the full current snapshot list (so a freshly connected viewer renders
the entire facility within a single round-trip), then forwards every
subsequent store event to all connected clients.

Inbound messages from clients are restricted to four types: `write`,
`querySamples`, `queryEvents`, `queryAudit`. Each carries a `reqId` so
the client can match the asynchronous response to the request.

Writes are dispatched to a `WriteExecutor` injected at construction time.
This kept the server side cleanly testable and lets the entrypoint compose
write semantics (gateway lookup, address validation, audit logging) in
one place.

### `HistoryStore` & `HistoryRecorder`

`better-sqlite3` with WAL mode. Three tables:

- `samples (id, card_key, metric, ts, value)` — per-metric time-series,
  indexed on `(card_key, metric, ts)` for range queries
- `events (id, card_key, ts, type, payload)` — discrete state transitions
  with a small JSON payload (e.g. door state from→to)
- `audit (id, ts, actor, card_key, address, before_value, after_value, note)`
  — every operator write, with the previous value snapshotted at the moment
  of the write

The `HistoryRecorder` subscribes to the snapshot store and writes through:

- Throttled samples (default 1 per metric per card per second) — the bus
  may run faster, but the DB doesn't need to
- Events on door-state transitions, online/offline edges, and error/warning
  bit transitions (with the bit indices that flipped, in the payload)

Hourly pruning of samples and events past a configurable retention; audit
is kept indefinitely.

## Desktop: read-only, theme-aware, localized

```
┌──────────────────────────────────────────────────────┐
│ App.tsx                                              │
│  • picks a route                                     │
│  • applies theme + language to the root container    │
│  • mounts a single WS client and keeps the store     │
│    populated                                         │
└──────────┬───────────────────────────────────────────┘
           │
   ┌───────▼─────────────────────────────────────────┐
   │ Zustand stores (client-only state)              │
   │  • scadaStore: cards, wsStatus, selectedKey,    │
   │    write & query actions                        │
   │  • cardNamesStore: per-card aliases (localStorage)│
   │  • prefsStore: theme + language (localStorage)  │
   └─────────────────────────────────────────────────┘
           │
           ▼
   View components — pure props in, pure markup out
   • PlantOverview, OverviewRail
   • CardDetail (Live | History)
   • Alarms, Timeline, SystemStatus
   • Sparkline (canvas), DoorViz, BitGrid, ConnBadge,
     CardNameEditor, Modal, Toast, ConnectingScreen
```

### Why Zustand

The data model is "one big snapshot store + a few small UI stores"; this
fits Zustand's idioms well. Components subscribe to slim slices, the store
applies WebSocket messages directly, and there's no need for a reducer
boilerplate or selectors-as-libraries layer.

### Why a hand-rolled canvas chart

Adding a chart library for ~120 lines of sparkline rendering is overkill
when the requirements are: tabular numeric labels on the axis, DPR-aware
crispness, gradient fill, and a single-color line. The chart respects the
active theme by reading CSS custom properties at draw time.

### Why no CSS framework

Operators need a single design system applied consistently across ~25
components. A set of CSS custom properties (themed, swappable in a single
class change) plus semantic class names (`.btn`, `.chip`, `.live-cell`,
`.kpi`) gives me one place to change a color and it propagates everywhere.
Tailwind-style utilities would have buried that signal in component markup.

### How i18n works

A `useT()` hook returns a translator bound to the active language. Strings
are organized by feature area (`page.*`, `card.*`, `alarms.*`, etc.) in a
single dictionary file with English and Turkish columns. The `door.*` and
`mode.*` keys are exposed as small hook helpers (`useDoorLabel`,
`useModeLabel`) so view components don't repeat the switch.

---

## Connection lifecycle

```
state machine, client side
═══════════════════════════════

  idle ──start──▶ connecting ──open──▶ open
                       │                  │
                       │                  │ socket.close
                       ▼                  ▼
                 reconnecting ◀──────── any drop
                       │
                       ├ reconnect attempt succeeds ──▶ open
                       │
                       └ user closes app ──▶ closed
```

Backoff: 1 s → 2 s → 4 s → 8 s → 16 s → 30 s → 30 s …

The UI never blocks on reconnect attempts. During `reconnecting`, the
overview keeps showing the last-known snapshot dimmed; the rail mini-detail
keeps updating numerics from cache (just no longer pulsing live). A
non-modal amber banner across the top tells the operator we're trying.

A "Retry now" button forces an immediate reconnect attempt, useful when
the operator knows the network just came back.

## Write path (when enabled)

```
operator edits a field
        │
        ▼
draft stored locally; field marks dirty
        │
        ▼
operator clicks "Save N changes"
        │
        ▼ for each dirty field:
        │   client encodes the new register value
        │   (preserving the other byte half for packed pairs)
        │   sends { type: "write", reqId, key, address, value }
        │
        ▼
server validates address is in known range
        │
        ▼
server calls writeExecutor:
  • dispatches FC06 to the appropriate gateway+slave
  • on ACK: snapshot.applyRead(key, now, { [addr]: value })
            (this fans out a delta to all viewers immediately)
  • writes audit row (before, after, ts)
  • emits writeAck { ok: true }
        │
        ▼
client clears the dirty draft on writeAck.ok
```

The optimistic snapshot update on a successful ACK is the trick that makes
writes feel instant: the value is "live" the moment the ACK lands; we
don't wait for the next poll round to read it back.

If the device rejects the write or the gateway times out, the executor
throws, the server emits `writeAck { ok: false, error }`, and the client
keeps the draft so the operator can retry or revert.

---

## What I'd build next

- A small bit-label dictionary (mapping each error/warning bit to a human
  description) so the operator sees `ERR.06 — Overload` instead of bit indices
- Bulk write: select N cards in the overview, edit one setting, write to all
- Server packaged as a Windows service / launchd daemon; auto-start on boot
- A signed Windows MSI in CI (GitHub Actions matrix), alongside the macOS
  `.dmg`
- An optional cloud sync layer: the server pushes anonymized telemetry to a
  central dashboard (separate consent flag; LAN-only stays the default)
