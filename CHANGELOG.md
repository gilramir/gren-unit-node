# Changelog

## 1.1.0

- Print results live instead of buffering the whole report: a suite's line is
  printed when that suite finishes, and under `-v` a test's line when that test
  finishes. Only the failure/error details and the summary still wait for the
  end of the run.
- `-v` now also prints each suite's line as a rollup after that suite's tests,
  closing the block. This is the only change to what a run prints; `--list`,
  `--junit-xml`, exit codes, and non-verbose output are byte-for-byte unchanged.

## 1.0.1

- Upgrade the gilramir/gren-argparse dependency to 2.0.0
Fix the test script.

## 1.0.0

- First release
