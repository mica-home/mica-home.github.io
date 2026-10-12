# How to Write Tests for MICA

This page is the authoring guide for MICA's test suites. It records the conventions
established by the drivers-layer pilot (the RNG and Keithley 2450 families): where tests
live, how they are named, how LUnit is invoked in CI, what a simulation-mode smoke test
is allowed to assert, and how fixtures must be written. Follow it for every new suite.

## 1. Naming and directory conventions

- Tests live in a `tests/` subdirectory next to the code under test:
  `drivers/<family>/tests/` for instrument drivers (same pattern as the existing
  `utility/Axis Tools/tests/`).
- All new files are written in English. Test case classes are named
  `<Device> <Scope> Tests` — the pilot classes are `RNG Driver Tests` and
  `Keithley 2450 Tests`. Member VIs are named `Test <Behavior>.vi`, for example
  `Test Config Parse Valid.vi` or `Test Sim Measure Smoke.vi`.
- JSON fixtures sit in the same directory, named `<family>_config_<variant>.json`
  (e.g. `rng_config_valid.json`). Run reports land in `tests/out/`; the final green
  report of a delivered suite may be committed there as evidence.
- The minimum matrix per family is: config-parse cases (valid, defaults, invalid) plus
  one simulation-mode smoke per template method (measure; source where the family
  supports it). Coverage targets are expressed as this family-times-case matrix, not as
  coverage percentages.

## 2. LUnit usage and the CI invocation

MICA uses LUnit (Astemes) for unit tests. A test class derives from LUnit's
`Test Case.lvclass`; assertions are members of that base (`Pass If Equal.vim`,
`Pass If Error.vim`, ...).

**Chain every assertion on the accumulator wire.** Only assertions wired in series on
the test-case wire that reaches the class Out terminal are collected. An assertion wired
off to the side neither counts nor fails — the suite stays green while asserting nothing.
This defect shipped in the pilot's first build and was invisible to all-green runs; only
a deliberate negative control exposed it.

The CI invocation (final form, as validated on the pilot channel):

```text
LabVIEWCLI -OperationName LUnit
  -AdditionalOperationDirectory "<LabVIEW>\vi.lib\Astemes\LUnit CLI"
  -Path <tests directory>
  -ReportPath <out>\lunit-junit.xml
  -PortNumber <VI Server port>
```

- `-Path` is scanned recursively. Test classes do **not** need to be members of
  `Lab_Super.lvproj` (or of any project); a class in a plain directory is discovered and
  run.
- Always pass `-PortNumber` explicitly. The LUnit CLI default is 3363 while this
  station's VI Server port is 3364; scripts must parse the value from `LabVIEW.ini` at
  run time instead of hard-coding it.
- `-Headless`, if used, must be the **last** argument and requires LabVIEWCLI 2026 Q1 or
  later.
- Use Windows backslash paths. The forward-slash form of the same command fails with
  error -350006 under Git Bash; backslash paths work from both Git Bash and PowerShell.
- Keep exactly one operation class named `LUnit` under the additional-operation
  directory. A duplicate residue trips error -350007 and is keyed on the class name, not
  the folder name.

Authoring recipe with the LabVIEW-MCP tooling (the route used for the pilot and for this
documentation set): create the test class parented on `Test Case.lvclass`
(`lvai_create_class`), restart LabVIEW once to clear the class-creation lock, then add
member methods with `lvai_lunit_add_test_method` (pane pattern 4815; the two class
terminals are derived from the class file name). Re-adding a member over an existing one
is safe.

**Two CI traps every test author and the test runner must know:**

1. `LabVIEWCLI` exits 0 even when a test fails. It prints the failing case to stdout and
   still reports "LUnit operation succeeded"; exit code 1 appears only for
   operation-level failures (wrong port, missing operation). Red/green must be derived
   from the fresh JUnit XML: sum the `failures` attributes and require at least one
   `testcase` element — a run that found no tests also reports green.
2. LUnit CLI never overwrites an existing report. A rerun writes a numbered sibling
   (`lunit-junit (1).xml`) and leaves the stale file at the named path, so reading
   `-ReportPath` after a rerun returns the *previous* run's verdict. Delete the report
   and its numbered siblings before every run, then read back the exact named path.

## 3. Raw/Sim conventions and what a Sim smoke test asserts

Drivers are written against a Raw/Sim split: the Raw state talks to a real instrument
(the `... Raw.vi` implementations), the Sim state simulates the device. Which state runs
is decided solely by the `simulation` boolean in the channel configuration, consumed by
the Base library's `Read simulation.vi`; the BaseDriver template methods (`Init`,
`Measure`, `Source`) dispatch to the selected implementation.

CI runs **strictly in simulation mode** — the runner has no instruments attached. A Sim
smoke test asserts exactly two things:

- the Init → Measure/Source → Close chain completes with **no error**, and
- the returned data has the **correct shape** (e.g. a one-element numeric result).

