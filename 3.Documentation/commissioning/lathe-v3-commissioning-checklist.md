# Lathe V3 Commissioning Checklist

Use this checklist before putting the XZA lathe under real cutting load. The
intent is to prove each safety and motion layer separately, then only advance
after the previous layer is stable and repeatable.

Current required posture:

- `lathe.enable_threading: false`
- `lathe.encoder_enable: false`
- `lathe.encoder_pulse_pin: NO_PIN`
- `lathe.encoder_index_pin: NO_PIN`
- Physical E-stop cuts drive/spindle power independently of FluidNC.
- No cutting tool installed until dry motion, homing, limits, and turret motion
  have passed.

## Pre-First-Motion FluidDial Operator Script

Complete this before the first powered motion in a commissioning session:

| Step | Pass | Notes |
| --- | --- | --- |
| Connect FluidDial to FluidNC and wait for stable connected state. | | |
| Open Status and confirm the lathe dashboard is active. | | |
| Confirm the displayed operator axes are X/Z/C. | | |
| Confirm encoder/threading state is visibly unsafe while encoder is disabled. | | |
| Confirm `ESP421` reports turret configured and tool confirmed, or initialize the physical turret station deliberately with `M61Qn`. | | |
| Confirm no pending or recoverable FluidDial lathe command is shown before enabling motion. | | |
| Use only one confirmed M6 command at a time; do not repeat M6 while FluidDial shows `Wait`, `Still waiting`, `Timed out`, or `Alarm during command`. | | |

## Stuck Pending M6 Recovery

If FluidDial shows a recoverable M6 error such as `Timed out` or
`Alarm during command`, do not immediately send another tool change.

| Step | Pass | Notes |
| --- | --- | --- |
| Stop motion and use physical E-stop if there is any uncertainty. | | |
| Verify the turret's physical station and lock state. | | |
| Verify FluidNC alarm/state and clear/reset only after the machine is safe. | | |
| Reinitialize or correct the active turret station deliberately with `M61Qn` if it no longer matches the physical turret. | | |
| Use FluidDial `Clear` only after the physical turret and FluidNC state are understood. | | |
| Send the next M6 only after FluidDial has no pending/recoverable lathe command. | | |

## Session Record

| Date | Operator | FluidNC commit/config | FluidDial commit | Result | Notes |
| --- | --- | --- | --- | --- | --- |
| | | | | | |

## 1. Power-On, No Motors, No Spindle

Goal: prove wiring identity, configuration load, pendant connection, and safe
power behavior before any actuator can move.

| Check | Pass | Notes |
| --- | --- | --- |
| Controller, drivers, spindle supply, and pendant wiring match the wiring notes and current config. | | |
| Physical E-stop removes power from stepper drives and spindle drive without relying on firmware. | | |
| Releasing E-stop does not automatically start motion or spindle output. | | |
| FluidNC boots with `maijker_xzact_mini_lathe.yaml` and reports no config load errors. | | |
| FluidNC reports the `maijker_5_station_turret` ATC driver in startup/config logs. | | |
| FluidDial connects over the intended transport and remains connected for 10 minutes. | | |
| FluidDial Status scene switches to the lathe dashboard when `ESP421` reports `Lathe enabled=true`. | | |
| FluidDial dashboard shows X/Z/C slots, not generic X/Y/Z, on the lathe config. | | |
| FluidDial dashboard shows encoder/threading unsafe while encoder is disabled. | | |
| Reset, feed hold, and connection loss behavior are understood before power is applied to drives. | | |

Stop if any item fails. Do not troubleshoot with motors powered until the no
motor checks are complete.

## 2. Motors Enabled, Spindle Disabled

Goal: prove homing, limits, and jog direction with the spindle incapable of
turning.

Preparation:

- Remove cutting tools or keep them clear of the work envelope.
- Disable spindle power at the drive or disconnect spindle motor power.
- Keep one hand on physical E-stop during first moves.
- Start with conservative jog increments.

