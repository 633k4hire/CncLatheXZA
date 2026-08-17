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
| Shared chuck | `shared_chuck: true`, `c_axis: 5` | C positioning and the CStepper spindle backend are mutually exclusive owners of one physical step/dir drive. |
| Threading | `enable_threading: false` | Threading must remain disabled until encoder feedback is proven. |
| Encoder | AS5047P enabled for commissioning; A/B/I `gpio.33`/`gpio.35`/`gpio.4`, 1000 PPR | GPIO4 is a direct input and avoids the GPIO39/SD-detect and GPIO25/26 TFT-buffer conflicts. Threading remains disabled until direction, index, and phase evidence pass. |
| Homing | X cycle 1, Z cycle 2 | Verify direction and switch polarity before full `$H`. |
| Turret | First-class `maijker_5_station_turret` ATC | Five software-dead-reckoned stations; no mechanical confirmation sensor is fitted. |
| E-stop/control inputs | All config control pins `NO_PIN` | Physical E-stop cuts power but has no FluidNC feedback and must be tested independently. |

## Axis Audit

### X Axis

Current settings:

- Axis path: `axes.x`
- Machine axis index: 0
- A4988 microstep DIP: `1 ON, 2 ON, 3 ON` (1/16)
- `steps_per_mm: 640`
- `max_rate_mm_per_min: 600`
- `acceleration_mm_per_sec2: 25`
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
- A4988 microstep DIP: `1 ON, 2 ON, 3 ON` (1/16)
- `steps_per_mm: 640`
- `max_rate_mm_per_min: 600`
- `acceleration_mm_per_sec2: 25`
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

### Turret ATC - Maijker 5-Station Toolchanger

Current settings:

- Config path: `maijker_5_station_turret`
- `station_count: 5`
- Step pin: `i2so.7`
- Direction pin: `gpio.5`
- `steps_per_station: 320`
- `step_rate_hz: 400`
- `overshoot_steps: 32`
- `lock_backoff_steps: 32` (matches the 32-step forward overshoot so the
  reverse seating move returns to the nominal station position)
- `require_confirmed_tool: true`
- `current_tool: 0`
- `sensor_pin: NO_PIN`

Physical validation required:

- Confirm `gpio.5` direction is stable before `i2so.7` step pulses.
- Confirm one driver station increment equals one actual turret station.
- Confirm forward overshoot and reverse lock/backlash move land repeatably.
- Confirm `M61Qn` initializes the physical station before first open-loop M6.
- Confirm `ESP421` reports turret configured/current/confirmed state.
- Confirm reset/alarm recovery procedure if a turret move is interrupted.

### C Axis - Commanded Spindle/C Axis

Current settings:

- Axis path: `axes.c`
- Machine axis index: 5
- Direct drive: 1:1, no belt reduction
- External driver microstep DIP: `S1 OFF, S2 ON, S3 OFF` (1/8)
- `steps_per_mm: 4.444444` steps/degree
- `max_rate_mm_per_min: 243000` (675 RPM)
- `acceleration_mm_per_sec2: 9000` (1500 RPM/s)
- `max_travel_mm: 100000`
- `soft_limits: false`
- Limit pins: `NO_PIN`
- Hard limits: false
- Step pin: `i2so.5`
- Direction pin: `i2so.6`

Important distinction:

- This is both the positioning drive and the open-loop spindle drive for the
  same physical chuck. `M3/M4/M5` use the CStepper backend on these pins.
- It is not spindle phase feedback.
- Threading and synchronized spindle behavior still require a real spindle
  encoder configured under `lathe.encoder_*`.
- `lathe.shared_chuck: true` makes firmware reject a block that requests C
  motion and spindle output together.
- `$ESP426=MODE=IDLE|C_POSITIONING|SPINDLE` selects exclusive ownership only
  while the controller, planner, and spindle output are stopped. It never
  starts motion.

Physical validation required:

- Confirm C jog direction and scale.
- Prove that simultaneous C/spindle commands fail closed and that M5 is
  required before changing ownership.
- Decide whether `$HC` is safe or should be avoided for this mechanical setup.

## Probe Audit

Current settings:

- Probe pin: `gpio.22`
- `check_mode_start: true`

Physical validation required:

- Confirm probe polarity before any motion.
- Confirm boot behavior with the expected probe wiring state.
- Confirm low-feed `G38.2` contact and no-contact failure behavior.
- Confirm bounded adapter requests through `$ESP427` report contact and final
  position truthfully.
- Confirm T5 probe/contact station repeatability before using T5 for touch-off.

## C-Stepper Spindle Audit

Current settings:

- `CStepper.axis: 5`
- `CStepper.cw_positive: true` (direction must still be commissioned)
- `CStepper.minimum_rpm: 50.0`
- `CStepper.maximum_rpm: 675.0`
- `CStepper.acceleration_rpm_per_sec: 1500.0`
- `CStepper.deceleration_rpm_per_sec: 100.0`
- `CStepper.operator_watchdog_ms: 12000`
- 1600 pulses/revolution from the 8x driver DIP setting and C scale
- The C positioning ceiling is 243000 degrees/min (675 RPM); FluidDial uses a
  lower 250 RPM default for continuous manual C jogging.
- 0.225 degrees per microstep
- Normal `M5` decelerates at 100 RPM/s all the way to rest. From 675 RPM the
  expected stop is approximately 6.75 seconds; safety faults stop immediately.
- `tool_num: 0`
- `off_on_alarm: true`
- `atc: maijker_5_station_turret`

