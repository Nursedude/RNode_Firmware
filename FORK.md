# Nursedude/RNode_Firmware — MeshForge fork

Branch `meshforge` = upstream `markqvist/RNode_Firmware` master plus the fixes
below. Each fix is small, board-agnostic where possible, and offered back
upstream. Wire format and KISS protocol are unchanged.

## How this fork is made

This fork is developed with an AI assistant (Claude, Anthropic) under the direction of
its maintainer (Nursedude / WH6GXZ). The maintainer found the problem, ran the bench
measurements on real hardware and made every decision. **The code changes on this branch
were written by the AI assistant.** Upstream treats machine-originated code as carrying
no valid licence grant, so these patches are NOT offered upstream as code. The
maintainer reports the findings in his own words instead. Commits written this way carry
a `Co-Authored-By: Claude` trailer; `11d51d7` and `0042d73` predate this note and lack it.

## Fixes

### Bounded interference-avoidance hold (2026-10-28)

**Problem.** `medium_free()` refuses to transmit while `interference_detected`
is true, and interference avoidance is on by default. The detector counts any
carrier-free level above `noise_floor + CSMA_INFR_THRESHOLD_DB` as interference,
and only re-learns the floor for levels below `CSMA_RFENV_RECAL_LIMIT_DB`
(-83 dBm). A STEADY level above that limit therefore blocks TX for as long as it
lasts.

**Seen on.** Heltec LoRa32 v3 (two units). Each reports a steady ~-71 dBm
"interference" at 903.625 MHz that no other radio, and no SDR inches away,
can see. The v3 generates it itself. A v4 at the same spot is clean.

**Measured, stock 1.86, interference avoidance ON.** 9/20 probes delivered.
11 consecutive probes were LOST while the v3 reported -71, and others were
delayed up to 26.7 s. With IA OFF: 20/20, max 0.41 s.

**Fix.** Track how long carrier-free interference has persisted, debounced
over gaps shorter than `CSMA_INFR_GAP_MS` (500 ms). Once it has lasted
`CSMA_INFR_MAX_HOLD_MS` (5 s), it no longer blocks TX. It is still reported to
the host as interference, and a real LoRa carrier (`dcd`) still blocks exactly
as before. The V4 LNA branch is unchanged.

**Measured, patched, interference avoidance ON.** 40/40 delivered, including
all 11 probes sent under -71, max 3.76 s (bounded by the cap).

**Second unit.** A second Heltec v3 ran as sender, received by an RF-only RNS instance (no LAN/TCP),
interleaved IA ON / OFF / ON: 20/20, 20/20, 20/20. With IA on, the worst cases were 5.05 s and
4.51 s, which is the cap releasing. With IA off, the max was 0.12 s.

**Builds:** heltec32_v3 and heltec32_v4 compile (ESP32 core 2.0.17).
