# Lathe V3 Current Config Audit

Audit target: `maijker_xzact_mini_lathe.yaml`

This audit records what the checked-in FluidNC config currently does and what
still must be verified on the physical machine. It is intentionally conservative:
the config can represent a plausible machine, but physical direction, travel,
limits, turret indexing, spindle behavior, and probe polarity still need staged
commissioning before cutting.

## Summary

| Area | Current state | Commissioning implication |
| --- | --- | --- |
| Lathe mode | `lathe.enable: true` | FluidDial should auto-detect lathe mode through `ESP421`. |
| X/Z operator axes | `x_axis: 0`, `z_axis: 2` | FluidDial maps X/Z/C display slots to machine axes 0/2/5. |
| Threading | `enable_threading: false` | Threading must remain disabled until encoder feedback is proven. |
| Encoder | `encoder_enable: false`, pulse/index `NO_PIN` | Dashboard should show encoder/threading unsafe. |
| Homing | X cycle 1, Z cycle 2 | Verify direction and switch polarity before full `$H`. |
| Turret | 5-tool HBridge/M6 macro | Requires manual active-tool initialization and dry turret tests. |
| E-stop/control inputs | All config control pins `NO_PIN` | Physical E-stop must be external and tested independently. |

## Axis Audit

### X Axis

Current settings:

- Axis path: `axes.x`
- Machine axis index: 0
- `steps_per_mm: 320`
- `max_rate_mm_per_min: 5000`
- `acceleration_mm_per_sec2: 500`
- `max_travel_mm: 60`
- `soft_limits: true`
- Homing cycle: 1
- Homing positive direction: true
- Homing machine position: `60.000`
- Limit pin: `gpio.36:low`
- Hard limits: true
- Step pin: `i2so.1`
- Direction pin: `i2so.2`

Physical validation required:

- Confirm positive/negative jog direction.
- Confirm limit polarity with manual switch actuation.
- Confirm homing direction and pull-off.
- Confirm the 60 mm travel value before relying on soft limits.

### Z Axis

Current settings:

- Axis path: `axes.z`
- Machine axis index: 2
- `steps_per_mm: 320`
- `max_rate_mm_per_min: 5000`
- `acceleration_mm_per_sec2: 500`
- `max_travel_mm: 90`
- `soft_limits: true`
- Homing cycle: 2
- Homing positive direction: true
- Homing machine position: `90.000`
- Limit pin: `gpio.34:low`
- Hard limits: true
- Step pin: `i2so.3`
- Direction pin: `i2so.4`

Physical validation required:

- Confirm positive/negative jog direction.
- Confirm limit polarity with manual switch actuation.
- Confirm homing direction and pull-off.
- Confirm the 90 mm travel value before relying on soft limits.

### A Axis - Turret Indexing

Current settings:

- Axis path: `axes.a`
- Purpose: tool changer axis
- `steps_per_mm: 320`
- `max_rate_mm_per_min: 1000`
- `acceleration_mm_per_sec2: 50`
- `max_travel_mm: 100000`
- `soft_limits: false`
- Limit pins: `NO_PIN`
- Hard limits: false
- Step pin: `i2so.7`
- Direction pin: `NO_PIN`
- Direction control: `digital0_pin: gpio.5` through `M62 P0` / `M63 P0`

Physical validation required:

- Confirm `M62 P0` and `M63 P0` drive the expected turret direction behavior.
- Confirm one macro station increment equals one actual turret station.
- Confirm the reverse lock/backlash move lands repeatably.
- Confirm reset/alarm recovery procedure if a turret move is interrupted.

### C Axis - Commanded Spindle/C Axis

Current settings:

- Axis path: `axes.c`
- Machine axis index: 5
- `steps_per_mm: 533.333`
- `max_rate_mm_per_min: 2000`
- `acceleration_mm_per_sec2: 75`
- `max_travel_mm: 100000`
- `soft_limits: false`
- Limit pins: `NO_PIN`
- Hard limits: false
- Step pin: `i2so.5`
- Direction pin: `i2so.6`

Important distinction:

- This is a commanded C axis from the older build.
- It is not spindle phase feedback.
- Threading and synchronized spindle behavior still require a real spindle
  encoder configured under `lathe.encoder_*`.

Physical validation required:

- Confirm C jog direction and scale.
- Confirm C commands do not conflict with spindle drive behavior.
- Decide whether `$HC` is safe or should be avoided for this mechanical setup.

## Probe Audit

Current settings:

- Probe pin: `gpio.22`
- `check_mode_start: true`

