# MICA System Architecture

MICA (Multi-Instrument Control and Automation) is the application defined by the `Lab_Super.lvproj`
project at the repository root. It is a multi-window, multi-instrument control program that drives
several interchangeable measurement schemes against a shared pool of laboratory instruments. The
system is written in LabVIEW 2026 on the Actor Framework (AF) pattern, extended with the MGI
`Panel Actor` and `Monitored Actor` base classes, and it swaps measurement cores at runtime through
a subpanel host.

This document was compiled from internal wave-analysis artifacts of this repository; each section
ends with a source line naming the artifact and section every claim was taken from. No claim was
re-derived from memory. For operation-level documentation — installing, running, and driving
measurements — see [getting-started.md](getting-started.md) and [user-manual.md](user-manual.md).

## 1. System overview

(source: wave1-findings §2; wave2-deltas §1)

- The repository holds 1,267 LabVIEW files: 1,003 VIs, 189 classes, 41 libraries, 33 typedef
  controls and 1 malleable VI, across twelve top-level directories (`app`, `manager`, `cores`,
  `drivers`, `panels`, `utility`, `Tools`, `controls`, `typedefs`, `configs`, `builds`, plus the
  project root with `Splash-Screen.vi`).
- The project file itself lists 267 items. The difference to the disk census is a counting rule,
  not missing code: most VIs and classes are members of the 41 libraries and are owned there
  (file-level inventory authoritative).
- Sources stay unpacked in the shipped build (`SourceOnly`), so executables reference source files
  rather than packing them into binaries.
- One process, many windows: the App actor owns the main window, measurement cores are nested panel
  actors hosted in a subpanel, and plotters, loggers and dialogs are separate managed panel actors.
- A unit-test configuration (NI Unit Test Framework properties) exists in the project file; no other
  test framework is configured.

## 2. Actor topology and messaging

(source: wave1-findings §1, §3; wave2-vi-findings §5-§7, §12, §13)

### 2.1 The four inheritance chains

```text
LabVIEW Object
└─ BaseDriver.lvclass (drivers/Base)              # drivers are plain classes, NOT actors

Actor Framework Message (vi.lib)
└─ 76 message classes, one <Name> Msg.lvclass each

MGI Monitored Actor (vi.lib)
├─ manager.lvclass (manager/manager)              # central hub; no Actor Core override
└─ Helper.lvclass (cores/Helper)                  # measurement-loop helper root

MGI Panel Actor (vi.lib)
├─ App.lvclass (app/App)                          # process root panel actor
├─ Outputers.lvclass ── Plotter.lvclass           # plotting family (panels/)
└─ MicaSubpanel.lvclass (panels/MicaSubpanel)     # subpanel host
   └─ Core.lvclass (cores/Core/Core)              # subpanel-embedded measurement core
      ├─ D-Y Map, Log, Map, Server, Sweep         # variant cores on the same template
      └─ paired helpers: D-Y Map Helper, Log Helper, Map Helper, Server Helper, Sweep Helper
```

1. `BaseDriver.lvclass` derives directly from LabVIEW Object. The driver tier is deliberately not
   made of actors: driver objects are composed and called by the core tier.
2. `Core.lvclass` reaches MGI `Panel Actor` through `MicaSubpanel.lvclass`, which carries the
   subpanel refnum. App and Core are therefore not siblings - Core sits one level deeper, and the
   subpanel host is what lets cores be swapped into the main window at runtime.
3. `manager.lvclass` and `Helper.lvclass` share the MGI `Monitored Actor` root. The manager has no
   `Actor Core.vi` override anywhere under `manager/`; its behavioral center is
   `manager/manager/launch core.vi` plus its `on_*` message handlers.
4. `App.lvclass` is itself a MGI `Panel Actor` and the root of the actor tree at runtime.

### 2.2 Message triplet convention

Every message is a folder of exactly three files: `<Name> Msg.lvclass` (the message class carrying
its payload as private data), a public dynamic-dispatch `Do.vi` (the AF override that executes the
action on the receiving actor), and a public static `Send <Name>.vi` (the enqueuer callers invoke).
There are 76 such message classes (app 6, manager 20, cores 16, panels 34; one artifact reports 77 -
the file-level inventory is authoritative). (source: wave1-findings §1, §8)

### 2.3 The seven core lifecycle messages

The Core tier exposes a consistent measurement lifecycle as messages, and the same verbs recur on
the App, manager and Helper tiers (Pre-measure, Post-measure, Measure, Update Drivers,
Force-stop-measure, On hot sim switch):

