---
title: How to Use
sidebar_label: How to Use
sidebar_position: 3
---

CRAI (Climate Reconstruction AI) can be executed either via the Command Line Interface (CLI) or directly within a Python script. The workflow is divided into two main phases: **training** a model on your data, and **evaluating** (infilling) missing values using that trained model.

## 1. Environment Setup

### 1.1 Get the Code

Clone the CRAI repository and navigate to the directory:

```bash
git clone https://github.com/FREVA-CLINT/climatereconstructionAI.git
cd climatereconstructionAI
```

### 1.2 Python Environment

Ensure your environment has the required dependencies (PyTorch, xarray, dask, etc.). The easiest way to get started is by creating an Anaconda environment.

For standard use:

```bash
conda env create -f environment.yml
```

Or, if you are training on GPUs:

```bash
conda env create -f environment-cuda.yml
```

The Climate Reconstruction AI model is provided as a package, so it must also be installed with pip in order to work as intended by the authors.

```bash
conda activate <ENVIRONMENT_NAME>
pip install .
```

### 1.3 Container

A Singularity image is also provided to avoid dependency issues on specific clusters. To build it, run the following command:

```bash
singularity build /home/singularity/crai_image.sif /home/singularity/<CONTAINER_NAME>.def
```

:::note
On most HPC clusters, it is not possible to build the image directly for security reasons. In this case, it is recommended to build it locally and then transfer it to where the code will be run.
:::

### 1.4 Verify the Installation

CRAI includes a pre-configured demo to help you understand the evaluation (infilling) process. This example infills missing monthly global temperature anomalies from the HadCRUT4 dataset for two specific historical dates: **September 1877** and **August 1893**.