It does not promise that the values are physically correct. Hardware truth stays with the
manual real-instrument smoke described in the user manual. Smoke fixtures use an
instrument address nothing on the runner answers, so an accidental Raw dispatch fails
loudly instead of silently passing.

## 4. Fixture rules

- Each family's `tests/` directory is **self-contained**: it carries its own minimal JSON
  fixtures, and tests never read `configs/production/` at run time. Production samples
  are shape references only. This prevents production config drift from silently
  changing what a test exercises.
- Every fixture **must** carry the `simulation` boolean — it is the only Sim/Raw switch.
- Key names are defined by what the parser actually reads (`Parse common-parameters.vi`
  of the family or of Base), not by what a config "should" contain. The measured key
  sets from the two pilot families:
  - Common keys: `model`, `name`, `short`, `address`, `channel`, `mode`, `simulation`,
    `extra-parameters`.
  - RNG `extra-parameters`: `non-block`.
  - Keithley 2450 `extra-parameters`: `autorange`, `autozero`, `compliance`,
    `non-block`.
  - Omitted keys fall back to parser defaults; the RNG defaults fixture, for instance,
    omits `address` and leaves `name`/`short` empty to pin those defaults.
- Invalid-branch fixtures are deliberately malformed JSON (e.g. a `trux` token inside
  `extra-parameters`) and pin the parser's measured error behavior (the JSONtext 402841
  error, with the parser's documented fallbacks) — an explicit recorded expectation, not
  an assumed crash.

## 5. CI behavior and blocking semantics

- `test.yml` runs the suites on push to `main` and `feat/testing` when code paths
  change; changes under `docs/**` do not trigger it (documentation has its own publish
  workflow, `.gitea/workflows/docs.yml`).
- Results are recorded honestly: there is no `continue-on-error` and no swallowed exit
  code. A summary step gated on `if: always()` re-throws the test exit code, so the job
  — and the commit status — show real red when a test fails.
- The lane is **non-blocking by design** for now: a red job is a real failing commit
  status, but it does not block push or merge (no required status checks, no release
  gate). Promoting the lane to a release gate is a separate, later decision.
- Verdicts come from the JUnit XML as described in section 2; the XML is echoed into the
  job log, which is the permanent record (no artifact upload).

## 6. Batch expansion status

These conventions come from the two-family pilot and were validated across the full
batch: all 30 remaining implementation families now carry suites (32 LUnit channels
plus the UTF suites, all green locally and in CI). Rolling suites out to the utility,
cores and UI layers remains gated on a separate owner decision.

## 7. Batch authoring route (measured, 2026-10-11)

The 30 batch suites were authored through one pipeline. Its steps and the traps they
surfaced are the reference for any future suite authoring:

1. Recon the family chain first (members, parse keys, the measured invalid-JSON error
   code, Sim feasibility) — never copy another family's expectations without measuring.
2. Author the test-method AIXML sources with placeholder stub calls; keep every
   assertion chained on the LUnit Test Case accumulator line.
3. Create the test class, then restart LabVIEW once (class-creation lock). Watch for
   the measured `lvai_create_class` defect that can duplicate every class entry in
   `Lab_Super.lvproj` (Error 74 on the next project open) — verify the LVClass count
   right after and restore the file if it doubled.
4. Generate members. The MCP add-method call reliably hits a 30-second client cap after
   its first phase and can then spin; the working completion is the shipped
   `lvlu_add_test_method.vi` helper run directly (per-stage error indicators must all
   be zero, terminals retyped = 2).
5. Swap stubs for real members one call per node, and re-point the seed constant at the
   SUBJECT class — not the test class. Two silent-failure traps: swapping the init
   socket to the wrong VI drops the config wires while every swap answer stays green,
   and a wrong seed class leaves the dynamic parse falling back to BaseDriver; the
   dynamic-resolution export check is only meaningful after the seed swap.
6. Re-apply icons after any regeneration; assert every member executable; then prove
   the suite with three local runs: GREEN, a fixture-mutation negative control (must
   go RED with exit 3), and the restored GREEN.
7. Register the family in `ci/run-tests.ps1` and commit one family per commit.

## 8. Early abort

The lane supports `MICA_TEST_EARLY_ABORT=1` (set by the workflow) or a local
`-EarlyAbort` switch: after the first infrastructure-level channel failure (timeout or
environment) the remaining LUnit directories are skipped so one wedging CLI call cannot
eat the job budget. Reports already produced are kept, the summary still prints every
recorded channel, and the exit code stays the most-severe outcome. A real test RED
never aborts early.

## 9. Known pinned driver defects

The suites pin six classes of measured driver defects as current behavior, each with a
KNOWN DRIVER BUG note and a flip-back comment (the drivers themselves were not
modified). They are tracked as issues #95-#100: the shared autozero `"once"` and the
LS155/LSM81 quoted-token branch fallthroughs, the LSM81 `Sqaure` label, the Lock-in
Amplifier `auto-phase` key claiming three booleans, the Controller-chain legacy
`wait-for` alias, and the recorded modal/unreachable coverage gaps. Fixing a driver
means flipping the corresponding pinned expectations in the affected suites.
