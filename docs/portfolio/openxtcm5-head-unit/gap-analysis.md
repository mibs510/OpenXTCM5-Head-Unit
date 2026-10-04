# OpenXTCM5 Portfolio Gap Analysis

**Purpose:** Identify what the first portfolio article can honestly claim today, what evidence is missing, and which project decisions need closure before a later build article. This is an editorial and engineering planning record, not a release checklist.

**Evidence reviewed, 2026-10-03:**

- Root project README and design-stage statement.
- Existing long-form portfolio draft and five Mermaid diagrams.
- Power-control rationale.
- RP2354 firmware implementation plan.
- Android bring-up plan.
- Modem and reverse-camera integration plans.
- KiCad project structure and schematic-sheet inventory.

## Executive assessment

The material supports a strong **pre-layout design-journal** article. It does not yet support a portfolio story framed as a working head unit, a completed board, a validated automotive design, or a production-ready Android platform.

The strongest evidence is the quality of the reasoning: separated ownership between CM5 and RP2354, explicit power policy, recovery design, display/touch off-state analysis, firmware design plans, and a documented review loop. The main weakness is the absence of physical evidence, measurement data, locked mechanical requirements, and a completed hardware/firmware/software contract.

The first article should therefore be presented as *Part 1: Architecture and Design Review*, not as a build reveal. A second article should wait for a fabricated prototype and measured bring-up results.

## What is publication-ready now

| Area | Evidence available | Safe portfolio claim |
| --- | --- | --- |
| Problem statement | The root README records observed overheating, lag, reverse-camera freezes, and repeated audio from the original head unit. | The project began as a response to observed reliability and thermal failures in an aftermarket Android head unit. |
| System architecture | CM5/RP2354 ownership is consistently described across the portfolio, power, firmware, and Android records. | The design intentionally separates Android/user-experience responsibilities from vehicle-facing power and control policy. |
| Power-policy reasoning | Branch-switch criteria, low-voltage/load-shed policy, and off-state risks are documented in detail. | The schematic was designed around explicit global and branch power states rather than arbitrary GPIO enables. |
| Display and touch review | The draft documents the panel-bias architecture, 0-ohm topology correction, GT911 bus change, and switched pull-ups. | Display/touch design review exposed real power-domain and back-power concerns that changed the design. |
| Recovery architecture | CM5-controlled RP2354 BOOTSEL/reset gates and the self-powered CM5 maintenance port are documented. | Recovery was treated as a hardware and firmware contract, with safe defaults and physical bench fallbacks. |
| Firmware architecture | The RP2354 implementation plan assigns power, vehicle I/O, recovery, faults, and service functions away from Android on the CM5. | Firmware design is part of the architecture: the proposed controller owns safety-relevant state and recovery while the CM5 owns the user-facing system. |
| Self-critique | The source documents identify unresolved hierarchy links, interface conflicts, and test requirements. | The project remains an explicit pre-layout hypothesis and records the work still required to validate it. |

## Evidence to gather later

The first revision does not need physical photographs or prototype data. Its primary visual is the public KiCad project displayed through KiCanvas. The items below are evidence gaps for the layout and prototype revisions, not blockers for publishing the schematic/design-journal revision.

| Missing evidence | Why it matters | Minimum useful asset or record |
| --- | --- | --- |
| Original head-unit context | Makes the “why” immediate and personal. | Optional dashboard/unit photo, temperature image, or captioned video showing the failure symptom. |
| Mechanical envelope | A head unit is constrained by a dashboard opening, connectors, mounting, cooling, and service access. | Dimensions, mounting points, connector-facing directions, display position, and a simple enclosure/floorplan sketch. |
| Selected display evidence | Panel selection affects DSI mode, FFC pinout, bias rails, reset/standby timing, and software driver selection. | Final panel part number, data sheet, flex drawing, resolution/refresh target, and evidence that the connector pinout was matched. |
| Board visual | Readers need a readable system-level artifact throughout the project. | KiCanvas project viewer for Revision 1; later, KiCanvas board viewer, floorplan, and a 3D/rendered PCB view. |
| Measured electrical behavior | Separates design intent from electrical proof. | Scope captures of startup, shutdown, branch recovery, rail margins, inrush, and actual crank behavior. |
| Thermal evidence | The project originated from audio-stage heat; the solution needs to address it experimentally. | Amplifier/board temperature plan, fan-placement rationale, then thermal images and logged temperatures under load. |
| Firmware evidence | The CM5-to-RP2354 transport and initial message set are designed, but implementation maturity and state-machine behavior need to be clear. | Versioned message/API contract, state-machine diagrams, error/status model, source/revision reference, and a test tool or early firmware trace. |
| Public source/provenance | A portfolio reader needs a stable way to verify what is current. | New public repository, license, public-safe source tree, and revision/date label. |

## Technical gaps that should remain explicit

### Layout blockers

1. **Hierarchical interfaces are not complete.** The display GPIO4/GPIO5 links and the CM5-to-RP2354 USB/BOOTSEL/reset parent-sheet links are still called out as pending.
2. **The audio architecture is selected but unvalidated.** The RP2354 is the direct USB Audio-to-I2S bridge: the CM5 hosts a composite UAC1-plus-management USB device, the RP2354 is I2S master, and the TLV320AIC3104 is the I2S slave. Remaining work is descriptor, PIO/DMA, clock-synchronization, mute/fault, ALSA, and Android validation rather than an unresolved ownership decision. See [RP2354 Firmware Platform Decision](../../rp2354-firmware-platform.md).
3. **The final panel is not locked.** Panel timing, rails, pinout, connector orientation, and the Linux/Android driver choice cannot be finalized without it.
4. **The CM5-to-RP2354 command/status protocol is designed but not implemented.** Internal USB, a composite UAC1-plus-CDC device, the attention-doorbell role, and an initial message set are defined. The CDC frame encoding, descriptors, both endpoint implementations, and fault/reconnection behavior still need to be built and tested.
5. **The modem control story needs reconciliation.** The modem plan, firmware plan, and Android bring-up plan need one source of truth for W_DISABLE#, reset ownership, default states, and actual USB interfaces.

