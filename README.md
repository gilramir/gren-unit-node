# gren-unit-node

A test framework for Gren Node applications. Write your tests in Gren, run them from the command line, and get a clear pass/fail report with per-test timing.

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
import Test.Unit as U


main : Node.SimpleProgram a
main =
    U.run
        { name = "my-tests"
        , version = "1.0.0"
        , suites = \_ -> [ mathSuite ]
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

Each test body is a function from the test fixture (ignored here with `_`) to a `Task String Expectation`. Use the standard `Expect.*` functions from `gren-lang/test` for assertions — the same ones you already know.

## Build and run

```bash
gren make src/Main.gren --output=app
node app
```

```
Starting tests


Ran 2 tests in 12 ms

OK — 2 passed
```

Add `-v` for a line per test:

```bash
node app -v
```

```
Math.addition                    ok    (6 ms)
Math.subtraction                 ok    (6 ms)


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

When tests need files, processes, or other external state, use `setUp` and `tearDown`. They run before and after each test, and **`tearDown` is guaranteed to run even if the test fails**, so resources are always cleaned up.

`U.Permissions` gives you `fs` (filesystem) and `childProcess` permissions. The framework acquires these for you and passes them to your suite builder — no `Init` wiring needed.

Here is a suite where each test gets a fresh temporary directory:

```gren
import FileSystem
import FileSystem.Path as Path exposing (Path)
import Task


tempDirSuite : U.Permissions -> U.Suite
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

Pass `tempDirSuite perms` alongside your other suites in `main`:

```gren
suites = \perms -> [ mathSuite, tempDirSuite perms ]
```

### Suite-level setup

`setUpSuite` and `tearDownSuite` run once for the whole suite — useful for expensive one-time work like locating a binary or opening a connection. The value `setUpSuite` produces is passed to every `setUp` call as its first argument.

```gren
U.suite
    { name = "CLI"
    , setUpSuite =
        -- locate the binary once; each test receives its path
        FileSystem.realPath perms.fs (Path.fromPosixString "../app")
            |> Task.map Path.toPosixString
            |> Task.mapError (\e -> "could not find app: " ++ Debug.toString e)
    , tearDownSuite = U.noTearDown
    , setUp = \appPath -> Task.succeed appPath
    , tearDown = U.noTearDown
    , tests = [ ... ]
    }
```

### Error channel

Every lifecycle function and every test body works in `Task String _`. When a task fails, put a human-readable message in the error channel — the framework prints it verbatim in the failure report.

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
