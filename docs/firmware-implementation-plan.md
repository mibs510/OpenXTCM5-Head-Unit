# OpenXTCM5 Firmware Implementation Plan

This document is the working hardware-to-firmware contract for the RP2354 controller. It records the current schematic intent, not a promise that every optional subsystem is part of the first bring-up.

## System Roles

- **CM5 / Android:** user interface, USB host, audio source and capture client, modem integration, and camera application.
- **RP2354:** real-time vehicle I/O, power sequencing, CAN control, front-panel acquisition, audio-codec control, fan control, and fault handling.
- **TCA9539A GPIO expander:** low-rate rail enables and active-low fault inputs over `SYS_I2C`.

## Firmware Platform and Audio-Bridge Decision

Rev A uses the **Raspberry Pi Pico SDK and TinyUSB** for RP2354 firmware. The first bridge does not require an RTOS: PIO and DMA own time-sensitive I2S movement, while the remaining application is an event-driven control plane for vehicle policy, codec configuration, USB management, and diagnostics. FreeRTOS remains an option only after the audio path is proven and a measured scheduling need exists.

The RP2354 is the bridge in the approved runtime topology:

```mermaid
flowchart LR
    CM5[CM5 / Android USB host] -->|UAC1 PCM playback| RP[RP2354\nTinyUSB + PIO/DMA]
    CM5 <-->|CDC ACM management| RP
    RP -->|I2S master| Codec[TLV320AIC3104]
    Codec -->|L/R analog| Amp[TDA7850]
    Amp --> Speakers[Four speaker channels]
```

The initial USB Audio Class 1 profile is **48 kHz, 16-bit, stereo playback**. It is paired with a CDC ACM management interface in one composite runtime USB device. Codec ADC/microphone capture is a later increment after playback is stable. The normal interface is different from explicit BOOTSEL recovery: BOOTSEL exits application firmware and re-enumerates as the RP2354 Boot ROM device.

Zephyr is deferred rather than rejected. The upstream Zephyr project currently has no maintained RP Pico PIO I2S driver; adopting it now would make this project owner of a custom I2S/DMA/full-duplex driver and its audio regression work. Revisit Zephyr only when upstream support exists or the project deliberately accepts that ownership. See [RP2354 Firmware Platform Decision](rp2354-firmware-platform.md) for the rationale, implementation sequence, and acceptance criteria.

## Power-Lifecycle Ownership

`ACC_OK` is an electrical observation. `SYS_HOLD` is a power-latch command. They must not have the same owner.

- `ACC_OK` tells the RP2354 that the vehicle ignition source is no longer present.
- RP2354 GPIO21 drives `SYS_HOLD` through the existing D701 diode-OR path. It is the sole owner of the finite grace period that keeps `+5V_SYS` alive after `ACC_OK` falls.
- The CM5 requests an orderly shutdown and reports its progress over the CM5-to-RP2354 management interface. It does **not** drive `SYS_HOLD` directly.
- The RP2354 must release `SYS_HOLD` after a bounded timeout even if the CM5, its USB link, or Android is unresponsive. A computer crash must never leave the main rail held on indefinitely from the vehicle battery.

`SYS_HOLD` is therefore a **shutdown lease**, not a sleep, suspend, or hibernation signal. The first board uses a clean shutdown followed by a cold boot on the next ACC event. Do not depend on suspend-to-disk or RAM retention: once `SYS_HOLD` is released, the 5 V system rail and CM5 RAM power are removed.

