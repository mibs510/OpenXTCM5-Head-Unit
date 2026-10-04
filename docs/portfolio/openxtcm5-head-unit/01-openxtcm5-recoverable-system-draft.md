---
title: "OpenXTCM5, Part 1: Designing an Automotive Head Unit as a Recoverable System"
slug: "openxtcm5-designing-a-recoverable-automotive-head-unit"
description: "A pre-layout engineering journal for a Raspberry Pi CM5 head unit: separating Android from vehicle power and firmware policy, designing recovery paths, and being explicit about what remains unproven."
status: "Draft — not published"
project_stage: "Pre-layout schematic and architecture review"
last_updated: "2026-10-03"
---

# OpenXTCM5, Part 1: Designing an Automotive Head Unit as a Recoverable System

> **Status at the time of writing:** OpenXTCM5 is a pre-layout design project. The schematic and system documentation have gone through iterative review, but there is no routed board, fabricated prototype, thermal result, vehicle-transient measurement, or validated Android image yet. This article describes the decisions and evidence available now; it does not claim a working replacement head unit.

I started OpenXTCM5 because I was tired of treating an aftermarket Android head unit as an appliance with no useful failure boundary. The unit in my vehicle overheats around its TDA7388 audio amplifier. Once temperature builds, Android becomes severely laggy, reverse-camera operation can freeze the system, and audio can repeat its last fragment until the unit is rebooted.

Replacing it with a faster sealed box would not teach me much. I wanted to understand the problem as a system: what happens when vehicle voltage changes, a display controller stops responding, an external USB device shorts, a camera needs to appear quickly in reverse, or Android is busy doing Android things while the hardware needs an immediate decision.

OpenXTCM5 is my in-progress, open-hardware attempt to build a more capable and repairable Android head unit around a Raspberry Pi Compute Module 5, or CM5. It is also an intentionally ambitious learning project. The value is not a finished board I can show today. It is the discipline of turning a frustrating field failure into architectural requirements, documenting the reasoning, finding weak assumptions early, and refusing to call an untested schematic a product.

## The design problem is larger than a CM5 carrier

At a glance, a head unit looks like a compute module, display, audio amplifier, a few connectors, and a power supply. In a vehicle, those blocks carry different responsibilities:

- The CM5 needs to run Android, present the display, enumerate USB devices, and host modem and camera applications.
- The vehicle electrical system needs a controlled response to ACC changes, low voltage, crank behavior, and external faults.
- The display requires more than MIPI DSI: it has logic power, analog bias rails, backlight control, reset timing, and a separate capacitive-touch interface.
- The audio stage needs a thermal plan and sequencing that prevents pops, undefined outputs, and a repeat of the original head unit's heat problem.
- Service and recovery need to work even when the main operating system does not.

Those are not independent design checkboxes. A display reset can create a back-power path. A USB port can influence crank behavior. A power switch can be useful for recovery—or it can add a failure state with no meaningful benefit. The first useful decision was therefore not a connector or an IC. It was deciding which controller owns each kind of behavior.

## Two processors, two different jobs

OpenXTCM5 deliberately separates the user-facing computer from vehicle-facing control.

- The **CM5** owns Android, the display pipeline, touch input, USB-host devices, modem integration, camera applications, and the human interface.
- The **RP2354** owns power sequencing, vehicle-state acquisition, CAN through a TCAN4550-Q1, GPIO-expander control, thermal and fan policy, audio-codec and amplifier control, and fault-aware recovery.

The CM5 could technically drive enables or poll vehicle GPIOs. That is not the point. Android is not the component I want solely responsible for deciding how the board behaves if a rail sags, a touch controller needs a genuine power cycle, or an ACC transition starts an orderly shutdown.

The split also creates a clearer software contract. Android should ask for actions and receive status through the defined CM5-to-RP2354 protocol; it should not silently treat a visible peripheral as permission to power it. The first version of that contract is designed but not implemented or tested, and a single wake or interrupt line is not a protocol.

<!-- Recommended visual: render diagrams/system-architecture.mmd as a static diagram. -->

## Firmware is part of the architecture, not a later add-on

The schematic only becomes a system when firmware gives its control and status signals defined behavior. I do not want the RP2354 to be a collection of GPIO writes that happen to turn things on. Its role is to be a deterministic supervisor between the vehicle, the board's power domains, and the Android computer.

The firmware design work therefore starts with ownership and failure behavior:

