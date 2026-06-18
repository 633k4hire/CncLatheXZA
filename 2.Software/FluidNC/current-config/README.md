# Current FluidNC lathe configuration

This folder contains the current FluidNC configuration artifacts for the Maijker
mini lathe after the FluidNC lathe feature work was brought forward.

Files:

- `maijker_xzact_mini_lathe.yaml` - MKS-DLC32 V2.1 machine config for the XZACt
  lathe build.
- `maijker_tool_change.gcode` - 5-tool turret M6 macro referenced by the config.

The physical spindle encoder is intentionally not enabled yet. The config keeps:

```yaml
lathe:
  encoder_enable: false
  encoder_pulse_pin: NO_PIN
  encoder_index_pin: NO_PIN
```

Recommended MKS-DLC32 V2.1 encoder candidates once the encoder is installed:

- `gpio.33` for encoder pulse input.
- `gpio.39` for encoder index input only if SD-card detect is not wired/needed.

Do not enable `lathe.enable_threading` until the spindle encoder reports live RPM,
index detection, angular phase, revolution count, and a non-stale/non-fault state
through FluidNC `ESP421`.
