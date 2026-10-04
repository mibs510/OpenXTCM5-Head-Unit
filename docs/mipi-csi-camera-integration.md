# MIPI CSI-2 Reverse Camera Integration Plan

**Status:** Future architecture and validation plan. The MS2106E CVBS-to-USB path remains the practical first-board and bring-up path. This plan must not block that path.

## 1. Goal

Replace the USB UVC capture subsystem with a native CM5 camera pipeline:

```text
Rear CVBS camera
  |
  |  CAMERA_12V_SW, CVBS input protection, AC coupling, and 75 ohm termination
  v
CVBS-to-MIPI CSI-2 decoder
  |
  |  MIPI CSI-2 clock lane plus one or two data lanes
  v
CM5 CAM0 or CAM1
  |
  v
Linux V4L2 / media-controller pipeline
  |
  v
Android camera provider / Camera HAL
  |
  v
Dedicated reverse-camera view

RP2354 REVERSE_OK ---> CM5 reverse event ---> camera-power and UI policy
```

The RP2354 continues to detect `REVERSE_OK`, control the camera supply policy, and notify the CM5. It does not carry video data.

## 2. Non-Negotiable Interface Boundary

The CM5 camera connector accepts MIPI CSI-2. It cannot accept BT.656, BT.601, generic DVP, or raw CVBS directly. A native-camera design therefore needs one of these conversion paths:

1. **Preferred:** CVBS to MIPI CSI-2 decoder.
2. **Fallback:** CVBS to BT.656/DVP decoder followed by a parallel-video-to-CSI-2 bridge.

The first option has fewer components and fewer clocks to coordinate. The second is only worth considering when it has a clearly better-supported decoder or a meaningful sourcing advantage.

An example of the preferred class is Analog Devices' ADV7280-M, which converts analog video to MIPI CSI-2. It is an architectural example, not a selected part or a BOM commitment.

## 3. Current MS2106E Path

The present camera circuit remains valuable:

- `CAMERA_12V_SW` supplies the external reverse camera.
- CVBS protection, coupling, and 75 ohm termination remain useful upstream infrastructure.
- `CAMERA_5V_SW` supplies the MS2106E USB capture subsystem.
- The CM5 receives a conventional UVC camera over USB and Android integration is comparatively straightforward.

Do not remove or repurpose `CAMERA_5V_SW` solely because a CSI-2 option is being investigated. A CSI decoder will need its own confirmed power tree and enable/reset sequence.

## 4. Decoder Selection Criteria

Select a candidate only after it satisfies all of the following:

1. Accepts the intended NTSC/PAL, 1 Vpp, 75 ohm composite-camera signal.
2. Outputs native MIPI CSI-2 with a lane count and lane rate within the CM5 camera-interface capability.
3. Has public electrical documentation, available evaluation material, and a realistic supply path.
4. Has an existing Linux V4L2 subdevice driver, or a credible route to one without NDA-only register documentation.
5. Documents required rails, reset, power-down, I2C control, reference clock, CSI lane configuration, and output format.
6. Can be proven with the actual vehicle camera before committing the main board to the replacement circuit.

Avoid choosing an IC solely because it produces CSI-2. The driver, device-tree binding, and Android camera exposure are the long poles in this project.

## 5. Planned Hardware Partition

### Retained external interface

Keep these functions independent of the capture architecture:

- `CAMERA_12V_SW` powers the camera only in the required reverse-camera policy state.
- The external connector keeps CVBS and `VGND` as a paired analog interface.
- ESD protection remains at the connector.
- The input network preserves AC coupling and the decoder-required 75 ohm termination on the decoder side of the coupling capacitor.

### New decoder subsystem

The future CSI decoder sheet should contain:

- Decoder-specific 3.3 V, 1.8 V, and/or analog rails with local decoupling exactly as required by its datasheet.
- I2C control path, including defined pull-ups, CM5-visible bus ownership, and address selection.
- Reset/power-down control with a known boot state.
- Required crystal, oscillator, or CM5-provided reference clock.
- A short, impedance-controlled CSI-2 connection from decoder to the selected CM5 camera connector.
- DNP options only where they serve a real bring-up purpose, such as optional clock components or test access.

`CAMERA_5V_SW` is not automatically part of this future power tree. If the selected decoder is powered from the system 3.3 V rail, use a dedicated decoder enable or a documented always-on policy rather than silently inheriting the MS2106E switch behavior.

## 6. Staged Validation Gates

### Gate 0: Preserve the known USB route

Bring up the MS2106E/UVC capture design first. This provides a usable reverse camera, validates the external camera, and prevents Android-camera work from blocking the board.

### Gate 1: Decoder bench proof

Build or obtain a small decoder test board. With the actual CVBS camera:

- Confirm lock on NTSC and expected image stability.
- Confirm I2C register access and reset behavior.
- Confirm the selected CSI-2 output format, lane count, and frame rate.
- Confirm the input-protection network does not degrade the composite signal.

### Gate 2: CM5 Linux capture

Use Raspberry Pi OS or another Linux environment before Android:

1. Add the decoder to the device tree and media graph.
2. Bind a V4L2 subdevice driver for the decoder.
3. Validate the CSI receiver and capture node with `media-ctl` and `v4l2-ctl`.
4. Prove stable repeated reverse-power cycles, not merely one successful boot.
5. Record a short capture, check color order, field handling, aspect ratio, and latency.

This gate is complete only when the camera works repeatedly on the real CM5 hardware without relying on the USB capture route.

### Gate 3: Android camera integration

Android is a separate project after Linux capture works:

1. Carry the working kernel and device-tree camera graph into the selected CM5 Android build.
2. Expose the V4L2 camera through an Android camera provider / Camera HAL implementation.
3. Add SELinux policy and permissions required by the provider.
4. Verify camera enumeration, preview, lifecycle handling, and repeated suspend/resume or power-cycle behavior.
5. Create a dedicated reverse-camera activity or service that responds to the CM5 reverse event.

Community Android images may not ship a usable Camera HAL for this custom camera graph. Treat that as the principal schedule risk, not as a small configuration task.

### Gate 4: Vehicle behavior and overlays

After a stable Android preview exists:

- Enable camera power only when reverse operation requires it.
- Allow a short decoder-lock delay before presenting the view.
- Use `REVERSE_OK` as the event source, not video-lock alone.
- Add steering-angle guidelines only after CAN decoding is stable.
- Calibrate guideline geometry with the installed camera and vehicle.

## 7. Decision Record

| Decision | Current position |
| --- | --- |
| First-board capture path | MS2106E USB UVC capture remains preferred. |
| Native CSI route | Future development path, evaluated separately from the first-board schematic. |
| Preferred architecture | Integrated CVBS-to-MIPI CSI-2 decoder. |
| BT.656/DVP direct to CM5 CSI | Not compatible; requires an additional CSI-2 bridge. |
| Video transport owner | Dedicated decoder/CM5 path, never the RP2354. |
| Reverse event owner | RP2354 detects and forwards `REVERSE_OK`; CM5 controls UI behavior. |

## 8. References

- [Raspberry Pi Compute Module documentation](https://www.raspberrypi.com/documentation/computers/compute-module.html)
- [Raspberry Pi camera documentation](https://www.raspberrypi.com/documentation/hardware/camera/computers/camera_software.html)
- [Android Camera HAL3 documentation](https://source.android.com/docs/core/camera/camera3)
- [Analog Devices ADV7280 data sheet](https://www.analog.com/media/en/technical-documentation/data-sheets/ADV7280.PDF)