- It samples the vehicle and board conditions that matter to a state transition, such as ACC state, power-good/fault indications, and thermal or subsystem health signals.
- It sequences enables and recovery actions with explicit prerequisites, timeouts, and safe defaults rather than treating every control net as an independent switch.
- It reports state and faults to the CM5, while remaining able to take a safe action if Android is slow, unavailable, rebooting, or gives no response.
- It keeps a physical and USB-based recovery route available for the RP2354 itself, instead of assuming the normal Android application path will always be healthy.

The selected normal CM5-to-RP2354 transport is the board's internal USB link. The RP2354 presents a composite runtime device: UAC1 carries audio separately from a CDC ACM channel for versioned management messages. The proposed active-low `RP2354_TO_CM5_ATTN_N` signal is only a doorbell asking the CM5 service to read the authoritative USB channel; it is not a substitute data path. Explicit BOOTSEL entry is different again: it exits normal application firmware and re-enumerates the RP2354 Boot ROM for maintenance.

The planned state machine needs to cover supervised startup, normal operation, orderly ACC-off shutdown, low-input or crank load shedding, subsystem recovery, and fault handling. The first message set is defined—such as `IGNITION_OFF_PENDING`, `SHUTDOWN_ACCEPTED`, `SHUTDOWN_READY`, `IGNITION_RESTORED`, `HELLO`, `HEARTBEAT`, and `STATUS`—but the frame implementation, protocol versioning, watchdog behavior, and host-side service still need to be built and tested. Those are design decisions that deserve the same review as a power rail or a connector—not details to postpone until after the PCB comes back.

### Rev A platform decision: Pico SDK and TinyUSB

I originally expected to use Zephyr for the RP2354. Zephyr is still a strong RTOS, but Rev A depends on an I2S audio bridge, and current upstream RP Pico support does not include a maintained PIO I2S driver. The upstream request for that driver was closed as not planned. Moving ahead with Zephyr would make this project responsible for a custom I2S driver, DMA integration, full-duplex behavior, clocking, underrun handling, and the regression work needed to trust it. That is too much unowned framework work to make the core audio path depend on before the first board has even been validated.

Rev A therefore uses the **Raspberry Pi Pico SDK** and **TinyUSB** instead. It does not require an RTOS initially: PIO and DMA handle time-sensitive I2S movement, while a small event-driven control application handles vehicle policy, codec configuration, USB management, and diagnostics. FreeRTOS remains a future option only if measured scheduling complexity justifies it after the audio path is proven.

The audio architecture is now explicit: the CM5 is the USB host; the RP2354 is a composite USB device providing **UAC1 playback** and a separate **CDC ACM management channel**; and the RP2354 is the I2S master driving the TLV320AIC3104 codec. The first milestone is deliberately narrow—48 kHz, 16-bit, stereo playback—before microphone capture, richer USB-audio profiles, or Android audio-policy work are allowed to expand the scope.

This choice removes an avoidable framework dependency; it does not remove the engineering risk. USB packet timing and I2S clocks are separate timing domains, so the firmware needs DMA-backed ring buffers, underrun/overrun counters, deliberate mute behavior, and long-duration testing. A larger buffer is not a clock-synchronization strategy. The prototype must establish whether the initial synchronous UAC1 approach holds over time or whether it needs an explicitly measured feedback or clocking strategy.

At this stage, the platform, audio topology, and implementation sequence are accepted design decisions—not claims of a completed control stack or working audio. When firmware enters the public repository, the article should identify the source revision and label what is merely designed, what runs in isolation, what runs on the interface board, and what has been tested in the vehicle.

## Power is a state machine, not a collection of net labels

The project has an always-on supervisory domain, a controlled main 5 V system rail, regular 3.3 V and 1.8 V rails, and a set of branch supplies. The principal 5 V stage is based on a TPS552892-Q1 buck-boost converter. The intent is to make the core system more resilient to vehicle voltage variation while retaining a small always-on domain for ACC sensing and controlled wake/shutdown.

Early in the design, I was tempted to add a load switch wherever I could imagine a firmware enable. That is not a sound power architecture. A rail should only be independently switched when it solves a specific problem:

1. It reaches an external connector and needs fault containment.
2. It sheds a meaningful load during a low-voltage event.
3. It lets firmware perform a real power-cycle recovery.
4. It prevents an off-state back-power path.
5. It creates a deliberate user-visible policy, such as camera power only when reverse operation requires it.

This rule kept the project from becoming a schematic filled with arbitrary enables. USB VBUS and the external camera supply qualify because they leave the board. The backlight qualifies because it is an effective early load to shed without immediately dropping the CM5. LCD bias and touch power qualify because each can benefit from a genuine recovery cycle. In contrast, switching every small logic rail would create more inrush, firmware states, and debugging surface without a proportional gain.