Physical validation required:

- Confirm probe polarity before any motion.
- Confirm boot behavior with the expected probe wiring state.
- Confirm low-feed `G38.2` contact and no-contact failure behavior.
- Confirm T5 probe/contact station repeatability before using T5 for touch-off.

## Spindle and HBridge Audit

Current settings:

- `HBridge.output_cw_pin: gpio.25`
- `HBridge.output_ccw_pin: gpio.26`
- `HBridge.enable_pin: gpio.27`
- `disable_with_s0: true`
- `s0_with_disable: true`
- `spinup_ms: 1000`
- `spindown_ms: 1000`
- `tool_num: 5`
- `speed_map: 0=0.000% 2000=100.000%`
- `off_on_alarm: true`
- `m6_macro: $SD/Run=maijker_tool_change.gcode`

Physical validation required:

- Confirm spindle direction for `M3` and `M4`.
- Confirm `M5`, `S0`, reset, alarm, and physical E-stop stop spindle output.
- Confirm `tool_num: 5` matches the actual turret count and FluidDial T1-T5 UI.

## M6 Macro Audit

Audit target: `maijker_tool_change.gcode`

Current behavior:

- Reads target tool from `#T`.
- Assumes `#<current_tool>` is already initialized.
- Does nothing if target equals current tool.
- Computes forward wraparound delta over 5 tools.
- Commands turret direction with `M62 P0`.
- Moves A forward by `delta + 0.1`.
- Commands reverse/lock direction with `M63 P0`.
- Moves A by `-0.076`.
- Updates `#<current_tool>` to target.

Commissioning risks:

- If `#<current_tool>` is not initialized correctly after boot, the first tool
  change can index to the wrong station.
- The macro has no physical turret sensor confirmation.
- The macro has no automatic recovery if reset/alarm occurs mid-change.
- The FluidDial V3 UI blocks duplicate M6 sends while pending, but it cannot
  prove the mechanical turret position without reliable FluidNC state and
  operator validation.

Required validation:

- Establish the boot-time procedure for initializing `#<current_tool>`.
- Dry-run every transition T1 through T5 and wraparound T5 to T1.
- Verify same-tool `Tn M6` performs no motion.
- Verify the active tool reported through `ESP421` matches the physical turret.

## Lathe Feature Audit

Current settings:

- `lathe.enable: true`
- `enable_css: true`
- `enable_feed_per_rev: true`
- `enable_threading: false`
- `min_css_diameter_mm: 1.000`
- `max_css_rpm: 2000.000`
- `x_axis: 0`
- `z_axis: 2`
- `feedback_stale_ms: 250`
- `encoder_enable: false`
- `encoder_pulse_pin: NO_PIN`
- `encoder_index_pin: NO_PIN`
- `encoder_pulses_per_rev: 1`

Commissioning interpretation:

- Lathe mode is available for UI/profile/status work.
- CSS and feed-per-rev modes are configured, but should be tested without
  cutting load first.
- Threading is explicitly disabled.
- Encoder values are placeholders and must not be treated as usable feedback.
- `ESP421` should report disabled/no feedback until encoder hardware is added.

## Control and Safety Inputs

Current config pins are all `NO_PIN`:

- `safety_door_pin`
- `reset_pin`
- `feed_hold_pin`
- `cycle_start_pin`
- `macro0_pin` through `macro3_pin`

Required machine safety posture:

- Physical E-stop must cut actuator power independently of FluidNC.
- Do not rely on the pendant, WiFi, UART, or FluidNC control pins as the only
  emergency stop path.
- Validate feed hold/reset behavior separately from the physical E-stop.

## Open Items Before Cutting

| Item | Owner | Status | Notes |
| --- | --- | --- | --- |
| Verify X/Z direction and homing behavior. | | Open | |
| Verify X/Z switch polarity and hard-limit behavior. | | Open | |
| Verify X/Z travel and soft-limit values. | | Open | |
| Define and document boot-time `#<current_tool>` initialization. | | Open | |
| Dry-run all turret transitions and same-tool no-op behavior. | | Open | |
| Verify C axis command behavior and whether `$HC` is allowed. | | Open | |
| Verify probe polarity, startup behavior, and low-feed `G38.2`. | | Open | |
| Verify spindle direction and alarm/E-stop shutdown. | | Open | |
| Install and validate encoder before enabling encoder feedback. | | Open | |
| Keep `lathe.enable_threading: false` until a separate encoder/threading checklist passes. | | Open | |
