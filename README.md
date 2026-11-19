# Land Use Classification with Sentinel‑2 and XGBoost (Dinamica EGO)

`classification_model.egomlx` is a Dinamica EGO model that:

1. searches the **Brazil Data Cube STAC catalog (INPE)** for a Sentinel‑2 L2A scene over the area of interest,
2. downloads 10 spectral bands and resamples the 20 m bands to 10 m,
3. samples random points inside the ground‑truth polygons and extracts the band values at each point,
4. trains an **XGBoost** classifier and evaluates it on a held‑out test set,
5. classifies the whole scene into **3 land‑use classes** and writes the result as a GeoTIFF.

---

## 1. Requirements

- **Dinamica EGO 8.14.1 or later**

---

## 2. Folder layout

```
<repository>/
├── classification_model.egomlx
├── inputs/                   ← provided with the repository
│   ├── inputs_table.csv
│   └── (area of interest and ground truth files)
└── outputs/                  ← created automatically when the model runs
```

Every path is resolved relative to the folder that holds the `.egomlx` file, so keep the model and the `inputs/` folder together.

---

## 3. Inputs

The `inputs/` folder comes with the repository. You don't need to create anything to run the model.

`inputs/inputs_table.csv` holds the run parameters:

| Parameter | Description |
|---|---|
| `Area of Interest` | Vector file with the area boundary used to search for the Sentinel‑2 scene. |
| `Ground Truth Data` | Polygon vector file with the labelled samples, in a `class` column (1 = water, 2 = vegetation, 3 = anthropic use). |
| `StartDate`  | STAC search period start date, as `yyyy-mm-dd`. |
| `EndDate`  | STAC search period end date, as `yyyy-mm-dd`. |
| `MaxCloudCover` | Highest scene cloud cover allowed, in percent. |

To run the model for another area or period, edit the `Value` column of this file. Keep the `Parameter` and `Type` columns as they are.

---

## 4. Running

1. Open `classification_model.egomlx` in Dinamica EGO 8.14.1 or later.
2. Run the model. Progress and Python messages appear in the Dinamica message log.

---

## 5. Outputs

Everything is written to `outputs/`:

```
outputs/
├── 1_STAC_Download/<scene_id>/        downloaded Sentinel-2 bands
├── 2_ResampledBands/<scene_id>/       20 m bands resampled to 10 m
├── 3_SampleDataset/
│   ├── random_points.gpkg
│   ├── scenes_datasets/<scene_id>.gpkg
│   ├── dataset_concat.gpkg
│   ├── samples_train.gpkg
│   └── samples_test.gpkg
├── 4_classifier/
│   ├── rf_classifier.pkl
│   ├── le_dir/rf_classifier.joblib
│   └── acc_metrics.csv
└── 5_inference/
    └── <scene_id>.tif        ← final land-use map
```

### 5.1 Final map: `5_inference/<scene_id>.tif`

This is the main result: a land‑use map with **3 classes**.

| Pixel value | Class |
|---|---|
| **1** | Water |
| **2** | Vegetation |
| **3** | Anthropic use |
| 0 | No data |

- **GeoTIFF, one band, `uint8`**, deflate‑compressed.
- **10 m pixels**, on the same grid, extent and coordinate system (UTM) as the Sentinel‑2 tile's B02 band.
- The **whole tile** is classified, not just the area of interest. To keep only your area, clip the map with the AOI in QGIS, ArcGIS or Dinamica.
- The classifier has no cloud mask, so cloudy areas still get one of the 3 classes. Check those areas before using the map.



### 5.2 Accuracy metrics: `4_classifier/acc_metrics.csv`

A two‑column table (`Metric`, `Value`) computed on `samples_test.gpkg`, rounded to 4 decimals:

| Metric | Meaning |
|---|---|
| Kappa Score | Cohen's kappa: agreement beyond chance (1 = perfect, 0 = no better than chance). |
| F1-Score | Weighted F1 across the 3 classes. |
| Recall | Weighted recall. |
| Accuracy | Overall share of test points classified correctly. |
| Precision | Weighted precision. |
| Balanced Accuracy | Mean recall per class, which is fairer when classes are imbalanced. |

Training and test points are split by polygon, so test points come from polygons the model never saw during training.