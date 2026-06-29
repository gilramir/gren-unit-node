# gren-unit-node

A test framework for Gren Node applications. Write your tests in Gren, orgranizing them into
test suites. Run them from the command line and get a pass/fail report with per-suite timing,
or per-test timing with `-v`.

**Example output: see a per-suite summary***
```
% node app

Import Statements                                    ok    8/8    (70 ms)
Expressions                                          ok    54/54  (471 ms)
Declarations                                         ok    5/5    (23 ms)
Comments                                             ok    39/39  (260 ms)
Kitchen sink                                         ok    11/11  (649 ms)


Ran 117 tests in 1473 ms

OK — 117 passed
```

**Example output: see a per-test details for just one suite***
```
$ node app -v 'Declarations.*'

Declarations.type aliases                            ok    (15 ms)
Declarations.union types                             ok    (6 ms)
Declarations.type signatures                         ok    (13 ms)
Declarations.ports                                   ok    (4 ms)
Declarations.effect module where { command = ... } renders each binding as one field ok    (7 ms)


Ran 5 tests in 45 ms

OK — 5 passed
```

## Setup

Create a `tests/` application directory alongside your package or application. Add `gilramir/gren-unit-node` as a dependency in `tests/gren.json`. For example:

```json
{
    "type": "application",
    "platform": "node",
    "source-directories": [ "src" ],
    "gren-version": "0.6.5",
    "dependencies": {
        "direct": {
            "gilramir/gren-unit-node": "1.0.0 <= v < 2.0.0",
            "gren-lang/core": "7.4.2",
            "gren-lang/node": "6.1.0",
            "gren-lang/test": "5.0.0"
        },
        "indirect": {
            "gren-lang/url": "6.0.0"
        }
    }
}
```

## Your first test

Create `tests/src/Main.gren`. The only required pieces are a `U.run` call in `main` and at least one suite containing the tests:

```gren
module Main exposing (main)

import Expect
import Node
import Task
import Test.Runner.UnitNode as U


main : Node.SimpleProgram a
main =
    U.run
        { name = "my-tests"
        , version = "1.0.0"
        , suites = [ mathSuite ]
        }


mathSuite : U.Suite
mathSuite =
    U.suite
        { name = "Math"
        , setUpSuite = U.noSuiteFixture
        , tearDownSuite = U.noTearDown
        , setUp = U.noFixture
        , tearDown = U.noTearDown
        , tests =
            [ U.test "addition" <| \_ ->
                Task.succeed (Expect.equal 4 (2 + 2))
            , U.test "subtraction" <| \_ ->
                Task.succeed (Expect.equal 1 (3 - 2))
            ]
        }
```

Each test body must return `Task String Expectation`.
`Expect.equal 4 (2 + 2)` is a pure expression — no effects — so `Task.succeed`
fits it into the `Task` type the framework expects.
Tests that do file I/O or spawn processes already return a `Task`,
so they use `Task.map` to produce the `Expectation` at the end instead.

For example, a test that reads a file uses `Task.map` to turn the bytes into an `Expectation`:

```gren
U.test "config file has expected content" <| \perms ->
    FileSystem.readFile perms.fs (Path.fromPosixString "config.txt")
        |> Task.mapError Debug.toString
        |> Task.map (\bytes ->
                Expect.equal "enabled=true\n" (Bytes.toString bytes |> Maybe.withDefault ""))
```

`Task.mapError` converts any filesystem error into the `String` the framework uses as a failure message. `perms` is a permissions record provided by `U.runWith` — see "Tests that need resources" below.

Use the standard `Expect.*` functions from `gren-lang/test` for assertions.

`U.run` requires no permissions. For tests that read files, spawn processes, or touch other external resources, use `U.runWith` — see below.

## Build and run

To run the tests, you can compile your test application and run it.
```bash
gren make Main --output=app
node app
```

or just have gren run it:
```
gren run Main
```

The output shows the test suites that were run, the result, the number of
tests passed out of the total number of tests, and the execution duration.
```
Math                                     ok    2/2    (12 ms)


Ran 2 tests in 12 ms

OK — 2 passed
```

Add `-v` for a line per test. Here, "gren run" cannot be used as it cannot
handle the additional options to the node application.

```bash
node app -v
```

Here you see the status and execution duration of each individual test.
```
Math.addition                            ok    (6 ms)
Math.subtraction                         ok    (6 ms)


Ran 2 tests in 12 ms

OK — 2 passed
```

## Selecting tests

