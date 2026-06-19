# gren-unit-node

A test framework for Gren Node applications. Write your tests in Gren, run them from the command line, and get a clear pass/fail report with per-suite timing (or per-test timing with `-v`).

## Setup

Create a `tests/` application directory alongside your package or application. Add `gilramir/gren-unit-node` as a dependency in `tests/gren.json`:

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

Create `tests/src/Main.gren`. The only required pieces are a `U.run` call in `main` and at least one suite containing named tests:

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

Each test body must return `Task String Expectation`. `Expect.equal 4 (2 + 2)` is a pure expression — no effects — so `Task.succeed` lifts it into the `Task` type the framework expects. Tests that do file I/O or spawn processes already return a `Task`, so they use `Task.map` to produce the `Expectation` at the end instead.

Use the standard `Expect.*` functions from `gren-lang/test` for assertions — the same ones you already know.

`U.run` requires no permissions. For tests that read files, spawn processes, or touch other external resources, use `U.runWith` — see below.

## Build and run

```bash
gren make src/Main.gren --output=app
node app
```

```
Math                                     ok    2/2    (12 ms)


Ran 2 tests in 12 ms

OK — 2 passed
```

Add `-v` for a line per test:

```bash
node app -v
```

```
Math.addition                            ok    (6 ms)
Math.subtraction                         ok    (6 ms)


Ran 2 tests in 12 ms

OK — 2 passed
```

## Selecting tests

Pass one or more glob patterns to run only matching tests. Test names are fully-qualified as `Suite.test`:

```bash
node app 'Math.*'        # every test in the Math suite
node app '*addition*'    # any test whose name contains "addition"
node app 'A.*' 'B.*'    # multiple patterns are a union
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
        , suites = \perms -> [ tempDirSuite perms ]
        }
```

Your `suites` function receives whatever record you passed to `done`.

### Per-test setup and teardown

Here each test gets a fresh temporary directory, created in `setUp` and removed in `tearDown`:

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

`setUpSuite` and `tearDownSuite` run once for the whole suite — useful for expensive one-time work like locating a binary. The value `setUpSuite` produces is passed to every `setUp` call as its first argument:

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

Every lifecycle function and every test body works in `Task String _`. When a task fails, put a human-readable message in the error channel — the framework prints it verbatim in the failure report.

Note: `U.runWith` always acquires its own `FileSystem.Permission` internally for `--junit-xml` support. If your tests also need filesystem access, acquire it separately in your `init`.

## Comparing output against a file

A common pattern is to keep expected output in a file under `testfiles/` and compare it against what your code produces. `FileSystem.readFile` returns a `Task`, so the comparison lives inside `Task.map`:

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

`Bytes.toString` returns a `Maybe String` because not all byte sequences are valid UTF-8; `Maybe.withDefault ""` is fine for text fixture files. `myRenderFunction` is whatever pure function you are testing — no effects needed on that side, so it just goes in the `let`.

## What happens when things go wrong

| Situation | Result |
|---|---|
| `setUpSuite` fails | Every test in the suite is marked **Errored**; `tearDownSuite` does not run |
| `setUp` fails | That test is marked **Errored**; its `tearDown` does not run |
| Test body fails an assertion | Test is marked **Failed** |
| Test body task errors | Test is marked **Errored** |
| `tearDown` fails on a passing test | Test becomes **Errored** |
| `tearDown` fails on an already-failed test | Original failure is kept |
| `tearDownSuite` fails | Recorded as a suite-level error alongside the test results |

All tests in a suite always run — there is no bail-out on first failure.

## Other CLI flags

```bash
node app --junit-xml results.xml   # also write JUnit XML (useful in CI)
```