The CM5 supports a configurable VPU-sleep shutdown behaviour, but that is not the same as a validated Android hibernation strategy. It would require keeping the main power domain alive and measuring the resulting parked-vehicle current. Treat it as a future, explicitly validated optimization rather than a first-board requirement. The CM5 data sheet instead provides `PWR_BUT` for a brief power-state request and recommends using `PMIC_ENABLE` only after the OS has shut down. [CM5 data sheet, system-control signals](https://datasheets.raspberrypi.com/cm5/cm5-datasheet.pdf)

## CM5 GPIO Assignment

The CM5 owns Android-facing interfaces and the deliberate recovery controls exposed to it. `GPIO_VREF` is strapped to the CM5 3.3 V rail, so the signals below use 3.3 V logic. I2C labels are bidirectional because both I2C devices may pull either open-drain line low.

| CM5 GPIO | Module pin | Net | Direction | Role |
| --- | --- | --- | --- | --- |
| GPIO2 | 58 | `~{TOUCH_RST}` | Output | GT911 active-low reset request |
| GPIO3 | 56 | `TOUCH_INT` | Bidirectional | GT911 interrupt and address-selection strap |
| GPIO4 | 54 | `~{LCD_RESET}` | Output | Reserved panel active-low reset, translated from the CM5 3.3 V domain to `LCD_1V8_SW`; pending final hierarchical connection |
| GPIO5 | 34 | `~{LCD_STBYB}` | Output | Reserved panel active-low standby control, translated to `LCD_1V8_SW`; pending final hierarchical connection |
| GPIO10 | 44 | `CM5_SDA` | Bidirectional | I2C1 data to the GT911 touch controller |
| GPIO11 | 38 | `CM5_SCL` | Bidirectional | I2C1 clock to the GT911 touch controller |
| GPIO17 | 50 | `RP2354_TO_CM5_ATTN_N` | Input, active low | Proposed RP2354 open-drain attention interrupt; wakes the CM5 service to read the management channel. This replaces the ambiguous provisional name `CM5_WAKE_OR_IRQ`. |
| GPIO22 | 46 | `RP2354_RESET_ASSERT` | Output | Active-high command to Q1001; asserting it pulls RP2354 `RUN` low |
| GPIO27 | 48 | `RP2354_BOOTSEL_ASSERT` | Output | Active-high command to Q1002; asserting it pulls RP2354 `QSPI_SS` low for BOOTSEL recovery |

The two RP2354 recovery controls are reserved on the child sheets but still need matching parent-sheet links when the hierarchy is completed. They are intentionally not direct connections to `RUN` or `QSPI_SS`.

## RP2354 GPIO Assignment

| GPIO | Net | Role |
| --- | --- | --- |
| GPIO0 | `UART0_TX` | Debug UART transmit |
| GPIO1 | `UART0_RX` | Debug UART receive |
| GPIO2 | `CAN_SPI_SCK` | TCAN4550 SPI clock |
| GPIO3 | `CAN_SPI_MOSI` | TCAN4550 SPI controller-out |
| GPIO4 | `CAN_SPI_MISO` | TCAN4550 SPI controller-in |
| GPIO5 | `CAN_CS` | TCAN4550 active-low chip select |
| GPIO6 | `CAN_INT` | TCAN4550 active-low interrupt |
| GPIO7 | `CAN_RESET` | TCAN4550 reset |
| GPIO8 | `CM5_PWR_BUT_ASSERT` | Proposed active-high command to an open-drain transistor in parallel with SW1401. A short assertion pulls CM5 `PWR_BUT` low as a controlled wake/shutdown fallback. |
| GPIO9-GPIO11 | `UNASSIGNED` | Available for a future board-level function; LIN is not part of the current design |
| GPIO12 | `I2S_BCLK` | Audio serial bit clock |
| GPIO13 | `I2S_LRCLK` | Audio left/right frame clock |
| GPIO14 | `I2S_DOUT` | Playback data to codec |
| GPIO15 | `I2S_DIN` | Capture data from codec |
| GPIO16 | `I2S_MCLK` | Codec master clock |
| GPIO17 | `GPIO_EXP_INT` | GPIO expander active-low interrupt |
| GPIO18 | `GPIO_EXP_RESET` | GPIO expander active-low reset |
| GPIO19 | `AMP_STBY_CTL` | Amplifier standby control, level-shifted to the amplifier domain |
| GPIO20 | `AMP_MUTE_CTL` | Amplifier mute control, level-shifted to the amplifier domain |
| GPIO21 | `SYS_HOLD` | Sole owner of the bounded main-5-V shutdown lease after `ACC_OK` falls; low releases the latch |
| GPIO22 | `FAN_PWM` | Four-wire fan PWM control |
| GPIO23 | `FAN_TACH` | Four-wire fan tachometer input |
| GPIO24 | `AUDIO_CODEC_RESET` | Active-high request to assert the codec's active-low reset through Q601 |
| GPIO29 | `IMMO_RELAY_EN` | Low-side immobilizer relay control |
| GPIO30 | `REVERSE_OK` | Reverse comparator status |
| GPIO32 | `RP2354_TO_CM5_ATTN_N` | Proposed active-low, open-drain CM5 attention interrupt; use CM5 GPIO17 / module pin 50, not as a data channel |
| GPIO34 | `ILLUM_OK` | Illumination comparator status |
| GPIO35 | `ACC_OK` | ACC comparator status |
| GPIO36 | `AUDIO_I2C_SDA` | Audio codec I2C data |
| GPIO37 | `AUDIO_I2C_SCL` | Audio codec I2C clock |
| GPIO38 | `SYS_I2C_SDA` | System I2C data |
| GPIO39 | `SYS_I2C_SCL` | System I2C clock |

## CM5-RP2354 Management and Shutdown Contract

The planned internal CM5-host-to-RP2354 USB connection, `MCU_USB_D+` / `MCU_USB_D-`, has two mutually exclusive modes:

1. **Normal operation:** the RP2354 enumerates as a composite device with a UAC1 audio function and a CDC ACM management function. The CM5 board-management daemon exchanges framed, versioned messages over CDC ACM without consuming more CM5 GPIOs; audio uses the independent UAC1 interface.
2. **Maintenance recovery:** when the CM5 asserts the BOOTSEL/reset controls, the same physical USB link re-enumerates as the RP2354 Boot ROM device. This is not the normal runtime protocol.

The USB management interface is the source of truth for commands and status. The proposed `RP2354_TO_CM5_ATTN_N` line is a low-latency, active-low open-drain doorbell only: it tells the CM5 to read USB immediately. It must not encode vehicle state or substitute for acknowledgements. Pull it up to the live CM5 GPIO domain, route it to CM5 GPIO17 / module pin 50, and ensure its inactive state is high.

Before routing, add the proposed `CM5_PWR_BUT_ASSERT` low-side transistor in parallel with SW1401. It is an RP2354-controlled fallback that briefly pulls CM5 `PWR_BUT` low; it is not connected to `SYS_HOLD`, and it must not be a push-pull drive into the CM5. Leave `PMIC_ENABLE` out of normal firmware control. It may only be used as a post-shutdown recovery mechanism if a later validation plan specifically requires it.

### Runtime Messages

The first protocol revision needs only a small, observable set of messages:

| Direction | Message | Meaning |
| --- | --- | --- |
| RP2354 -> CM5 | `IGNITION_OFF_PENDING` | `ACC_OK` fell; includes the configured shutdown deadline. |
| CM5 -> RP2354 | `SHUTDOWN_ACCEPTED` | Android has accepted the request and begun its orderly shutdown. |
| CM5 -> RP2354 | `SHUTDOWN_READY` | Filesystems and services are quiesced; the RP2354 may release `SYS_HOLD`. Sent immediately before the CM5's final power-off action. |
| RP2354 -> CM5 | `IGNITION_RESTORED` | ACC returned before the rail was released; cancel a still-reversible shutdown. |
| RP2354 -> CM5 | `POWER_CUT_IMMINENT` | Optional final notice before a timeout-forced release. |
| Both directions | `HELLO`, `HEARTBEAT`, `STATUS` | Protocol version, liveness, and power/fault state. |

Use a versioned frame with a message type, sequence number, and explicit payload length. The RP2354 retains the policy decision even when the link is down.

### ACC-Off Sequence

The initial configurable values are deliberately conservative: a 30-second total shutdown lease and a short final-cut notice. They are firmware configuration values, not fixed electrical timing requirements.

```mermaid
sequenceDiagram
    participant ACC as ACC_OK comparator
    participant RP as RP2354 power manager
    participant CM5 as CM5 board-management daemon
    participant Rail as SYS_HOLD / +5V_SYS

    ACC->>RP: ACC_OK falls
    RP->>Rail: Assert SYS_HOLD; start bounded deadline
    RP->>CM5: IGNITION_OFF_PENDING over runtime USB
    RP->>CM5: Pulse ATTN_N if needed
    CM5-->>RP: SHUTDOWN_ACCEPTED
    Note over CM5: Quiesce Android and flush storage
    CM5-->>RP: SHUTDOWN_READY
    RP->>Rail: Release SYS_HOLD after short guard time
    Rail-->>CM5: Main 5 V removed; next start is cold boot

    Note over RP,Rail: No ACCEPTED or READY before deadline: release SYS_HOLD anyway
```

If `ACC_OK` returns while the CM5 has not crossed its final shutdown boundary, RP2354 sends `IGNITION_RESTORED` and retains the rail. If the CM5 has already completed shutdown while ACC remains high, RP2354 can issue a brief `CM5_PWR_BUT_ASSERT` pulse to request a normal CM5 start. Critical low-input protection may shorten or bypass the normal lease; it must still leave `SYS_HOLD` released when the RP2354 resets.

## Amplifier Fan Control

`GPIO22` controls the high-power, four-wire fan at `J1301`. This is the amplifier-cooling fan and is electrically separate from the low-power CM5 fan header, `J1401`. `J1401` is controlled and monitored by the CM5 through its dedicated `FAN_PWM` and `FAN_TACHO` pins; it is not part of RP2354 fan policy.

Use a **25 kHz, active-low PWM configuration** for `J1301`: full fan speed means Q1301 is off and the fan PWM line is high. Q1301 is an open-drain pull-down, so RP2354 `FAN_PWM` high turns Q1301 on and pulls the fan line low. Firmware must therefore invert its gate-drive duty convention: a 100% fan request holds `GPIO22` low, leaving Q1301 off continuously; a 0% request holds it high, pulling the fan line low continuously. Validate the selected fan's permitted minimum duty cycle and startup behavior during board bring-up.

`GPIO23` reads the tachometer from `J1301` through the high-power fan-control sheet. Keep this amplifier-fan tach path named `FAN_TACH` and distinct from the CM5 header's `FAN_TACHO` signal.

## CM5-to-RP2354 Firmware Update and Recovery

The recovery route reserves a CM5 USB-host path to the RP2354 USB data pair, `MCU_USB_D+` and `MCU_USB_D-`. The matching parent-sheet USB link still needs to be completed. The CM5 also owns two active-high command signals that operate local, low-side MOSFET gates on the RP2354 sheet:

- `RP2354_RESET_ASSERT` drives Q1001. A high turns Q1001 on and pulls RP2354 `RUN` low through R1004.
- `RP2354_BOOTSEL_ASSERT` drives Q1002. A high turns Q1002 on and pulls `QSPI_SS` low through R1012.

The CM5 must never drive `RUN` or `QSPI_SS` directly. The gate pulldowns, R1006 and R1013, keep both controls released while the CM5 is reset or its pins are high impedance. SW1001 and SW1002 remain independent local recovery controls.

### USB BOOTSEL Sequence

This is a development and recovery path, not a normal vehicle-start sequence.

1. Confirm the RP2354 3.3 V rail and CM5 USB-host path are powered. Configure both CM5 control GPIOs inactive: output low or high impedance.
2. Assert `RP2354_BOOTSEL_ASSERT` high. Allow a configurable settling delay; 1 ms is a suitable initial firmware value.
3. Assert `RP2354_RESET_ASSERT` high, holding RP2354 `RUN` low. Hold reset for a configurable dwell; begin with 1 ms.
4. Release `RP2354_RESET_ASSERT` by driving it low or returning the GPIO to input mode.
5. Keep `RP2354_BOOTSEL_ASSERT` high through the Boot ROM sample of `QSPI_SS`, then release it. The RP2350 documentation describes this sample as occurring shortly after reset, so use a conservative, firmware-configurable 10 ms initial hold and confirm it on the bench.
6. Wait for the RP2354 USB boot device to enumerate, then transfer the firmware image using a supported host flow such as PICOBOOT or UF2.
7. Before any normal reboot, ensure both commands are inactive. If the update flow times out or the CM5 resets, firmware must release both controls rather than leave the RP2354 held in reset or BOOTSEL.

Holding `QSPI_SS` low during reset selects the RP2350-family BOOTSEL path. With the standard configuration, the Boot ROM then enters the USB bootloader because `QSPI_SD1` is internally pulled low. The recovery clocking path requires a suitable 12 MHz XOSC configuration; validate the populated crystal and its load network before relying on USB recovery in production hardware.

References: [RP2350 Datasheet, Section 5.2.8](https://pip-assets.raspberrypi.com/categories/1214-rp2350/documents/RP-008373-DS-2-rp2350-datasheet.pdf?disposition=inline) and [Hardware Design with RP2350, Section 3.1](https://pip-assets.raspberrypi.com/categories/1214-rp2350/documents/RP-008280-DS-1-hardware-design-with-rp2350.pdf?disposition=inline).

## CM5 Storage Recovery Through J1402

`J1402` is the CM5's external USB 2.0 **device/UFP** maintenance port. It is intended for a directly connected development host to recover or program the selected non-Lite CM5's onboard eMMC/flash when `nRPIBOOT` is asserted. This is separate from the internal CM5-host-to-RP2354 BOOTSEL route above.

The board must be powered normally from its vehicle or bench supply before J1402 is connected. J1402 VBUS is not allowed to power `+5V_SYS`, and firmware must never assume that a PC or powered USB hub can supply board operating current. Recovery validation must prove the complete sequence: apply board power, assert `nRPIBOOT`, enumerate on the host, write and verify the image, release `nRPIBOOT`, then power-cycle into a normal boot.

J1402 is not an Android runtime USB host port. Its hardware uses independent 5.1 kOhm UFP Rd pull-downs on CC1 and CC2, leaves VBUS NC, and keeps CM5 `CC1`/`CC2` NC. Any future VBUS attachment indication must be high impedance and protected.

## GPIO Expander Assignment

The TCA9539A runs on `SYS_I2C`; all listed `FLT` inputs are active low.

| Pin | Net | Direction | Role |
| --- | --- | --- | --- |
| P00 | `LCD_BIAS_EN` | Output | Enable LCD 5 V bias switch |
| P01 | `LCD_1V8_EN` | Output | Enable LCD 1.8 V switch |
| P02 | `TOUCH_3V3_EN` | Output | Enable touch 3.3 V switch |
| P03 | `BACKLIGHT_EN` | Output | Enable backlight subsystem |
| P04 | `USB_VBUS_EN` | Output | Enable front USB VBUS |
| P05 | `CAMERA_5V_EN` | Output | Enable camera decoder 5 V rail |
| P06 | `CAMERA_12V_EN` | Output | Allow camera 12 V rail; `REVERSE_OK` remains the hardware interlock |
| P07 | `MODEM_PWR_EN` | Output | Enable modem 3.3 V power switch |
| P10 | `MODEM_PERST_ASSERT` | Output | Assert modem `PERST#` through the open-drain transistor |
| P11 | `STATUS_LED` | Output | Board status LED |
| P12 | `LCD_BIAS_FLT` | Input | LCD bias switch fault |
| P13 | `TOUCH_3V3_FLT` | Input | Touch rail switch fault |
| P14 | `USB_VBUS_FLT` | Input | USB VBUS switch fault |
| P15 | `CAMERA_5V_FLT` | Input | Camera 5 V switch fault |
| P16 | `CAMERA_12V_FLT` | Input | Camera 12 V switch fault |
| P17 | `MODEM_3V3_FLT` | Input | Modem 3.3 V switch fault |

## Audio Codec and Amplifier

### Single-Ended Codec Line Outputs

The TLV320AIC3104 is configured for **single-ended** line output into the TDA7850. Firmware must configure the DAC routing for the positive line outputs only; do not enable a differential line-output path for the first board revision.

| Codec output | TDA7850 inputs | Speaker channels |
| --- | --- | --- |
| `LINE_OUT_L+` / `LEFT_LOP` | `IN1`, `IN3` | Front Left and Rear Left |
| `LINE_OUT_R+` / `RIGHT_LOP` | `IN2`, `IN4` | Front Right and Rear Right |

`LINE_OUT_L-` and `LINE_OUT_R-` are intentionally not routed to the amplifier. This duplicates stereo left/right playback into the corresponding front and rear amplifier channels. Keep the codec and amplifier muted until codec clocks, DAC routing, and output common-mode have settled.

### Codec Control and I2S

The TLV320AIC3104 is controlled over `AUDIO_I2C_SDA` and `AUDIO_I2C_SCL`. The RP2354 is the intended I2S clock master for the codec:

- `I2S_MCLK`, `I2S_BCLK`, and `I2S_LRCLK` are clocks from RP2354 to codec.
- `I2S_DOUT` carries playback samples from RP2354 to codec.
- `I2S_DIN` carries microphone capture samples from codec to RP2354.
- `AUDIO_CODEC_RESET` drives Q601. A GPIO high pulls the codec reset pin low and asserts reset; GPIO low releases it through the codec-side 1.8 V pull-up.

Use PIO plus DMA ring buffers between USB packets and the I2S state machines. USB packet timing and codec clocks are independent, so firmware must expose underrun and overrun counters and mute the amplifier for stream loss, codec reset, USB disconnect, or an audio-data fault. Do not treat a larger buffer as a substitute for a measured clock-synchronization policy. The initial UAC1 playback bring-up may use a simple synchronous profile; any observed long-run drift must result in an explicit, validated synchronization decision before audio is declared ready.

### Audio Startup and Shutdown

1. Assert codec reset and hold the amplifier muted and in standby.
2. Enable/configure codec clocks, I2C registers, microphone bias, ADC/PGA, DAC, and the single-ended L+/R+ output paths.
3. Start I2S streams.
4. Release codec reset, allow the documented analog settle interval, then release amplifier standby and mute in the TDA7850-required order.
5. On shutdown, mute the amplifier first, stop streams, then assert codec reset if its rail is being removed.

## Rail and Peripheral Sequencing Rules

- `5V_SYS_GOOD` qualifies system-rail startup. The RP2354 must not enable optional loads until the system rail is valid.
- Use the GPIO expander only for low-rate enables and fault observation, not timing-critical audio or CAN signals.
- Before disabling `TOUCH_3V3_SW`, drive `TOUCH_RST` low or configure it high impedance. The 1 kOhm resistor limits accidental injection but must not be relied on to protect an unpowered touch controller indefinitely.
- Keep LCD logic power, LCD bias, touch power, backlight, USB VBUS, camera rails, and modem power independently recoverable. A subsystem fault should permit a targeted off/on reset without dropping the CM5.
- Camera 12 V requires both firmware permission (`CAMERA_12V_EN`) and the reverse-state interlock (`REVERSE_OK`).
- A modem power-up sequence enables `MODEM_3V3_SW`, waits for the rail to stabilize, releases `PERST#`, and monitors `MODEM_3V3_FLT`.

## Bring-Up Order

1. Validate RP2354 debug UART, SWD, and `SYS_I2C`.
2. Validate the GPIO expander reset, interrupt, rail enables, and fault polarities.
3. Enumerate the composite UAC1-plus-CDC ACM device on a Linux host; validate management traffic while 48 kHz, 16-bit stereo playback runs.
4. Bring up the audio codec over I2C and PIO/DMA I2S with the amplifier held muted/standby.
5. Validate codec line output at the amplifier inputs before connecting speakers, then prove mute/standby sequencing and the fan policy.
6. Bring up CAN/SPI and vehicle state handling while collecting audio underrun/overrun counters during stress testing.
7. Validate RP2354 USB BOOTSEL recovery from the CM5, including normal boot, the manual switches, host-controlled entry, enumeration, programming, and timeout release.
8. Add capture audio, display/touch sequencing, camera control, modem power/reset, and Android integration incrementally.
