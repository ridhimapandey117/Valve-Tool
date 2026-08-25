# Valve Tool

A Python-based image analysis tool for automatically segmenting and measuring heart valve leaflets from grayscale scans. The tool isolates individual leaflets from a single image, separates leaflets that are touching using watershed segmentation, and extracts physical measurements (area, perimeter, length) in centimeters to help quantify leaflet symmetry.

## Overview

Given a cropped grayscale image of valve leaflets, this notebook:

1. Enhances and denoises the image
2. Builds a binary mask isolating the leaflets from the background
3. Removes irrelevant regions touching the edge of the frame
4. Separates touching/overlapping leaflets using watershed segmentation
5. Measures each leaflet's area, perimeter, and major axis length
6. Converts pixel measurements to real-world units (cm, cm²)
7. Reports symmetry metrics across leaflets

## Pipeline

### 1. Image Preprocessing
- **Upload & Grayscale**: Image is uploaded (via Google Colab) and converted to grayscale.
- **Crop**: The image is manually cropped to isolate the leaflet region as tightly as possible.
- **CLAHE Contrast Enhancement**: `cv2.createCLAHE` boosts local contrast to make leaflet edges more distinct.
- **Bilateral Filtering**: `cv2.bilateralFilter` reduces noise while preserving edges.

### 2. Binary Mask Creation
- **Otsu Thresholding**: Automatically finds a threshold to separate leaflets from background.
- **Morphological Cleanup**: Binary dilation and opening (`skimage.morphology`) remove noise and smooth the mask.
- **Hole Filling**: `binary_fill_holes` ensures leaflets are solid, continuous regions.
- **Edge Cleanup**: Any connected region touching the bottom of the frame (e.g. leftover valve housing) is removed via `regionprops` bounding boxes.

### 3. Leaflet Separation (Watershed Segmentation)
Touching leaflets are separated using a distance-transform-based watershed approach:
- `distance_transform_edt` converts the binary mask into a "distance-to-background" map, turning each leaflet into a hill peaking at its widest point.
- `peak_local_max` finds these peaks, which serve as seed markers for each leaflet (`min_distance` controls how close two peaks can be before being treated as separate leaflets).
- `watershed` floods outward from each marker (on the inverted distance map) until floods from different markers meet, producing a uniquely labeled region for each leaflet.

### 4. Measurement Extraction
For each labeled leaflet, `regionprops` and `cv2.findContours` are used to extract:
- Area (pixels)
- Perimeter
- Major axis length (pixels)

### 5. Pixel-to-Real-World Conversion
Pixel measurements are converted to physical units using fixed calibration constants:
- `pixel_edge = 0.0338 cm/pixel`
- `pixel_area = 0.001142 cm²/pixel`

These are used to compute each leaflet's real-world area (cm²) and length (cm).

### 6. Symmetry Analysis
The tool reports:
- Mean and standard deviation of leaflet area
- Area asymmetry rate (std/mean, where 0 = perfectly symmetric)
- Max/min area ratio
- Coefficient of variation for leaflet length
- Pairwise percent differences in area between all leaflets

## Requirements

```
numpy
pandas
opencv-python
matplotlib
seaborn
scikit-image
scikit-learn
scipy
```

## Usage

1. Open the notebook in Google Colab (or a local Jupyter environment with the upload cell adapted accordingly).
2. Run the upload cell and select your leaflet scan image.
3. Adjust the crop dimensions in the **CROP IMAGE** cell so the image is tightly framed around the leaflets.
4. Run the remaining cells in order. Intermediate plots let you sanity-check each preprocessing step (grayscale, enhanced, denoised, binary mask).
5. Review the final segmentation overlay and printed area/length/symmetry statistics.

## Notes

- The `min_distance` parameter in the watershed step may need tuning depending on how close together leaflets are in a given scan.
- The pixel-to-cm calibration constants are specific to the imaging setup used to capture the source scans and should be recalculated if the camera, resolution, or working distance changes.
- A K-means clustering step is included in the notebook to document an earlier approach that was explored but is not required for the final pipeline.
