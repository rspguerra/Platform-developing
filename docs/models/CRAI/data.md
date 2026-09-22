---
title: Data 
sidebar_label: Data
---
There are multiple datsets used for the different stages of development of this project. In the following table we can see an Overview of in which stage was each dataset used:

<figure>
![CRAI-dataset](/img/crai/CRAI-datasets.png)
<figcaption>**Overview of the number of samples, temporal coverage and utilization of the datasets.** *The temporal coverage reported here corresponds to the time period used in the study. Ballot symbols indicates the usage of each dataset for three specific tasks: training of the AI model,
evaluation of the AI model, analysis of the dataset.*</figcaption>
</figure>

The HadEX-CAM datasets used for the evaluation and analysis are available in Zenodo: [https://zenodo.org/records/13891336](https://zenodo.org/records/13891336).


Noticeably, the CMIP6 dataset is used on all the models stages. The exact CMIP6 ensemble members used are provided in this table:
<figure>
![CRAI-ensemble](/img/crai/CRAI-CMIP6-ensemble.png)
<figcaption>**List of CMIP6 models and members used to create the training, validation
and test datasets.**</figcaption>
</figure>

Next, we provide a guide to obtain the CMIP6 dataset for the training and evaluation using [ESMValTool](https://www.esmvaltool.org/) to download and preprocess them.
 
### 1. Install ESMValTool
 
Install ESMValTool and its core engine into a dedicated Conda environment:
 
```bash
conda create -n esmvaltool -c conda-forge esmvaltool
conda activate esmvaltool
```
 
Full installation instructions are available in the [ESMValTool documentation](https://docs.esmvaltool.org/en/latest/quickstart/installation.html).
 
### 1.1 Configure local paths
 
Generate the default user configuration file:
 
```bash
esmvaltool config get_config_user
```
 
This creates `~/.config/esmvaltool/config-user.yml`. Open it and set the following fields. The values shown below reflect the recommended settings for this project, adjust the paths to your own system:
 
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

In the `esmval` module there is an example configuration, `config-user.yml`. This is the configuration that was used for obtaining the datasets. Of course, the paths should be modified. 

 
### 1.2 Configure ESGF search nodes
 
If the default ESGF node is unavailable or missing data, create `~/.esmvaltool/esgf-pyclient.yml` with additional search nodes:
 
```yaml
search_connection:
  urls:
    - https://esgf-node.llnl.gov/esg-search
    - https://esgf.ceda.ac.uk/esg-search
  distrib: true
```
 
---
 
## 2. Understanding the Recipe
 
The CRAI recipe (`CRAI_recipe.yml`) is structured in four blocks:
 
**`documentation`** — metadata describing the recipe's purpose and authors.
 
**`datasets`** — specifies the exact CMIP6 ensemble members to retrieve. 
 
**`preprocessors`** — defines the spatial and mathematical operations applied to the raw data before it reaches the diagnostic script.

**`diagnostics` / `scripts`** — maps datasets to preprocessors, then passes the standardised output to the desired script, which in this case is a placeholder with no operations.
 
---
 
## 4. Run the Recipe
 
Once all data is in place and paths are configured:
 
```bash
conda activate esmvaltool
esmvaltool run CRAI_recipe.yml
```
 
The output NetCDF store will be written to the `work_dir` subdirectory inside `output_dir`. Update `paths.yml` in the SEASGen experiment configs to point to this path before running the model.
 
---
 
## 5. Preprocessing data into the indices

Once each model data is downloaded it must be organized as it follows:

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
Inside of each model the structure must be as:

```bash
 `-- historical
     `-- r10i1p1f1 (member code)
         `-- day (frequence)
             `-- tasmax (variable)
                 `-- ["gr" or "gn"] (grid)
                     `-- "v..." 
```

With each model folder corretly in place and filled with the downloaded netCDF (.nc) data containing daily tasmax and tasmin for each one of the years from 1940 to 2022, run the code python + cdo pipeline for preprocessing and index calculation code `build_all_cmip_indices.sh`. 

> **Note 1**: If you are in a HPC, run the preprocessing code through a slurm script.

>**Note 2:** If the `build_all_cmip_indices.sh` gets interrupted during its execution, verify in which model member it happened and delete the incomplete file. The code has a file verification that identifies already created output files before runing the pipeline only for the missing ones, if a file creation is interrupted this incomplete file will be counted as an existing one and will break the training pipeline execution afterwards.

The preprocessing pipeline will produce 4 netCDF file (.nc) for each model member one per index (TN10p, TN90p, TX10p and TX90p), following the naming pattern `{index}_{model}_{member}_{starting_year}-{ending_year}.nc`. Those files are the ones used in the training and validation pipeline of the model.



