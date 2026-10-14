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
| Tests collected | 1,248 in 34 modules, at the latest commit | `pytest --collect-only`, re-run 2026-10-13 |
| Latest full-suite run | 1,229 passed, 0 failed, 2 skipped, 209 subtests, 662 s, with the packaged build present | project progress record, Phase 10 (2026-09-20) |
| Tests added since that run | 17 (release-script tests, not marked `slow`) | repository history; 1,229 + 2 skipped + 17 = 1,248 |
| Fast suite | 1,127 passed, 0 failed, 2 skipped, 192 subtests, 150 s | re-run 2026-10-13. Excludes the 119 tests marked `slow` and includes the 17 added since the full run |
| Legacy vs desktop parity | 22 cases, all passing | part of the full-suite run (marked `slow`); scope below |
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

### Points and counts

Two different sets of points are involved, and they are never added together:

- **4 reference corners** for geometry. Step 1 takes the outer corner of each
  of the four ArUco markers. If one marker isn't detected, it infers that
  corner from the other three. These 4 points define the perspective warp.
- **13 anatomical landmarks** on the wing. Step 4's DeepLabCut model predicts
  them, each with x, y and a confidence value.

## What is verified and what is not

| Kind of evidence | What it covers | Status |
|---|---|---|
| **Automated tests** | Pipeline contracts, desktop behaviour, failure and restart handling, the packaged executables | Passed on the development machine (figures above) |
| **Parity** | One synthetic collection driven through both front ends in the same environment. Corrected, hand-corrected and standardized images are SHA-256 identical, and metadata is row-identical. The landmark CSV is identical through a **stand-in** analysis process | Passed. It shows that both front ends call the same science the same way. It does **not** compare real DeepLabCut output, and it uses no real specimen photos |
| **Packaged-build acceptance** | Steps 1–4 from the built executables, with real DeepLabCut inference on 10 synthetic images | 23 of 23 checks, on the development machine |
| **Hands-on acceptance** | A person using the app on the real Windows platform | Repeated sessions during development. Each defect found became a regression test |
| **Real specimens** | One approved LF/RF pair, run from source | Shown on [the real-example page](REAL_EXAMPLE.md). It is not a validation |

### Still open

- **Clean-machine acceptance:** a run on a second Windows account with no
  Python on `PATH` and a fresh user profile is still required before
  `2.0.0-rc1` becomes `2.0.0`. Until then, nothing here shows that the
  package works on a machine other than the one it was built on.
- **Code signing:** deliberately deferred. The first launch on a new machine
  shows a SmartScreen prompt.

### Known limitations

- **CPU only.** The bundled PyTorch is the CPU build, so Step 4 takes
  seconds per image (70 s for 10 images in the acceptance run).
- **Windows only** for the packaged build. The source also runs elsewhere.
- **About 3.2 GB**, mostly PyTorch and the model.
- **One collection at a time** per workspace.
- **No resume.** An interrupted Step 4 starts over (see
  [ENGINEERING_DECISIONS.md](ENGINEERING_DECISIONS.md#4-an-interrupted-analysis-is-re-run-never-resumed)).
