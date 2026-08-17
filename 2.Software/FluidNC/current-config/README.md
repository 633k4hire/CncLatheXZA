# Current FluidNC lathe configuration

This folder contains the current FluidNC configuration artifacts for the Maijker
mini lathe after the FluidNC lathe feature work was brought forward.

Files:

- `maijker_xzact_mini_lathe.yaml` - MKS-DLC32 V2.1 machine config for the XZACt
  lathe build. The five-tool turret uses FluidNC's `maijker_5_station_turret`
  ATC driver rather than an SD-card `m6_macro`.
- `maijker_xzact_mini_lathe_encoder_bench.yaml` - bench-only encoder capture
  config for AS5600/StepperSpindle pulse and index validation. It enables
  encoder capture on `gpio.33`/`gpio.4`, keeps threading disabled, and should
  not replace the safe default config until the signals are scoped.
- `lathe-v3-config-audit.md` - current config audit, commissioning risks, and
  open physical validation items.

Firmware must be built with the FluidNC `maijker_wifi` PlatformIO environment,
which omits unused onboard-OLED code so the firmware and bundled filesystem fit
the standard two-slot 4 MiB layout. The matching firmware's TAMS contract is
`$ESP425` read-only telemetry, `$ESP426` exclusive shared-chuck ownership
selection, and `$ESP427` bounded X/Z probing. See the audit before
commissioning; the turret has no mechanical confirmation sensor and the
physical E-stop has no controller feedback.

Legacy reference:

- [`../../../3.Documentation/legacy/maijker_tool_change_legacy.gcode`](../../../3.Documentation/legacy/maijker_tool_change_legacy.gcode)

Commissioning checklist:

- [`lathe-v3-commissioning-checklist.md`](../../../3.Documentation/commissioning/lathe-v3-commissioning-checklist.md)

The AS5047P commissioning profile is staged with threading still disabled:

```yaml
lathe:
  enable_threading: false
  encoder_enable: true
  encoder_pulse_pin: gpio.33
  encoder_b_pin: gpio.35
  encoder_index_pin: gpio.4
  encoder_direction_invert: false
  encoder_pulses_per_rev: 1000
```

MKS-DLC32 V2.1 AS5047P inputs:

- `gpio.33` for A/pulse input; the TFT must not be attached.
- `gpio.35` for B/direction on the otherwise-unused Y-limit input.
- `gpio.4` for I/index on the direct J2/I2C pin. This avoids both the DLC32
  `gpio.39`/SD-detect conflict and the one-way TFT buffer on GPIO25/26, so the
  SD card may remain installed.

Do not enable `lathe.enable_threading` until the spindle encoder reports live RPM,
index detection, angular phase, revolution count, and a non-stale/non-fault state
through FluidNC `ESP421`. If hand rotation reports reversed direction, change
only `encoder_direction_invert`; do not swap A and B after pulse/index proof.
