# OpenXTCM5 Portfolio Publication Roadmap

**Status:** Working editorial and engineering roadmap  
**Last updated:** 2026-10-03  
**Publishing model:** One evolving OpenXTCM5 portfolio article. Each phase appends evidence and decisions to the same project story rather than pretending a pre-layout schematic is a completed product.

## Core rule

Every revision must distinguish among:

- **Designed:** represented in the current schematic, layout, or architecture.
- **Reviewed:** checked against requirements, datasheets, reference designs, calculations, ERC, or design review.
- **Implemented:** present in versioned firmware or software source, but not necessarily integrated with hardware.
- **Tested:** demonstrated on physical hardware with a stated setup and result.

No later result should be used to imply that an earlier stage had already been validated. If a prototype disproves a schematic, layout, or architectural assumption, the article should record the change and why it was made.

## Revision 1 — Schematic, architecture, and firmware design

**Goal:** Publish the current design-journal portion of the article after the KiCad project is prepared and pushed to a new public repository.

**What this revision covers**

- The original head-unit reliability and thermal problem that motivated the project.
- System ownership: CM5 for Android and user-facing functions; RP2354 for vehicle I/O, power policy, recovery, fault handling, audio control, and thermal policy.
- Firmware ownership and the reasoning behind it: why the RP2354, rather than Android on the CM5, owns power sequencing, vehicle-facing state, faults, and recoverability.
- The intended firmware state machines and contracts: global/branch power states, safe startup and shutdown defaults, reset and BOOTSEL recovery, fault/status reporting, and the planned CM5-to-RP2354 command/status boundary.
- The Rev A platform choice: Raspberry Pi Pico SDK and TinyUSB rather than Zephyr, because the audio bridge needs PIO/DMA I2S work that Zephyr does not currently maintain upstream for RP Pico devices. The article should explain that this defers an unowned driver project; it does not claim a working audio path.
- Decisions that are deliberately deferred: the concrete CDC frame encoding and descriptor implementation, Android service boundary, update workflow, watchdog behavior, clock-synchronization strategy, and hardware-in-the-loop test harness. These should be named as open design work, not implied to exist already.
- Global and branch power-state reasoning, including ACC behavior, controlled shutdown, low-voltage load shedding, external-port protection, and recovery-oriented rail switching.
- Display, panel-bias, and GT911 touch design review, including the off-state and back-power lessons that changed the intended design.
- CM5 and RP2354 recovery paths, including the self-powered maintenance port and RP2354 BOOTSEL/reset controls.
- Explicit status: pre-layout schematic and design intent, not validated hardware.

**Primary visual evidence**

- The KiCad project displayed interactively through KiCanvas on the portfolio site.
- Static architecture or state-machine diagrams only where they clarify ownership or sequencing; photographs are optional for this revision.
- Selected firmware state-machine, recovery, and CM5-to-RP2354 interface diagrams where they make a firmware decision more legible than prose.

**Before publishing**

1. Prepare a clean public project directory and create/push the new repository.
2. Decide and add the repository license, README, current project status, and a public-safe source tree.
3. Check the public repository for vendor-provided documents, third-party libraries, private paths, local backup archives, and material that should remain local.
4. Confirm the published schematic uses the intended Production variant and label the article with the KiCad revision/date it reflects.
5. Add the interactive KiCanvas project viewer to the article and verify that the root schematic, child sheets, and project board file all open.
6. Review the firmware plans against the article and label each decision as designed, reviewed, implemented, or tested. Do not let a detailed implementation plan read as completed firmware.

**Publish condition**

The article may be published when the repository and KiCanvas viewer are ready. Prototype photos, layout work, and measurement data are not prerequisites; the article must simply remain candid that they do not exist yet.

## Revision 2 — Layout and physical implementation

**Goal:** Append a dedicated Layout section after the schematic subsections once the board is placed and routed far enough for design decisions to be meaningful.

**What this revision adds**

- Board outline, display and connector placement, mechanical constraints, service access, and cooling strategy.
- Power-entry, buck-boost, high-current, and external-connector placement decisions.
- Separation of switching power, analog audio, CM5, modem/RF, CAN, MIPI DSI, USB, PCIe, camera, and other sensitive paths.
- The firmware consequences of physical choices: interrupt ownership, reset and enable timing, fault inputs, status LEDs, connector/service access, test points, and which signals should be observable during bring-up.
- Stack-up, impedance, return-path, copper, thermal, creepage, and component-orientation decisions as applicable.
- Tradeoffs, compromises, routing constraints, and any places where the layout changed the schematic or architecture.

**Primary visual evidence**

- The routed board displayed interactively through KiCanvas.
- Board renders or KiCad screenshots only when a static image makes a placement or routing decision easier to explain.

**Append only when**

- The major interfaces and connector orientations are locked.
- The board floorplan is intentional rather than temporary.
- The layout claims can be supported by the actual board file, stack-up assumptions, and routing rules.

## Revision 3 — Prototype, bring-up, and corrections

**Goal:** Append the physical evidence after Rev A boards are ordered, assembled, and tested. Begin bring-up with the power domain, then move deliberately through the remaining subsystems.

**What this revision adds**

- Board-arrival, population, fixture, and bench-test photographs.
- Power-domain bring-up first: protected input, always-on domain, main buck-boost rail, 3.3 V and 1.8 V rails, branch enables, fault reporting, and safe defaults.
- Measured results for each circuit as it earns trust: rail startup, shutdown, inrush, recovery, display/touch, USB, camera, modem, audio, fan, CAN, and vehicle behavior.
- Firmware integration evidence: state-transition traces, command/status exchanges, fault-injection results, recovery timing, diagnostics, and the differences between bench simulation and the real board.
- Oscilloscope captures, current/temperature data, firmware logs, and test conditions where they substantiate a claim.
- A candid record of every issue that requires a schematic, layout, architecture, firmware, or test-plan change.

**Correction format**

For every meaningful discrepancy, record:

1. What was designed or expected.
2. What the prototype actually did.
3. The evidence collected.
4. The likely cause and confidence level.
5. The change made or deferred for the next revision.
6. What must be retested afterward.

This section is not a postmortem only for failures. It should also record validated assumptions and explain why a result increases confidence.

## Article structure after all three revisions

1. Why OpenXTCM5 exists
2. Architecture and controller ownership
3. Firmware architecture, state machines, and controller contract
4. Schematic and subsystem design review
5. Interactive schematic/project viewer
6. Layout and physical implementation
7. Interactive board viewer
8. Prototype assembly and staged bring-up
9. Measurements, firmware corrections, and Rev B implications
10. Current status and next engineering milestone

## Standing editorial guardrails

- Do not describe the design as automotive-qualified, vehicle-proven, production-ready, or reliable until the corresponding tests exist.
- Preserve the difference between a design choice, a reviewed choice, and a measured result.
- Keep future camera, modem, and Android work scoped to the maturity actually reached.
- Treat changes as evidence of learning, not material to hide. The article should make the engineering process legible.
