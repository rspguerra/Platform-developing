---
title: How to Use
sidebar_label: How to Use
sidebar_position: 3
---

Considering that the main goal of this guide is to make the pipeline easily and fully reproducible, it focuses on container execution, since this works in both HPC and local execution scenarios.

## 1. Environment Setup

### 1.1 Get the Code

The project repository is available on GitLab. To get started, clone the project and navigate to the directory:

```bash
git clone https://gitlab.earth.bsc.es/rpereir1/HGRF-HeatWaves.git
cd HGRF-HeatWaves
```

### 1.2 Python Environment

The dependencies required to execute this project, chosen for good package compatibility, are:

| Package | Version |
|---|---|
| Python | 3.8 |
| pip | latest (unpinned) |
| hydra-core | latest (unpinned) |
| numpy | latest (unpinned) |
| cartopy | 0.20.3 |
| matplotlib | 3.4.3 |
| xclim | 0.31.0 |
| xarray | latest (unpinned) |
| netcdf4 | 1.5.8 |
| pandas | 1.3.5 |
| cftime | 1.5.1 |
| seaborn | 0.9.0 |
| scikit-gstat | 1.0.1 |
| scipy | 1.7.3 |
| xgboost | 1.5.1 |
| shap | 0.36.0 |
| scikit-learn | latest (unpinned) |
| pydantic | 2.12.5 |
| setuptools | 67.8.0 |

This project uses uv for dependency management.

1. **Install uv** (if not installed):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

:::note
Restart your shell to ensure uv is in your PATH.
:::

2. **Create a local virtual environment:**

```bash
[ -d ".venv" ] || uv venv
```

3. **Initialize and sync the environment:**

```bash
uv init
uv sync
```

4. **Activate the venv:**

```bash
source .venv/bin/activate
```

### 1.3 Container

The container is built with Singularity from a *.def* file available in the repository. For the container scenario, the environment was installed using miniconda3 24.1.2-0.

```bash
# .def file
Bootstrap: docker
From: continuumio/miniconda3:24.1.2-0

%labels
    Author Bruna
    Version 1.0
    Description feat_env - main analysis environment

%post
    # Update conda and set channels
    conda update -n base -c conda-forge conda -y
    conda config --add channels conda-forge
    conda config --set channel_priority strict

    # Create environment
    conda create -n feat_env -y \
        python=3.8 \
        pip \
        hydra-core \
        numpy \
        cartopy=0.20.3 \
        matplotlib=3.4.3 \
        xclim=0.31.0 \
        xarray \
        netcdf4=1.5.8 \
        pandas=1.3.5 \
        cftime=1.5.1 \
        seaborn=0.9.0 \
        scikit-gstat=1.0.1 \
        scipy=1.7.3 \
        xgboost=1.5.1 \
        shap=0.36.0 \
        scikit-learn \
        pydantic \
        omegaconf \


    # Activate env and install pip packages
    . /opt/conda/etc/profile.d/conda.sh
    conda activate feat_env

    # Clean up to reduce image size
    conda clean -afy
    pip cache purge

%environment
    . /opt/conda/etc/profile.d/conda.sh
    conda activate feat_env
    export PATH="/opt/conda/envs/feat_env/bin:$PATH"
```

```bash
# Container build command
singularity build featsel_image.sif featsel_image.def
```

Once built, the container is ready to be used with the following command, using the season and region as arguments:

```bash
# Script execution using the singularity container
singularity exec --nv -B <PROJECT_ROOT_DIRECTORY> featsel_image.sif python3 -u testsFS_2022_refactored.py season="$season" region="$region"
```

## 2. Configuration

### 2.1 Configuration Files

This project uses Hydra for configuration management, which strictly separates the code logic from the experimental settings. You should not edit the Python code to change target regions; instead, use the configuration files or CLI overrides.

The main configuration entry point is `conf/config.yaml`.

### 2.2 Parameter Reference

| Parameter | Description | Example |
| :--- | :--- | :--- |
| `preprocessing.base_dir` | Code root directory | `path/to/project/root/HGRF-HeatWaves` |
| `preprocessing.wrkdir` | Preprocessed data output directory | `${preprocessing.base_dir}/preprocessed_data/BSC` |
| `preprocessing.srcdir_month` | Raw ERA5 data directory | `path/to/raw/era5/` |
| `preprocessing.srcdir_era5land` | Raw ERA5-Land data directory | `path/to/raw/era5land/` |
| `preprocessing.srcdir_seas5` | Raw SEAS5 data directory | `path/to/raw/seas5/` |
| `preprocessing.prepdir` | Directory of the ERA5 and ERA5-Land preprocessing code | `${preprocessing.base_dir}/preprocessing/BSC` |
| `preprocessing.co2_dir` | Directory of the NOAA CO2 preprocessing code | `${preprocessing.base_dir}/preprocessing/CO2` |
| `experiment.season` | The meteorological season to forecast. | `is_JJA` |
| `experiment.region` | The European target region to evaluate. | `is_wmed` |
| `experiment.input_path` | Absolute path to the input CSV/data file containing predictors and targets. | `/path/to/data.csv` |
| `experiment.output_dir` | Directory where the output CSVs (SHAP values, predictions, selected features) will be saved. | `/path/to/results/` |

#### CLI overrides

Hydra allows you to override any config value directly from the command line.

Run for a specific season and region:

```bash
# With the virtual environment
uv run python testsFS_2022_refactored.py season='is_DJF' region='is_scan'

# With the Singularity container
singularity exec --nv featsel_image.sif python3 -u testsFS_2022_refactored.py season='is_DJF' region='is_scan'
```

