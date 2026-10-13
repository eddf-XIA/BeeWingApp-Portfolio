# Architecture

← [Back to the overview](../README.md)

This page describes how the application is put together and why. It covers
responsibilities and contracts, not code. The production source is private.

<img src="../assets/diagrams/architecture.svg" alt="Layered architecture diagram">

## 1. Layers and the one-way rule

| Layer | Responsibility | Never does |
|---|---|---|
| **Presentation** | Seven pages in a `QStackedWidget` behind a sidebar. Reusable widgets (cards, status badges, stat tiles, path fields), a token-based QSS design system, and five explanatory animations. | Touch the filesystem when shown, or run pipeline code |
| **Controllers** | Start and stop work, stage and validate folder edits, install results into state, and rebuild the manual queue on launch. | Hold widgets or pixels |
| **State** | Workspace paths, an immutable Qt-free snapshot of the workflow, per-step task state (progress, counts, log), and the manual queue. | Decide scientific questions |
| **Workers** | Run jobs on a `QThreadPool`. Jobs are plain Python functions given a `report()` callable. Exceptions become failure signals. | Import Qt inside a job |
| **Pipeline** | Discovery, folder-safety rules, correction, standardization, analysis, progress model. | Import Qt or know a GUI exists |
| **Analysis child** | Load DeepLabCut and PyTorch, run inference, emit events. | Share memory or an event loop with the window |

Tests enforce several of these rules directly. Pages may only reach the
pipeline through controllers. No desktop module may reimplement path
comparisons that belong to the folder-safety module. Worker job bodies may not
call OpenCV.

## 2. Why the science is kept apart

The four scientific steps were already validated in an earlier front end built
with Streamlit. The desktop migration had a hard rule: **don't change the
science unless a test proves a defect.** So the pipeline was frozen as a
shared package that both front ends call. That gave the strongest test
available: run both front ends over the same synthetic collection in separate
workspaces, then compare every output.

| Compared | Result |
|---|---|
| Folder detection, Step 1 accounting, manual queue identities | identical |
| Corrected and hand-corrected images | SHA-256 identical, file by file |
| Metadata rows and specimen-ID map | identical, row by row |
| Standardized images | SHA-256 identical |
| Landmark CSV (through a stand-in analysis child) | SHA-256 identical |
| Raw photographs | untouched in both runs |

## 3. Long-running work that survives navigation

- **Pages are built once, at startup.** Switching pages only changes an index,
  and a test fails if the window code ever removes or deletes a page. Typed
  text, half-placed corners and progress bars survive any amount of switching.
- **Each background step has one controller, owned by the window.** Two
  controllers would mean two ways to start the same work. The single
  controller refuses a second run while one is active.
- **Late results are dropped.** Each controller numbers its requests. A slow
  scan that finishes after a newer one is discarded, so old data can't
  overwrite fresh data.
- **Results can't be garbage-collected.** A task the caller didn't keep a
  reference to used to lose its result. Tasks are now held in a registry until
  their result has been delivered.
- **Status depends on the whole pipeline.** While Step 1 runs, later steps
  show "waiting" rather than claiming to be ready.

## 4. Step 4: inference in a separate process

DeepLabCut and PyTorch take over ten seconds just to import, use gigabytes of
memory, and can crash in native code. All of that happens in a separate
process (`BeeWingAnalysis.exe` when packaged). A supervisor job on a pool
thread runs it.

<img src="../assets/diagrams/step4-process.svg" alt="Step 4 sequence diagram">

- **Mechanism:** a supervised `subprocess.Popen`, not `QProcess`. The
  supervisor already runs on a pool thread, where a blocking read of a pipe
  costs nothing, and the job stays Qt-free like the others.
- **Channel:** the child copies its real stdout to a private handle before any
  import, then points `sys.stdout` at stderr. Events are newline-delimited
  JSON, flushed after every line (`start`, `progress`, `image`, `log`,
  `result`, `done`, `error`). The parent reads stderr on its own thread and
  keeps it as log text.
- **Progress that doesn't lie:** the progress bar follows a stage-based model.
  Each stage owns a fixed share of the bar, the bar never moves backwards, and
  it reaches 100% only after the output has been checked. A separate per-image
  "x of N" count is shown alongside it.
