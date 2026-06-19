# gren-unit

An xUnit / Python-`unittest`-style test framework for Gren.

> **Status: working proof-of-concept.** The package and the
> `examples/cli-tests/` app both compile with `gren 0.6.5`, and the example runs
> (all features below exercised: fixtures, glob/name selection, verbose timing,
> JUnit XML, and pass/fail/error reporting with correct exit codes). It is a v1
> skeleton, not a polished release — the `Reason` renderer covers only the common
> matchers, output is built once rather than streamed live, and there are no
> tests of the framework itself yet.

## Why this, and not the effectful runner

`blaix/gren-effectful-tests` (used in `gren-format-lib/tests`) is great for pure
+ Task-backed assertions, but its execution model runs every Task first and
evaluates the pure `Expectation`s last. That makes four `unittest` staples
awkward or impossible: lifecycle hooks whose **timing** matters, *noticing*
setup/teardown failures, per-test timing, and machine-readable output.

`gren-unit` owns its own loop over `Task`, so it can offer all of them. It still
reuses `gren-lang/test`'s `Expect` matchers — it pulls the structured failure
out of an `Expectation` with `Test.Runner.getFailureReason` — so you don't learn
a new assertion library.

## Features

| Feature | Where |
|---|---|
| Per-**suite** `setUpSuite` / `tearDownSuite` (run once) | `Test.Unit.suite` |
| Per-**test** `setUp` / `tearDown` (run each test) | `Test.Unit.suite` |
| Lifecycle failures recorded as real errors (not skips/crashes) | `Test.Unit.Runner` |
| Select tests by name or glob on the CLI | `Test.Unit.Glob`, `…/Program` |
| Verbose output: per-test status + execution time | `Test.Unit.Report` |
| JUnit XML output (`--junit-xml`) | `Test.Unit.JUnit` |
| Proper exit codes (0 all-pass, 1 otherwise) | via `Argparse.Program` |

## The lifecycle (matches unittest exactly)

```
setUpSuite                       once; throw ⇒ every test Errored, no tearDownSuite
  ├ for each selected test:
  │   setUp                      throw ⇒ test Errored, no body, no tearDown
  │     ├ body → Expectation     assertion ⇒ Passed/Failed; task error ⇒ Errored
  │     └ tearDown               ALWAYS once setUp succeeded; throw on a passing
  │                              test ⇒ that test becomes Errored
  └ tearDownSuite                throw ⇒ recorded as a suite-level error
```

"Always runs `tearDown`" is guaranteed *structurally*: each body is reduced to a
`Task Never Outcome` before `tearDown` is chained, so it can't short-circuit.

## CLI

```bash
gren make src/Main.gren --output=app

node app                       # run everything
node app 'Format.*'            # glob by qualified name "Suite.test"
node app '*in place*' 'Glob.*' # multiple patterns = union
node app -v                    # verbose: status + ms per test
node app --list                # print qualified names, don't run
node app --junit-xml out.xml   # also write JUnit XML
```

## Writing tests

```gren
import Test.Unit as U
import Expect

arithmetic : U.Suite
arithmetic =
    U.suite
        { name = "Arithmetic"
        , setUpSuite = U.noSuiteFixture
        , tearDownSuite = U.noTearDown
        , setUp = U.noFixture
        , tearDown = U.noTearDown
        , tests =
            [ U.test "adds" <| \_ ->
                Task.succeed (Expect.equal 4 (2 + 2))
            ]
        }

main : Node.SimpleProgram a
main =
    U.run
        { name = "my-tests"
        , version = "1.0.0"
        , suites = \perms -> [ arithmetic ]
        }
```

`perms : U.Permissions` (`{ fs, childProcess }`) is acquired by the framework
and handed to your suite builder, so fixtures can touch the filesystem and spawn
processes without any `Init` boilerplate. See `examples/cli-tests/` for a port
of two real `gren-format` CLI tests, including a temp-dir-per-test fixture.

## Module map

| Module | Role |
|---|---|
| `Test.Unit` | public facade: `suite`, `test`, `run`, fixture defaults |
| `Test.Unit.Program` | CLI wiring (argparse), report + XML orchestration, exit code |
| `Test.Unit.Runner` | the execution loop + lifecycle semantics |
| `Test.Unit.Internal` | erased `Suite` type + result types (cycle-breaker) |
| `Test.Unit.Report` | console reporter (plain + verbose, ANSI) |
| `Test.Unit.JUnit` | JUnit XML serialization |
| `Test.Unit.Reason` | `Expectation` failure → message |
| `Test.Unit.Glob` | shell-style `*`/`?` matcher (exposed, unit-testable) |

## Known limitations (inherent to Gren)

* **No auto-discovery.** No reflection ⇒ you list suites/tests explicitly (same
  as the effectful runner). Names are explicit strings.
* **No per-test crash isolation.** Catchable failures are *Task* failures. A
  pure runtime crash (`Debug.todo`, kernel abort) takes the process down — only
  true subprocess isolation would prevent that, which is heavier than v1 wants.
* **Serial only.** Tests run in order; no parallelism (keeps fixtures simple).
