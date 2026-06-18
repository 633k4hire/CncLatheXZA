# CncLatheXZA

Micro-manufacturing CNC lathe for batch machining.

This repository holds the physical machine artifacts for the XZA lathe: CAD,
wiring notes, FluidNC configuration, commissioning records, BOM material, and
generated G-code outputs. Reusable firmware work lives in the sibling firmware
repositories in the `XZA-Lathe` super-repo.

## Current FluidNC Config

The active machine configuration is in
[`2.Software/FluidNC/current-config`](2.Software/FluidNC/current-config):

- [`maijker_xzact_mini_lathe.yaml`](2.Software/FluidNC/current-config/maijker_xzact_mini_lathe.yaml)
  - MKS-DLC32 V2.1 FluidNC config for the current XZACt mini lathe build.
- [`maijker_tool_change.gcode`](2.Software/FluidNC/current-config/maijker_tool_change.gcode)
  - five-tool turret `M6` macro referenced by the config.
- [`lathe-v3-config-audit.md`](2.Software/FluidNC/current-config/lathe-v3-config-audit.md)
  - audit of the checked-in config and the physical validation items that remain
    open before cutting.

## Commissioning

Use the staged
[`Lathe V3 Commissioning Checklist`](3.Documentation/commissioning/lathe-v3-commissioning-checklist.md)
before putting the machine under real cutting load.

Threading is intentionally disabled until a physical spindle encoder is
installed and low-speed feedback is verified through FluidNC `ESP421`.

## Reference Images

![XZA lathe](https://github.com/633k4hire/CncLatheXZA/assets/17692800/dbd7f1f0-2fb8-444c-b307-a07dbbfa5788)
![Lathe v2](https://github.com/user-attachments/assets/c82f8129-68f9-4a93-b50b-f9c98cac6c39)
