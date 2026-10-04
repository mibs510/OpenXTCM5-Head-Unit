# OpenXTCM5

OpenXTCM5 is an in-progress open hardware project for building a more capable, repairable Android head unit around a Raspberry Pi Compute Module 5.

## Why This Exists

This began as both a learning project and a response to the inexpensive Android head units that are common in aftermarket vehicle installs. The unit currently in use overheats around its TDA7388 audio amplifier. Once heat builds up, Android becomes severely laggy, reverse-camera use can freeze the system, and audio can repeat the last fragment played until the unit reboots.

The aim is not just to replace that unit, but to understand and improve the entire design: protected automotive power, controlled startup and shutdown, display integration, vehicle I/O, communications, camera capture, audio, and thermal management. A 1U server fan is planned for active amplifier cooling so the audio stage is no longer allowed to quietly cook the rest of the system.

## Current State

The project is in the schematic-design stage. There is no PCB layout or validated prototype yet. Design decisions are being documented as the schematic develops, with a longer build article planned later.

## Documentation

See [docs/README.md](docs/README.md) for the current design notes and integration plans.

## Repository Layout

* `*.kicad_sch` - KiCad schematic sheets
* `OpenXTCM5-Head-Unit.kicad_*` - top-level KiCad project files
* `docs/` - design rationale and implementation plans
* `lib/` - locally managed design references and library material

## Project Status

This is a personal, evolving engineering project. The schematic and documentation should be treated as work in progress until the design is built and verified on hardware.