The intended operating states are simple to describe but substantial to implement:

- With the vehicle off, the protected input and low-IQ always-on domain remain available while the main system stays off.
- An ACC event enables the main buck-boost stage. Once the system rail is valid, the RP2354 can bring up branches in an intentional order.
- On ACC removal, the system requests an orderly CM5 shutdown, disables high-load and external branches, and releases the system hold after shutdown or a defined timeout.
- During a low-input or crank event, the design prioritizes the core computer and sheds load predictably: mute the amplifier, remove the backlight, disable USB VBUS and camera power, then consider panel bias only if needed.

None of that is validated hardware behavior yet. It is the policy the prototype must challenge with real measurements.

<!-- Recommended visual: render diagrams/power-state-machine.mmd as a static diagram. -->

## The display taught me to treat “off” as an electrical state

The display section exposed most of the assumptions I did not yet know I was making. The planned panel requires four-lane MIPI DSI, switched 1.8 V panel logic, translated reset and standby signals, a dedicated backlight path, a separate capacitive-touch flex, and analog panel-bias rails.

The analog bias is generated by a TPS65150. Its input is independently switched so firmware can remove the LCD-bias domain during a defined recovery sequence. The device produces the panel rails typically named AVDD, VGH, VGL, and VCOM.

One small schematic detail is representative of the kind of review I want this project to encourage. The regulator-side positive rail is named VVS/AVDD and connects through R309, a 0-ohm link, to the panel-side AVDD net. Using separate net names is deliberate. If both sides were called AVDD, KiCad would electrically connect around the resistor and the link would stop functioning as a meaningful isolation or assembly option.

That lesson is more valuable than the resistor itself: component symbols and net labels are not documentation decorations. They define topology. The portfolio version of this project should show decisions like that, including the mistakes I caught before they became PCB mistakes.

The touch circuit reinforced the same point. The GT911 is on its own flex connector; it is not an optional feature of the LCD connector. Its I2C pull-ups need to live on the switched touch rail. A previously considered CM5 I2C pair had fixed pull-ups to an always-on 3.3 V rail, which would have left an unpowered touch controller connected to a live bus. Moving the touch bus to another CM5 I2C-capable GPIO pair makes a meaningful power-off state possible—provided firmware first leaves reset, interrupt, and I2C lines safe.

That is the distinction I am trying to learn throughout OpenXTCM5: “power removed” is not the same as “subsystem electrically isolated.”

<!-- Recommended visual: render diagrams/display-touch-stack.mmd as a static diagram. -->

## Firmware updates and recovery are part of the hardware design

The CM5 runs the product experience, but the RP2354 needs a deliberate in-system firmware-update path. The current plan reserves a CM5 USB-host link to the RP2354 and adds two CM5-controlled, active-high gate commands that allow the CM5 to request the RP2354 bootloader without manual access to the board:

- RP2354_RESET_ASSERT drives a local MOSFET that pulls the RP2354 RUN pin low.
- RP2354_BOOTSEL_ASSERT drives a second MOSFET that pulls QSPI_SS low during reset to request BOOTSEL.

The CM5 does not drive either RP2354 pin directly. Gate pulldowns keep both commands released while the CM5 boots or its GPIOs are high impedance, and local pushbuttons remain available for bench firmware updates and recovery.

This is a small circuit, but it changed how I think about firmware updates. A dependable update process starts with reset behavior, boot straps, safe defaults, USB enumeration, timeouts, and a physical fallback—not merely a software utility. The same circuit also provides a recovery path if normal RP2354 firmware is unhealthy. The first implementation will use deliberately conservative, configurable timing and will need bench validation against the actual RP2354 clocking and USB path.

The same principle applies to the external USB-C maintenance connector. It is a self-powered CM5 device/UFP port for storage recovery and provisioning. It does not power the board. Its VBUS pin is intentionally not connected to the system power rail; the head unit must already be powered from a proper vehicle or bench supply. This prevents a development host or powered hub from becoming an accidental, underspecified power architecture.

<!-- Recommended visual: render diagrams/rp2354-firmware-update.mmd as a static diagram. -->

## What the project has proven—and what it has not

The honest project boundary is important.

**Supported by the current schematic and written design records:**