| Check | Pass | Notes |
| --- | --- | --- |
| X jog positive moves in the expected operator direction. | | |
| X jog negative moves in the expected operator direction. | | |
| Z jog positive moves in the expected operator direction. | | |
| Z jog negative moves in the expected operator direction. | | |
| C jog commands the intended commanded spindle/C axis without unexpected spindle drive behavior. | | |
| X limit input changes state when manually actuated before homing. | | |
| Z limit input changes state when manually actuated before homing. | | |
| `$HX` homes X in the expected direction, backs off, and leaves the axis clear of the switch. | | |
| `$HZ` homes Z in the expected direction, backs off, and leaves the axis clear of the switch. | | |
| `$HC` is either confirmed safe for the current C setup or intentionally not used. | | |
| `$H` homes all enabled homing axes in the expected sequence. | | |
| Soft limits prevent motion beyond configured X travel. | | |
| Soft limits prevent motion beyond configured Z travel. | | |
| FluidDial Jog scene shows X/Z/C and sends X/Z/C jog commands. | | |
| FluidDial Home scene shows X/Z/C and sends `$HX`, `$HZ`, `$HC`, or `$H` as selected. | | |
| Feed hold and reset stop motion as expected during a safe test move. | | |

Stop if direction, limit polarity, or homing behavior is uncertain.

### Low-speed jog vibration gate

Do not treat maximum driver current as a normal operating target. Excess
current adds heat and can make each commanded step excite the machine more
strongly; it does not repair a resonance, a coarse microstep setting, or a
binding slide.

Before changing a driver switch, record for X and Z:

- Plug-in driver make and exact part number.
- Motor rated phase current.
- Driver current-limit setting or measured Vref.
- All three DLC32 microstep DIP positions.
- Present `steps_per_mm` and a measured 10 mm travel result.

Change DIP switches only with drive power removed. Use the exact driver's
truth table because A4988, DRV8825, and TMC-family modules do not share one
universal switch table. When changing from one microstep ratio to another,
preserve scale with:

```text
new_steps_per_mm = old_steps_per_mm * new_microsteps / old_microsteps
```

At the commissioned `640 steps/mm`, a `0.01 mm` command contains 6.4 step
pulses. The matching X and Z A4988 DIP setting is `1 ON, 2 ON, 3 ON` (1/16).
Verify actual travel with an indicator; do not change `steps_per_mm` without
the matching physical DIP change.

| Low-speed check | Pass | Notes |
| --- | --- | --- |
| Driver current is set from the motor and driver ratings, not simply to maximum. | | |
| X and Z microstep DIP settings and matching `steps_per_mm` are recorded. | | |
| A single `0.01 mm` jog is smooth enough for setup work and lands repeatably. | | |
| Repeated `0.1 mm` jogs do not produce stop/start hammering. | | |
| A continuous `G1` low-feed move is smooth; otherwise inspect current, phase wiring, coupler alignment, gib preload, and screw binding. | | |
| The motor and driver remain within acceptable temperature after a 10-minute hold/motion test. | | |

## 3. Dry Motion, No Tooling

Goal: prove repeatable coordinated motion at low risk.

| Check | Pass | Notes |
| --- | --- | --- |
| `G0 X...` and `G0 Z...` small moves match the expected direction and scale. | | |
| `G1 X... F...` and `G1 Z... F...` low-feed moves match the expected direction and scale. | | |
| Returning to a known point repeats within acceptable mechanical tolerance. | | |
| A low-feed diagonal X/Z move follows the expected path. | | |
| C axis commanded moves are understood and do not conflict with spindle control. | | |
| Alarm recovery does not leave the machine in a confusing modal state. | | |
| FluidDial DRO tracks FluidNC position during motion and after reset. | | |

Do not install a cutting tool until dry motion is repeatable.

## 4. Five-Tool Turret

Goal: prove first-class turret ATC behavior before any turret station can crash into the
workpiece, chuck, or machine.

Preparation:

- Keep the turret physically clear of the spindle/chuck/work envelope.
- Mark each turret station.
- Initialize the current station intentionally with `M61Qn` before the first
  automatic tool change after boot unless `maijker_5_station_turret.current_tool`
  is deliberately configured to a verified station.
- If FluidNC's confirmed tool state is wrong, the first real turret motion can
  index the wrong station. Treat this as a hard commissioning gate, not a
  convenience setting.

