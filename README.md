<div align="center">

# Bee Wing Lab: computer vision for bumblebee wing morphometrics

**Houchunxiao (Byron) Xia** · research software for a university entomology research group

**A Windows desktop application that turns raw specimen photographs into analysis-ready wing-landmark coordinates: OpenCV marker-based perspective correction, a human-in-the-loop fallback, and DeepLabCut pose estimation in an isolated process.**

`Python` · `OpenCV` · `PySide6 / Qt 6` · `DeepLabCut 3` · `PyTorch` · `pytest` · `PyInstaller`

<img src="assets/real/real_workflow.gif" width="720" alt="Animated walkthrough on a real bumblebee forewing: original photo, three detected markers and the inferred fourth corner, corrected image, standardized orientation, 13 model-predicted landmarks">

<sub>A real bumblebee left forewing going through the real pipeline: original photo → markers found → perspective-corrected → standardized → 13 landmarks predicted by the trained model (model output, not validated ground truth).</sub>

</div>

---

## The 60-second version

| | |
|---|---|
| **Problem** | Wing-shape studies need the same anatomical points measured on hundreds of specimen photos. The photos are hand-held, so they are tilted, at different scales, and a mix of left and right wings. Marking points by hand is slow and inconsistent. |
| **What I built** | A four-step pipeline and desktop app: **(1)** detect the four ArUco markers on the specimen card and use their **4 outer corners** to warp each photo to a fixed-scale square, **(2)** send any photo the detector can't handle to a fast manual 4-click correction queue, **(3)** put left and right wings in the same orientation, **(4)** run a DeepLabCut pose model I trained, which writes **13 anatomical landmarks** per wing (x, y, confidence) to CSV. |
| **Hard parts** | Keeping the UI responsive during multi-minute PyTorch inference. Never losing or duplicating a measurement across crashes, restarts and re-runs. Shipping all of it to lab computers with no Python installed. |
| **Evidence** | **1,248** automated tests collected. Latest full-suite run: **1,229 passed, 0 failed**, 2 skipped, including 29 tests that launch the packaged executables. On a synthetic collection, the desktop app's images and landmark CSV were **SHA-256 byte-identical** to the original Streamlit app's. The packaged build checks itself and **refuses to ship** if it doesn't match its source. |
| **Status** | Release candidate `2.0.0-rc1`, built and checked on the development machine. **Not yet done:** the clean-machine acceptance run on a second Windows account, which is required before `2.0.0`. Code signing is deferred. |

> **About this repository.** This is a public portfolio. One approved pair of real forewing photos (left and right of the same specimen) is shown, with the application's genuine outputs for that pair. Specimen identifiers and image metadata have been removed. Every other image is synthetic. The production source code, all other research images, the training data and labels, the trained model weights and all scientific results are **private**.

---

## A real specimen, end to end

<img src="assets/real/real_pair_stages.jpg" alt="Real left and right forewings: original photo with three detected markers and the inferred fourth corner, perspective-corrected image, standardized orientation">

<img src="assets/real/real_pair_landmarks.jpg" alt="Close-ups of the 13 model-predicted landmarks on both standardized wings, with the low-confidence landmark 1 drawn as a grey ring">

**Full resolution:** [side-by-side LF/RF comparison of all four stages (4483 × 2470)](assets/real/real_pair_full_resolution.jpg)

The pair was run through the real application (Steps 1, 3 and 4), with the trained DeepLabCut model doing the inference. **Nothing in these images was drawn or adjusted by hand.** The marker outlines come from the pipeline's own detector. The landmark positions and confidence values are read straight from the CSV the model wrote.

What the run shows:

- **Step 1 corrected both photos automatically, even though it detected only 3 of the 4 markers on each.** The top-right marker is visible in both photos, but the detector didn't pick it up. The pipeline inferred that corner from the other three (amber ring, dashed edges), so neither photo needed manual correction.
- **Step 3 put both wings the same way round.** The left wing was mirrored and the right wing was rotated 180°. Both now have the base on the left and the tip on the right, so the 13 landmarks line up between them.
- **Step 4 predicted all 13 landmarks on both wings** (2 images in 13.9 s on a laptop CPU, including model load). These are **model predictions, not validated ground truth**. The other 12 on each wing come with likelihood ≥ 0.77. **Landmark 1, near the wing base, comes out at only 0.03 (LF) and 0.05 (RF).** It is drawn as a grey hollow ring rather than hidden. The output keeps the model's own confidence for every point, so weak predictions can be found and checked downstream.