Before running the demo, ensure you have installed `climatereconstructionai` as detailed in [Python Environment](#12-python-environment).

**Directory structure**

Navigate to the demo folder in your repository. It contains:

- `demo_args.txt`: a text file containing all the pre-configured input arguments.
- `outputs/`: an empty directory where your final reconstructed files will be saved.
- `images/`: a directory containing pre-generated visual comparisons.
- `../data/test/`: contains the input climate dataset (`tas_hadcrut_187709_189308.nc`), which has a spatial resolution of 2.5º × 5º (lat × lon).

**Executing the demo**

Because the paths inside `demo_args.txt` are relative, you must run this command from inside the demo directory.

```bash
crai-evaluate --load-from-file demo_args.txt
```

## 2. Configuration

### 2.1 Configuration Files

To run the package, you must execute its built-in commands, `crai-train` and `crai-evaluate`, followed by the required arguments for each command, which are listed and described in the README.md found in the CRAI repository. The arguments are passed through one text file per pipeline (training and evaluation); examples for each are given in [Running the Model](#4-running-the-model).

### 2.2 Parameter Reference

The CLI arguments are broken down into logical categories below.

#### File paths

| Flag | Description |
|---|---|
| `--data-root-dir` | The main folder containing your `data/`, `val/`, and `test/` folders. |
| `--mask-dir` | Directory containing your explicit mask datasets. |
| `--data-names` / `--mask-names` | Comma-separated lists of your NetCDF filenames. |
| `--log-dir` / `--snapshot-dir` | Where TensorBoard logs and intermediate snapshot images are saved during training. |
| `--evaluation-dirs` | *(Evaluation only)* Where the final infilled NetCDF files will be saved. |

#### Hardware and performance

| Flag | Description |
|---|---|
| `--device` | Choose between `cuda` (GPU) or `cpu`. |
| `--multi-gpus` | Distribute training across multiple available GPUs. |
| `--batch-size` | Number of samples processed before the model updates (adjust based on your GPU memory). |
| `--n-threads` | Number of CPU workers for loading data. |
| `--lazy-load` | Crucial for massive datasets; loads data into memory only when needed rather than all at once. |

#### Model architecture

| Flag | Description |
|---|---|
| `--encoding-layers` / `--pooling-layers` | Defines the depth of the neural network. |
| `--conv-factor` | Sets the number of channels in the deepest layer of the network. |
| `--attention` | Enables the attention module, helping the model focus on specific spatial features. |
| `--disable-skip-layers` | Removes skip connections in the U-Net (usually not recommended, but available for testing). |

#### Training hyperparameters

| Flag | Description |
|---|---|
| `--max-iter` | The maximum number of training steps. |
| `--lr` | The learning rate (how aggressively the model updates its weights). |
| `--loss-criterion` | Choose the mathematical function used to calculate errors (e.g., Mean Absolute Error of the hole region). |
| `--early-stopping-patience` | Stops training automatically if the validation loss hasn't improved after this many checks, preventing overfitting. |
| `--normalize-data` | Normalizes your input climate data to a mean of 0 and standard deviation of 1 before passing it to the network. |

#### Evaluation and output

| Flag | Description |
|---|---|
| `--model-dir` / `--model-names` | *(Evaluation only)* Points the software to the specific trained models you want to use for infilling. |
| `--min-bounds` / `--max-bounds` | Forces the AI's output to stay within realistic physical limits (e.g., setting a max bound of 100 for a percentage index). |
| `--partitions` / `--maxmem` | *(Evaluation only)* If your dataset is too large to evaluate at once, these options split the data along the time coordinate to prevent out-of-memory crashes. |
| `--plot-results` | Automatically generates image plots of the reconstructed time indices for quick visual inspection. |

## 3. Data Preparation

Download and preprocess the data as described in [Data](./data.md) before continuing.

### 3.1 NetCDF files

CRAI expects your NetCDF (`.nc`) climate datasets to be referenced as a list in JSON files, stored in specific sub-directories depending on which phase you are running:

- `${ROOT}/data/train/train_<IDX>.json` and `${ROOT}/data/val/val_<IDX>.json`: used during **training**.
- `${ROOT}/data/test/test_<IDX>.json`: used during **evaluation**.

:::note
`--data-root-dir` is only used to locate the JSON files (`<data-root-dir>/train/<data-names>`); the file paths inside the JSON files are used as written. One JSON file per index is required, because `--data-types` applies to all the files listed in a JSON file.
:::

**Version 1.0.4:** The data split is made when the JSON lists are created, based on the total number of models and consequently the total number of monthly samples. As stated in the paper, *37 models (50616 monthly samples) are used for training, 7 models (9576 monthly samples) for validation and 1 model (1368 monthly samples)* for testing, giving a total of *45 models and 61560 monthly samples*.

**Version 1.0.3:** In this version, only one file can be referenced for each of the training, validation and testing steps. A CDO pipeline must therefore be executed to merge the `.nc` files and generate one single file for each step.

:::note
In this reproduction, the random sampling was done member-wise, filling each of the training, validation and testing lists with the file paths.
:::

:::warning
The paper results were obtained using version 1.0.3 of the CRAI package, which is now at version 1.0.4, so mind the setup differences between versions and possible discrepancies in the output values.
:::

### 3.2 Mask files

Mask files are reference `.nc` files that provide the missing-data structure (`1` for valid data, `0` for missing data) used to simulate the original HadEX files in the training and evaluation pipelines. They must match the dimensions of your climate datasets. If you don't provide explicit mask files, CRAI can automatically extract them from the NaN values in your climate dataset. How to build them is described in [Data](./data.md).

The mask files can be stored in a mask directory inside the project root, as in the example below, and referenced in the execution arguments with `--mask-dir` and `--mask-names`.

- `${ROOT}/data/mask/`

## 4. Running the Model

### 4.2 Training

To run the training, type the following command:

```bash
crai-train -f arguments/training_arguments_TN10p.txt

# or, with the singularity image:
singularity exec --nv <CONTAINER_NAME>.sif crai-train -f <PROJECT_ROOT_PATH>/arguments/training_arguments_TN10p.txt
```

```python
from climatereconstructionai import train
train()
```

The model must be trained separately for each index (TN10p, TN90p, TX10p and TX90p), so the arguments file must be adapted to each index. To achieve the same training setup as the paper *for the TN10p index*, type the following arguments inside the arguments file:

```bash
--data-root-dir
<PROJECT_ROOT_PATH>/data/
--data-names
train_TN10p.json
--val-names
val_TN10p.json
--data-types
TN10p
--log-dir
<PROJECT_ROOT_PATH>/logs/TN10p/
--snapshot-dir
<PROJECT_ROOT_PATH>/snapshots/TN10p/
--batch-size
16
--n-threads
32
--max-iter
1000000
--log-interval
100
--save-model-interval
100000
--lr
5e-5
--mask-dir
<INPUT_DATA_PATH>/masks/
--mask-names
mask_TN10p.nc
--shuffle-masks
--conv-factor
128
--loss-criterion
1
--min-bounds
0
--max-bounds
100
--encoding-layers
3
--pooling-layers
0
--normalize-data
--loop-random-seed
2000
--cuda-random-seed
1000
--steady-masks
steady_sea_TN10p.nc
```

:::note
This is an example with the required arguments and values for TN10p. Remember to update it to your folder paths, particularities and data.
:::

### 4.3 Evaluation

The evaluation pipeline uses each checkpoint stored in the folder given by `--model-dir` and listed in `--model-names` as an ensemble, generating an evaluation output for each of the array runs. You can follow the example below for an evaluation arguments file.

The flag `--plot-results` determines which timesteps are plotted to show the infilling results. Feel free to change it as you prefer, keeping in mind that 0 is the first timestep (the first month of the period) and the last one is the last month of the period. The total will vary depending on the number of timesteps in your data, which can be verified with the CDO command `cdo sinfon <file_name>.nc`.

```bash
--data-root-dir
<PROJECT_ROOT_PATH>/data/
--data-names
test_TN10p.json
--data-types
TN10p
--model-dir
<PROJECT_ROOT_PATH>/snapshots/TN10p
--model-names
run_01/ckpt/final.pth,run_02/ckpt/final.pth,run_03/ckpt/final.pth,run_04/ckpt/final.pth,run_05/ckpt/final.pth,run_06/ckpt/final.pth,run_07/ckpt/final.pth,run_08/ckpt/final.pth,run_09/ckpt/final.pth,run_10/ckpt/final.pth,run_11/ckpt/final.pth,run_12/ckpt/final.pth,run_13/ckpt/final.pth,run_14/ckpt/final.pth,run_15/ckpt/final.pth,run_16/ckpt/final.pth,run_17/ckpt/final.pth,run_18/ckpt/final.pth,run_19/ckpt/final.pth,run_20/ckpt/final.pth
--evaluation-dirs
outputs/TN10p/array/
--eval-names
TN10p_20_eval
--device
cuda
--log-dir
<PROJECT_ROOT_PATH>/logs/TN10p/eval/
--use-train-stats
--batch-size
16
--n-threads
32
--plot-results 0,200,400,600,800,1000,1200
--mask-dir
<INPUT_DATA_PATH>/masks/
--mask-names
croped_mask_TN10p.nc
--conv-factor
128
--encoding-layers
3
--pooling-layers
0
--normalize-data
--min-bounds
0
--max-bounds
100
--steady-masks
steady_sea_TN10p.nc
```

Once the arguments are set, the evaluation pipeline can be executed with the command:

```bash
crai-evaluate -f <PROJECT_ROOT_PATH>/arguments/evaluation_arguments_TN10p.txt

# or, with the singularity image:
singularity exec --nv <CONTAINER_NAME>.sif crai-evaluate -f <PROJECT_ROOT_PATH>/arguments/evaluation_arguments_TN10p.txt
```

:::note
The evaluation arguments differ from the training arguments in some aspects, so it is good practice to have one file per pipeline. For more configuration flags, check the [Parameter Reference](#22-parameter-reference).
:::

:::warning
When a JSON file lists several `.nc` files, `evaluation.py` (which starts from `ds_index=0`) overruns the time dimension of the first file. Use JSON files with a single file each for evaluation, and run one evaluation per file.
:::

### 4.4 Ensemble and Parallel Runs

The paper states that the results are the average of 20 different model trainings, which means that the training pipeline must be run 20 times and generate 20 *"final checkpoints"* (`final.pth`). To run an array of jobs with different `--loop-random-seed` and `--cuda-random-seed` values and save the outputs in different folders, for the TN10p index for example, the following script can be used:

```bash
#!/bin/bash
#SBATCH --job-name=crai_train_TN10p
#SBATCH --output=logs/slurm/process_%A_%a.out
#SBATCH --error=logs/slurm/process_%A_%a.err
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=32      # Fixes the DataLoader warning
#SBATCH --time=24:00:00
#SBATCH --account=
#SBATCH --partition=            # Puts you in the GPU nodes
#SBATCH --qos=
#SBATCH --gres=gpu:1
#SBATCH --array=1-20%10

BASE=<PROJECT_ROOT_PATH>
ARGS=$BASE/arguments/training_arguments_TN10p.txt
SIF=$BASE/<CONTAINER_NAME>.sif
IDX=TN10p

i=$SLURM_ARRAY_TASK_ID
n=$(printf '%02d' "$i")

singularity exec --nv "$SIF" crai-train  -f "$ARGS" \
	 --loop-random-seed $((1000 + i)) \
	 --cuda-random-seed $((2000 + i)) \
	 --snapshot-dir "$BASE/snapshots/$IDX/run_$n/" \
	 --log-dir "$BASE/logs/$IDX/run_$n/"

```

:::note
Adjust the script to your folder paths and particularities.
:::

Once the array job has finished and each training run has written its final checkpoint, you can move on to the evaluation pipeline.

## 5. Outputs

Once the evaluation is complete, CRAI will generate 5 NetCDF (`.nc`) files and the requested number of PNG images inside the `outputs/` folder. The output files and their content are listed below:

1. `<output_name>_gt.nc`: NetCDF file corresponding to the original dataset (gt = ground truth).
2. `<output_name>_mask.nc`: mask corresponding to the missing values.
3. `<output_name>_image.nc`: original dataset with the mask of missing values applied.
4. `<output_name>_output.nc`: dataset where both valid and missing values are infilled.
5. `<output_name>_infilled.nc`: original dataset with the missing values replaced by the values from `<output_name>_output.nc`.
6. `<output_name>_infilled.png`: plot of the first timestep of `<output_name>_infilled.nc`.

## 6. Figures and Tables

The paper presents its results in 2 tables and 7 figures, which show the following:

- **Table 2:** Evaluation of the reconstruction methods using the test dataset (CMIP6 data).
- **Table 3:** Evaluation of the reconstruction methods using the ERA5 dataset.
- **Figure 1:** Comparison of the temporal and spatial root mean square error (RMSE) between CRAI and Kriging for the reconstruction of ERA5.
- **Figure 2:** Comparison of the regional means of TX90p and TN10p.
- **Figure 3:** TX90p hot extreme event from November 1947.
- **Figure 4:** Linear trends (in days/10 years) of TX90p for the period 1901–2018.
- **Figure 5:** Linear trends (in days/10 years) of TN10p for the period 1901–2018.
- **Figure 6:** TX90p for a reported heatwave event (September 1911).
- **Figure 7:** TN10p for a reported coldwave event (February 1929).

### 6.1 Figures

To generate all the plots, the following must be referenced in the plots section of the configuration file:

- `tx90p_era5_gt_dir`: ERA5 ground truth output *TX90p_20_eval_era5_gt.nc*, generated by the model evaluation with ERA5 input for the TX90p index.
- `tn10p_era5_gt_dir`: ERA5 ground truth output *TN10p_20_eval_era5_gt.nc*, generated by the model evaluation with ERA5 input for the TN10p index.
- `tx90p_cam_dir`: The paper's Zenodo HadEX-CAM reference for TX90p, `original_TX90p_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc`.
- `tn10p_cam_dir`: The paper's Zenodo HadEX-CAM reference for TN10p, `original_TN10p_mon-eu-32x32_hadexcam_infilled_all_1901-2018.nc`.
- `tx90p_hadex3_dir`: HadEX3 original data obtained from the [CEDA archive](https://data.ceda.ac.uk/badc/ukmo-hadobs/data/derived/MOHC/HadOBS/HadEX3/v3-0-4/), *HadEX3-0-4_tx90p_mon_1981-2010.nc*, regridded to 32x32 with CDO.
- `tn10p_hadex3_dir`: HadEX3 original data obtained from the [CEDA archive](https://data.ceda.ac.uk/badc/ukmo-hadobs/data/derived/MOHC/HadOBS/HadEX3/v3-0-4/), *HadEX3-0-4_tn10p_mon_1981-2010.nc*, regridded to 32x32 with CDO.
- `tx90p_crai_dir`: HadEX-CAM infilled output *TX90p_20_eval_HadEX_infilled.nc*, generated by the model evaluation with HadEX-CAM input for the TX90p index.
- `tn10p_crai_dir`: HadEX-CAM infilled output *TN10p_20_eval_HadEX_infilled.nc*, generated by the model evaluation with HadEX-CAM input for the TN10p index.


:::note
Set the correct configuration paths for your local system.
:::

Once the configuration paths are set, run `paper_plots.py` to plot all the figures with the same formatting as the paper.

### 6.2 Tables

The paper's evaluation relies on two main statistics: the Root Mean Squared Error (RMSE) between the predicted value and the ground truth, and the Spearman Rank-Order Correlation Coefficient (SROCC). They are calculated with the following functions:

:::note
These statistics are not included in the public CRAI repository (`evaluation.py` only produces the `.nc` output files), so they are computed separately from the evaluation outputs.
:::

#### RMSE

```python
def RMSE_calculation(ds_output, ds_truth, ds_mask, land):
    mask = (ds_mask == 0)
    mask_eval = mask & land            # land broadcasts over time

    RMSE = np.sqrt(np.mean((ds_output[mask_eval] - ds_truth[mask_eval])**2))

    return RMSE
```

Where:

**`ds_output`:** The model's `<output_name>_output.nc` file, listed in [Outputs](#5-outputs).

**`ds_truth`:** The `<output_name>_gt.nc` file, listed in [Outputs](#5-outputs).

**`ds_mask`:** The `<output_name>_mask.nc` file, listed in [Outputs](#5-outputs).

**`land`:** The reference grid file that distinguishes land from sea, so that no sea cell is evaluated.

#### SROCC

```python
def SROCC_calculation(ds_output, ds_truth, ds_mask, land):
    mask = (ds_mask == 0)
    mask_eval = mask & land            # land broadcasts over time
    
    # Per-month
    monthly_rhos = [stats.spearmanr(ds_output[t][mask_eval[t]], ds_truth[t][mask_eval[t]]).statistic
                    for t in range(ds_truth.shape[0]) if mask_eval[t].sum() > 2]
    
    # Pooled method for SROCC --> single Spearman over all reconstructed values pooled across space and time
    pooled_rhos = stats.spearmanr(ds_output[mask_eval], ds_truth[mask_eval]).statistic

    return monthly_rhos, pooled_rhos
```

Where:

**`ds_output`:** The model's `<output_name>_output.nc` file, listed in [Outputs](#5-outputs).

**`ds_truth`:** The `<output_name>_gt.nc` file, listed in [Outputs](#5-outputs).

**`ds_mask`:** The `<output_name>_mask.nc` file, listed in [Outputs](#5-outputs).

**`land`:** The reference grid file that distinguishes land from sea, so that no sea cell is evaluated.
