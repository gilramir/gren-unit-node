# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`gilramir/gren-unit-node` — a Gren **package** (`platform: node`) providing an
xUnit / Python-`unittest`-style test framework: suite- and test-level fixtures,
CLI test selection by glob, per-test timing, and JUnit XML output. Public API is
`Test.Runner.UnitNode`; `Test.Runner.UnitNode.Glob` and
`Test.Runner.UnitNode.Testing` are the only other exposed modules (see
`gren.json`).

The parent directory `/home/gram/prj/gren-format/` has its own CLAUDE.md
covering the `gren-format` formatter repos — that is a *sibling* project, not
this one. This repo is its own git repo and its own devbox project.
`examples/cli-tests/` is a port of `gren-format`'s black-box CLI tests and is
the only link between them.

`README.md` is the authoritative user-facing documentation (setup, fixtures,
lifecycle-failure table, CLI flags, design rationale). Read it before changing
public API or lifecycle semantics.

## Build & Test

Everything runs through devbox (`devbox.json` pins `gren@0.6` and `nodejs@22`).
Devbox scripts always start at the repo root, so the ones that touch `tests/`
or `examples/` `cd` there themselves.

```bash
devbox run build          # `gren make` at the root — compiles the package, writes docs.json
cd tests && ./run-tests.sh    # build + run the self-test suite (this is the main gate)
devbox run test           # same suite via `gren run Main` (no CLI flags possible)
devbox run examples       # build examples/cli-tests into examples/cli-tests/app
```

`run-tests.sh` forwards its arguments to the built binary, so a single test or
suite is selected with the framework's own glob patterns:

```bash
cd tests
./run-tests.sh -v                       # per-test lines with timing
./run-tests.sh 'Glob.*'                 # one suite
./run-tests.sh '*backtrack*'            # one test (patterns are a union)
./run-tests.sh --list                   # qualified names only, run nothing
```

The self-tests use gren-unit-node to test gren-unit-node: `tests/src/Main.gren`
wires up `GlobTests` (pure glob matcher cases) and `RunnerTests` (lifecycle and
selection). `RunnerTests` builds an *inner* `U.Suite`, runs it through
`Test.Runner.UnitNode.Testing.runSuite`/`runSuiteFiltered`, and asserts on the
returned `SuiteResult` — that is the only reason `Testing` is exposed. When
adding a lifecycle behavior, add its case there as an inner suite rather than
by observing console output.

`tests/gren.json` and `examples/cli-tests/gren.json` depend on the package via
`"local:.."` / `"local:../.."`, so editing `src/` and re-running the tests is
enough — there is no separate package-install step.

## Architecture

Pipeline for one run: CLI flags → `Selection` + `Reporter` → each `Suite.run` →
`SuiteResult`s → console tail + optional JUnit XML → exit code.

Console output is **live**: body lines are printed as results land (per test
under `-v`, per suite otherwise), and only the failure details and summary wait
for the run to finish. See "Live output" below.

- **`UnitNode.gren`** — the public facade. `suite`/`test` build values; `run`
  and `runWith` just forward to `Program`. `suite` is where **fixture-type
  erasure** happens: a `suite { … }` call is generic over a suite-fixture and a
  test-fixture type, and both variables are closed over inside the produced
  `run : Selection -> Task Never SuiteResult`. Gren has no existentials or type
  classes, so this closure is the only way an `Array Suite` can hold groups
  whose fixtures have unrelated types. Anything that would need those types back
  out cannot be added to the public API without breaking this.
- **`Internal.gren`** — the erased `Suite(..)`, `Selection`, `Reporter`,
  `Outcome`, `TestResult`, `SuiteResult`. Exists to break the `UnitNode` ↔
  `Runner` import cycle (both need the erased representation).
  `Suite.testNames` is precomputed so `--list` and selection work *without*
  running any `setUpSuite`.
- **`Runner.gren`** — the execution loop, still generic (it is called from
  inside `suite`, before erasure). Tests run strictly serially, and each
  finished test is handed to the `Reporter` before the next one starts.
