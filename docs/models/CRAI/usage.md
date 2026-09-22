---
title: How to use
---

CRAI (Climate Reconstruction AI) can be executed either via the Command Line Interface (CLI) or directly within a Python script. The workflow is divided into two main phases: **training** a model on your data, and **evaluating** (infilling) missing values using that trained model.

## 1. Installation and setup

Ensure your environment has the required dependencies (PyTorch, xarray, dask, etc.). The easiest way to get started is by creating an Anaconda environment.

For standard use:
```bash
conda env create -f environment.yml
```

Or, if you are training on GPUs:
```bash
conda env create -f environment-cuda.yml

```

A singularity image is also provided for avoiding dependency issues with specific clusters. To mount it and execute it, run the following commands:

```bash
singularity build /home/singularity/crai_image.sif /home/singularity/<CONTAINER_NAME>.def
```
>**Note**. In most HPC clusters, for security reasons it is not possible to mount the image directly there. In this case, it is recommended to mount it locally, and then transfer it to where the code will be ran.

The Climate Reconstruction AI model is presented as a package, so it must also be installed with pip in order to work as intended by the authors.

```bash
conda activate <ENVIRONMENT_NAME>
pip install .
```

----

## 2. Data preparation

CRAI expects your NetCDF (`.nc`) climate datasets to be referenced as a list in a json files stored in specific sub-directories based on which phase you are running:

- `${ROOT}/data/train/training_data.json` and `${ROOT}/data/val/val_data.json` — used during **training**.
- `${ROOT}/data/test/test_data.json` — used during **evaluation**.

**Masks:** missing values are defined by mask files (`1` for valid data, `0` for missing data). These must match the dimensions of your climate datasets. If you don't provide explicit mask files, CRAI can automatically extract them from the NaN values in your climate dataset.

The data split is made in the moment that the json lists are created based on the total number of models and consequently the total number of monthly samples, as stated in the paper *37 models data for training (50616 monthly samples), 7 (9576 monthly samples) models for validation and 1 (1368 monthly samples)* for testing, giving us the total of *45 models and 61560 monthly samples*.

> **Note:** In the reproduction made the random sampling was made memberwise filing the each one of the training, validation and testing lists with the files path. 

---

## 3. Running the software

In order to run the package you must execute its built-in commands crai-train, and crai-evaluate followed by the required arguments for each command, which are listed and described in the README.md found in the CRAI repository. 

### 3.1 Training:

To execute the training type the following command:

```bash
crai-train -f arguments/arguments.txt

# or, if the singularity image was used:
singularity exec --nv <CONTAINER_NAME>.sif crai-train -f arguments/arguments.txt
```
```python
from climatereconstructionai import train
train()
```
The model must be separetely trained for each index (TN10p, TN90p, TX10p and TX90p) so the arguments.txt file must be adapted to each index. To achieve the same training setup as the papper *for the TN10p index*, type the following arguments inside the arguments.txt file:

```bash
--data-root-dir
/path/to/project/root/data/
--data-names
train_TN10p.json
--val-names
val_TN10p.json
--data-types
TN10p
--log-dir
/path/to/project/root/logs/TN10p/
--snapshot-dir
/path/to/project/root/snapshots/TN10p_1/
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
/path/to/datasets/folder/masks/
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
```
>**Note:** This is an example with the required arguments and values, remember to update to your folder paths and particularities.

The paper states that the results obtained were the averaged result of 20 different model trainings, which means that the training pipeline must be ran 20 times and generate 20 *"best checkpoints"*. To run an array of jobs with different and randomized `--loop-random-seed` and `--cuda-random-seed` and save the outputs in different folders for the index TN10p for example, the following script can be used:

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

BASE=/path/to/project/root/
ARGS=$BASE/arguments/train_args_TN10p.txt
SIF=/path/to/project/root/<CONTAINER_NAME>.sif
IDX=TN10p

i=$SLURM_ARRAY_TASK_ID
n=$(printf '%02d' "$i")

singularity exec --nv "$SIF" crai-train  -f "$ARGS" \
	 --loop-random-seed $((1000 + i)) \
	 --cuda-random-seed $((2000 + i)) \
	 --snapshot-dir "$BASE/snapshots/$IDX/run_$n/" \
	 --log-dir "$BASE/logs/$IDX/run_$n/"