| Message | Role |
|---|---|
| Init Drivers | prepare the driver tier for a run |
| Pre-measure | per-run preparation before the measurement loop starts |
| Measure | trigger one measurement round through the driver tier |
| Post-measure | per-run teardown after the measurement loop ends |
| Force-stop-measure | abort a running measurement immediately |
| On hot sim switch | flip the stack between real (Raw) and simulated (Sim) instruments at runtime |
| Close Drivers | release the driver tier after the run |

The Core message set additionally contains `Set Driver Ready`, `Update Drivers` and
`On Enable Selection Changed`, which carry configuration and UI state between tiers. Handler
internals were not individually inspected in this pass; the roles above are read from the message
names and their cross-tier usage. (source: wave1-findings §1, §7.11)

### 2.4 Runtime composition

App (top panel actor) drives the manager - it holds the manager's enqueuer in its private data. The
manager launches and stops managed actors and panels and registers "adapters" for them:
`manager/adapter/adapter.lvclass` plus four per-panel adapters (ConfigViewer, DataLogger, DataTipper,
Plotter), each overriding the hook set `on_measure.vi` / `on_pre_measure.vi` / `on_post_measure.vi` /
`on_update_config.vi`. The adapters decouple the manager from concrete panel classes. Cores run
inside the MicaSubpanel host, and each core pairs with a helper actor that runs the actual
measurement loop. Managed panel actors: ConfigViewer, DataLogger, DataTipper, the Outputers/Plotter
family, WaitingModal, About, Check for Update, HistoryLogger, CoreSelectionPanel, QuickAccessBar.