| Check | Pass | Notes |
| --- | --- | --- |
| Boot with `current_tool: 0` blocks `Tn M6` until `M61Qn` initializes the physical station. | | |
| `ESP421` reports `Turret configured=true` and `Turret tool confirmed=true` after initialization. | | |
| `T1 M6` from known tool 1 performs no motion. | | |
| `T2 M6` advances exactly one station and locks consistently. | | |
| `T3 M6`, `T4 M6`, and `T5 M6` each advance the expected number of stations from known state. | | |
| `T1 M6` from T5 wraps forward to T1 as expected. | | |
| Repeating `Tn M6` for the active tool performs no motion. | | |
| Backlash compensation and lock move leave each station repeatable. | | |
| Reset or alarm during tool change leaves the operator with a clear recovery procedure. | | |
| FluidDial Tools page shows T1-T5 and labels T5 as Probe. | | |
| FluidDial confirms tool change before sending `Tn` and `M6`. | | |
| FluidDial shows `Wait` while the M6 action is pending and blocks duplicate M6 sends. | | |
| FluidDial clears the pending state after `ESP421` reports the requested active tool. | | |
| T5 is treated as the contact/probe station, not a normal cutting tool. | | |

Do not cut with turret-indexed tools until active tool tracking is trusted after
power cycles, resets, and alarms.

## 5. Probe and Manual Touch-Off

Goal: prove contact sensing and the manual `ESP423` touch-off flow without
automated probe-to-offset conversion.

| Check | Pass | Notes |
| --- | --- | --- |
| Probe input is electrically verified with no machine motion. | | |
| Probe polarity matches FluidNC expectations. | | |
| Probe check at startup is intentional and does not block boot with the expected wiring state. | | |
| A very low-feed `G38.2` test move stops on contact and alarms safely on no-contact. | | |
| T5 probe/contact station is mechanically repeatable. | | |
| FluidDial Probe scene remains available and profile-aware. | | |
| FluidDial Touch Off page displays current machine X/Z positions. | | |
| FluidDial X touch-off defaults to the current `ESP421` diameter/radius mode. | | |
| FluidDial asks for confirmation before sending `[ESP423]`. | | |
| `[ESP423]` updates are first tested with harmless reference values and verified through `ESP421`. | | |
| The operator understands that V3 does not automatically convert a `G38.2` result into `[ESP423]`. | | |

## 6. Spindle Enabled, No Cutting

Goal: prove spindle command behavior before putting tool pressure on the machine.

| Check | Pass | Notes |
| --- | --- | --- |
| Physical E-stop removes spindle power. | | |
| `M3 S...`, `M4 S...`, and `M5` behave as expected at low speed. | | |
| `S0` and disabled spindle states behave as expected with the HBridge config. | | |
| FluidNC alarm stops spindle output because `HBridge.off_on_alarm` is true. | | |
| CSS mode commands are understood, but no cutting is performed during this check. | | |
| Feed-per-rev commands are understood, but threading remains disabled. | | |
| FluidDial dashboard displays programmed/effective RPM and measured RPM as available. | | |

## 7. Encoder and Threading Gate

Goal: define the future gate. Do not execute threading validation until encoder
hardware is installed and the config is intentionally changed.

Required before changing threading state:

- Encoder pulse and index wiring installed.
- Encoder pins updated from `NO_PIN` to verified hardware pins.
- `lathe.encoder_enable: true` only after low-speed feedback tests pass.
- `lathe.encoder_pulses_per_rev` set to the real encoder value.
- `ESP421` reports live measured RPM.
- `ESP421` reports index detection when an index channel is installed.
- `ESP421` reports angular position/revolution state without stale or fault
  flags at low spindle speed.
- FluidDial dashboard shows encoder feedback healthy.
- Only then consider a separate threading validation checklist.

| Check | Pass | Notes |
| --- | --- | --- |
| Encoder pulse input verified at low speed. | | |
| Encoder index input verified at low speed. | | |
| `ESP421` measured RPM is stable and plausible. | | |
| `ESP421` stale/fault flags remain clear during low-speed spindle tests. | | |
| `lathe.enable_threading` remains false until all encoder checks pass. | | |

## Release Gate

The machine is not ready for real cutting until sections 1 through 6 have passed
in order and the failures, if any, have been corrected in config, wiring, or
procedure. Threading remains out of scope until section 7 has a separate passed
encoder validation record.
