# tooling — tools, configurations, typedefs and build outputs

- **Main class:** none — this module is class-free. `Tools/` contains exactly seven standalone
  top-level VIs; `typedefs/`, `configs/`, `inifiles/`, `builds/` and `ci/` hold no LabVIEW classes at
  all.
- **Entry VIs:** each tool VI is its own entry point —
  `Tools/Config Editor/ConfigEditor.vi` and `Tools/Updater/Updater.vi` (the two in subdirectories), plus
  `Tools/2450 continuous.vi`, `Tools/CrOCl Device Capacitance Calculator.vi`,
  `Tools/KE26XX Config Source & Measure Tool.vi`, `Tools/LS336 T Control.vi`,
  `Tools/VISA Instrument List.vi`.

## Tools (7 VIs, each backed by an EXE build specification)

| Tool VI | Build spec | Notes |
|---|---|---|
| `Tools/VISA Instrument List.vi` | VISA Instrument List.exe | Purpose implied by name; internals not inspected in this pass. |
| `Tools/LS336 T Control.vi` | LS336 T Control.exe | Standalone temperature control front end for the LakeShore LS336 driver; internals not inspected in this pass. |
| `Tools/Config Editor/ConfigEditor.vi` | Config Editor.exe | Editor for the JSON configuration documents described below. |
| `Tools/KE26XX Config Source & Measure Tool.vi` | KE26XX Config Source & Measure Tool.exe | Standalone source/measure front end for the Keithley 26xx family. |
| `Tools/CrOCl Device Capacitance Calculator.vi` | CrOCl Device Capacitance Calculator.exe | Purpose implied by name; internals not inspected in this pass. |
| `Tools/2450 continuous.vi` | 2450 continuous.exe | Continuous-measure front end (`Run when opened = false`); its build spec additionally bundles the KE26XX tool VI and still carries the KE26XX file description/internal name/product name — a copy-spec leftover, recorded rather than silently "fixed". |
| `Tools/Updater/Updater.vi` | Updater.exe | Self-updater; command-line arguments enabled; packaged into the Installer together with the app. |

None of the tools participates in the application startup chain; each is an independent executable for
interactive use. The six plain tool EXEs are not bound to an ini item, so LabVIEW generates a same-named
`.ini` in their output directory at build time.

## configs — JSON instrument configurations

`configs/` holds the JSON instrument-configuration documents that drive a MICA session (channel lists,
driver selection, measurement parameters). `typedefs/channel type.ctl` — a cluster of name, short,
address, channel index, model, mode and `<JSON>extra-parameters` — is the serialized currency of these
files, mirroring the BaseDriver private fields one-to-one. The JSON schema of these files, the
extra-parameter key tables and sanitized worked examples are documented in
[user-manual.md](../user-manual.md), "Channel configuration reference" and "Worked examples". Two
groups exist:

- `configs/*.json` at the folder root: working configurations (e.g. `config.json`,
  `config_lab_test*.json`).
- `configs/examples/`: 20 `ex_*.json` example configurations are registered in the project file
  (ex_2450_SR830, ex_6220_2182, ex_dual_2450, ex_LS336, ex_LS336_dual_2636, ex_Mercury_ips,
  ex_Mercury_itc, ex_Mercury_itc_heliox, ex_Nova_2400, ex_OminiL, ex_OminiL_2450, ex_PPMS, ex_SR830,
  ex_SR865, ex_LS155_AC, ex_LS155_DC, ex_LSM81_ACS_ACM, ex_LSM81_ACS_LIA, ex_LSM81_DCS_DCM,
  ex_LSM81_DC_CM_input_bias). The folder carries 24 JSON files on disk — `ex_2182.json` and three
  non-`ex_` files (`2602B.json`, `2636B.json`, `2636B_LS155DC_M81DC.json`) exist on disk but are not
  among the 20 registered documents.

## inifiles — launcher configuration

Four ini files, each bound to a build specification: `Launcher-Debug.ini`, `Launcher-Release.ini`,
`Launcher-Simulation.ini` (one per Launcher EXE variant) and `Updater.ini`. The three Launcher specs
differ only in the bound ini and output directory; their source lists are identical.

## typedefs — the central shared typedef folder

Five parsed, bindable typedefs shared across all tiers:

| File | Kind | Structure |
|---|---|---|
| `typedefs/Axis Name Type.ctl` | enum (u16) | Physical Name / Short Name / Long Name / Mixed Name |
| `typedefs/Channel Config.ctl` | cluster | Input / Output / Enable, each an array |
| `typedefs/Channels.ctl` | cluster | Input / Output, each an array |
| `typedefs/Source.ctl` | **strict** enum | Current / Voltage — the repository's only strict typedef |
| `typedefs/channel type.ctl` | cluster | name, short, address (strings), channel (i32), model, mode, `<JSON>extra-parameters`; control label is "Cluster", not the file name |

For the remaining 28 `.ctl` files scattered through `app/`, `cores/`, `drivers/`, `utility/` and
`controls/`, see `docs/modules/utility.md` and `docs/modules/cores.md`.

## builds — build output area

`builds/` currently contains five output directories — Launcher-Debug, Launcher-Release,
Launcher-Simulation, MICA Installer, Updater — matching five of the eleven build specifications declared
in the project file; the six tool-EXE output directories are built on demand and absent on disk. The
project declares 10 EXE specs plus 1 Installer (3 Launchers from `Splash-Screen.vi`, 6 tools,
Updater, and the MICA Installer packaging Launcher-Release + Updater). The complete 11-specification
table with sources, output paths, ini bindings and version fields lives in
`docs/architecture.md`, section "Build Specifications".

## ci — continuous integration

The repository's `ci/` directory currently holds a single environment-definition file (`ci/vm.env`);
no pipeline definitions are tracked there, and CI as such is out of scope for this documentation pass.

## Notes for readers

- The tool EXEs, the Launchers and the Updater are all declared enabled in the project file (no
  per-spec disable flag exists).
- Path caveat: item URLs and build output paths in the project file carry a `../` prefix that does not
  resolve literally against the project's location at the repository root; actual outputs land in
  `builds/`. Discussed in `docs/architecture.md`.
- Documentation baseline: the project-file parse and disk cross-checks under `.omz/tmp/lvmcp-docs/`
  (lvproj-inventory, inventory, typedefs, wave findings). Tool internals beyond ConfigEditor's and
  Updater's build-spec metadata were not deep-read in this pass.