The full coordinate table and the details of how the images were prepared are on **[the real-example page](docs/REAL_EXAMPLE.md)**.

---

## How it works

<img src="assets/diagrams/workflow.svg" alt="Workflow: raw photographs, Step 1 ArUco correction, Step 2 manual fix for failures, Step 3 orientation standardization, Step 4 landmark inference, results CSV">

| Step | Computer-vision task | Engineering idea |
|---|---|---|
| **1 · Correct** | Find the four ArUco markers on the specimen card and take the outer corner of each, giving the 4 reference corners. If one marker is missed, infer its corner from the other three. Warp it to a fixed square so every pixel has the same physical size. | Runs on a worker thread. Progress comes from the real work, image by image. Metadata writes are idempotent, so re-running never duplicates a row. |
| **2 · Fix by hand** | When markers are hidden, blurred or out of frame, a person clicks the four outer corners and the same warp is applied. | A native canvas whose coordinates are always image pixels, at any window size. After a restart, the queue is rebuilt from what is on disk. |
| **3 · Standardize** | Mirror left wings and rotate right wings 180° so every wing faces the same way before landmarking. | Up-to-date status is checked by filename, not by count. One new image makes the step "out of date" even when the totals match. |
| **4 · Analyse** | A DeepLabCut 3 model (HRNet backbone, PyTorch, CPU) predicts 13 anatomical wing-vein landmarks, each with x, y and a confidence value. | Runs in a separate process with a private JSON event channel. Success is checked on disk by the parent. |

### When detection fails (synthetic example)

The real pair didn't need Step 2, so the fallback is shown on a synthetic card.

<table>
<tr>
<td width="50%"><img src="assets/synthetic/06_detection_failed.jpg" alt="Synthetic photo: two markers hidden"><br><sub><b>Exception.</b> Glare and a fingertip hide two markers, so the photo goes to the manual queue.</sub></td>
<td width="50%"><img src="assets/synthetic/07_manual_corners.jpg" alt="Four corners clicked by hand"><br><sub><b>Step 2.</b> A person clicks four corners in a fixed order. The same warp follows.</sub></td>
</tr>
</table>

More pictures, including every application page on a synthetic collection, are in **[the visual walkthrough](docs/WALKTHROUGH.md)**.

---

## The application

<img src="assets/screenshots/step2_manual_fix.png" alt="Step 2 manual fix page: synthetic photo with four corners placed and an outline, corner-order guide on the right">

<table>
<tr>
<td width="50%"><img src="assets/screenshots/overview.png" alt="Overview page"><br><sub><b>Overview.</b> Counts come from disk, not memory.</sub></td>
<td width="50%"><img src="assets/screenshots/step1_finished.png" alt="Step 1 finished"><br><sub><b>Step 1.</b> 14 attempted, 12 automatic, 2 sent to Step 2.</sub></td>
</tr>
</table>

<sub>Screenshots of the real application, captured on the native Windows platform at 1920×1080 against a synthetic 14-wing collection. Paths point at a temporary demo folder, and the model folder is an empty placeholder. The affiliation line in the sidebar was blanked.</sub>

---

## Architecture

<img src="assets/diagrams/architecture.svg" alt="Layered architecture: presentation, controllers, state and background work, a Qt-free scientific pipeline; DeepLabCut in an isolated child process; disk as the source of truth">