If you want to run one or more tests, but not **all** the tests,
pass one or more glob patterns on the command-line.
Test names are fully-qualified as `Suite.test`. Most likely you will
want to esacpe the glob patterns with single-quotes to avoid your shell from
expanding them, looking for files.

```bash
node app 'Math.*'        # every test in the Math suite
node app '*addition*'    # any test whose name contains "addition"
node app 'A.*' 'B.*'     # multiple patterns are a union
node app --list          # print all qualified names, don't run
```

## Tests that need resources

When tests need files, processes, or other external state, use `U.runWith` to acquire permissions, then use `setUp` and `tearDown` to manage per-test resources. **`tearDown` is guaranteed to run even if the test fails**, so resources are always cleaned up.

### Acquiring permissions

`U.runWith` takes an `init` field that follows Gren's `Init.await` continuation style: call `done` with whatever permissions your tests need. The most common case is filesystem access:

```gren
import FileSystem
import FileSystem.Path as Path exposing (Path)
import Init
import Node
import Task
import Test.Runner.UnitNode as U


type alias Perms =
    { fs : FileSystem.Permission }


main : Node.SimpleProgram a
main =
    U.runWith
        { name = "my-tests"
        , version = "1.0.0"
        , init =
            \done ->
                Init.await FileSystem.initialize <| \fs ->
                    done { fs = fs }
        , globalSetUp = U.noFixture
        , globalTearDown = U.noTearDown
        , suites = \perms -> [ tempDirSuite perms ]
        }
```

Your `suites` function receives whatever record you passed to `done`. Use
`U.noFixture` and `U.noTearDown` as no-ops when you don't need a global
lifecycle.

### Per-test setup and teardown

Here each test gets a fresh temporary directory, created in `setUp` and
removed in `tearDown`. The framework runs `setUp` before each test and
passes whatever its `Task` resolves to as the argument to that test's
body — so `\dir ->` receives the `Path` that `setUp` produced.

```gren
tempDirSuite : Perms -> U.Suite
tempDirSuite perms =
    U.suite
        { name = "TempDir"
        , setUpSuite = U.noSuiteFixture
        , tearDownSuite = U.noTearDown
        , setUp =
            \_ ->
                FileSystem.makeTempDirectory perms.fs "my-tests"
                    |> Task.mapError (\e -> "setUp: " ++ Debug.toString e)
        , tearDown =
            \dir ->
                FileSystem.remove perms.fs { recursive = True } dir
                    |> Task.map (\_ -> {})
                    |> Task.mapError (\e -> "tearDown: " ++ Debug.toString e)
        , tests =
            [ U.test "creates a file" <| \dir ->
                let
                    file =
                        Path.append (Path.fromPosixString "hello.txt") dir
                in
                FileSystem.writeFile perms.fs (Bytes.fromString "hello") file
                    |> Task.map (\_ -> Expect.pass)
                    |> Task.mapError (\e -> "write failed: " ++ Debug.toString e)
            ]
        }
```

### Suite-level setup

`setUpSuite` and `tearDownSuite` run once for the whole suite — useful
for expensive one-time work like locating a binary. The value `setUpSuite`
produces is passed to every `setUp` call as its first argument:

```gren
U.suite
    { name = "CLI"
    , setUpSuite =
        FileSystem.realPath perms.fs (Path.fromPosixString "../app")
            |> Task.map Path.toPosixString
            |> Task.mapError (\e -> "could not find app: " ++ Debug.toString e)
    , tearDownSuite = U.noTearDown
    , setUp = \appPath -> Task.succeed appPath
    , tearDown = U.noTearDown
    , tests = [ ... ]
    }
```

### Global setup and teardown

`globalSetUp` and `globalTearDown` run once around the entire test run
— before any suite starts and after all suites finish. Both receive
the same `perms` record as `suites`. Use them for expensive one-time work
that spans all suites, such as starting a server or seeding a database:

```gren
main : Node.SimpleProgram a
main =
    U.runWith
        { name = "my-tests"
        , version = "1.0.0"
        , init =
            \done ->
                Init.await FileSystem.initialize <| \fs ->
                Init.await ChildProcess.initialize <| \cp ->
                    done { fs = fs, childProcess = cp }
        , globalSetUp = \perms -> startTestServer perms.childProcess
        , globalTearDown = \perms -> stopTestServer perms.childProcess
        , suites = \perms -> [ apiSuite perms ]
        }
```

If `globalSetUp` fails, no suites run and `globalTearDown` is skipped. If
`globalTearDown` fails, the program exits with an error after the report
has been printed.

