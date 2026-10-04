# SIM7600NA-H Android Integration Plan

**Status:** Planning and validation guide. Validate the purchased module firmware and its USB composition before Android platform work begins.

This document covers the **SIMCom SIM7600NA-H-PCIE** modem with a Raspberry Pi Compute Module 5 (CM5) running a community Android image.

## Confirmed Module Boundary

The selected non-`A` Series-PCIE variant is not the analog-audio variant.

| Pin | Non-`A` Series-PCIE function | OpenXTCM5 use |
| --- | --- | --- |
| 1, `WAKE#/MICP` | Host wake-up I/O | Optional future modem wake signal. Do not route to the electret microphone. |
| 3, `MICN` | No connect | No connect. |
| 5, `EARP` | No connect | No connect. |
| 7, `EARN` | No connect | No connect. |

The separate `Series-PCIEA` variant adds analog microphone and receiver connections. It must not be assumed to be electrically or functionally equivalent to the selected SIM7600NA-H-PCIE module.

## Audio Ownership

The front-panel electret microphone remains part of the head-unit audio path:

```text
Electret microphone -> TLV320AIC3104 codec mic input / ADC -> USB audio bridge -> CM5
CM5 media -> USB audio bridge -> I2S -> TLV320AIC3104 DAC -> analog amplifier -> speakers
```

Do not fit a microphone bias, coupling, or analog receiver circuit for the standard SIM7600NA-H-PCIE pins above. Do not raw-parallel an electret microphone between two codecs.

Whether the exact purchased modem exposes a USB Audio Class interface remains a validation item, not a board dependency. Multiple USB Audio Class devices do not inherently conflict on USB, but Android routing and telephony integration would still require deliberate platform work.

## First-Board Scope

1. USB modem enumeration.
2. USB serial AT-command control.
3. Mobile data using the networking interface exposed by the module firmware.
4. Controlled modem power, reset, and fault reporting.
5. RIL/HAL investigation only after basic Linux-level control and data are stable.

Reliable Android telephony and call audio are outside the first-board acceptance criteria. They require an explicitly validated voice path and Android integration.

## Hardware Controls

| Signal | Owner | Purpose |
| --- | --- | --- |
| `MODEM_PWR_EN` | GPIO expander P07 | Enables the switched modem 3.3 V rail. |
| `MODEM_PERST_ASSERT` | GPIO expander P10 | Holds or releases modem reset according to the module timing requirements. |
| `MODEM_3V3_FLT_N` | GPIO expander P17 | Reports the modem rail power-switch fault. |
| `MODEM_WAKE_N` | TBD | Optional wake indication after its polarity, voltage range, and power-off behavior are measured from the actual module. |

Do not connect `MODEM_WAKE_N` directly to a 3.3 V controller input until its behavior is confirmed. Add level translation or protection if its limits require it.

## Validation Gates

Before committing to RIL, Android audio policy, or app work:

1. Identify every USB interface exposed by the exact modem and firmware.
2. Identify the working AT-command serial interface. Do not assume a particular `/dev/ttyUSB*` number.
3. Validate SIM detection, registration, APN configuration, and data connectivity from Linux.
4. Query supported vendor commands, for example with `AT+CLAC`, before relying on a modem-specific command.
5. Measure `WAKE#/MICP` voltage, polarity, and state through boot, normal operation, sleep, and rail-off conditions.
6. Treat USB networking, RIL/HAL integration, IMS/telephony, and call audio as independent milestones.

## Android and Linux Baseline

The baseline kernel work should cover only the functions that are actually enumerated:

```ini
CONFIG_USB_SERIAL=y
CONFIG_USB_SERIAL_OPTION=y
CONFIG_USB_NET_DRIVERS=y
CONFIG_USB_NET_CDCETHER=y
CONFIG_USB_NET_QMI_WWAN=y
CONFIG_USB_NET_RNDIS_HOST=y
```

Add audio kernel or Android audio-policy support only if the production-intent modem demonstrably exposes a usable audio interface and the final voice design needs it.

## Future Voice Options

1. Source and validate an explicit analog-audio `Series-PCIEA` module.
2. Use a modem with documented and accessible USB voice-audio support.
3. Design a deliberate codec or USB-audio bridge for call media after the Android telephony path is proven.

The first revision prioritizes stable data, power sequencing, and a clean head-unit media path over an unverified modem-call-audio design.
