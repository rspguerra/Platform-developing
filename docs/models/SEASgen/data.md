---
title: Data
sidebar_label: Data
sidebar_position: 2
---

## 1. Datasets

### 1.1 Data Sources

This project uses [ESMValTool](https://www.esmvaltool.org/) to download and preprocess the climate datasets required for training and evaluating the SEASGen model. Two data sources are needed: ERA5 reanalysis data and CMIP6 model output. The necessary scripts and recipes are provided in the `esmval` package in the source code, but **ESMValTool** requires a different environment from the rest of the project.

## 2. Environment Setup

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
  native6: /path/to/esmvaltool_data  # ERA5 data directory (see Section 3)

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

### 2.3 Configure ESGF search nodes (for CMIP6)

If the default ESGF node is unavailable or missing data, create `~/.esmvaltool/esgf-pyclient.yml` with additional search nodes:

```yaml
search_connection:
  urls:
    - https://esgf-node.llnl.gov/esg-search
    - https://esgf.ceda.ac.uk/esg-search
  distrib: true
```

## 3. Data Acquisition

ERA5 data must be downloaded manually from the Copernicus CDS and placed into a specific directory structure before ESMValTool can use it.

### 3.1 Set up the CDS API

Create an account at [https://cds.climate.copernicus.eu/](https://cds.climate.copernicus.eu/). Find your API key under *Your Profile → API Key* and save it:

```bash
nano ~/.cdsapirc
```

```text
url: https://cds.climate.copernicus.eu/api
key: YOUR-API-KEY-HERE
```

### 3.2 Download the data

Install `era5cli` and download the required variables:

```bash
pip install era5cli

mkdir -p ~/era5_staging
cd ~/era5_staging

era5cli monthly \
  --variables 2m_temperature sea_surface_temperature total_precipitation geopotential \
  --startyear 1950 \
  --endyear 2022 \
  --format netcdf
```

### 3.3 Organise the directory structure

ESMValTool requires ERA5 files to follow a specific `native6` directory tree. Create the structure and move the downloaded files:

```bash
mkdir -p esmvaltool_data
cd esmvaltool_data

mkdir -p Tier3/ERA5/1/mon/tas
mkdir -p Tier3/ERA5/1/mon/tos
mkdir -p Tier3/ERA5/1/mon/pr
mkdir -p Tier3/ERA5/1/mon/zg

mv ~/era5_staging/*2m_temperature*.nc       Tier3/ERA5/1/mon/tas/
mv ~/era5_staging/*sea_surface_temperature*.nc Tier3/ERA5/1/mon/tos/
mv ~/era5_staging/*total_precipitation*.nc  Tier3/ERA5/1/mon/pr/
mv ~/era5_staging/*geopotential*.nc         Tier3/ERA5/1/mon/zg/
```

### 3.4 Clean and reformat

Files downloaded by `era5cli` have some metadata and variable names that clash with the `ESMValTool` recipe. Run the following script (provided in the `esmval` package in the source code) to fix them:

```bash
python clean.py --data-dir /path/to/esmvaltool_data
```

The script produces a corrected copy of the data. Update `native6` in `config-user.yml` to point to this cleaned directory.

## 4. Preprocessing

### 4.1 Understanding the Recipe

The SEASGen recipe (`recipe_seasgen.yml`) is structured in four blocks:

**`documentation`**: metadata describing the recipe's purpose and authors.

**`datasets`**: specifies the exact data to retrieve. CMIP6 ensembles (CanESM5, MIROC6, MPI-ESM1-2-LR, MIROC-ES2L, CNRM-CM6-1) are defined using YAML anchors to avoid repetition. ERA5 is referenced as a `native6` dataset.

**`preprocessors`**: defines the spatial and mathematical operations applied to the raw data before it reaches the diagnostic script:
- `regrid`: regrids all variables to a uniform 5×5 degree grid using bilinear interpolation (area-weighted for precipitation).
- `extract_levels`: extracts specific pressure levels (300 hPa and 500 hPa) from the geopotential data.

**`diagnostics` / `scripts`**: maps datasets to preprocessors, then passes the standardised output to `preprocessor.py`, which merges all variables into a single zarr store ready for model training.

### 4.2 Run the Recipe

Once all data is in place and paths are configured:

```bash
conda activate esmvaltool
esmvaltool run recipe_seasgen.yml
```

## 5. Final Data Structure

The output zarr store will be written to the `work_dir` subdirectory inside `output_dir`. Update `paths.yml` in the SEASGen experiment configs to point to this path before running the model.
