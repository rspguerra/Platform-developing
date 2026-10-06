---
title: How to use
---


## 1. Installation

Considering that the main goal of this guide is to make the pipeline easily and fully reproducible, it will focus on container execution, once it is achievable both in HPC and local execution scenarios.

### Setup Repository
The project repository is available in github at this link. To get started, clone the project and navigate to the directory:
```
git clone repository_url
cd repository_name
```

### 1.1 Dependencies list

The dependencies required to execute this project considering good packages compactibilities are:

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
| pydantic |2.12.5 |
| setuptools | 67.8.0 |


### 1.2 Environment Setup

This project uses uv for dependency management.
1. **Install uv** (if not installed):
```
curl -LsSf https://astral.sh/uv/install.sh | sh
```
_Note: Restart your shell to ensure uv is in your PATH._

2. **Create a Local Virtual Environment:**
```
[ -d ".venv" ] || uv venv
```
3. **Init and sync Environment:** 
```
uv init
uv sync
```
4. **Activate venv**
```
source .venv/bin/activate
```

### 1.3 Singularity Container setup

Container build with singularity, from a *.def* file available within the repository files. For the container scenario, the environment was installed using miniconda3: 24.1.2-0.


```bash
# .Def file
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
$ singularity build featsel_image.def 
```

Once built the container is ready to be used by the command, using as arguments season and region:

```bash
# Scrip execution using singularity container
singularity exec --nv -B featsel_image.sif python3 -u testsFS_wfwd_FIXED_FEATURES_refactored.py -s "$season" -r "$region"
```

---


## 2. Configuration
This project uses Hydra for configuration management. This strictly separates the code logic from experimental settings. You should not edit the Python code to change target regions; instead, use the configuration files or CLI overrides.

The main configuration entry point is conf/config.yaml.

### 2.1 Configuration Parameters

|Parameter |Description |	Example|
| :--- | :--- | :--- |
|`season`|	The meteorological season to forecast.	|`is_JJA`|
|`region`	|The European target region to evaluate.	|`is_wmed`|
|`input_path` |	Absolute path to the input CSV/data file containing predictors and targets.	|`/path/to/data.csv`|
|`output_dir` |	Directory where the output CSVs (SHAP values, predictions, selected features) will be saved.|`/path/to/results/`|


### How to use (CLI Overrides)

Hydra allows you to override any config value directly from the command line.

Run for a specific season and region:

```
uv run python testsFS_2022_refactored.py season='is_DJF' region='is_scan'
```

Run with a custom input path and output directory:

```
uv run python testsFS_2022_refactored.py input_path="/new/path/data.csv" output_dir="./new_results"
```

---

## 3. Feature Selection, Prediction & SHAP computing

The workflow for this project follows a pipeline designed to replicate the data-driven methodology from the associated paper. The process is divided into three distinct stages:

- **Data Loading:** Retrieves the combined climate dataset (containing both predictors and the target variable) for a specific season and region via the ClimateDataset class.

- **Feature Selection (GHGA):** Uses a Guided Hybrid Genetic Algorithm (GHGA) wrapped around a Random Forest (RF) base learner. It minimizes a cost function to select an ensemble of the best feature subsets (predictors).

- **SHAP Computing & Evaluation:** Uses tree-based SHAP explainers to compute feature importance scores across the ensemble of "good" models, outputting predictions, R-squared (R2) scores, and aggregated SHAP values to CSV files.

Full Pipeline Script (testsFS_2022_refactored.py)

This main script executes the entire lifecycle end-to-end, over the preprocessed input data. It utilizes Hydra, meaning you can modify its behavior via the command line without changing the source code.

Example SLURM Script for parallelization:
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

SCRIPT_PATH="<PROJECT_ROOT_DIRECTORY>/main_script.py"

export UV_LINK_MODE=copy

uv run --no-sync python3 -u <PROJECT_ROOT_DIRECTORY>/testsFS_2022_refactored.py season=$CURRENT_SEASON region=$CURRENT_REGION

# Or through the container
singularity exec --nv -B <PROJECT_ROOT_DIRECTORY>/featsel_image.sif python3 -u <PROJECT_ROOT_DIRECTORY>/testsFS_2022_refactored.py season=$CURRENT_SEASON region=$CURRENT_REGION

```
>Note: This will run using the default inputs defined in conf/config.yaml. Specific file paths must be configured for your environment (see below).

---

## 4. Outputs & Visualization

The pipeline automatically generates several CSV files in your designated output_dir. These files capture the ensemble predictions and feature importance:

    - `[label]_FS_goodModels.csv`: Contains a boolean matrix of the features selected by the top-performing models in the GHGA ensemble.

    - `[label]_pred_shap_ensMean.csv`: Contains the ensemble mean predictions (ypred) and the mean SHAP values for each feature across the time series.

    - `[label]_pred_shap_ensStd.csv`: Contains the standard deviation of the predictions and SHAP values, useful for evaluating ensemble uncertainty.

    - `[label]_FI_shap_ens.csv & [label]_FS.csv`: Summarizes the overall Feature Importance (Mean Absolute SHAP) across the dataset, along with the out-of-bag R2 score indicating model performance.

### Analysis Notebooks

To reproduce the figures from the paper, the project provides a notebook.
