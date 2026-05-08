# Screenshots — what to add, what to redact

Add UI snapshots here as `.png` files. The README at the repo root
references this folder; recruiters will browse them after reading the
overview.

## Suggested set

1. `01-overview-light.png` — Plant overview, light theme, ~10–20 cards
2. `02-overview-dark.png` — Plant overview, dark theme
3. `03-card-detail-live.png` — Card detail with door visualization,
   live values grid, no errors
4. `04-card-detail-error.png` — Card detail with error bits set,
   showing the bit-grid decode
5. `05-history-trends.png` — History view with motor current sparkline
6. `06-system-pulse.png` — System tab with Plant Pulse, gateway list,
   theme/language toggle visible
7. `07-overview-rail.png` — Overview with right rail showing selected-
   card preview
8. `08-overview-tr.png` — Overview in Turkish, just to demonstrate i18n

## Redaction checklist before posting

Robodor branding and RF15 / RF30 product names are fine to keep visible.
The remaining items to review:

- [ ] **No real network IPs / hostnames** in the System tab, server URL
      modal, or audit log. Use `127.0.0.1`, `192.168.0.x`, or `<redacted>`.
- [ ] **No real operator names** in the audit log entries (the actor
      column). Blank or generic ("operator.a") is fine.
- [ ] **No real customer / facility names** if the screenshots happen
      to capture them in card aliases or window titles.
- [ ] **Cmd-Shift-5 → Capture** at native retina; export at the size
      you want displayed; compress with `pngcrush` or similar before
      committing.

## How to take clean screenshots

The simulator running locally with the redactions above produces
realistic-looking UI without leaking anything. Two terminals:

```bash
# Terminal 1: server with simulator
cd scada-server
SCADA_USE_SIMULATOR=1 SCADA_SIMULATOR_SLAVES=1,2,3,4,5,6,7,8,9,10 \
  npx tsx src/index.ts

# Terminal 2: desktop
cd scada-desktop
npm run tauri:dev
```

Wait a few seconds for the slow-poll to fill the snapshot, then capture.