- **Checks before launch:** unsafe folder layout, a missing model, or no input
  images means the child is never started. Images that are corrected but not
  yet standardized produce a warning, not a refusal.
- **Verified outcome:** success means a non-empty results file *written by
  this run*. Both the child and the parent check this.
- **Cancellation:** only Step 4 can be cancelled, because it is long and owns a
  process. Steps 1 and 3 write whole images quickly, and stopping them halfway
  through a file would be the riskier choice.

## 5. Data integrity

- **Raw photographs are read-only.** The app writes nothing next to or inside
  the raw folder.
- **Folder topology contract.** No output folder may be the raw folder, sit
  inside it, or contain it, and two outputs may not share a folder. All
  comparisons use resolved, case-normalised paths, so `..`, mixed slashes and
  Windows case differences can't get past the rule. The rule is checked
  independently at the UI, the controller, the state object and each worker,
  and folders are only ever created by an explicit Save.
- **Idempotent metadata.** Each row's identity is (species, specimen id, wing
  type). Re-running skips identical rows and updates changed ones (e.g. auto →
  manual) instead of appending duplicates. Rewrites go through a temporary file
  and an atomic replace.
- **Stable specimen IDs.** A persistent map assigns each specimen a numeric ID
  once, and later runs reuse it.
- **No output re-read as input.** Corrected outputs are never rediscovered as
  raw inputs, so there is no `fixed_fixed_*`.
- **"Nothing processed" is not success.** A run that attempted zero images is
  reported as a failure with a reason, never as a green banner.

## 6. Recovery after a restart

The Step 2 queue isn't saved. It is **rebuilt** at launch from three facts on
disk. A wing is pending if:

1. it is an eligible raw wing photograph,
2. it has no corrected output, and
3. its specimen already has an ID, which proves Step 1 tried it and failed.

The third condition keeps photos added since the last run out of the
hand-correction queue; those belong to Step 1. Rebuilding is read-only, runs
off the GUI thread, and gives way if Step 1 has already filled the queue.

## 7. Packaging and deployment

<img src="../assets/diagrams/release-pipeline.svg" alt="Packaging pipeline">

- **PyInstaller, one-directory mode**, chosen after a 90-line test build showed
  that NumPy, OpenCV, pandas, PyTorch, DeepLabCut and PySide6 all survive
  freezing. One-file mode was rejected: it would unpack gigabytes on every
  launch.
- **Two executables, one shared runtime.** The window has no console. The
  analysis child does, because its event channel needs a real stdout.
- **Three locations, one resolver.** Read-only resources sit inside the bundle.
  The model folder sits *beside* the executables, so it can be replaced
  without a rebuild. User configuration goes to per-user storage, or beside
  the program in portable (USB) mode. Environment overrides let tests simulate
  a frozen layout without freezing.
- **Nothing is imported from outside the bundle.** The child's import path is
  cleared, so a packaged build can't silently run untested code from the
  machine it's on.
- **Failures are written down.** A crash in a windowless app is written to a
  log and named in one dialog. During self-check, no dialog is shown at all,
  because a blocked dialog would hang an automated build.
- **Self-check, fingerprint, release gate.** See
  [ENGINEERING_DECISIONS.md](ENGINEERING_DECISIONS.md#7-a-build-that-proves-it-is-the-source).

## Technology

| Area | Choices |
|---|---|
| Computer vision | OpenCV (ArUco detection, perspective transforms), NumPy |
| Pose estimation | DeepLabCut 3 (PyTorch engine), HRNet backbone, CPU inference |
| Desktop | PySide6 / Qt 6: `QMainWindow`, `QStackedWidget`, `QGraphicsView`, `QThreadPool`, QSS |
| Data | pandas, CSV with multi-level headers |
| Testing | pytest + unittest, headless Qt, real Windows platform for visual acceptance, Playwright for the legacy web front end |
| Packaging | PyInstaller (one-dir), a self-check mode in the built app, scripted acceptance and release gates |
| Earlier front end | Streamlit (kept as a tagged, runnable release) |
