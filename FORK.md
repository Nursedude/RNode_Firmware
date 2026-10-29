# Nursedude/RNode_Firmware — MeshForge fork

Branch `meshforge` = upstream `markqvist/RNode_Firmware` master plus the changes below,
so RNodes work reliably in the MeshForge domain. Wire format and KISS protocol are unchanged.
Developed with an AI assistant (Claude) under the maintainer's direction.

## Bounded interference-avoidance hold (2026-10-28)

- **What:** a Heltec LoRa32 v3 reports a steady, self-generated ~-71 dBm "interference".
  With interference avoidance on, it holds or drops its own transmissions for as long as
  that lasts (bench: 9/20 delivered, 11 lost in a row, delays up to 26.7 s).
- **Why:** `medium_free()` refuses TX while `interference_detected` is true, and the noise
  floor never re-learns a level above `CSMA_RFENV_RECAL_LIMIT_DB` (-83 dBm). So a steady
  level above that blocks TX indefinitely.
- **Our solution:** interference blocks TX only while it is transient. After
  `CSMA_INFR_MAX_HOLD_MS` (5 s) of continuous carrier-free interference, debounced over
  `CSMA_INFR_GAP_MS` (500 ms), it stops blocking. It is still reported to the host, DCD
  still blocks, and the V4 LNA branch is unchanged. Two v3 units with IA on: 40/40 and
  40/40 delivered, worst case bounded by the cap.
