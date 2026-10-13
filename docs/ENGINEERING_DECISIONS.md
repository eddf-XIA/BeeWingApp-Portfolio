# Engineering decisions and the problems behind them

← [Back to the overview](../README.md)

The private project keeps an architecture decision log of 36 records plus 3
recorded findings. Below are eight of the most instructive, each told as
*context → what went wrong or was at stake → decision → how it is checked*.

---

## 1. A stray `print` must not be able to break the progress channel

**Context.** DeepLabCut reports progress only through a tqdm bar on stderr,
and prints other text to stdout. A JSON event stream sharing either stream
would be mixed with half-drawn progress bars.

**Decision.** Before importing anything, the child copies the real stdout to
a private handle, writes events only there, and points `sys.stdout` at stderr.
Everything any library prints goes to stderr, which the parent keeps as log
text. Every event is flushed right away. A buffered channel would deliver a
whole run's history at the end, which looks exactly like a frozen app.

**Alternatives rejected.** Events on stderr: that is where the tqdm bar lives.
A socket or temp file: more moving parts for no gain. Running inference in a
thread: a native PyTorch crash would take the window down with it.

**Checked by.** A stand-in child that prints to stdout, writes to stderr, and
puts a deliberately broken line on the event channel. The run still succeeds,
and the broken line shows up as log text.

## 2. "Success" is a fact about the filesystem, not a message

**What went wrong.** During robustness testing, a run that was interrupted
left a CSV behind. The next run could report that file as its own success.

**Decision.** Before each run, the parent records the size and modification
time of every existing result file. Success requires a non-empty file that
this run wrote or changed. A child that names a file it never wrote has
failed.

**Checked by.** Regression tests for an earlier run's file, a file the child
names but never writes, an empty CSV, and a child that exits without a word.

## 3. Pressing Stop before the process exists

**What went wrong.** If the researcher pressed Stop within moments of
pressing Start, `cancel()` found no process handle yet, terminated nothing,
and the child launched anyway.

**Decision.** The cancel flag and the process handle are protected by one
lock. The launch path checks the flag while holding that lock.

**Checked by.** A test that stops before the child exists and asserts that no
child is ever started.

## 4. An interrupted analysis is re-run, never resumed

**At stake.** Resuming is the obvious feature. But DeepLabCut gives no
per-image checkpoint, so a partial results file can't be shown to contain each
image exactly once.

**Decision.** No resume. A landmark set that might quietly be missing
specimens is worse than no landmark set, because nothing downstream could tell.
Re-running costs minutes, while publishing an incomplete analysis could cost a
result.

**Checked by.** A test that fails if a resume path appears in the analysis
worker, so adding one means deleting a test and noticing the decision.

## 5. Manual-correction coordinates are image pixels at any zoom

**At stake.** The four clicked corners feed straight into the perspective
warp. If the coordinates depended on zoom level or window size, every
hand-fixed image would be subtly wrong.

**Decision.** The photo is added to the scene at native size, at the origin,
unscaled. Only the view transform changes. Corner handles ignore the
transform, so they keep a constant on-screen size without scaling anything
that holds a coordinate.

**Two traps found along the way.** Fitting the image while its page was still
hidden locked in a scale of about 0.03 instead of about 0.6. Fitting with the
scrollbars briefly turned off computed a scale for a viewport that wouldn't
exist once they came back. Both looked the same on screen, and neither showed
up in tests, so both are recorded.

**Checked by.** View → scene → view round trips at 0.4×, 1×, 2.5× and 7×. A
test that placed corners don't move when the zoom changes. The hand-corrected
JPEG matches a direct pipeline call byte for byte.

**Later product decision.** After using the tool, zoom was removed from manual
correction entirely. A larger, always-fitted canvas worked better. The
coordinate rule didn't change.

## 6. The slow test suite was a leak, not the cost of the science

**Context.** The suite had grown to 776 tests and about **21 minutes**. The
working assumption was that real image processing was the cost.

**Measurement.** `--durations=0` told a different story. One widget test file
took **1.4 s** on its own but **454.7 s** inside the full run, and one trivial
focus-policy test took **80 s**. Cost tracked a test's position in the run,
not what the test did.

**Cause.** Tests correctly scheduled widgets for deletion with `deleteLater`.
But deletion happens only when an event loop delivers the event, and a test
run has no event loop. So the number of live widgets grew all run long, and
every `setStyleSheet` call re-styled all of them. Runtime grew quadratically.

**Decision.** A conftest hook delivers pending deletions after every test. No
assertion was changed. Result: **1,334.9 s → 299 s** for the same 776 tests.
Fixing two real defects that surfaced in what was still slow brought the full
suite under four minutes.

## 7. A build that proves it is the source

**What went wrong.** A rebuilt package failed with a `TypeError`, a call that
no longer existed in the source. A file had been rewritten in place, ending
at the same length and without its mtime changing. Python's import system
and the freezer both judge freshness by (mtime, size), so both bundled stale
bytecode. Every other packaging check passed.

**Decision.**
1. Purge cached bytecode before freezing.
2. Fingerprint the application source **by content**.
3. Ship the fingerprint inside the bundle.
4. Have the built app's `--self-check` report it back.
5. Fail the build if the two differ.

A separate release gate refuses to stamp a folder if the source has changed
since the build, if the model folder is a development junction pointing at
the build machine, or if the self-check fails. It refuses rather than warns.

**Checked by.** A test that reproduces the exact failure conditions (same
mtime, same size, different content) and asserts that the fingerprint
changes. The release gate caught a real mismatch the first time it ran, right
after a version bump.

## 8. Ask whether anything is cut off, not whether the arithmetic fits

**What went wrong.** The "no page scrolls" guarantee was checked by comparing
a page's minimum size hint with its height. That check reported a problem on
a page that was fine, and passed a page that was visibly cutting a heading in
half.

**Decision.** A layout audit asks the question directly. Is a wrapped label
given less height than its text needs? Is any child's edge past its parent's?
Is anything below the bottom of the page? Bounded scroll regions are exempt,
and the text rule runs only where font metrics are real (the headless
platform reports none). The audit is itself tested against deliberately
broken layouts.

**Result.** It found three real defects, and every page is clean across 7
pages × 4 window sizes × 2 states (56 combinations). The same audit runs
inside the packaged build's self-check.

---

### Smaller decisions that shaped the product

- **Don't save the Step 2 queue position.** A saved index is a claim about the
  world that stops being true as soon as anything else changes. Rebuild it
  from disk instead.
- **Space confirms a fix, the arrow keys move through the queue,** and holding
  Space confirms exactly once. Shortcuts apply to the whole page, so a
  focused child widget can't swallow them, and they do nothing while a text
  field has focus.
- **Pages are fixed layouts, not scrolling documents.** Every page fits at
  1280×720, 1440×900, 1920×1080 and 2560×1440.
- **Known legacy limitations stay documented, not silently "fixed".** Changing
  filename parsing or specimen identity would renumber existing data. That is
  a scientific decision, not an engineering one, so it is recorded with
  options for the research group to decide later.
