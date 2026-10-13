# Visual walkthrough (synthetic data)

← [Back to the overview](../README.md)

Everything below was made from **synthetic** inputs. An invented wing is
drawn on a card with four public ArUco markers (OpenCV `DICT_4X4_50`), and a
hand-held photo is simulated. The application screenshots show the real
program processing a 14-wing synthetic collection in a temporary folder.

## The idea, step by step

| | |
|---|---|
| <img src="../assets/synthetic/01_raw_photo.jpg" width="420" alt="raw"> | **Raw photograph.** The card is tilted, seen in perspective, and lit unevenly. The pixel scale differs from photo to photo. |
| <img src="../assets/synthetic/02_markers_detected.jpg" width="420" alt="detected"> | **Markers detected.** The outer corner of each marker defines a quadrilateral whose real-world size is known. |
| <img src="../assets/synthetic/03_corrected.jpg" width="300" alt="corrected"> | **Corrected.** A perspective warp maps that quadrilateral to a fixed square, so every corrected image has the same mm-per-pixel scale. |
| <img src="../assets/synthetic/04_standardized.jpg" width="300" alt="standardized"> | **Standardized.** A left wing is mirrored, so it faces the same way as every other wing. |
| <img src="../assets/synthetic/05_landmarks_illustrative.jpg" width="300" alt="illustrative points"> | **Landmarks (illustrative).** In the real application a trained pose-estimation model predicts 13 landmarks per wing. The points here are junctions of the drawn veins, **not model output**. |

### When detection fails

| | |
|---|---|
| <img src="../assets/synthetic/06_detection_failed.jpg" width="420" alt="failed"> | Glare and a fingertip hide two markers. Rather than guess, the photo goes to a queue for a person. |
| <img src="../assets/synthetic/07_manual_corners.jpg" width="420" alt="manual"> | The researcher clicks the four outer corners in a fixed order: top-left, top-right, bottom-right, bottom-left. |
| <img src="../assets/synthetic/08_manual_corrected.jpg" width="300" alt="manual corrected"> | The same warp produces the corrected image. |
| <img src="../assets/synthetic/09_rf_standardized.jpg" width="300" alt="rf standardized"> | A right wing is rotated 180° to match. |

## The application

Captured from the real program on the native Windows platform at 1920×1080.
The only change was blanking the affiliation line in the sidebar. The model
folder is an empty placeholder, so no inference is shown.

### Folders: one choice, safe by construction
<img src="../assets/screenshots/folders.png" alt="Folders page">

The researcher picks only the folder that holds the per-species folders. Every
output goes into a sibling `BeeWing Output` folder. A layout where outputs
would overlap the photographs can't be saved, and the reason is shown.

### Overview: counts from disk
<img src="../assets/screenshots/overview.png" alt="Overview page">

### Step 1: automatic correction
<img src="../assets/screenshots/step1_finished.png" alt="Step 1 finished">

14 wing photos attempted: 12 corrected automatically, 2 sent to Step 2, and 7
full-body photos skipped by design.

### Step 2: manual correction
<img src="../assets/screenshots/step2_manual_fix.png" alt="Step 2 with four corners placed">

Four corners placed on a photo whose markers were hidden. The guide teaches
corner order by *position on the card*, not by marker pattern, so it still
works on a rotated card.

<img src="../assets/screenshots/step2_gallery.png" alt="Corrected-image gallery">

The corrected-image gallery. Any correction can be sent back to the queue
without deleting the existing file.

### Step 3: orientation
<img src="../assets/screenshots/step3_finished.png" alt="Step 3 finished">

### Step 4: landmark analysis
<img src="../assets/screenshots/step4_ready.png" alt="Step 4 ready">

The page states which model a run will use. Here it is a placeholder folder,
because no real model is included. The readout on the right is part of the
app's built-in illustrated guide, not a result.
