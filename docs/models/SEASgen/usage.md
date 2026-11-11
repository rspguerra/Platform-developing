---
title: How to Use
sidebar_label: How to Use
sidebar_position: 3
---

## 1. Environment Setup

### 1.1 Get the Code

The project repository is hosted on GitLab at [this link](https://gitlab.earth.bsc.es/es/seasgen). To get started, clone the project and navigate into the directory:

```bash
git clone https://gitlab.earth.bsc.es/es/seasgen.git
cd seasgen
```

### 1.2 Python Environment

You can set up your environment using `uv` (recommended for faster dependency resolution) or standard `pip`.

**Option A: Using `uv` (recommended)**

Execute the following commands in your terminal:

```bash
# Install uv (if not already installed)
command -v uv &> /dev/null || curl -LsSf https://astral.sh/uv/install.sh | sh

# Create a .venv local virtual environment (if it doesn't exist)
[ -d ".venv" ] || uv venv

# Install the repository dependencies
uv sync

# Activate the virtual environment
source .venv/bin/activate
```

**Option B: Using `pip`**

```bash
pip install .[all]
```

### 1.3 Container

The container is built with Singularity from the `seasgen_env.def` file below. It installs exactly the package versions recorded in the repository's `uv.lock` file (e.g. PyTorch 2.9.1 with CUDA 12.8, NumPy 2.4.0, xarray 2025.12.0, zarr 3.1.5), on Python 3.12. The `seasgen` package is installed in editable mode, so the image always runs the code and configuration files of the repository folder you bind to it, and it only needs to be rebuilt when the dependencies change.

```bash
# seasgen_env.def
Bootstrap: docker
From: python:3.12-slim

%files
    . /opt/seasgen

%post
    set -e

    # git is needed by uv to fetch the tinysunset dependency (pinned git commit)
    apt-get update
    apt-get install -y --no-install-recommends git ca-certificates
    rm -rf /var/lib/apt/lists/*

    # uv, using the image's Python only (no Python downloads at build time)
    pip install --no-cache-dir uv
    export UV_PYTHON_DOWNLOADS=never
    export UV_PROJECT_ENVIRONMENT=/opt/venv
    export UV_LINK_MODE=copy

    # Optional: if BSC's GitLab is not reachable from the build machine, clone
    # tinysunset into the seasgen folder beforehand and uncomment the next line:
    # git config --global url."file:///opt/seasgen/tinysunset".insteadOf "https://earth.bsc.es/gitlab/lpalma/tinysunset.git"

    # Install exactly the versions recorded in uv.lock (without the dev tools)
    cd /opt/seasgen
    uv sync --frozen --no-dev --python /usr/local/bin/python3.12

    # Clean up to reduce image size
    uv cache clean
    rm -rf /root/.cache

%environment
    export PATH=/opt/venv/bin:$PATH
    export VIRTUAL_ENV=/opt/venv

%runscript
    cd /opt/seasgen
    exec python run_experiment.py "$@"

%test
    python -c "import torch, xarray, zarr, tinysunset, seasgen; print('torch', torch.__version__, '| CUDA build:', torch.version.cuda, '| GPU visible:', torch.cuda.is_available())"
```

To build the container run the command bellow on singularity or apptainer.

```bash
# Singularity build
singularity build seasgen_env.sif seasgen_env.def

# Apptainer build
apptainer build seasgen_env.sif seasgen_env.def
```


## 2. Configuration

### 2.1 Configuration Files

Experiment parameters, datasets, and model architectures are controlled via YAML configuration files (e.g., `configs/your_experiment.yml`). A specific file, `paths.yml`, is dedicated to configuring the different paths required, so to replicate an experiment you only need to modify this file.

#### Variables to modify

* **`global_cmip6_inputs_conf` / `targets_conf`:** Absolute paths to the CMIP6 training data (e.g., `.../inputs.zarr`).
* **`global_era5_inputs_conf` / `targets_conf`:** Absolute paths to the ERA5 observational data for testing/validation.
* **`outputs_dir`:** The directory where your trained models, logs, and plots will be saved.
* **`preprocessed_data_dir`:** The directory used for staging temporary or intermediate dataset files.

### 2.2 Parameter Reference

Beyond the file paths, the YAML experiment files dictate the exact nature of the study:

* **Datasets (`dataset_cmip6_train`, `dataset_era5`, etc.):** Define the variables used (`tas`, `pr`, `tos`), spatial domains (latitude/longitude bounds), the specific months targeted, and which climate model members are included.
* **Model (`model`):** Defines the neural network architecture (e.g., `vitcVAE`), embedding dimensions, input shapes, and residual blocks.
* **Trainer (`trainer`):** Sets hyperparameters including `batch_size`, learning rate (`lr`), `epochs`, loss terms, and dropout scheduling.
* **Verification (`verification`):** Specifies the statistical metrics computed against the ERA5 baseline during the testing phase.

## 3. Data Preparation

`seasgen` relies on preprocessed climate data saved as Zarr stores. The [Data section](./data.md) has a guide on how to obtain these datasets using ESMValTool. You must link the output directories of your ESMValTool pipelines to the path variables in the `paths.yml` file.

## 4. Running the Model

### 4.1 Pipeline Overview

A single run of `run_experiment.py` executes the whole lifecycle, following the methodology of the associated paper:

- **Data Loading:** Reads the preprocessed CMIP6 (training) and ERA5 (testing/validation) Zarr stores defined in `paths.yml`, for the variables, spatial domain and months set in the experiment YAML file.
- **Signal Decomposition:** A locally estimated scatterplot smoothing (LOESS) trend is regressed to isolate the long-term anthropogenic forcing, so that the model is explicitly tasked with predicting only the detrended, interannual fluctuations.
- **Training:** The conditional Variational Autoencoder (two Vision Transformer encoders and a CNN decoder) is trained on the CMIP6 data with the hyperparameters set in the `trainer` block.
- **Testing:** The trained model generates an ensemble of `forecast_members=125` probabilistic hindcasts for each target year, and the final forecast is assembled by recombining the cVAE's interannual prediction with the deterministic LOESS trend.
- **Verification:** The hindcasts are compared against the ERA5 observations with the metrics set in the `verification` block, over the verification periods 1985–2021 and 2001–2021, with and without detrending.


### 4.2 Training and Evaluation

Training and validation are executed using the `run_experiment.py` script.

**Single GPU / CPU:**

By default, experiments run on a single processing unit.

```bash
python run_experiment.py --conf_file=configs/YOUR_YAML_FILE.yml --nprocs=1
```

**Multi-GPU (distributed training):**

For distributed training, set the `--nprocs` parameter to the number of GPUs available on your single HPC node.

```bash
python run_experiment.py --conf_file=configs/YOUR_YAML_FILE.yml --nprocs=4
```

### 4.5 Computational Resources

| Metric | Usage |
| :--- | :--- |
| **Status** | `COMPLETED` |
| **Time (Wallclock)** | 06:36:03 |
| **Memory (Peak RAM)** | ~56.4 GB |
| **CPU Processing Time** | 07:13:51 |
| **Hardware** | 1x GPU |

## 5. Outputs

Once the script completes the training and testing phases, `seasgen` automatically generates the evaluation outputs. All results are saved in the `outputs_dir` specified in your YAML file.

Inside your output directory, you will find:

* **Model Checkpoints:** Saved `.pt` files of your trained models (e.g., `best_model.pt`, `final_model.pt`).
* **Loss Curves:** PNG images (e.g., `loss_curves_0.png`) showing the training and validation loss progression over the epochs.
* **Forecast Outputs:** `.nc` (NetCDF) files containing the generated hindcasts (`hcst`) alongside the matching observations (`obs`).
* **Verification Maps:** Geographic map plots displaying your specified verification metrics (like correlation or ROC scores) evaluated over the test periods.

All outputs are written under the experiment directory, whose path is defined by `conf.exp_dir` in `paths.yml`. A typical run produces the following structure:

```
exps/
└── <experiment_name>-<timestamp>/
    ├── train_loss_0.pkl
    ├── val_loss_0.pkl
    ├── loss_curves_0.png
    ├── loss_curves_by_term_0.png
    ├── models/
    │   └── <model_name>_0
    ├── output/
    │   ├── <model_name>_<var>_<dataset>_hcst_fcst.nc
    │   └── <model_name>_<var>_<dataset>_obs_fcst.nc
    └── imgs/
        └── <model_name>/
            └── metrics_<var>_<years>_<detrend>_fcst.png
```

#### `train_loss_0.pkl` / `val_loss_0.pkl`

Serialised Python lists containing the training and validation loss values at each epoch. Load them with:

```python
import pickle
with open("train_loss_0.pkl", "rb") as f:
    train_loss = pickle.load(f)  # list of floats, one per epoch
```

#### `loss_curves_0.png`

Plot of training and validation loss over epochs.

#### `loss_curves_by_term_0.png`

Same as above, but broken down by individual loss terms.

#### `models/<model_name>_0`

Saved PyTorch model state dict. This is the checkpoint loaded during the test phase. The filename suffix `_0` corresponds to the training run index.

#### `output/<model_name>_<var>_<dataset>_hcst_fcst.nc`

NetCDF file containing the model's hindcast predictions. Dimensions are `(lon, lat, time, ensemble)`: the model generates an ensemble of `forecast_members=125` probabilistic predictions for each target year.

One file is produced per output variable (e.g. `tas`, `pr`, `tos`, `zg500`, `zg300`).

#### `output/<model_name>_<var>_<dataset>_obs_fcst.nc`

NetCDF file containing the corresponding ERA5 observations, inverse-transformed to physical units. Used as the verification reference. Written only once per variable (subsequent runs with the same dataset skip writing if the file already exists).

## 6. Figures and Tables

### 6.1 Figures

#### `imgs/<model_name>/metrics_<var>_<years>_<detrend>_fcst.png`

Verification metric plots, generated automatically at the end of each run. One image is produced per combination of:

- **variable** (`tas`, `pr`, etc.)
- **verification period** (`1985-2021` or `2001-2021`)
- **detrending** (`detrend` or `no_detrend`)

### 6.2 Tables

{/* TODO: describe how to reproduce the paper's tables, if any. */}