Run with a custom input path and output directory:

```bash
# With the virtual environment
uv run python testsFS_2022_refactored.py input_path="/new/path/data.csv" output_dir="./new_results"

# With the Singularity container
singularity exec --nv featsel_image.sif python3 -u testsFS_2022_refactored.py input_path="/new/path/data.csv" output_dir="./new_results"
```

## 3. Data Preparation

Download the data as described in [Data](./data.md) before continuing.

### 3.1 Preprocessing

Once downloaded and saved as monthly averages, the ERA5 and NOAA input data must be preprocessed and saved in the project's `preprocessed_data` folder. To do so:

1. Set the correct paths in the configuration file `conf/config.yaml`.
2. Run the script `launch_preprocessing_pipeline.sh`.

After the preprocessing pipeline has run, the `preprocessed_data` folder will be created with the following structure:

```bash
preprocessed_data/
|-- BSC
|   |-- benchmark
|   |-- localpred
|   |-- remotepred
|   `-- target
`-- CO2
```

### 3.2 Input table creation

With all the preprocessed files generated and stored in the `preprocessed_data` directory, execute the `make_the_table_n90_HWP.py` script in the project's `src/` folder. This Python script will generate two CSV tables:

1. **`the_table2022_hwprop.csv`**: contains the calculated Heatwave Propensity (the seasonal sum of heatwave magnitudes), based on the Russo et al. (2014) index.
2. **`the_table2022_dabv90.csv`**: contains the calculated Tx90n index, the number of days whose Tmax is above the smoothed 90th percentile.

These two tables are the inputs required by the models. Once they are correctly generated, the Feature Selection pipeline, combining the Genetic Algorithm and the Random Forest, can be executed.

## 4. Running the Model

### 4.1 Pipeline Overview

The workflow for this project follows a pipeline designed to replicate the data-driven methodology of the associated paper. The process is divided into three distinct stages:

- **Data Loading:** Retrieves the combined climate dataset (containing both the predictors and the target variable) for a specific season and region via the ClimateDataset class.
- **Feature Selection (GHGA):** Uses a Guided Hybrid Genetic Algorithm (GHGA) wrapped around a Random Forest (RF) base learner. It minimizes a cost function to select an ensemble of the best feature subsets (predictors).
- **SHAP Computing & Evaluation:** Uses tree-based SHAP explainers to compute feature importance scores across the ensemble of "good" models, outputting predictions, R-squared (R2) scores, and aggregated SHAP values to CSV files.

### 4.2 Training and Evaluation

The main script, `testsFS_2022_refactored.py`, executes the entire lifecycle end-to-end over the preprocessed input data. It uses Hydra, meaning you can modify its behavior via the command line without changing the source code (see [CLI overrides](#cli-overrides)).

### 4.4 Ensemble and Parallel Runs

Example SLURM script for running the 40 region-season experiments in parallel:

```bash
#!/bin/bash
#SBATCH --job-name featsel
#SBATCH --time=01:00:00
#SBATCH --array=0-39

SEASONS=('is_MAM' 'is_JJA' 'is_SON' 'is_DJF')
REGIONS=('is_ceur' 'is_cmed' 'is_eeur' 'is_emea' 'is_emed' 'is_scan' 'is_ukbn' 'is_wceu' 'is_wmed' 'is_wsib')

SEASON_IDX=$((SLURM_ARRAY_TASK_ID / 10))

REGION_IDX=$((SLURM_ARRAY_TASK_ID % 10))

CURRENT_SEASON=${SEASONS[$SEASON_IDX]}
CURRENT_REGION=${REGIONS[$REGION_IDX]}

echo "Running task $SLURM_ARRAY_TASK_ID with season=$CURRENT_SEASON and region=$CURRENT_REGION"

SCRIPT_PATH="<PROJECT_ROOT_DIRECTORY>/testsFS_2022_refactored.py"

export UV_LINK_MODE=copy

uv run --no-sync python3 -u "$SCRIPT_PATH" season=$CURRENT_SEASON region=$CURRENT_REGION

# Or through the container
singularity exec --nv -B <PROJECT_ROOT_DIRECTORY> <PROJECT_ROOT_DIRECTORY>/featsel_image.sif python3 -u "$SCRIPT_PATH" season=$CURRENT_SEASON region=$CURRENT_REGION

```

:::note
This will run using the default inputs defined in `conf/config.yaml`. Specific file paths must be configured for your environment (see [Configuration](#2-configuration)).
:::

## 5. Outputs

The pipeline automatically generates several CSV files in your designated `output_dir`. These files capture the ensemble predictions and feature importance:

- `[label]_FS_goodModels.csv`: contains a boolean matrix of the features selected by the top-performing models in the GHGA ensemble.
- `[label]_pred_shap_ensMean.csv`: contains the ensemble mean predictions (ypred) and the mean SHAP values for each feature across the time series.
- `[label]_pred_shap_ensStd.csv`: contains the standard deviation of the predictions and SHAP values, useful for evaluating ensemble uncertainty.
- `[label]_FI_shap_ens.csv` & `[label]_FS.csv`: summarize the overall Feature Importance (mean absolute SHAP) across the dataset, along with the out-of-bag R2 score indicating model performance.

## 6. Figures and Tables

### 6.1 Figures

To reproduce the figures from the paper, the project provides a notebook.

{/* TODO: name the notebook and list which paper figures and tables it reproduces. */}

### 6.2 Tables

{/* TODO: describe how to reproduce the paper's Table 2 (coefficients of determination). */}
