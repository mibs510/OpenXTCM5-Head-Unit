# OpenXTCM5 Documentation

These notes explain the decisions that a schematic alone cannot capture. They are living design records, not production guarantees.

| Document | Purpose |
| --- | --- |
| [Android Bring-Up](android-bring-up.md) | Staged CM5 Android, kernel, device-tree, display, touch, USB, modem, audio, camera, and vehicle-integration plan. |
| [Power Control Rationale](power-control-rationale.md) | Power-domain decisions, sequencing, fault handling, switch ownership, and the current RP2354 control allocation. |
| [Firmware Implementation Plan](firmware-implementation-plan.md) | Hardware-to-firmware contract, including GPIO allocation and RP2354 USB recovery. |
| [RP2354 Firmware Platform Decision](rp2354-firmware-platform.md) | Rev A Pico SDK/TinyUSB decision, RP2354 USB Audio-to-I2S bridge contract, validation gates, and Zephyr revisit criteria. |
| [SIM7600NA-H Android Integration Plan](sim7600na-h-android-integration.md) | Planned modem USB, Android, UAC call-audio, and validation work for the SIM7600NA-H-PCIE. |
| [MIPI CSI-2 Reverse Camera Integration Plan](mipi-csi-camera-integration.md) | Staged future plan for native CVBS-to-CSI-2 capture, CM5 Linux media bring-up, Android camera exposure, and the retained MS2106E fallback. |
| [OpenXTCM5 Portfolio Draft](portfolio/openxtcm5-head-unit/openxtcm5-head-unit.md) | Long-form design journal with editable Mermaid diagrams, written for a personal development portfolio. |

## Reading Order

1. Start with the repository [README](../README.md) for the project purpose and current state.
2. Use the Android bring-up guide as the top-level implementation order for CM5 software and kernel work.
3. Read the power-control rationale before changing a switched rail or its firmware behavior.
4. Read the RP2354 firmware-platform decision before starting MCU firmware. It defines the direct CM5 USB to RP2354 to I2S audio path and the boundary between runtime audio, board management, and BOOTSEL recovery.
5. Use the modem integration plan as a staged validation checklist. It contains proposed implementation material that must be confirmed on the actual modem firmware and Android build.
6. Treat the MIPI CSI-2 reverse-camera plan as a later architecture path. It intentionally preserves the MS2106E USB path as the first-board fallback.

More design records will be added as major subsystems settle.

## Naming Convention

- Project-authored documents and directories use lowercase kebab-case: `firmware-implementation-plan.md`.
- `README.md` retains its conventional repository-documentation name.
- Vendor-supplied documents and their source directories retain their published part-number names so their provenance remains obvious.
