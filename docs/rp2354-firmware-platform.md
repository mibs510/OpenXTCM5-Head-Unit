# RP2354 Firmware Platform Decision

**Status:** Accepted for Rev A implementation. Hardware, descriptors, and timing still require bench validation.  
**Decision date:** 2026-10-03

## Decision

Rev A firmware for the RP2354 will use the **Raspberry Pi Pico SDK and TinyUSB**. The first audio-bridge implementation will not require an RTOS. PIO and DMA handle time-sensitive I2S transfer; the remaining work is a small event-driven control application for vehicle policy, codec control, USB management, and diagnostics.

The RP2354 is the USB audio bridge. The approved signal path is:

```mermaid
flowchart LR
    CM5[CM5 / Android\nUSB host] -->|UAC1 isochronous PCM\n48 kHz, 16-bit, stereo playback| RP[RP2354\nTinyUSB + PIO/DMA]
    CM5 <-->|CDC ACM management\nversioned control protocol| RP
    RP -->|MCLK, BCLK, LRCLK, DOUT, DIN\nPIO + DMA, I2S master| Codec[TLV320AIC3104\nI2S slave / DAC / ADC]
    Codec -->|Analog L/R| Amp[TDA7850]
    Amp -->|Four amplified channels| Harness[Rear connector\nvehicle harness / speakers]
```

`AUDIO_I2C_SDA` and `AUDIO_I2C_SCL` configure the TLV320AIC3104. They are a control path, not part of audio sample transport.

## Why This Platform

The Pico SDK gives the project a maintained RP2350-family build environment and a direct path to PIO and DMA. `pico-extras` contains a PIO I2S implementation, and the Pico examples include TinyUSB USB-audio examples for both RP2040 and RP2350 families. TinyUSB supplies the USB device implementation and supports USB Audio Class 1 and 2.

Zephyr remains a good RTOS and is not rejected as a future option. Its current upstream RP Pico support does not include an RP PIO I2S driver; the upstream request for one was closed as not planned. A Zephyr migration would therefore make this project responsible for a custom I2S driver, DMA integration, full-duplex behavior, clocking, underrun handling, and long-duration audio validation before audio could be treated as stable. Rev A should not make its core audio path dependent on that unowned work.

The selected path keeps the earliest firmware small and testable:

| Concern | Rev A implementation |
| --- | --- |
| USB device | TinyUSB composite device |
| Playback transport | USB Audio Class 1, isochronous OUT |
| Board management | CDC ACM, versioned framed protocol |
| Audio clocks and samples | RP2354 PIO plus DMA ring buffers |
| Codec configuration | Pico SDK I2C driver on `AUDIO_I2C` |
| Vehicle and fault policy | Pico SDK control loop, GPIO/interrupt/DMA events |
| RTOS | Not required initially; FreeRTOS is a later option if measured scheduling complexity justifies it |

Exact Pico SDK and TinyUSB revisions must be pinned in the firmware manifest or submodule configuration before work starts. The project should keep a short record of any local changes to TinyUSB descriptors or Pico PIO programs.

## USB Runtime Contract

The CM5 is the USB host and the RP2354 is the USB device. In normal operation, one physical `MCU_USB_D+` / `MCU_USB_D-` connection enumerates a composite runtime device with independent functions:

1. **USB Audio Class 1:** mandatory Rev A playback interface. The starting profile is 48 kHz, 16-bit, stereo PCM playback from Android to the RP2354.
2. **CDC ACM:** board-management interface for ignition, power, fault, configuration, and firmware status messages. It is separate from the audio data stream.
3. **Future capture interface:** add codec ADC or microphone capture only after playback is stable. It must be negotiated and tested as a separate increment, not assumed to work because playback works.

The Android-facing choice is intentionally conservative. AOSP documents USB Audio Class 1 support as the baseline USB-audio target. UAC2 can be investigated later, but Rev A playback must not depend on a host-specific UAC2 behavior or implicit-feedback implementation.

BOOTSEL recovery is a distinct, explicit state. When the CM5 asserts the documented reset and BOOTSEL controls, the RP2354 leaves normal application firmware and re-enumerates through its Boot ROM. It is not a third runtime interface and must never be entered as part of ordinary audio or vehicle operation.

## Audio Ownership and Timing

The RP2354 is the I2S master. It drives `I2S_MCLK`, `I2S_BCLK`, and `I2S_LRCLK`; it transmits playback samples on `I2S_DOUT` and receives future capture samples on `I2S_DIN`. The TLV320AIC3104 is the I2S slave.