### Multiple permissions

Chain `Init.await` calls to acquire more than one permission:

```gren
init =
    \done ->
        Init.await FileSystem.initialize <| \fs ->
        Init.await ChildProcess.initialize <| \cp ->
            done { fs = fs, childProcess = cp }
```

### Error channel

Every lifecycle function and every test body works in `Task String _`.
When a task fails, put a human-readable message in the error channel;
the framework prints it verbatim in the failure report.

Note: `U.runWith` always acquires its own `FileSystem.Permission` internally for
`--junit-xml` support. If your tests also need filesystem access,
acquire it separately in your `init`.

## Comparing output against a file

A common pattern is to keep expected output in a file in a directory
(in this example, `testfiles/`)
and compare it against what your code produces. `FileSystem.readFile`
returns a `Task`, so the comparison lives inside `Task.map`:

```gren
import Bytes
import FileSystem
import FileSystem.Path as Path
import Task
import Expect
import Test.Runner.UnitNode as U


goldenSuite : Perms -> U.Suite
goldenSuite perms =
    U.suite
        { name = "Golden"
        , setUpSuite = U.noSuiteFixture
        , tearDownSuite = U.noTearDown
        , setUp = U.noFixture
        , tearDown = U.noTearDown
        , tests =
            [ U.test "render matches expected output" <| \_ ->
                let
                    expectedFile =
                        Path.fromPosixString "testfiles/expected-output.txt"
                in
                FileSystem.readFile perms.fs expectedFile
                    |> Task.mapError (\e -> "could not read expected file: " ++ Debug.toString e)
                    |> Task.map
                        (\bytes ->
                            let
                                expected =
                                    Bytes.toString bytes |> Maybe.withDefault ""

                                actual =
                                    myRenderFunction someInput
                            in
                            Expect.equal expected actual
                        )
            ]
        }
```

`Bytes.toString` returns a `Maybe String` because not all byte sequences
are valid UTF-8; `Maybe.withDefault ""` is fine for text fixture
files. `myRenderFunction` is whatever pure function you are testing —
no effects needed on that side, so it just goes in the `let`.

## What happens when things go wrong

| Situation | Result |
|---|---|
| `globalSetUp` fails | No suites run; program exits with error; `globalTearDown` does not run |
| `setUpSuite` fails | Every test in the suite is marked **Errored**; `tearDownSuite` does not run |
| `setUp` fails | That test is marked **Errored**; its `tearDown` does not run |
| Test body fails an assertion | Test is marked **Failed** |
| Test body task errors | Test is marked **Errored** |
| `tearDown` fails on a passing test | Test becomes **Errored** |
| `tearDown` fails on an already-failed test | Original failure is kept |
| `tearDownSuite` fails | Recorded as a suite-level error alongside the test results |
| `globalTearDown` fails | Program exits with error after the report is printed |

All tests in a suite always run — there is no bail-out on first failure.

## Other CLI flags

```bash
node app --junit-xml results.xml   # also write JUnit XML (useful in CI)
```

## Design notes

Why another Gren test runner?

This gren-unit-node package exists because I wanted to include timing
information and setup and teardown functions.  So we can't run all the
test tasks first and the evaulate the resulting `Expectation` values
together. In the gren-unit-node model, the **ordering and timing**
of individual steps matters:

- **Per-test setUp and tearDown.** A tearDown that must release the
  resource its test acquired and do so before the next setUp runs.

- **Setup failures as first-class outcomes.** When setUp fails, the right
  behavior is to mark that test as Errored, skip its body entirely, and still
  run tearDown for any fixtures that did get created.

- **Per-test timing.** Knowing how long each individual test took requires
  observing when each one starts and finishes, which is difficult to recover
  after a batch run.

- **Machine-readable output.** JUnit XML and similar formats need per-test
  outcomes with lifecycle data attached — the same information that per-test
  timing requires.

gren-unit-node addresses these by owning a sequential loop over `Task`. Each test
runs completely — setUp → body → tearDown — as a single composed `Task` before
the next test begins. A failing setUp becomes a `Task` error the runner catches
and records as an Errored outcome; tearDown is chained in a way that is
structurally guaranteed to run even when the body fails.

The assertion API is the same: gren-unit-node reuses `gren-lang/test`'s
`Expect.*` matchers throughout, pulling structured failure information out of an
`Expectation` via `Test.Runner.getFailureReason`. You do not need to learn a new
assertion library to use it.
