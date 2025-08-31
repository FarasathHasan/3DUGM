# Unified CA–DNN Urban Growth & Verticalization Framework

This repository contains a unified cellular automata (CA) + deep neural network (DNN) framework for simulating horizontal urban expansion and vertical building volume growth from 
multi-temporal raster inputs. The main script ties together: entropy-based rule generation, CA transition simulation, neighborhood/distance kernels, and a DNN to predict building 
volume (vertical growth) for newly urbanized cells.

> **Important:** This is a single, unified framework. **Do not** run script sections seperately. Configure file paths and parameters and then run the framework end-to-end.

---

## Repository structure (suggested)

```
/ (root)
├─ README.md
├─ Python Script.py        # main script (your provided notebook/script saved as a .py)
├─ requirements.txt
├─ data/
│  ├─ 2015_cleaned.tif  # Cleaned is referred to as the raster file after removing the null values
│  ├─ 2020_cleaned.tif
│  ├─ cbd_cleaned.tif
│  ├─ ntl_cleaned.tif
│  ├─ road_cleaned.tif
│  ├─ restricted_cleaned.tif
│  ├─ pop_cleaned.tif
│  ├─ slope_cleaned.tif
│  ├─ amenitykernel_cleaned.tif
│  ├─ commercialkernel_cleaned.tif
│  ├─ industrailkernel_cleaned.tif
│  ├─ building_volume_cleaned.tif
│  └─ 2025_cleaned.tif   # used here as study-area / accuracy reference
├─ outputs/
│  ├─ veryhighurbansparawlNEW.tif
│  ├─ kernel_component_shopping.tif
│  ├─ kernel_component_industrial.tif
│  ├─ kernel_component_residential.tif
│  ├─ newvolme.tif
│  └─ building_heights_suburbanization.tif
└─ docs/
   └─ notes.md
```

---

## Quick overview of code objects (mapping to filenames/variables)

Use this table to check the input files you must provide and where they map to the script variables.

| Variable name in script | Meaning / expected raster content                                                                           | Example filename in `data/`            |
| ----------------------- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `file1`, `file2`        | Base landcover at time1 and time2 (single-band with integer class codes; urban class=1 in the code)         | `2015_cleaned.tif`, `2020_cleaned.tif` |
| `cbd`                   | CBD distance kernel or influence raster                                                                     | `cbd_cleaned.tif`                      |
| `ntl`                   | Night-time lights (if used) — brightness or index raster                                                    | `ntl_cleaned.tif`                      |
| `road`                  | Road proximity or road density raster                                                                       | `road_cleaned.tif`                     |
| `restricted`            | Restriction mask raster (0 = free, >0 = restricted)                                                         | `restricted_cleaned.tif`               |
| `pop01`                 | Population density raster                                                                                   | `pop_cleaned.tif`                      |
| `slope`                 | Terrain slope raster (degrees or percent)                                                                   | `slope_cleaned.tif`                    |
| `amenities`             | Amenity kernel raster used for kernel decomposition                                                         | `amenitykernel_cleaned.tif`            |
| `commercial`            | Commercial kernel raster                                                                                    | `commercialkernel_cleaned.tif`         |
| `industrail`            | Industrial kernel raster                                                                                    | `industrailkernel_cleaned.tif`         |
| `building_volume`       | Observed building volume dataset (for training/validation of vertical model)                                | `building_volume_cleaned.tif`          |
| `study_area_file`       | Binary mask raster (1 = inside study area; 0 outside) used to restrict simulation + for accuracy assessment | `2025_cleaned.tif`                     |

> Filenames above match the variable names used in the provided script — keep them or update the variable assignment block at the top of the script.

---

## Environment & dependencies

Suggested minimal Python environment. Exact versions depend on your system (GDAL and TensorFlow versions are especially sensitive):

* Python 3.8–3.11
* numpy
* scipy
* rasterio or GDAL (GDAL is used in the script via `osgeo.gdal`)
* scikit-learn
* tensorflow (2.x)
* matplotlib
* seaborn (optional, for plotting)
* xgboost (optional; code falls back if unavailable)