USB packets and I2S clocks do not arrive in the same scheduling domain. Firmware must use DMA-backed playback ring buffers between TinyUSB and the PIO state machine, record underrun and overrun counters, and keep the amplifier muted during stream acquisition, loss, reset, or a detected data fault. Buffer size, clock rate, and any future feedback mechanism are prototype measurements, not assumptions to bury in a descriptor.

The first milestone may use a simple, synchronous UAC1 implementation to establish electrical correctness and stable playback. If long-duration testing exposes clock drift or buffer walk, the project must choose and document a synchronization strategy before declaring audio ready. That could be a clock scheme compatible with the selected codec and USB profile, explicit feedback, or another measured solution. It must not be papered over by increasing buffer depth indefinitely.

## Implementation Sequence

1. Establish a reproducible Pico SDK and TinyUSB build for the RP2354, including UART/SWD debug and a unique firmware version report.
2. Enumerate a composite USB device with CDC ACM and a UAC1 playback interface on a Linux host before involving Android.
3. Configure the codec over `AUDIO_I2C`; verify reset polarity and hold the TDA7850 muted and in standby.
4. Generate I2S clocks and playback data through PIO/DMA. Confirm the clock/data relationship at the codec pins with a logic analyzer or oscilloscope.
5. Play a 48 kHz, 16-bit stereo test stream through the codec and observe clean L/R analog output at the TDA7850 inputs while the amplifier remains muted.
6. Release amplifier standby and mute only after stream and codec common-mode are stable. Validate the four speaker outputs with a benign load before vehicle speakers.
7. Stress playback while exercising CAN traffic, GPIO-expander activity, fan control, and normal board-management traffic. Capture underrun, overrun, reset, and USB reconnect counters.
8. Add capture only after the playback and fault-recovery acceptance criteria pass.
9. Bring the verified ALSA-visible UAC1 device into Android audio policy, then repeat boot, suspend/resume, ACC-off, and forced-power-cut tests.

## Acceptance Criteria

Rev A audio is not considered ready until it meets all of these checks:

- The CM5 enumerates the intended UAC1 and CDC ACM interfaces repeatedly across cold boots and USB reconnects.
- Continuous 48 kHz, 16-bit stereo playback produces no audible drops, stuck samples, or amplifier pops during a sustained stress run.
- I2S clocks and sample framing remain correct at the codec pins under management, CAN, and fan-control activity.
- The amplifier remains muted for absent clocks, codec reset, USB disconnect, RP2354 reset, and a reported DMA/audio-stream fault.
- An `ACC_OK` shutdown, timeout-forced power removal, and subsequent cold start return the bridge to a known muted state without requiring a manual cable reconnect.
- Management messages remain responsive while audio is active, but a management fault never causes arbitrary data to be emitted to the codec.

## Zephyr Revisit Criteria

Zephyr should be reconsidered only when one of the following is true:

- Upstream Zephyr gains maintained RP2350 PIO I2S support suitable for this full-duplex, DMA-backed use case; or
- the project explicitly commits to owning a custom Zephyr driver and its regression suite; or
- a later firmware scope benefits enough from an RTOS to justify porting a proven Pico SDK audio engine without changing the electrical or USB contract.

That review must preserve the same USB descriptors, audio clocking evidence, recovery behavior, and acceptance criteria. A framework migration is not a reason to reopen a working hardware interface casually.

## Known Documentation and Schematic Follow-Up

Any older annotation that describes an external USB-to-I2S bridge, CP2615, or PCM3120 is obsolete for this architecture and should be removed or replaced during the next schematic-text cleanup. The source of truth is the direct RP2354-to-TLV320AIC3104 I2S topology recorded here and in the firmware implementation plan.

## References

- [Raspberry Pi Pico SDK](https://github.com/raspberrypi/pico-sdk)
- [Raspberry Pi Pico examples](https://github.com/raspberrypi/pico-examples)
- [Pico Extras PIO I2S implementation](https://github.com/raspberrypi/pico-extras/blob/master/src/rp2_common/pico_audio_i2s/audio_i2s.pio)
- [TinyUSB](https://github.com/hathach/tinyusb)
- [Zephyr RP Pico PIO I2S driver request](https://github.com/zephyrproject-rtos/zephyr/issues/103035)
- [Zephyr next-generation USB device stack](https://docs.zephyrproject.org/latest/services/connectivity/usb/device_next/usb_device.html)
- [Android USB audio documentation](https://source.android.com/docs/core/audio/usb)