```
>**Note:** Adjust the code to your folder paths and particularities.


### 3.2 Evaluation (infilling):
```bash
crai-evaluate -f arguments/arguments.txt
# or, if the singularity image was used:
singularity exec --nv <CONTAINER_NAME>.sif crai-evaluate -f arguments/arguments_TN10p.txt
```
```python
from climatereconstructionai import evaluate
evaluate()
```

Because CRAI has many configuration options, typing them all in the terminal can be tedious. You can save all your parameters in a plain text file and load them at runtime using the `-f` / `--load-from-file` flag:

```bash
crai-train -f my_config.txt
```

---

## 4. Configuration guide

The CLI arguments are broken down into logical categories below.

### File paths

| Flag | Description |
|---|---|
| `--data-root-dir` | The main folder containing your `data/`, `val/`, and `test/` folders. |
| `--mask-dir` | Directory containing your explicit mask datasets. |
| `--data-names` / `--mask-names` | Comma-separated lists of your NetCDF filenames. |
| `--log-dir` / `--snapshot-dir` | Where TensorBoard logs and intermediate snapshot images are saved during training. |
| `--evaluation-dirs` | *(Evaluation only)* Where the final infilled NetCDF files will be saved. |

### Hardware and performance

| Flag | Description |
|---|---|
| `--device` | Choose between `cuda` (GPU) or `cpu`. |
| `--multi-gpus` | Distribute training across multiple available GPUs. |
| `--batch-size` | Number of samples processed before the model updates (adjust based on your GPU memory). |
| `--n-threads` | Number of CPU workers for loading data. |
| `--lazy-load` | Crucial for massive datasets; loads data into memory only when needed rather than all at once. |

### Model architecture

| Flag | Description |
|---|---|
| `--encoding-layers` / `--pooling-layers` | Defines the depth of the neural network. |
| `--conv-factor` | Sets the number of channels in the deepest layer of the network. |
| `--attention` | Enables the attention module, helping the model focus on specific spatial features. |
| `--disable-skip-layers` | Removes skip connections in the U-Net (usually not recommended, but available for testing). |

### Training hyperparameters

| Flag | Description |
|---|---|
| `--max-iter` | The maximum number of training steps. |
| `--lr` | The learning rate (how aggressively the model updates its weights). |
| `--loss-criterion` | Choose the mathematical function used to calculate errors (e.g., Mean Absolute Error of the hole region). |
| `--early-stopping-patience` | Stops training automatically if the validation loss hasn't improved after this many checks, preventing overfitting. |
| `--normalize-data` | Normalizes your input climate data to a mean of 0 and standard deviation of 1 before passing it to the network. |

### Evaluation and output

| Flag | Description |
|---|---|
| `--model-dir` / `--model-names` | *(Evaluation only)* Points the software to the specific trained models you want to use for infilling. |
| `--min-bounds` / `--max-bounds` | Forces the AI's output to stay within realistic physical limits (e.g., setting a max bound of 100 for a percentage index). |
| `--partitions` / `--maxmem` | *(Evaluation only)* If your dataset is too large to evaluate at once, these options split the data along the time coordinate to prevent out-of-memory crashes. |
| `--plot-results` | Automatically generates image plots of the reconstructed time indices for quick visual inspection. |


---

## 5. Infilling process

CRAI includes a pre-configured demo to help you understand the evaluation (infilling) process. This example infills missing monthly global temperature anomalies from the HadCRUT4 dataset for two specific historical dates: **September 1877** and **August 1893**.

Before running the demo, ensure you have installed `climatereconstructionai` as detailed in the [Installation](#1-installation-and-setup) section.

### Directory structure

Navigate to the demo folder in your repository. It contains:

- `demo_args.txt` — a text file containing all the pre-configured input arguments.
- `outputs/` — an empty directory where your final reconstructed files will be saved.
- `images/` — a directory containing pre-generated visual comparisons.
- `../data/test/` — contains the input climate dataset (`tas_hadcrut_187709_189308.nc`), which has a spatial resolution of 2.5º × 5º (lat × lon).

### Executing the demo

Because the paths inside `demo_args.txt` are relative, you must run this command from inside the demo directory.

```bash
crai-evaluate --load-from-file demo_args.txt
```

---

## 6. Outputs

Once the evaluation is complete, CRAI will generate 5 NetCDF (`.nc`) files and 1 PNG image inside the `outputs/` folder. What each file represents is detailed in the [Overview](index.md).