The core-family template, verified by reading the Core, Map and Sweep `Actor Core.vi` overrides:
each core collects its run controls into the `GUI_refnums` typedef cluster, disables its Start/Stop
buttons, launches its Helper as a nested actor, cross-wires the two enqueuers (each writes the
other's enqueuer into its own private data), then calls the parent method to enter the event loop.
Plotter repeats the nested pattern one level down, hosting Figure, VariableViewer and AxisManager
subpanels and cross-wiring their enqueuers.

`manager/manager/launch core.vi` is the Core-switch orchestrator: in its No-Error case it stops any
previous Core (`Send Normal Stop`, then wait up to 5000 ms; a dialog plus error clear if the old
Core refuses to quit), launches the new Core as a nested panel actor of the manager, sends the new
Core an `Update Drivers` message carrying the Channel Config, and - if the manager's `Driver Ready?`
flag is set - replays a `Set Driver Ready` message so the new Core inherits the previous readiness
state.

![manager launch core.vi block diagram](images/manager-launch-core.png)
*Figure 1. The manager's Core-switch orchestrator: stop the previous Core, launch the new one as a
nested panel actor, then push Channel Config and replay Driver Ready as messages.*

## 3. Module map

(source: wave1-findings §2)

- **app/** - 30 VIs, 7 classes, 1 library. The App panel actor: GUI refnum registry, logging, menu
  handling and core switching; contains the process entry point `app/Launch App.vi` and 6 message
  classes.
- **manager/** - 81 VIs, 26 classes, 1 library. The central Monitored Actor hub with 20 message
  classes; it launches and stops managed actors and panels and registers adapters (1 base plus 4
  panel adapters).
- **cores/** - 116 VIs, 29 classes, 12 libraries. Six measurement cores (Core, D-Y Map, Log, Map,
  Server, Sweep), their helper actors, the `cores/load_actors.vi` actor class registry and per-core
  config typedefs; Core is the subpanel-embedded base class. Two naming quirks to know when reading
  this tree: a D-Y Map class file exists twice on disk under the same qualified name, and the class
  constants inside `load_actors.vi` read "D-T Map" while the files are named "D-Y Map" - left
  unresolved in this pass.
- **drivers/** - 228 VIs, 65 classes, 7 typedefs. Sixteen instrument families under the BaseDriver
  base class with a Raw/Sim operation split, plus the `drivers/load_drivers.vi` driver class
  registry.
- **panels/** - 193 VIs, 53 classes, 19 libraries. Fifteen UI panel libraries including the Plotter
  plotting subsystem (8 libraries), the MicaSubpanel subpanel host, WaitingModal, and panel
  utility VIs.
- **utility/** - 107 VIs, 3 classes, 8 libraries, 1 malleable VI. The AppConfig GOOP-style singleton,
  the LV-Argparse CLI framework, the Channel Config class, Axis/Layout/Panel/Path/Version tool
  libraries, the ZolixOminiSpec vendor library, loose numeric/format helpers, and the raw
  `utility/measure.vi` SCPI helper.
- **Tools/** - 7 VIs. Seven standalone tool VIs, six of which are the sources of the six tool EXE
  build specifications.

Supporting tiers outside the seven application directories: `controls/` (QControl-style enhanced
controls), `typedefs/` (five central shared typedefs, including `Source.ctl`, the repository's only
strict typedef), and `configs/` (20 example JSON configuration documents registered in the project).

## 4. Measurement data flow

(source: wave1-findings §1, §6; wave2-vi-findings §8, §9, §11, §15)

1. **Configuration.** Channel setups live as JSON documents (see the examples in `configs/`). The
   `channel type.ctl` typedef mirrors the BaseDriver private fields one-to-one and is the currency
   of these JSON files. The manager holds the active Channel Config in its private data.
2. **Into the core.** When a core is launched or reconfigured, the manager pushes the configuration
   as an `Update Drivers` message to the core's enqueuer - config arrives as a message, not set
   pre-launch (measured in launch core.vi).
3. **Cores and helpers.** The measurement cores (e.g. Sweep, Map) collect run parameters such as
   sweep rate, time per point, limits and channel selection from their front panels into per-core
   config typedefs (`cores/Sweep/Sweep Config.ctl`, `cores/Map/Map Config.ctl`,
   `cores/D-Y Map/D-Y Map Config.ctl`) and hand them to their helper actors, which run the timing
   loop.
4. **Drivers.** Helper loops call `drivers/Base/Measure.vi`, the single public measure entry. It is
   a template method: three nested case structures dispatch over simulation (Raw/Sim) and
   blocking/non-blocking, call one of four dynamic-dispatch hooks (`Measure Raw`, `Measure Sim`,
   `Non-blocking Measure Raw`, `Non-blocking Measure Sim`), then run the per-driver
   `Handle Error.vi` hook. Blocking results are returned and cached in the driver's Data Cache, so a
   later `Get non-blocking measure result.vi` still returns the latest data immediately on
   instruments that only support blocking measurement. The generic write-then-read primitive is
   `drivers/Base/Query.vi`: read the driver's stored VISA address, VISA Write, wait (OpenG wait),
   VISA Read. Per-model hooks (e.g. the 2450 source hook) delegate to vendor instrument libraries
   that own the actual VISA session.
5. **Panels.** Results flow to the panel tier through the manager's adapters, which fan the
   measure/pre-measure/post-measure/update-config events out to the registered panels. The Plotter
   family renders traces (Figure / VariableViewer / AxisManager); DataLogger covers the data-logging
   side, and MICA data files - TAB-delimited with at least one header line, first column label
   discarded - are read back by `utility/parse MICA data file.vi`.

## 5. Startup chain and launcher variants

(source: wave1-findings §5, §7.10; wave2-vi-findings §1-§4)

Three launcher build specifications share one source (`Splash-Screen.vi`) and differ only in the
bound INI file and the output directory:

| Launcher | Bound INI | Purpose |
|---|---|---|
| Launcher-Debug | inifiles/Launcher-Debug.ini | development builds |
| Launcher-Release | inifiles/Launcher-Release.ini | shipped builds |
| Launcher-Simulation | inifiles/Launcher-Simulation.ini | simulated-instrument builds |

Simulation is a first-class architectural feature, not a build-time afterthought: driver operations
come in Raw/Sim pairs, BaseDriver carries a `simulation` Boolean field, and `On hot sim switch`
messages exist at the App, manager and Core tiers, so a running system can flip between real and
simulated instruments.

The startup chain, phase by phase (the loader steps inside the splash screen run asynchronously, so
a strict intra-phase order is not claimed):

1. `Launcher-<variant>.exe` starts `Splash-Screen.vi` with the variant INI.
2. The splash screen resolves its own location (`Get App Path.vi`), shows a transient window, and
   asynchronously preloads both class registries via strict, start-asynchronous calls:
   `cores/load_actors.vi` returns the registry of every launchable actor class - one class constant
   per actor, covering App, manager, the six cores, their helpers and the panel libraries, with link
   tables spanning 22 libraries - and `drivers/load_drivers.vi` returns the driver class registry,
   roughly 68 class constants covering the sixteen instrument families. Both registries use the same
   error-case idiom and act as the application's static plugin tables. The splash screen then makes
   itself transparent and closes.
3. The application is started by `app/Launch App.vi`, whose inclusion in every Launcher spec is
   explicit in the build manifest: it builds the App actor object (window title "Multi-Instrument
   Control and Automation (MICA)") and launches it as the root panel actor via the MGI panel
   launcher.
4. The App actor drives the manager, which launches the managed panel actors and the first Core
   into the subpanel.

`utility/measure.vi`, despite its name, is not part of this chain - it is a standalone VISA
source-measure utility included here only as the example of "utility VIs call driver libraries
directly".

![app splash screen](images/app-splash-screen.png)
*Figure 2. Splash-Screen.vi - transient startup window that asynchronously preloads the actor and
driver registries while the application starts.*

![cores load actors.vi block diagram](images/cores-load-actors.png)
*Figure 3. cores/load_actors.vi - the actor class registry consulted during startup; its error case
passes through, its No-Error case enumerates one class constant per launchable actor.*

## 6. Build artifacts

(source: wave1-findings §4; wave2-deltas DELTA 2.1 - file-level inventory authoritative)

All 11 build specifications are enabled in the project file. Every EXE spec also produces a Support
Directory next to the executable.

| # | Spec | Type | Output | Top-level source | Bound INI | Output on disk |
|---|---|---|---|---|---|---|
| 1 | VISA Instrument List | EXE | VISA Instrument List.exe | Tools/VISA Instrument List.vi | auto-generated | not built yet |
| 2 | LS336 T Control | EXE | LS336 T Control.exe | Tools/LS336 T Control.vi | auto-generated | not built yet |
| 3 | Config Editor | EXE | Config Editor.exe | Tools/Config Editor/ConfigEditor.vi | auto-generated | not built yet |
| 4 | KE26XX Config Source & Measure Tool | EXE | KE26XX Config Source & Measure Tool.exe | Tools/KE26XX Config Source & Measure Tool.vi | auto-generated | not built yet |
| 5 | CrOCl Device Capacitance Calculator | EXE | CrOCl Device Capacitance Calculator.exe | Tools/CrOCl Device Capacitance Calculator.vi | auto-generated | not built yet |
| 6 | 2450 continuous | EXE | 2450 continuous.exe | Tools/2450 continuous.vi | auto-generated | not built yet |
| 7 | Launcher-Debug | EXE | Launcher.exe | Splash-Screen.vi | inifiles/Launcher-Debug.ini | built |
| 8 | Launcher-Release | EXE | Launcher.exe | Splash-Screen.vi | inifiles/Launcher-Release.ini | built |
| 9 | Launcher-Simulation | EXE | Launcher.exe | Splash-Screen.vi | inifiles/Launcher-Simulation.ini | built |
| 10 | Updater | EXE | Updater.exe | Tools/Updater/Updater.vi | inifiles/Updater.ini | built |
| 11 | MICA Installer | Installer | install.exe | packages Launcher-Release + Updater into user folder MICA | n/a | built |

"Not built yet" reflects the output area at documentation time: only the three Launcher, Updater and
Installer output directories exist on disk; the six tool EXE directories do not. The specs are
declared enabled either way, so the tool EXEs read as build-on-demand.

Details worth knowing when reading build output:

- The three Launcher specs have identical source lists (configs, drivers, cores, manager, panels,
  utility, App.lvlib including Launch App.vi); they differ only in INI and output directory.
- The six tool EXEs are not bound to an explicit INI item, so LabVIEW generates a same-named INI in
  the output directory.
- The "2450 continuous" spec still carries the KE26XX file description and internal name and also
  bundles the KE26XX tool VI - a copy-spec leftover, recorded here rather than silently "fixed".
- The Installer records an effective source count of two (Launcher + Updater) while legacy source
  tags remain in the project XML; it ships the NI-VISA, LabVIEW 2026, NI-DAQmx and NI-488.2 runtimes
  and reports product version 1.0.0.
- The Updater enables command-line arguments.

## 7. Error handling and logging

(source: wave2-vi-findings cross-cutting findings; wave1-findings §1, §2)

Error handling follows one uniform idiom: every VI guards its work with a Case Structure keyed on
the error cluster, and the Error frame passes the cluster through untouched. On top of that base:

- Drivers run a per-driver `Handle Error.vi` hook after every Measure dispatch, so each instrument
  family defines its own error policy.
- The manager treats a failed Core shutdown as recoverable: it shows a one-button dialog ("Previous
  Core still running before switching to new Core.") and clears the error so the new launch
  proceeds.
- The Plotter merges property and message errors with Merge Errors before registering its events.
- The raw SCPI utility path (`utility/measure.vi`) runs an instrument error query after each read.

Logging is spread over three cooperating pieces: the App actor carries a dedicated Logging user
event its panels can raise; the HistoryLogger panel keeps the human-visible history; and the Log
core family (cores/Log with its Log Helper - the actor registry groups DataLogger, ManualLog and
DataTipper with it) covers the measurement-data logging side. Generic format and numeric helpers
live in `utility/`.