Example `requirements.txt` snippet:

```
numpy
scipy
matplotlib
seaborn
scikit-learn
rasterio
gdal
tensorflow>=2.4
xgboost
```

### Installing GDAL

* On Ubuntu/Debian: `sudo apt-get update && sudo apt-get install -y gdal-bin python3-gdal` (you may still need `pip install GDAL==<matching-version>` depending on your pip wheel)
* On Windows/macos: use conda (`conda install -c conda-forge gdal rasterio`) or follow the OS-specific GDAL installation instructions.

> If you hit `ImportError` for GDAL/OSGeo, install GDAL through conda first or use prebuilt wheels matching your Python version.

---

## How to prepare your data

1. Ensure all rasters are aligned (same projection, resolution, and dimensions). The script checks dimensions and raises an error if they differ. The `performChecks()` methods rely
   on identical raster size.
2. Use single-band GeoTIFFs with consistent integer class values for landcover (`1` used as urban class in code). If urban class in your data is different, either change
   the `urban_class` argument in `EntropyRuleGenerator` / `fitmodel` or reclassify your rasters.
3. The `study_area_file` must be a binary mask raster (1 inside study area, 0 outside). The script uses this to limit simulation and validation.
4. If you need to rename files to match the variable names in the script, do so or update the path variables at the top of the script.

---

## Running the framework

1. Edit the top-of-script variables to point to your local paths in `data/` (the block in the script that sets `file1`, `file2`, `cbd`, ...).
2. Make sure `os.chdir(...)` in the script points to your project working directory or remove it and use absolute paths.
3. Run the full script end-to-end. Example:

```bash
python Python Script.py
```

**Why run end-to-end?**
The pipeline trains entropy rules, runs the CA simulation, extracts new cell kernels, and trains/predicts building volume in a chained workflow. Running partial blocks can leave 
objects uninitialized (for example, `rule_generator` or the trained DNN models) and cause errors.

---

## Expected outputs

* `veryhighurbansparawlNEW.tif` — predicted landcover after simulation (single-band raster where urban cells = 1). ##This can be chnaged as per the scnario which is handling
* `kernel_component_*.tif` — predicted kernel density components (shopping/industrial/residential) for newly urbanized cells.
* `newvolme.tif` — predicted building volumes for newly urbanized cells.
* `building_heights_suburbanization.tif` — building height map converted from predicted volume (m).

Paths and filenames are set in the script; change `exportPredicted()` and `export_*()` calls if you want different names or folders.

---

## Accuracy assessment & validation

* The script uses `model.checkAccuracy(reference_array)` to compare the predicted new urban cells against the `study_area_file` reference.
* For building volume validation, the DNN returns RMSE, MAE and R² when `building_volume_new.compare_with_actual()` is called (ensure you pass the correct actual raster path).
* Carefully check masks — both model and reference must be compared only inside the study area and for cells that were non-urban at simulation start.

---

## Common pitfalls & troubleshooting

* **Dimension mismatch / `performChecks()` error**: Make sure projection, extent, resolution and number of rows/columns are identical among rasters. Reproject/resample
  using `gdalwarp` or `rasterio` if needed.
* **Incorrect urban class**: If your urban class is not `1`, either change the code variable `urban_class` or reclassify your landcover rasters.
* **GDAL import issues**: Prefer using conda (`conda install -c conda-forge gdal rasterio`) when GDAL `pip` wheels fail.
* **Memory issues**: Training DNNs on large rasters can be memory- and time-intensive. Consider sampling training pixels or using a GPU.

---

## Recommended improvements (for contributors)

* Add CLI argument parsing so users pass input/output paths instead of editing the script.
* Split the monolith into modular entry points (prepare-data, train-rules, run-ca, train-volume, predict-volume) guarded by sanity checks — but keep the default `run_all` behavior
  intact so users can run the unified model.
* Replace hard-coded filenames with a YAML/JSON config file for reproducibility.
* Add unit tests for raster reading and rule-generation functions.

---

## License & contact

This repository is provided as-is for research and educational use. 

---


