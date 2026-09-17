---
title: Data
---

## 1. Data

### 1.1 Dataset

This study uses data from *ERA5 post-processed daily statistics on pressure levels from 1940 to present* to train the RMM-VAE model and calculate the k-means. The data can be downloaded from the [Copernicus](https://cds.climate.copernicus.eu/datasets/derived-era5-pressure-levels-daily-statistics?tab=download) web portal.

### 1.2 Variables

The following variables were filtered from the dataset:

| Variable | ERA5 Long name | ERA5 Short name |
|----------|----------------|-----------------|
| 2m surface air temperature (SAT) | 2m_temperature | t2m |
| Volumetric soil water layer 1 (Soil Moisture) |  volumetric_soil_water_layer_1 | swvl1 |
| Total precipitation | total_precipitation | tp | 
| Northward component of the wind (500 hPa) | v_component_of_wind | v |
| Eastward component of the wind (500 hPa) | u_component_of_wind | u |


The Northward and Eastward components of wind were used to calculate the 500 hPa streamfunction (ψ) through CDO.

> **Note**: The CDO pipeline for the stramfunction was not supplied by the author so we relly only on the Zenodo already calculated field `sf500_NAE_1950-2024_r144x72.nc`.


### 1.3 Time intervall

Since the paper works with the period between 1950 to 2022, the dataset used as source must contain this complete intervall.

### 1.4 Preprocessing

The repository counts with a complete preprocessing pipeline code `pre_processing.py`, that is used along the notebooks to prepare the data for ploting and calculations.