- A defined division of responsibility between Android/CM5 and RP2354 vehicle/power policy.
- A reviewed rationale for global power control, branch switching, load shedding, and off-state risks.
- A planned display, touch, modem, audio, camera, CAN, and maintenance-recovery architecture.
- A firmware ownership model and planned supervisor state machine that keep vehicle-facing policy out of the Android application layer.
- An accepted Rev A audio/firmware direction: Pico SDK plus TinyUSB, a composite UAC1-and-CDC runtime device, and RP2354 PIO/DMA as the I2S master for the TLV320AIC3104.
- Explicit documentation of several design corrections: touch-bus pull-up ownership, panel-bias topology, recovery-gate behavior, and KiCad production-variant review.
- A staged plan that keeps the MS2106E USB UVC camera route as the first-board path while treating native CSI-2 capture as later work.

**Not yet proven:**

- A PCB layout, stack-up, or routing solution for MIPI, USB, PCIe, switching loops, analog audio, and vehicle I/O.
- Prototype operation, thermal behavior, amplifier performance, or vehicle-transient survival.
- The final display panel, its data sheet, its exact timing, and the real-world validity of the panel power sequence.
- Android device-tree integration, camera behavior, modem composition, call audio, or a working CM5-to-RP2354 control protocol.
- RP2354 firmware behavior on the actual interface board, composite USB enumeration, UAC1 playback, PIO/DMA clock integrity, command/status exchanges with the CM5, timeout behavior, fault injection, or hardware-in-the-loop coverage.
- Long-duration audio synchronization, mute/standby behavior, codec output quality, or proof that the selected UAC1 profile survives management, CAN, fan-control, and USB-reconnect activity.
- Real crank behavior, reverse-camera time-to-first-frame, USB fault handling, touch recovery, or modem transmit-burst margin.

Calling out the unproven work is not a disclaimer pasted onto an otherwise finished-looking project. It is the central engineering fact. The schematic is a hypothesis with more structure than a breadboard, not validation.

## What I learned before layout

This work has already improved how I review embedded systems.

I learned to ask what happens when a device is off, not only when it is on. I learned that a net name is not proof of a control function, and that a 0-ohm resistor only matters if the net topology preserves it. I learned that an automotive-qualified suffix, a lower component cost, and actual environmental validation are separate decisions. I learned that a controller should own the behavior it can still perform when the operating system is busy or recovering.

I also learned that design tools can mislead when I do not inspect the active configuration. In KiCad, a pass through the default variant made several values appear to be missing even though they existed in the Production variant. The correction was not to overwrite the schematic; it was to make the selected variant part of the review evidence.

These are useful lessons precisely because they emerged before a board was fabricated. The goal is not to portray every correction as a triumph. It is to leave a trail that lets the next review start from a better question.

## What comes next

The immediate milestone is not publication. It is closing the gaps that block a reliable layout:

1. Complete and review every parent-sheet interface, especially CM5 display control and RP2354 USB/recovery links.
2. Define and version the CM5-to-RP2354 command/status contract, including safe defaults, timeouts, fault reporting, and update/recovery behavior.
3. Implement and bench-prove the accepted audio topology: composite UAC1 plus CDC enumeration, codec I2C/reset behavior, PIO/DMA clocks, ring-buffer telemetry, and mute/standby sequencing.
4. Lock the actual display panel, flex drawings, board outline, connector orientation, and mechanical constraints.
5. Perform rail-current, inrush, thermal, and copper calculations with the production-intent parts.
6. Floorplan high-current, high-speed, analog, RF, and thermal regions before detailed routing makes the decisions expensive.
7. Create a bench-validation matrix before layout freeze, then test the design in the vehicle environment that motivated it.

The eventual build article should be a different article. It should contain board renders, prototype photos, scope captures, temperature data, current measurements, startup and crank traces, and a record of where the first hardware revision disagreed with this design journal.

For now, OpenXTCM5 is the first part of that story: a system being designed to be understandable, recoverable, and honest about its evidence.

## Suggested portfolio metadata

- **Tags:** Embedded Systems, Electrical Engineering, KiCad, Raspberry Pi CM5, RP2354, Raspberry Pi Pico SDK, TinyUSB, Firmware Architecture, Automotive Electronics, Power Sequencing, Hardware Architecture, CAN, Android Bring-Up
- **Feature-image direction:** A clean board-level system architecture graphic over a subdued photograph of the original aftermarket head unit or vehicle dashboard. Do not use a PCB render as the feature image until layout exists.
- **Recommended supporting visuals:** rendered system-architecture, power-state, display/touch, and BOOTSEL-recovery diagrams; a photographed original-unit thermal/failure context; later, schematic excerpts and measured prototype evidence.
