# OpenXTCM5 Agent Guide

## Project Context

OpenXTCM5 is an in-progress open-hardware Android automotive head unit built around a Raspberry Pi CM5 and an RP2354 real-time controller. The repository contains the hardware design, design records, and will later contain firmware and CM5/Android software.

Treat all design material as evolving intent until it has been validated on hardware. Do not present a documented plan as a proven behavior.

## Repository Map

- `pcb/`: KiCad project, hierarchical schematic sheets, board file, and local libraries.
- `docs/`: design rationale, decision records, and integration plans.
- `docs/portfolio/`: long-form project journal and its Mermaid diagrams.
- `README.md`: concise project overview; it should link to detailed records rather than duplicate them.

## KiCad Rules

- The active design variant is **`Production`**. Check it before reviewing values, generating a netlist/BOM, or diagnosing missing component values. The `<Default>` variant can intentionally show placeholder values and is not evidence of data loss.
- The user normally owns schematic edits. Do not modify `.kicad_sch`, `.kicad_pcb`, symbols, footprints, or project settings unless the user explicitly asks for that change.
- Do not overwrite, revert, or clean up existing KiCad changes. Assume uncommitted edits are intentional user work.
- Ignore autosave, lock, local-state, and backup artifacts for design review unless the user specifically asks to recover them.
- When reviewing the schematic, report findings using the hierarchical sheet name plus concrete landmarks such as reference designator, net name, connector pin, or nearby component.
- The user prefers sequential reviews that stop at the first material problem instead of accumulating a long list of speculative issues.

## Documentation Rules

- Keep the README architectural and concise. Put implementation details, firmware protocols, timing, validation steps, and design rationale in `docs/`.
- PCB release history belongs in the release-note table on page 1 of `pcb/OpenXTCM5-Head-Unit.kicad_sch`. `X*` PCB revisions are unreleased and not production-ready.
- Do not invent a repository-wide revision number. Hardware, RP2354 firmware, and CM5/Android software may be revised independently. Give firmware and software their own changelogs when those deliverables are introduced.
- Keep related records synchronized when an accepted architecture decision changes. At minimum, check `README.md`, `docs/README.md`, the relevant firmware plan, Android bring-up plan, and portfolio note for stale claims.
- Use lowercase kebab-case for project-authored documentation names. Preserve vendor filenames and directories.
- Mermaid diagrams belong inline in the document they explain; portfolio diagram source belongs under `docs/portfolio/openxtcm5-head-unit/diagrams/`.

## Accepted Architecture Decisions

- CM5 owns Android, Linux drivers, display, touch, user-facing USB applications, modem/camera applications, and audio policy.
- RP2354 owns real-time vehicle policy, `SYS_HOLD`, vehicle input processing, CAN FD, branch power/fault control, codec control, amplifier behavior, and thermal policy.
- `SYS_HOLD` is an RP2354-owned bounded shutdown lease. Rev X0 targets clean shutdown and cold boot; do not describe it as hibernation or RAM-retention support.
- The normal internal CM5-to-RP2354 USB link is composite: UAC1 audio plus CDC ACM board management. RP2354 BOOTSEL recovery is an explicit, separate re-enumeration mode.
- The RP2354 is the USB Audio-to-I2S bridge and I2S master. It uses Pico SDK plus TinyUSB for the initial implementation; PIO/DMA drive I2S to the TLV320AIC3104. Zephyr is deferred, not rejected. See `docs/rp2354-firmware-platform.md`.
- J1402 is a self-powered CM5 USB-C maintenance/UFP data port. Its VBUS pin is not a board power source.

## Verification and Communication

- Prefer narrow, relevant checks over broad cleanup. Do not claim ERC, layout, firmware, Android, or hardware validation has passed unless it was actually run and its scope is clear.
- If a design change alters ownership, sequencing, an external connector, a power rail, or a safety-relevant behavior, update the relevant design record in the same task.
- Use exact net names and active-low notation from the schematic. In prose, state the physical behavior as well as the logical name when polarity could be confusing.