### Firmware and integration decisions

1. **Implement and exercise the RP2354-to-CM5 contract.** The initial commands, status/fault events, ownership of wake/reset behavior, and bounded shutdown behavior are now designed. Define the frame encoding and versioning, then prove what each side does if the other is absent or unresponsive.
2. **Turn power-policy diagrams into an implementable state-machine contract.** Record transition inputs, rail/output actions, timing constraints, exit criteria, fault paths, and safe default states. The article can show the design now; prototype evidence must later verify it.
3. **Separate implemented code from planned firmware.** The portfolio needs revision-specific labels for firmware that exists in source, firmware that has run on a bench, and firmware that has exercised the actual interface board.
4. **Plan observability before bring-up.** Specify which UART/USB logs, fault counters, status LEDs, test points, and host-side tools will prove a state transition or recovery event.
5. **Define the firmware test ladder.** Start with host-side/unit tests and controlled GPIO emulation where useful, then progress to board-level fault injection and hardware-in-the-loop tests. A manual bench result should not be presented as full coverage.

### Pre-layout engineering work

1. Confirm board outline, enclosure constraints, mounting, connector orientations, service access, and display/camera flex routing.
2. Create the PCB stack-up and routing rules before placement. MIPI DSI, USB, PCIe, switching loops, audio, CAN, modem/RF considerations, and high-current paths need placement-driven—not cosmetic—layout decisions.
3. Calculate and record rail current, inrush, switch limits, copper width, thermal dissipation, and worst-case vehicle-voltage behavior for the production-intent BOM.
4. Reconcile the selected commercial TPS65150 with environmental expectations, or document the qualification rationale and test plan.
5. Complete ERC using the Production variant, then disposition every remaining finding as corrected or intentionally waived with a reason.

### Prototype-validation gates

1. Normal power-on, ACC-off orderly shutdown, and controller-reset safe-state behavior.
2. Low-input and real crank capture at BATT+, protected input, +5V_SYS, main rails, and branch rails.
3. Display-bias sequencing, backlight enable timing, panel reset/standby behavior, and repeated display recovery.
4. Touch enumeration, rotation, suspend/resume, and deliberate rail-off recovery without I2C or GPIO back-powering.
5. USB VBUS short/reconnect behavior and enumeration of each intended USB topology branch.
6. RP2354 BOOTSEL recovery from CM5 and from the physical pushbuttons, including timeout release.
7. Camera power, UVC lock, reverse event, and measured time to first usable frame.
8. Modem rail margin under transmit bursts, reset/power recovery, USB enumeration, data, and GNSS.
9. Codec/amplifier mute/standby sequencing, thermal performance, fan policy, audio output quality, and fault response.
10. Vehicle CAN behavior and any safety-relevant separation from the immobilizer-related circuit before road use.

## Narrative gaps to fill with Connor

These answers will make the eventual article more personal and more defensible:

1. Which vehicle and head-unit form factor is the first board meant to serve? What physical constraints were measured from the dash and original unit?
2. What exact display panel, camera, amplifier load, speakers, and SIM7600 module are on hand today?
3. Which parts of the schematic were personally designed from first principles versus adapted from vendor reference circuits or existing CM5 carrier material?
4. Is the source intended to become public? If so, what license, repository URL, and public-safe BOM/document subset should be used?
5. Do photos, thermal observations, error videos, reverse-camera symptoms, or teardown images of the original unit exist?
6. What would count as a successful Rev A: reliable Android display/touch, data modem, reverse camera, audio, CAN observation, or a narrower subset?
7. Which design decisions changed most sharply during review, beyond the touch pull-up, panel-bias link, and recovery-gate examples already documented?
8. Which RP2354 firmware decisions are already represented in source, and which remain design plans only? Which CDC frame details, host-service behavior, and audio synchronization strategy still need decisions before bring-up?

## Recommended article sequence

| Article | Publish condition | Focus |
| --- | --- | --- |
| **Revision 1 — Schematic, architecture, and firmware design** | Public repository and KiCanvas schematic/project viewer are ready. | Problem, architecture, power and firmware ownership, recovery design, proposed state machines/contracts, review process, and candid pre-layout limits. |
| **Revision 2 — Layout and physical implementation** | Board outline, panel, connectors, floorplan, and meaningful routing are ready. | Mechanical constraints, placement, stack-up, high-speed/analog partitioning, and layout tradeoffs. |
| **Revision 3 — Prototype, bring-up, and corrections** | Rev A boards have been built and tests have begun. | Power-domain-first testing, photos, measurements, failures, corrections, and Rev B implications. |

## Editorial guardrails

- Describe the current work as **pre-layout**, **reviewed design intent**, or **planned architecture**. Do not imply field reliability or automotive qualification.
- Keep the original head-unit symptoms clearly labeled as personal observations, not a universal diagnosis of every unit using the same amplifier.
- Preserve the distinction between the first-board USB UVC camera route and the future CSI-2 architecture.
- Treat firmware diagrams and implementation plans as design evidence unless their revision, execution environment, and test result are stated.
- Treat automotive safety, CAN, and immobilizer-related functions as controlled-development work with defined bench and vehicle validation boundaries.
- Use real future measurements to correct this article rather than retroactively smoothing over failed assumptions.
