# MICA Documentation Index

MICA (Multi-Instrument Control and Automation) is the multi-window, multi-instrument
measurement control program defined by the `Lab_Super.lvproj` project at the repository
root. This directory is the documentation set for that application.

## Documents

| Document | Summary |
| --- | --- |
| [architecture.md](architecture.md) | System-level architecture: Actor Framework topology and message conventions, the module map, the measurement data flow, the startup chain with its launcher variants, build artifacts, and error handling and logging. |
| [getting-started.md](getting-started.md) | Installation-to-first-measurement guide: prerequisites, install, first launch in Simulation mode, and where data lands. |
| [user-manual.md](user-manual.md) | Full operator reference: UI tour, the four measurement modes, channel-configuration reference with code-adjudicated instrument tables, worked examples, data files, logs and troubleshooting, maintenance and update, notes and limitations. |

### Modules

| Document | Summary |
| --- | --- |
| [modules/app.md](modules/app.md) | The application root actor and process entry point: the top-level Panel Actor that owns GUI refnums, menu handling, core switching and logging, launched by `app/Launch App.vi`. |
| [modules/manager.md](modules/manager.md) | The central hub and core switcher: a Monitored Actor with the largest message inventory of any tier, concentrating its behavior in `launch core.vi` and its message handlers. |
| [modules/cores.md](modules/cores.md) | The measurement core family: twelve libraries, six interchangeable cores and six helper actors, hosted as subpanel-embedded panel actors and registered in `cores/load_actors.vi`. |
| [modules/drivers.md](modules/drivers.md) | The instrument driver tier: sixteen instrument families of plain, non-actor classes under the `BaseDriver` root, registered by `drivers/load_drivers.vi` and composed by the cores. |
| [modules/panels.md](modules/panels.md) | The UI panel actors and the plotting subsystem: `MicaSubpanel`, the panel-side root that lets measurement cores swap into the main window's subpanel, plus the Plotter libraries. |
| [modules/utility.md](modules/utility.md) | Configuration, parsing and shared tool libraries: the `AppConfig` singleton, the standalone `measure.vi` and `parse MICA data file.vi` utilities, and shared helpers. |
| [modules/tooling.md](modules/tooling.md) | Tools, configurations, typedefs and build outputs: seven standalone tool VIs, typedefs, configs and inifiles, build specifications and CI assets; this module defines no classes. |

> A legacy Chinese manual draft (`manual/zh-cn.md` in the repository) predates these guides and
> is superseded by user-manual.md.

## Images

All screenshots and block-diagram renders live in `images/`. Documents reference them
with relative paths: `images/<file>.png` from this directory, `../images/<file>.png`
from `modules/`.

The `images/ui-*` files are application UI screenshots captured in the application's
simulation-mode build (no hardware attached); they illustrate `getting-started.md` and
`user-manual.md` and are referenced from those documents. The table below covers the
block-diagram renders.

| Image | Owning module | Shown |
| --- | --- | --- |
| `images/app-splash-screen.png` | app | The splash screen front panel that preloads the actor and driver registries. |
| `images/cores-load-actors.png` | cores | Block diagram of the measurement-core actor registry. |
| `images/manager-launch-core.png` | manager | Block diagram of the manager's `launch core.vi`. |
| `images/drivers-load-drivers.png` | drivers | Block diagram of the driver class registry. |
| `images/drivers-basedriver-measure.png` | drivers | Block diagram of a `BaseDriver` measure method. |
| `images/panels-plotter-actor-core.png` | panels | Block diagram of the Plotter actor core. |
| `images/utility-measure.png` | utility | Block diagram of `utility/measure.vi`. |
| `images/utility-parse-mica-data-file.png` | utility | Block diagram of `utility/parse MICA data file.vi`. |

## Scope and conventions

- The documents in this directory were produced by read-only parsing of the repository
  with LabVIEW-MCP tooling: a file-level census, project-tree reads, VI Server property
  reads, AIXML exports and block-diagram renders. No source file was modified to produce
  them, and no claim was derived from memory.
- Target environment: LabVIEW 2026, Actor Framework extended with the MGI `Panel Actor`
  and `Monitored Actor` base classes.
- Coverage is honest by construction: only regions that were actually opened and read
  are documented. Where analysis did not reach a region, the documents say nothing about
  it rather than speculate; `architecture.md` additionally cites its source artifact at
  the end of every section.
- All cross-links are relative paths within `docs/`, so the directory can be moved or
  copied as a unit without breaking them.
- `getting-started.md` and `user-manual.md` derive their operational content from the project's
  official user manual (English rewrite, sanitized); every technical enumeration they make —
  instrument families, models, modes, config keys, INI keys, file format — was verified against
  the repository sources, and conflicts are annotated in the user manual.
- This site auto-publishes: changes pushed to the `docs` branch are deployed here by CI
  (see `.gitea/workflows/docs.yml` in the repository).
