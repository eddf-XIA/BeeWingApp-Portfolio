# The real example: one forewing pair, genuine outputs

← [Back to the overview](../README.md)

<img src="../assets/real/real_pair_stages.jpg" alt="Real pair: detection, correction, standardization">

<img src="../assets/real/real_pair_landmarks.jpg" alt="Real pair: model-predicted landmarks">

**[Full-resolution comparison of both wings, all four stages](../assets/real/real_pair_full_resolution.jpg)** (4483 × 2470).

## What this is, and what it isn't

- **It is** one real bumblebee specimen's left and right forewing photos,
  approved for public demonstration, run through the real application. Every
  output shown is what the software produced.
- **It isn't** a validation. One pair says nothing about the model's accuracy
  across a collection. No accuracy, error or agreement statistics are reported
  anywhere in this portfolio.

## How the images were prepared

1. **Identifiers removed.** The two files were copied out of the collection
   and renamed in a neutral folder. The original specimen code, collection date
   and species aren't shown.
2. **Metadata stripped losslessly.** EXIF (15 tags, including the camera
   model) and every other metadata segment were removed **without
   re-encoding**. The decoded pixels were checked to be identical to the
   originals.
3. **Run through the real application**, from source, in a temporary
   workspace:
   - Step 1 (correction) and Step 3 (standardization) through the pipeline the
     app uses;
   - Step 4 through the real separate analysis process, with the trained
     model.
4. **Figures made only from those outputs.**
   - Marker outlines come from the pipeline's own detection functions.
   - The inferred corner is identified by checking which corner of the
     resolved square doesn't coincide with any detected marker's corner.
   - Landmarks are plotted at the CSV coordinates.
   - The only additions are labels, the colour coding by likelihood, and
     resizing or cropping for display.

## The run

| Step | Result |
|---|---|
| 1 · Correct | 2 attempted, **2 corrected automatically**, 0 sent to manual correction. The detector found **3 of 4 markers on each photo**. The top-right marker is visible but wasn't detected, so its corner was **inferred** from the other three. |
| 3 · Standardize | LF mirrored, RF rotated 180°; 2 written |
| 4 · Analyse | 2 images, 13 landmarks each, **13.9 s** including model load (laptop CPU); result file verified on disk |

## Model-predicted landmarks (not validated ground truth)

Coordinates are pixels in the standardized 1500 × 1500 image. The likelihood
is the model's own confidence for each point, on a 0–1 scale.

| # | LF x | LF y | LF likelihood | RF x | RF y | RF likelihood |
|---|---|---|---|---|---|---|
| 1 | 213.8 | 553.6 | **0.029** | 72.0 | 810.3 | **0.049** |
| 2 | 440.1 | 704.6 | 0.778 | 311.3 | 876.6 | 0.826 |
| 3 | 454.0 | 618.7 | 0.853 | 281.5 | 796.7 | 0.799 |
| 4 | 475.5 | 479.6 | 0.821 | 239.3 | 652.8 | 0.810 |
| 5 | 570.0 | 518.6 | 0.911 | 332.1 | 656.8 | 0.929 |
| 6 | 603.5 | 571.5 | 0.992 | 383.7 | 694.4 | 0.912 |
| 7 | 714.1 | 592.9 | 0.989 | 489.4 | 674.9 | 0.975 |
| 8 | 775.0 | 522.3 | 0.864 | 506.0 | 585.6 | 0.918 |
| 9 | 685.2 | 498.0 | 0.837 | 423.6 | 594.3 | 0.831 |
| 10 | 836.4 | 635.1 | 1.000 | 615.6 | 662.2 | 0.961 |
| 11 | 809.2 | 638.1 | 1.000 | 593.3 | 676.9 | 1.000 |
| 12 | 692.6 | 715.7 | 0.971 | 531.8 | 790.2 | 0.913 |
| 13 | 837.1 | 521.1 | 1.000 | 559.3 | 562.9 | 0.996 |

Rounded to one decimal place (coordinates) and three (likelihood) from the
CSV.

## Reading it as an engineer

- **The low-confidence point is the interesting one.** Landmark 1 is the one
  nearest the wing base, close to the edge of the wing area visible in both
  photos. The model reports it at about 0.03–0.05. I'm not claiming a cause
  from a single pair. The pipeline keeps every point with its confidence
  instead of dropping or hiding weak ones, so downstream analysis can filter,
  flag or re-check them. It is shown here as it came out.
- **Orientation standardization is doing its job.** The two wings are mirror
  images of each other. After Step 3 the same-numbered landmarks fall in the
  same region of both images, even though the original photos were taken at
  different angles and crops.
- **Three markers were enough.** Step 1 tolerates one missed marker by
  completing the square from the other three. That is why neither photo
  needed the manual-correction queue.
