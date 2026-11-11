---
title: Data
sidebar_label: Data
sidebar_position: 2
---

## 1. Datasets

### 1.1 Data Sources
Multiple datasets are used for the different stages of development of this project. The following table gives an overview of the stage in which each dataset was used:

<figure>
![CRAI-dataset](/img/crai/CRAI-datasets.png)
<figcaption>**Overview of the number of samples, temporal coverage and utilization of the datasets.** *The temporal coverage reported here corresponds to the time period used in the study. Ballot symbols indicate the usage of each dataset for three specific tasks: training of the AI model, evaluation of the AI model, and analysis of the dataset.*</figcaption>
</figure>

The HadEX-CAM datasets used for the evaluation and analysis are available on Zenodo: [https://zenodo.org/records/13891336](https://zenodo.org/records/13891336).

Notably, the CMIP6 dataset is used in all the model stages. The exact CMIP6 ensemble members used are provided in this table:

<figure>
![CRAI-ensemble](/img/crai/CRAI-CMIP6-ensemble.png)
<figcaption>**List of CMIP6 models and members used to create the training, validation and test datasets.**</figcaption>
</figure>

## 2. Environment Setup

Next, we provide a guide to obtain the CMIP6 dataset for training and evaluation, using [ESMValTool](https://www.esmvaltool.org/) to download and preprocess it.

### 2.1 Install ESMValTool

Install ESMValTool and its core engine into a dedicated Conda environment:

```bash
conda create -n esmvaltool -c conda-forge esmvaltool
conda activate esmvaltool
```

Full installation instructions are available in the [ESMValTool documentation](https://docs.esmvaltool.org/en/latest/quickstart/installation.html).

### 2.2 Configure local paths

Generate the default user configuration file:

```bash
esmvaltool config get_config_user
```

This creates `~/.config/esmvaltool/config-user.yml`. Open it and set the following fields. The values shown below reflect the recommended settings for this project; adjust the paths to your own system:

```yaml
# Where plots, NetCDF output, and logs are written
output_dir: /path/to/output

# Auxiliary data (masks, orography, etc.)
auxiliary_data_dir: /path/to/aux

# Where ESMValTool looks for (and downloads) raw data
rootpath:
  CMIP6: /path/to/cmip6_downloads   # ESMValTool will download CMIP6 data here

# Separate directory for ESGF downloads (can be same as rootpath CMIP6)
download_dir: /path/to/cmip6_downloads

drs:
  CMIP6: default
  native6: default

# Search ESGF only when data is not already available locally
search_esgf: when_missing

# Keep intermediate preprocessed files for debugging
remove_preproc_dir: false

# Fail if a requested dataset is missing
skip_nonexistent: false

# Parallelism
max_parallel_tasks: 1

log_level: info
compress_netcdf: false
save_intermediary_cubes: false
```

The `esmval` module contains an example configuration, `config-user.yml`. This is the configuration that was used to obtain the datasets. The paths must be modified.

### 2.3 Configure ESGF search nodes

If the default ESGF node is unavailable or missing data, create `~/.esmvaltool/esgf-pyclient.yml` with additional search nodes:

```yaml
search_connection:
  urls:
    - https://esgf-node.llnl.gov/esg-search
    - https://esgf.ceda.ac.uk/esg-search
  distrib: true
```

## 3. Data Acquisition

### 3.1 Understanding the Recipe

The CRAI recipe (`CRAI_recipe.yml`) is structured in four blocks:

**`documentation`**: metadata describing the recipe's purpose and authors.

**`datasets`**: specifies the exact CMIP6 ensemble members to retrieve.

**`preprocessors`**: defines the spatial and mathematical operations applied to the raw data before it reaches the diagnostic script.

**`diagnostics` / `scripts`**: maps datasets to preprocessors, then passes the standardised output to the desired script, which in this case is a placeholder with no operations.

### 3.2 Run the Recipe

Once all data is in place and paths are configured:

```bash
conda activate esmvaltool
esmvaltool run CRAI_recipe.yml
```

The output NetCDF store will be written to the `work_dir` subdirectory inside `output_dir`. The raw CMIP6 files are downloaded into `download_dir`, following the directory structure shown in Section 4.1.

## 4. Preprocessing

### 4.1 Extreme Indices Calculation

Once each model's data is downloaded, it must be organized as follows:

```bash
raw_downloads/
`-- CMIP6
    `-- CMIP
        |-- AWI
        |   `-- AWI-CM-1-1-MR
        |-- CNRM-CERFACS
        |   `-- CNRM-CM6-1-HR
        |-- EC-Earth-Consortium
        |   |-- EC-Earth3
        |   |-- EC-Earth3-AerChem
        |   |-- EC-Earth3-CC
        |   `-- EC-Earth3-Veg
        |-- MOHC
        |   `-- HadGEM3-GC31-MM
        `-- MPI-M
            `-- MPI-ESM1-2-HR
```

Inside each model, the structure must be:

```bash
 `-- historical
     `-- r10i1p1f1 (member code)
         `-- day (frequency)
             `-- tasmax (variable)
                 `-- ["gr" or "gn"] (grid)
                     `-- "v..."
```

With each model folder correctly in place and filled with the downloaded NetCDF (.nc) data containing daily tasmax and tasmin for each of the years from 1940 to 2022, run the Python + CDO pipeline for preprocessing and index calculation, `build_all_cmip_indices.sh`.

:::note
If you are on an HPC system, run the preprocessing code through a Slurm script.
:::

:::warning
If `build_all_cmip_indices.sh` is interrupted during its execution, check which model member it stopped at and delete the incomplete file. The code checks which output files already exist and runs the pipeline only for the missing ones. If a file creation is interrupted, the incomplete file will be counted as an existing one and will break the training pipeline afterwards.
:::

### 4.2 Masks

Mask files were created to simulate the missing-value patterns of the original HadEX files over the other datasets used for training and evaluation. To reproduce the paper correctly, these masks must be built from the original HadEX data supplied by the authors in the Zenodo repository.

#### 4.2.1 Original data missing-value masks

This procedure must be executed for each of the original data files `original_<IDX>_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc` available on Zenodo (one for each IDX). To create the mask files, follow this sequence of CDO commands:

1. Divide the file by itself to obtain a pattern of 1 and NaN values: `cdo div original_<IDX>_...nc original_<IDX>_...nc div_<IDX>.nc`
2. Turn the missing values into 0: `cdo setmisstoc,0 div_<IDX>.nc mask_<IDX>.nc`
3. Delete the intermediate file: `rm div_<IDX>.nc`
4. Verify the mask information using the CDO info command. Expect Miss=0, min 0 and max 1 at every one of the 1416 timesteps (1901–2018): `cdo info mask_<IDX>.nc`

```bash
#!/bin/bash

ROOT="<DATA_ROOT_DIRECTORY>"
DATA_DIR="$ROOT/data"
MASK_DIR="$ROOT/masks"
LOGS_DIR="$ROOT/logs"
LOGFILE="$LOGS_DIR/mask_creation_$(date +%Y%m%d_%H%M%S).log"
exec > >(tee -a "$LOGFILE") 2>&1

echo 'Creating masks dir if do not exist'
mkdir -p $MASK_DIR
mkdir -p $LOGS_DIR


echo 'Creating masks'

FILE_TX90p="$DATA_DIR/original_TX90p_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc"
FILE_TX10p="$DATA_DIR/original_TX10p_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc"
FILE_TN90p="$DATA_DIR/original_TN90p_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc"
FILE_TN10p="$DATA_DIR/original_TN10p_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc"

FILES=(TX90p TX10p TN90p TN10p)


for F in "${FILES[@]}";do
        varname="FILE_${F}"
        echo "Creating mask for $F"
        echo "Dividing $F over itself"
        echo '--------------------------------------------------------------'
        cdo div ${!varname} ${!varname} ${MASK_DIR}/div_${F}.nc
        echo 'Turning missing info into 0'
        echo '--------------------------------------------------------------'
        cdo setmisstoc,0 ${MASK_DIR}/div_${F}.nc ${MASK_DIR}/mask_${F}.nc
        echo 'Deleting intermadiate file'
        echo '--------------------------------------------------------------'
        rm ${MASK_DIR}/div_${F}.nc
        echo 'Verifing info'
        echo '--------------------------------------------------------------'
        cdo info ${MASK_DIR}/mask_${F}.nc | head -n 10
done

echo 'Mask creations finished'
```

#### 4.2.2 ERA5 evaluation masks

For the ERA5 evaluation in the paper, the time interval is shorter, going from 1940 to 2018 (948 steps), so the masks already built from the original files (Section 4.2.1) must be cropped to the required time window.

1. Start from the original masks `mask_<IDX>.nc` (1416 steps).
2. Crop them to the ERA5 window to obtain a mask with the same length and calendar months: `cdo seldate,1940-01-01,2018-12-31`
3. Verify the ERA5 mask file. It must have 948 steps, from 1940-01 to 2018-12: `cdo sinfon ERA5_mask_<IDX>.nc`

#### 4.2.3 Steady masks

This is an auxiliary mask that must be passed with `--steady-masks` during the evaluation procedure. It flags the sea grid cells as NaN and excludes them from the evaluation map plots and calculations, following the same patterns as the authors. For these masks, the reference files are the original CRAI outputs supplied by the authors in the Zenodo repository, and the pipeline is executed in Python instead of CDO.

1. For each file `crai_<IDX>_mon-eu-32x32_hadexcam_infilled_all-avg_1901-2018.nc`:
2. Create a land identifier, with NaN values as 0 and all others as 1, for a single timestep `time = 0`: `~np.isnan(bs_mask[idx].isel(time=0).values)`
3. Create a sea identifier to map all the sea: `sea = (~land).astype("float32")`
4. Check that land and sea match the proportions of the map you are using. For the paper, land = 763 and sea = 261.
5. Check that the latitude and longitude coordinates of the training data match the mask ones with minimum error.
6. Create the steady mask using the sea values and the coordinate values from the training dataset:

```python
    steady = xr.Dataset(
        {f"{idx}": (("time", "latitude", "longitude"), sea[None])},
        coords={"time": train.time.values[:1],
                "latitude": train.latitude.values,
                "longitude": train.longitude.values},
    )
```

7. Copy the training attribute metadata to the steady coordinates:

```python
steady.latitude.attrs = train.latitude.attrs.copy()
steady.longitude.attrs = train.longitude.attrs.copy()
steady.time.attrs = train.time.attrs.copy()
```

8. Save the mask.

The full pipeline can be written as follows:

```python

# steady_mask_creation.py

import xarray as xr
import numpy as np
from pathlib import Path
import os

MASK_ROOT = "<DATA_ROOT_DIRECTORY>"
GRID_REFF = "masks/Zenodo_refferences/"

file_name = "{src}_{idx}_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc"
index = ["TN10p", "TN90p", "TX10p", "TX90p"]
corr_dit = {"original": "crai", "all": "all-avg"}
source = ["original", "crai"]

print(corr_dit["all"])

# ---------------------------------------------------------------
# Steady mask creation
# ---------------------------------------------------------------
src = "original"
version = "all"

for idx in index:
    TRAINING_DATA = f"{MASK_ROOT}{GRID_REFF}{src}_{idx}_mon-eu-32x32_hadexcam_infilled_{version}_1901-2018.nc"
    BASE_MASK = f"{MASK_ROOT}{GRID_REFF}{corr_dit[src]}_{idx}_mon-eu-32x32_hadexcam_infilled_{corr_dit[version]}_1901-2018.nc"
    print(f"Opening grid Reference file for {idx}")
    print(TRAINING_DATA)
    print(f"Opening land & sea reference file for {idx}")
    print(BASE_MASK)

    train = xr.open_dataset(TRAINING_DATA)
    bs_mask = xr.open_dataset(BASE_MASK)

    print(f"VARIABLES TRAINING DATA: {train.data_vars}")
    print(f"DIMENSIONS TRAINING DATA: {train.dims}")
    print(f"VARIABLES MASK DATA: {bs_mask.data_vars}")
    print(f"DIMENSIONS MASK DATA: {bs_mask.dims}")

    assert np.allclose(bs_mask.latitude.values, train.latitude.values)
    assert np.allclose(bs_mask.longitude.values, train.longitude.values)

    land = ~np.isnan(bs_mask[idx].isel(time=0).values)
    sea = (~land).astype("float32")
    print(f"LAND values: {np.sum(land == True)}")
    print(f"SEA values: {np.sum(sea == True)}")

    steady = xr.Dataset(
        {f"{idx}": (("time", "latitude", "longitude"), sea[None])},
        coords={"time": train.time.values[:1],
                "latitude": train.latitude.values,
                "longitude": train.longitude.values},
    )

    print(steady.data_vars)
    print(steady.dims)

    steady.latitude.attrs = train.latitude.attrs.copy()
    steady.longitude.attrs = train.longitude.attrs.copy()
    steady.time.attrs = train.time.attrs.copy()

    OUT = f"{MASK_ROOT}{GRID_REFF}steady"
    os.makedirs(OUT, exist_ok=True)
    steady.to_netcdf(f"{OUT}/steady_sea_{idx}.nc")
    print("Writen at", f"{OUT}/steady_sea_{idx}.nc")

# ---------------------------------------------------------------
# Verification (relative path: run the script from MASK_ROOT)
# ---------------------------------------------------------------
for idx in index:
    test = xr.open_dataset(f"masks/Zenodo_refferences/steady/steady_sea_{idx}.nc")

    sm = test[idx].isel(time=0).values
    print("sea (1):", int((sm == 1).sum()))    # expect 261
    print("land (0):", int((sm == 0).sum()))   # expect 763
    print("NaN:", int(np.isnan(sm).sum()))     # expect 0

```

## 5. Final Data Structure

The preprocessing pipeline produces 4 NetCDF (.nc) files for each model member, one per index (TN10p, TN90p, TX10p and TX90p), following the naming pattern `{index}_{model}_{member}_{starting_year}-{ending_year}.nc`. These files are the ones used in the training and validation pipeline of the model.