- **`Program.gren`** — CLI on top of `gilramir/gren-argparse`
  (`Program.runRootWithContext`): flag parsing, `--list`, glob selection,
  `globalSetUp`/`globalTearDown`, live printing, JUnit write, exit code.
  Color is on only with a TTY and no `NO_COLOR`.
- **`Report.gren`** — pure rendering, split by *when* it can be produced:
  `suiteLine`/`verboseLine` render one result each and are printed live;
  `tail` renders the failure details and summary and needs every result.
- **`JUnit.gren`** — pure string building to JUnit XML; the caller does the write.
- **`Reason.gren`** — renders a failed `Expectation` by pulling the structured
  `Test.Runner.Failure.Reason` out via `Test.Runner.getFailureReason`. Only the
  common constructors are spelled out; the rest fall through to a generic
  `Debug.toString` line — filling more in is cosmetic and incremental.
- **`Glob.gren`** — two-cursor backtracking `*`/`?` matcher over `Array Char`,
  anchored at both ends. No regex dependency.
- **`Testing.gren`** — white-box access for the self-tests only; it re-declares
  `Outcome`/`TestResult`/`SuiteResult` with constructors exposed and converts
  from the `Internal` types. Keep it in sync with `Internal` when those change.

### Live output

The body is streamed, the tail is not. Three things keep that honest, and all
three are easy to break:

- **The line renderers must stay pure functions of a single result.** Column
  widths are constants (`String.padRight 52`), not measured across the run. Any
  line that depended on another suite's result would make streamed output
  diverge from buffered output.
- **Every `TestResult` must reach the reporter exactly once.** `runTest` reports
  the normal path; `runSuite`'s `setUpSuite`-failed branch builds its `Errored`
  results directly and reports them via `reportEach`. Adding a third way to
  produce a `TestResult` means adding a third report call — miss it and verbose
  mode silently drops those lines (this happened during implementation, on the
  branch you least want to lose output from).
- **`Reporter` is `Task Never {}` on purpose.** It cannot fail, so it can never
  short-circuit the lifecycle chain in `Runner` — reporting can't change what
  runs. Keep it that way.

`Program.tailSeparator` reproduces the blank lines `Report.render` used to get
from its `"\n\n\n"` section join; `Verbose` needs one fewer because each suite
block already ends with a blank. If you change the body's spacing, change it
there too. The self-tests don't cover console formatting — verify by diffing
real output (build a throwaway app with a failing test, an erroring test, and a
failing `setUpSuite`, and compare against the previous build).

### Invariants that the design rests on

- **Every lifecycle step and every test body works in `Task String _`.** The
  error channel is deliberately `String` so the runner prints the user's own
  message verbatim instead of mangling effect errors through `Debug.toString`.
  Do not widen it to a type variable.
- **`tearDown` running is structural, not hopeful.** In `Runner.lifecycle` the
  body is reduced to `Task Never Outcome` (via `attempt`, which moves the error
  into the success channel as a `Result`) *before* `tearDown` is chained, so a
  failing body cannot short-circuit cleanup. Keep new lifecycle work inside that
  `Task Never` discipline — `Never` and `String` error channels will not unify
  across `andThen`.
- **Lifecycle-failure semantics mirror Python's unittest** and are tabulated in
  README.md ("What happens when things go wrong"). The sharp edges: a failed
  `setUp` means *no body and no `tearDown`* (tearDown only pairs with a
  completed setUp); a failed `setUpSuite` errors every test and skips
  `tearDownSuite`; a `tearDown` failure only converts an otherwise-`Passed` test
  to `Errored` and never buries an existing failure. Changing any row means
  updating the README table and the `RunnerTests` case together.
- **All tests in a suite always run** — there is no bail-out on first failure,
  and there is no `Skipped` outcome (deselected tests produce no result at all).
- Exit code is 0 only when every selected test passed; a bad flag or a failed
  `--junit-xml` write is a *task* failure (stderr + exit 1), which is the split
  `Argparse.Program` already implements.

## Releasing

Version lives in `gren.json`; note the change in `CHANGELOG.md` in the same
commit (see `f8f0fa1`). The public API surface is the `exposed-modules` list —
adding to it is a minor bump, changing existing signatures is a major one.
