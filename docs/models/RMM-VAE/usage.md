---
title: How to use
---

## 1. Installation

### 1.1 Container

To run the notebooks, we provide a container configuration file `.def` to be used to build the container itself. The first time you use it, you'll need to mount it and bind it to a local folder.

1. Navigate to the folder containing the `.def` file (for example, your Downloads folder):

```bash
   cd ~/Downloads
```

2. Build the container using the `.def` file:

```bash
   singularity build rmm_env2.sif rmm_env2.def
```



> **Note:** adjust the paths above (`<PROJECT_ROOT_DIRECTORY>` and `rmm_env2.sif`) to match where you've saved the project and the image on your own machine.

> **Note**: move the `.sif` file to your project root directory once the building is done for a simple execution.

---

## 2. Configuration

### Configuring paths

Paths and constants used throughout the notebooks are set in a single `config.yaml` file. Before running anything, update this file to match your local folder structure.

```yaml
# config.yaml
paths:
  project_root: '<PROJECT_ROOT_DIRECTORY>'
  data_path: '<PROJECT_ROOT_DIRECTORY>/DATA'
  figs_path: '<PROJECT_ROOT_DIRECTORY>/figures'

constants:
  target_variable_name: 'var167'
  g0: 9.80665
  extended_summer_months: [6, 7, 8]
```

**Field reference:**

| Field | Description |
|---|---|
| `project_root` | Root folder of the project |
| `data_path` | Folder containing the input data |
| `figs_path` | Folder where generated figures will be saved |
| `target_variable_name` | Name of the target variable used in the model |
| `g0` | Gravitational constant (m/s²) |
| `extended_summer_months` | Months considered as "extended summer" (June, July, August) |

> **Note:** make sure the paths in `config.yaml` point to the location in which you saved the folders on your machine before running the notebook. The `config.yml` file is in the source directory of the project.

> **Note**: feel free to set the paths as you wish, but we advise to follow the centered on root directory strategy to make the reproduction more intuitive.

## 3. Running the scripts


### 3.1 Inside Jupyter

Run the container, binding your local project folder to your project root directory inside the image:

```bash
   singularity run --bind <PROJECT_ROOT_DIRECTORY> rmm_env2.sif
```

Once the container starts, open the link it prints in your terminal to access the notebook interface.

### 3.2 Bash scripts

For a HPC running it is possible to run it through a single batch script all the paper steps. To do so, build a bash script with the following commands, they will execute all the figure notebooks in sequence and generate all the figures and data used in the paper analysis.

```bash

module load singularity

export CARTOPY_DATA_DIR=<PROJECT_ROOT_PATH>/cartopy/shapefiles/natural_earth/10m/physical/ne_10m_coastline.*

singularity exec --nv rmm_env2.sif jupyter nbconvert --to notebook --execute Figure1_ERL_refactored.ipynb --output Figure1_ERL_refactored_out.ipynb --ExecutePreprocessor.timeout=-1

singularity exec --nv rmm_env2.sif jupyter nbconvert --to notebook --execute Figure2_ERL_refactored.ipynb --output Figure2_ERL_refactored_out.ipynb --ExecutePreprocessor.timeout=-1

singularity exec --nv rmm_env2.sif jupyter nbconvert --to notebook --execute Figure3_ERL_refactored.ipynb --output Figure3_ERL_refactored_out.ipynb --ExecutePreprocessor.timeout=-1

singularity exec --nv rmm_env2.sif jupyter nbconvert --to notebook --execute Figure4_ERL_refactored.ipynb --output Figure4_ERL_refactored_out.ipynb --ExecutePreprocessor.timeout=-1

singularity exec --nv rmm_env2.sif jupyter nbconvert --to notebook --execute Figure5_a_b_Table1_ERL_refactored.ipynb --output Figure5_a_b_Table1_ERL_refactored_out.ipynb --ExecutePreprocessor.timeout=-1

singularity exec --nv rmm_env2.sif jupyter nbconvert --to notebook --execute Figure5_c_d_ERL_refactored.ipynb --output Figure5_c_d_ERL_refactored_out.ipynb --ExecutePreprocessor.timeout=-1

```

> **Note**: Remember to adjust all the paths into your system folder particularities.

