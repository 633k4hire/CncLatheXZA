# Spindle Encoder GPIO4 Hand Test — 2026-08-17

## Test posture

- FluidNC application: `c07cdab5`
- Maintained configuration: `181472cd`
- FluidDial: `dfa278a` (unchanged)
- AS5047P A/B/I: `gpio.33` / `gpio.35` / `gpio.4`
- Encoder resolution: 1000 pulses/revolution
- Threading: disabled
- Spindle motor power: disconnected
- Spindle rotation: manual only
- Telemetry: read-only `/api/v1/lathe/status`; no motion commands issued

## Result

The test began at zero A and Index counts. Matthew rotated the spindle through
multiple complete turns in both directions. The final observation was:

- A pulse count: 9,726
- Index count: 10
- Direction: observed CW and CCW
- Settled Index intervals: exactly 1,000 A pulses per revolution
- Encoder fault: false throughout

One 883-pulse Index interval occurred across the deliberate direction reversal.
That partial path is expected when reversing between Index crossings; subsequent
same-direction intervals returned to exactly 1,000 pulses.

## Wiring conclusion

GPIO4 successfully receives the AS5047P Index signal. GPIO39 failed because it
is shared with DLC32 SD-card detect, and GPIO25 failed at the TFT connector
because that path is behind a one-way output buffer. Keep Index on direct GPIO4,
leave the SD card installed, and do not use GPIO25/26 as encoder inputs.

This test accepts low-speed A/B/I capture only. It does not enable or qualify
threading, which remains gated on powered-spindle speed tests and a separate
threading commissioning plan.