Physical validation required:

- Confirm `M3` is physical CW and `M4` is physical CCW; invert
  `cw_positive` only if that test proves the labels are reversed.
- Confirm 0.5, 1.0, and 5.0 RPM against a marker or tachometer.
- Confirm `M5`, reset, alarm, operator-link timeout, and physical E-stop stop
  the pulse stream.
- Confirm C positioning resumes at the dead-reckoned spindle stop angle.
- The large driver is set to 3.0 A / 3.2 A peak (`S4 OFF, S5 ON, S6 OFF`);
  verify that against the motor nameplate before extended holding tests.

## First-Class Turret ATC Audit

Audit target: FluidNC `maijker_5_station_turret` ATC driver

Current behavior:

- Handles `T1` through `T5` through the normal `Tn` + `M6` path.
- Blocks M6 until the current turret station is confirmed by config, `M61Qn`,
  or future sensor home.
- Does nothing if target equals current confirmed tool.
- Computes forward wraparound delta over 5 tools.
- Drives direction internally on `gpio.5`.
- Pulses the turret step input internally on `i2so.7`.
- Moves forward by native station steps plus overshoot.
- Moves reverse by native lock/backoff steps.
- Updates FluidNC current tool only after the ATC reports success.
- Reports turret state through `ESP421`.
- Reports the station as `software_dead_reckoning` and never mechanically
  confirmed while `sensor_pin` is `NO_PIN`.

Commissioning risks:

- If the confirmed tool state is initialized incorrectly after boot, the first
  tool change can index to the wrong station.
- The current config has no physical turret sensor confirmation.
- Open-loop recovery after reset/alarm still requires physical station
  inspection and deliberate `M61Qn` reinitialization.
- The FluidDial V3 UI blocks duplicate M6 sends while pending, but it cannot
  prove the mechanical turret position without reliable FluidNC state and
  operator validation.

Required validation:

- Establish the boot-time procedure for initializing the current station with
  `M61Qn`.
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
- `max_css_rpm: 675.000`
- `x_axis: 0`
- `z_axis: 2`
- `shared_chuck: true`
- `c_axis: 5`
- `feedback_stale_ms: 250`
- `encoder_enable: true`
- `encoder_pulse_pin: gpio.33`
- `encoder_b_pin: gpio.35`
- `encoder_index_pin: gpio.4`
- `encoder_direction_invert: false`
- `encoder_pulses_per_rev: 1000`

Commissioning interpretation:

- Lathe mode is available for UI/profile/status work.
- CSS and feed-per-rev modes are configured, but should be tested without
  cutting load first.
- Threading is explicitly disabled.
- Encoder values describe the installed AS5047P commissioning wiring but must
  not be treated as accepted feedback until the low-speed checklist passes.
- `ESP421` should report stale/unavailable while the encoder is disconnected,
  then live RPM/direction/index/phase during hand rotation.
- `$ESP425` is the adapter/HMI digital-twin snapshot. X/Z positions and tool
  offsets are millimeters, C position is degrees, and executing file line
  provenance comes from the planner block.

## Control and Safety Inputs

Current config pins are all `NO_PIN`:

- `safety_door_pin`
- `reset_pin`
- `feed_hold_pin`
- `cycle_start_pin`
- `macro0_pin` through `macro3_pin`
- `fault_pin`
- `estop_pin`
- `homing_button_pin`

Required machine safety posture:

- Physical E-stop must cut actuator power independently of FluidNC.
- Do not rely on the pendant, WiFi, UART, or FluidNC control pins as the only
  emergency stop path.
- The firmware and adapter must report E-stop feedback as unavailable; a
  software stop/reset is not an emergency stop.
- Validate feed hold/reset behavior separately from the physical E-stop.

## Open Items Before Cutting

| Item | Owner | Status | Notes |
| --- | --- | --- | --- |
| Verify X/Z direction and homing behavior. | | Open | |
| Verify X/Z switch polarity and hard-limit behavior. | | Open | |
| Verify X/Z travel and soft-limit values. | | Open | |
| Define and document boot-time `M61Qn` confirmed-tool initialization. | | Open | |
| Dry-run all turret transitions and same-tool no-op behavior. | | Open | |
| Verify shared-chuck ownership, C direction/scale, and whether `$HC` is allowed. | | Open | |
| Verify probe polarity, startup behavior, and low-feed `G38.2`. | | Open | |
| Verify spindle direction and alarm/E-stop shutdown. | | Open | |
| Install and validate encoder before enabling encoder feedback. | | Open | |

## Adapter and HMI contract

The matching FluidNC branch exposes three bounded commands:

```text
$ESP425
$ESP426=MODE=IDLE|C_POSITIONING|SPINDLE
$ESP427=PROBE,AXIS=X|Z,DISTANCE=<signed-mm>,FEED=<mm-per-minute>
```

`ESP425` is read-only. `ESP426` and `ESP427` require administrator
authentication and reject extra or malformed fields. They do not provide an
arbitrary remote G-code write path. The exact schema, units, nullable fields,
conditions, and stop semantics are in the FluidNC repository document
`docs/tams-fluidnc-telemetry-v1.md`.

Build this board with the `maijker_wifi` PlatformIO environment. It omits the
unused onboard-OLED implementation so the firmware and bundled filesystem fit
the standard 4 MiB layout while retaining two OTA application slots.
| Keep `lathe.enable_threading: false` until a separate encoder/threading checklist passes. | | Open | |
