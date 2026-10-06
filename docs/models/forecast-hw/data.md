---

title: Data

---

## 1. Datasets

This project works with 4 different data sources: 
- [ERA5](https://doi.org/10.24381/cds.50314f4c) (Hersbach et al. 2020): For Oceanic data - model input
- [ERA5-Land](https://doi.org/10.24381/cds.e2161bac) (Muñoz-Sabater et al. 2021): For Land Surface data - model input
- [SEAS5](https://doi.org/10.24381/cds.7e37c951) (Johnson et al. 2019): Used as benchmark comparison for the model predictions
- [NOAA](https://gml.noaa.gov/ccgg/trends/data.html): CO2 concentration data - model input (to be checked)

The lsit of candidate drivers chosen by the author in each of the input datasets were:

#### Table 1. List of candidate drivers

| Predictor | Acronym | Component | Source |
|---|---|---|---|
| CO2 concentration | CO2 | Global | NOAA |
| Surface soil moisture (0–7 cm) | swvl1 | Land surface | ERA5 Land |
| Soil moisture layer 2 (7–28 cm) | swvl2 | Land surface | ERA5 Land |
| Soil moisture layer 3 (28–100 cm) | swvl3 | Land surface | ERA5 Land |
| Surface soil temperature (0–7 cm) | stl1 | Land surface | ERA5 Land |
| Soil temperature layer 2 (7–28 cm) | stl2 | Land surface | ERA5 Land |
| Soil temperature layer 3 (28–100 cm) | stl3 | Land surface | ERA5 Land |
| Surface latent heat flux | slhf | Land surface | ERA5 Land |
| Surface sensible heat flux | sshf | Land surface | ERA5 Land |
| Surface solar radiation | ssr | Land surface | ERA5 Land |
| Snow water equivalent | sd | Land surface | ERA5 Land |
| 15-day trend of slhf | mlhf | Land surface | ERA5 Land |
| 15-day trend of sshf | mshf | Land surface | ERA5 Land |
| Eastern Equatorial Pacific SST | eepo | Ocean | ERA5 |
| Central Equatorial Pacific SST | cepo | Ocean | ERA5 |
| Western Equatorial Pacific SST | wepo | Ocean | ERA5 |
| Southeast Asia Seas SST | seas | Ocean | ERA5 |
| Eastern Indian SST | eind | Ocean | ERA5 |
| Western Indian SST | wind | Ocean | ERA5 |
| Equatorial Atlantic SST | eatl | Ocean | ERA5 |
| Caribbean Sea SST | cars | Ocean | ERA5 |
| Southern North Atlantic SST | snao | Ocean | ERA5 |
| Mid North Atlantic SST | mnao | Ocean | ERA5 |
| Northern North Atlantic SST | nnao | Ocean | ERA5 |
| Mediterranean Sea SST | meds | Ocean | ERA5 |
| North Pacific SST | npac | Ocean | ERA5 |
| Alaska-Canada Pacific coast SST | acnp | Ocean | ERA5 |
| Arctic Ocean SST | arco | Ocean | ERA5 |
| Barents Sea SST | bars | Ocean | ERA5 |
| Siberian ice shelf concentration | sibs | Sea Ice | ERA5 |
| Alaska ice shelf concentration | alsk | Sea Ice | ERA5 |
| Canadian ice shelf concentration | cans | Sea Ice | ERA5 |
| Barents and Kara seas ice concentration | baka | Sea Ice | ERA5 |


>**Note: ERA5, ERA5-LAnd and SEAS5 data were downloaded from the C3S Climate Data Store after registration and acceptance of the licensing terms, and postprocess from hour to monthly averaged values before being used in the preprocessing pipelines presented bellow.**

## 2. Data preprocessing

### 2.1 ERA5 data

The data preprocessing for ERA5, ERA5-Land and SEAS5 is done through the usage of Climate Data Operators (CDO) and netCDF Operators (NCO) in bash scripts stored at the *preprocessing folder*. The preprocessing is done globaly for ERA5 and regionaly for ERA5-Land and SEAS5.

| Filename                          | Description                                                        |
|------------------------------------|---------------------------------------------------------------------|
| era5_sic_global.sh                | Builds sea ice concentration climatology and anomaly predictors for 4 Arctic regions |
| era5_sst_global.sh                | Builds SST climatology and anomaly predictors for 16 ocean regions |
| era5land_slhf_euregions.sh        | Builds surface latent heat flux (SLHF) climatology/anomaly predictors for EU regions |
| era5land_sshf_euregions.sh        | Builds surface sensible heat flux (SSHF) climatology/anomaly predictors for EU regions |
| era5land_ssr_euregions.sh         | Builds surface solar radiation (SSR) climatology/anomaly predictors for EU regions |
| era5land_stl1_euregions.sh        | Builds soil temperature layer 1 (STL1) climatology/anomaly predictors for EU regions |
| era5land_stl2_euregions.sh        | Builds soil temperature layer 2 (STL2) climatology/anomaly predictors for EU regions |
| era5land_stl3_euregions.sh        | Builds soil temperature layer 3 (STL3) climatology/anomaly predictors for EU regions |
| era5land_swe_euregions.sh         | Builds snow water equivalent (SWE) climatology/anomaly predictors for EU regions |
| era5land_swvl1_euregions.sh       | Builds soil water volume layer 1 (SWVL1) climatology/anomaly predictors for EU regions |
| era5land_swvl2_euregions.sh       | Builds soil water volume layer 2 (SWVL2) climatology/anomaly predictors for EU regions |
| era5land_swvl3_euregions.sh       | Builds soil water volume layer 3 (SWVL3) climatology/anomaly predictors for EU regions |
| era5land_tmax_euregions.sh        | Builds max temperature (Tmax) climatology/anomaly predictors for EU regions |
| fix_locpred_dates.sh              | Resets the time axis (3-month steps) for swvl2/swvl3 local predictor files across EU regions and ensembles |
| seas5_tmax_euregions.sh           | Builds SEAS5 forecast tasmax climatology/anomaly benchmark data (2003–2022) by season for EU regions |
|||

Once the data is downloaded and stored the header configuration variables must be adjusted to the correct system names and paths:

```bash
iniyr=1950
endyr=2022
varera=<VARIABLE_NAME_AT_ERA5>
var=<VARIABLE_SHORT_NAME_USED>
srcdir=<RAW_SOURCE_DATA_BASE_DIRECTORY>
flddir=<RAW_SOURCE_DATA_FILE_PATH> #Complements srcdir
lsmdir=<LAND_SEA_MASK_FILE_DIRECTORY>
wrkdir=<WORKING_DIRECTORY>
pdcdir=<PREDICTORS_FILES_WRITING_DIRECTORY>
```

>**Note:** The variables `varera` is the ERA5 correct short name in case of adjustments needed from the downloaded files and `var` is the name used in the pipeline. The table bellow correlates both to make it clear to all users.

Once the variables are correctly filled in each one of the preprocessing scripts they can be executed. The preprocessing scripts are located inside the folder preprocessing, as listed bellow:

| Name | var | varera | Script |
|---|---|---|---|
| Surface soil moisture (0-7 cm) | swvl1 | swvl1 | `era5land_swvl1_euregions.sh` |
| Soil moisture layer 2 (7-28 cm) | swvl2 | swvl2 | `era5land_swvl2_euregions.sh` + `fix_locpred_dates.sh` |
| Soil moisture layer 3 (28-100 cm) | swvl3 | swvl3 | `era5land_swvl3_euregions.sh` + `fix_locpred_dates.sh` |
| Surface soil temperature (0-7 cm) | stl1 | tsl1 | `era5land_stl1_euregions.sh` |
| Soil temperature layer 2 (7-28 cm) | stl2 | tsl2 | `era5land_stl2_euregions.sh` |
| Soil temperature layer 3 (28-100 cm) | stl3 | tsl3 | `era5land_stl3_euregions.sh` |
| Surface latent heat flux | slhf | hfls | `era5land_slhf_euregions.sh` |
| Surface sensible heat flux | sshf | hfss | `era5land_sshf_euregions.sh` |
| Surface solar radiation | ssr | rss | `era5land_ssr_euregions.sh` |
| Snow water equivalent | swe | snd | `era5land_swe_euregions.sh` |
| 15-day trend of slhf (mlhf) | slhf | hfls | `era5land_slhf_euregions.sh` (`cdo trend` → `_m.nc`) |
| 15-day trend of sshf (mshf) | sshf | hfss | `era5land_sshf_euregions.sh` (`cdo trend` → `_m.nc`) |
| 15-day trend of ssr (mssr) | ssr | rss | `era5land_ssr_euregions.sh` (`cdo trend` → `_m.nc`) |
| Sea surface temperature (16 regions: eepo … bars) | tos | — | `era5_sst_global.sh` |
| Sea ice concentration (cans, alsk, sibs, baka) | sic | — | `era5_sic_global.sh` |

### 2.2 NOAA CO2 data

The CO2 data follows a different preprocessing path in relation to the ERA5. This data is obtained in .csv format with starting date on 1958, requiring the following steps before the preprocessing script execution:

1. Data download.
2. Data adjustment to seasons trimesters.
3. Linear regression over 1958 to 1968 to generate 1950 to 1957 years missing data.
4. Downloaded data season shaped concatenation with infered values to build input dataset for the preprocessing pipeline.

Once executed the output dataset will be a `.csv` file containing the columns *"Year", "Season", "Conc"* with Seasons composed by the months initials and Conc by the the average CO2 concentration values from the year 1950 to 2022. This dataset can now be feed to the preprocessing pipeline to generate the *the_table.csv* file that will be inputed for the model.

>**Note:** NOAA data is passive to revision and updates so the linear regression values obtained for the 1950 to 1958 period may differ slighly if executed from files downloaded in defferent periods.