Three rules shape the design. They are explained in **[ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

1. **The science never imports the GUI.** The correction, standardization and analysis code is plain Python, shared unchanged by the original Streamlit app and the new desktop app. That is what made an exact comparison between the two possible.
2. **Navigation can't interrupt work.** Every page is built once. Switching pages only changes a stack index. Long-running steps belong to controllers owned by the window, so a running job doesn't depend on which page is visible.
3. **The disk is the source of truth.** Corrected images, metadata and the specimen-ID map are the record. After a restart, the app rebuilds its state from them. No UI position is ever saved, because a saved position goes wrong the moment anything else changes.

<img src="assets/diagrams/step4-process.svg" alt="Sequence diagram: GUI thread, supervisor pool thread, child process with DLC and PyTorch; events on a private channel; verification on disk">

---

## Engineering highlights

Each item is one problem I hit, how I fixed it, and how the fix is checked. Full write-ups are in **[ENGINEERING_DECISIONS.md](docs/ENGINEERING_DECISIONS.md)**.

- **A JSON event channel that a stray `print` can't break.** DeepLabCut prints to stdout and draws progress bars on stderr. The child process copies the real stdout to a private handle before importing anything, then sends `sys.stdout` to stderr. Events and noise can't mix, and a non-JSON line becomes a log line, never progress.
- **"Success" must be proved by a file.** The parent records each result file's size and timestamp before a run. A leftover CSV from an interrupted run doesn't count as success for the next one. Found as a real defect, fixed, and covered by a regression test.
- **A Stop race, closed.** Pressing Stop in the moment before the child process existed used to stop nothing, and the analysis started anyway. The cancel flag and the process handle now share one lock.
- **No "resume" button, on purpose.** DeepLabCut gives no per-image checkpoint, so a partial CSV can't be shown to be complete. A landmark set that might be missing specimens is worse than none, so an interrupted run starts over. A test fails if anyone adds a resume path.
- **Clicked coordinates stay exact.** The photo sits in the scene at native size, unscaled, and only the view transform changes. Clicked points are image pixels at any zoom level. Round-trip tests check this at 0.4×, 1×, 2.5× and 7×.
- **A slow test suite was really a memory leak.** 776 tests took **1,335 s**. Measuring showed the cost growing with each test's position in the run, not with the science. Widgets scheduled for deletion were never destroyed, and every stylesheet change re-styled all of them. One reaping hook brought the same tests, with the same assertions, down to **299 s**.
- **The build checks itself.** A rebuild once shipped stale bytecode: the source changed but kept the same mtime and size, so every cache check passed. Builds now purge caches, fingerprint the source by content, embed the fingerprint, and fail if the built executable reports a different one.
- **Layout checks that actually find clipping.** The old size check gave wrong answers both ways. It was replaced with a geometry audit ("is any text cut off, is any child outside its parent?") that runs on 7 pages × 4 window sizes × 2 states. It found three real defects.

<img src="assets/diagrams/release-pipeline.svg" alt="Packaging pipeline: fingerprint, freeze, place model, self-check, acceptance, release gate">

---

## Verified engineering metrics

These numbers come from the project's own test records and from runs I repeated while preparing this page. They are **engineering** measurements, made on synthetic data except for the one real pair. They are not scientific or model-accuracy results: a single pair's confidence values are not a validation, and no accuracy figures are reported here. Where each one comes from is in [TESTING.md](docs/TESTING.md).

| Metric | Value |
|---|---|
| Automated tests collected | **1,248** across 34 test modules, at the latest commit |
| Latest full-suite run (2026-09-20) | **1,229 passed, 0 failed**, 2 skipped (opt-in real-model run; one focus test the headless platform can't host). It ran before 17 release-script tests were added |
| Fast suite, re-run on 2026-10-13 | 1,127 passed, 0 failed, 2 skipped, in 150 s. It includes those 17 tests and excludes the 119 marked `slow` |
| Legacy vs desktop parity | On one synthetic collection: SHA-256 **byte-identical** corrected, hand-corrected and standardized images, row-identical metadata, and a byte-identical landmark CSV. Step 4 was compared through a stand-in analysis process, not real DeepLabCut inference |
| Page switch, idle | **1.03 ms** average over 140 switches |
| Page switch while Step 1 processes a collection | mean 25.2 ms, worst 98.7 ms over 235 switches (budget 250 ms) |
| Workspace scan, 2,100 images | ~209 ms, moved off the GUI thread for that reason |
| Test-suite runtime after fixing the widget leak | 1,335 s → 299 s, same 776 tests |
| Packaged build self-check | 35 checks, 0 failed |
| End-to-end acceptance from the packaged executables | 23 of 23 checks, including real DeepLabCut inference: **70 s for 10 synthetic images** on CPU |
| Real forewing pair through the app (run from source) | 2 of 2 auto-corrected, 2 standardized, 13 landmarks each; Step 4 in **13.9 s** including model load |
| Layout audit | 56 page × size × state combinations, 0 clipping problems |
| Design record | 36 architecture decision records, 3 recorded findings |

**Automated checks are not real-machine acceptance.** Everything above ran on the development machine. The clean-account acceptance run is still open, the build is unsigned, and Step 4 runs on CPU only. The full list of limitations is in [TESTING.md](docs/TESTING.md#what-is-verified-and-what-is-not).

---

## My role and how it was built

I built this software for a university entomology research group, as its sole human developer.

**My own technical work**

- **Image processing in Python and OpenCV.** I wrote the original research code as notebooks: ArUco detection, ordering the 4 outer corners and inferring a missed one, the perspective warp to a fixed-scale square, the manual 4-click fallback, and LF/RF orientation standardization. That code became the pipeline both front ends share unchanged.
- **The landmark model.** I labelled the training images, then trained, evaluated and ran the 13-landmark DeepLabCut 3 model (HRNet backbone, PyTorch) that Step 4 uses. The training data, labels, weights and evaluation results stay private.
- **Reproducible GPU training on UW–Madison's CHTC.** I built an Apptainer container (DeepLabCut 3 with CUDA) from DeepLabCut's official image. I wrote the HTCondor submit files and a job script that sets the project up on whichever GPU node it lands on. Training, evaluation and batch inference are scripted as jobs, submitted over SSH from the cluster's Linux access point.

**What I directed, with AI-assisted implementation**

- **Desktop architecture.** I specified the move from Streamlit to PySide6 and its constraints: the validated science must not change, the old app stays runnable until parity is proven, pages persist, Steps 1 and 3 run in the background, manual correction uses a native `QGraphicsView`, and DeepLabCut runs in an isolated process. I also set the phase plan and acceptance criteria, and reviewed the decision log.
- **Data consistency and error recovery.** Raw photos are never modified, and a run that processed nothing never counts as success. A lab researcher reported a run that silently processed zero images and later re-read its own outputs as new photos. I reproduced it, and it led to the folder-safety rules and output checks described in [ARCHITECTURE.md](docs/ARCHITECTURE.md#5-data-integrity). Product calls such as "no saved queue position" and "no resume for an interrupted analysis" were mine.
- **Testing and deployment.** I reviewed every checkpoint and tested each build by hand. That hands-on testing found defects the automated suite had passed, such as touchpad zoom, focus-driven scrolling and a silently dead button, and each became a regression test. I ran the PyInstaller packaging, the self-check and the release gate for the Windows build.

**AI assistance.** An AI coding agent (Claude Code) wrote much of the application's code, tests and documentation from my specifications, and every checkpoint went through my review. 32 of the private repository's 33 commits are co-authored by it.

---

## What is public and what is private

| Public here | Private (not in this repository) |
|---|---|
| Design, architecture and decision write-ups | Production source code and its git history |
| Diagrams I drew for this page | All other specimen photographs and lab data |
| **One approved real forewing pair** and its genuine outputs (identifiers and metadata removed) | Training images, landmark labels, model weights |
| Synthetic images (a drawn wing on a card with public ArUco markers) | Specimen identifiers, collection metadata, species and dates |
| Screenshots of the app running on synthetic data | Model evaluation and any scientific results |
| Engineering and test metrics | The packaged application and its archives |

## Rights and reuse

No license is offered. The ownership and licensing of the underlying research software are still being confirmed with the research group. This repository is viewable but not reusable; contact me before reusing any part of it.

## Contact

Houchunxiao (Byron) Xia. Reach me through [my GitHub profile](https://github.com/eddf-XIA).
