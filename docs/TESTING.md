# Testing and verification

← [Back to the overview](../README.md)

## Strategy

The test suite answers four different questions, each at a different distance
from the code.

| Layer | Question | Examples |
|---|---|---|
| **Pipeline and contracts** | Is the science unchanged, and are the data rules kept? | metadata idempotence, folder-overlap rules, filename-based staleness, Step 1 accounting, the landmark CSV schema with its four header levels |
| **Desktop behaviour** | Does the app behave correctly under navigation, failure and restart? | state survives navigation, duplicate runs refused, late results dropped, every step fails the same way (terminal state, reason, log kept, retry possible) |
| **Parity** | Does the new front end produce exactly what the old one did? | two full workflows over the same synthetic collection, compared by SHA-256 and row by row |
| **Artefact** | Does the packaged build itself work on a machine with no Python? | tests that start the real executables; self-check; end-to-end acceptance with real inference |

### Synthetic data only

The suite builds its own fixtures at runtime, including images carrying real
ArUco markers, so correction really runs end to end without any specimen
photograph. Destructive tests run in temporary workspaces. Step 4 is tested
with a **stand-in child process** that emits the same events as the real one.
That lets the process boundary, event parsing, output checks and failure
handling be tested in about two minutes without loading a model. One real
inference test is opt-in.

### Testing what Qt doesn't report

Several failures are silent in Qt, so they have dedicated tests:

- a widget `objectName` that no stylesheet rule matches;
- a dynamic property changed without re-polishing, so the change never shows;
- fractional lengths, or CSS properties Qt ignores;
- hard-coded sizes that clip under display scaling;
- a `clicked` signal passing `checked` into a slot that takes no arguments.
  That bug made a real button do nothing, with no error and no log line.

### Hands-on acceptance testing

The suite was green each time a person sat down and used the app, and each
session still found real defects:

- precision touchpads couldn't zoom;
- a focus change scrolled the page under the pointer;
- arrow keys stopped working when focus was in the gallery;
- the *Add model* button silently did nothing;
- a guide's text was clipped at narrow column widths.

Each is now a regression test. Visual acceptance runs on the real Windows
platform, because the headless platform reports no font families and fake
font metrics, so text never overflows there.

## Verified metrics and where each comes from

| Metric | Value | Source |
|---|---|---|
| Tests collected | 1,248 in 34 modules | `pytest --collect-only`, re-run 2026-10-13 |
| Last full gate | 1,229 passed, 0 failed, 2 skipped, 209 subtests | project progress record, Phase 10 |
| Fast suite | 1,127 passed, 0 failed, 2 skipped, 192 subtests, 150 s | re-run 2026-10-13 (excludes 119 tests marked `slow`) |
| Tests against the packaged artefact | 29 passed | release-candidate record |
| Self-check of the release candidate | 35 checks, 0 failed | release-candidate record |
| End-to-end acceptance from the artefact | 23 of 23; real inference 70 s for 10 synthetic images (CPU) | Phase 10 record |
| Window open from the packaged build | 3.1 s | acceptance record |
| Page switch, idle | 1.03 ms mean over 140 switches | performance table |
| Page switch during Step 1 | mean 25.2 ms, median 23.1 ms, worst 98.7 ms over 235 switches | performance table |
| Page switch during Step 3 | worst 37.4 ms | performance table |
| Step 1 `start()` returns | 8.8 ms for a 120-image collection | performance table |
| Workspace scan, 2,100 images | ~209 ms (now on a worker thread) | decision record ADR-011 |
| Suite runtime fix | 1,334.9 s → 299 s, 776 tests | decision record ADR-020 |
| Layout audit | 56 combinations, 0 problems | decision record ADR-036 |
| Packaged distribution | about 3.2 GB, most of it PyTorch and the model | release-candidate record |

All timings were measured on the development laptop. They describe how
responsive the software is, not how accurate the model is. No model-accuracy,
validation or scientific results are reported in this portfolio.

### Still open

- **Clean-machine acceptance:** a run on a second Windows account with no
  Python on `PATH` and a fresh user profile is still required before
  `2.0.0-rc1` becomes `2.0.0`.
- **Code signing:** deliberately deferred. The first launch on a new machine
  shows a SmartScreen prompt.
