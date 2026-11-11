---
title: Data
sidebar_label: Data
sidebar_position: 2
---

## 1. Datasets

### 1.1 Data Sources

This study uses data from *ERA5 post-processed daily statistics on pressure levels from 1940 to present* (wind components at 500 hPa) and *ERA5 post-processed daily statistics on single levels from 1940 to present* (surface variables) to train the RMM-VAE model and calculate the k-means clusters.

### 1.2 Variables

The following variables were filtered from the dataset:

| Variable | ERA5 Long name | ERA5 Short name |
|----------|----------------|-----------------|
| 2m surface air temperature (SAT) | 2m_temperature | t2m |
| Volumetric soil water layer 1 (Soil Moisture) | volumetric_soil_water_layer_1 | swvl1 |
| Total precipitation | total_precipitation | tp |
| Northward component of the wind (500 hPa) | v_component_of_wind | v |
| Eastward component of the wind (500 hPa) | u_component_of_wind | u |

The northward and eastward components of the wind were used to calculate the 500 hPa streamfunction (ψ) with CDO.

:::note
The CDO pipeline for the streamfunction was not supplied by the authors, so we rely only on the field already calculated and available on Zenodo, `sf500_NAE_1950-2024_r144x72.nc`.
:::

### 1.3 Spatial and Temporal Coverage

Since the paper covers the period from 1950 to 2022, the source dataset must contain this complete interval.

## 3. Data Acquisition

The data can be downloaded from the Copernicus web portal: [pressure levels](https://cds.climate.copernicus.eu/datasets/derived-era5-pressure-levels-daily-statistics?tab=download) (`u`, `v`) and [single levels](https://cds.climate.copernicus.eu/datasets/derived-era5-single-levels-daily-statistics?tab=download) (`t2m`, `swvl1`, `tp`).

## 4. Preprocessing

The repository includes a complete preprocessing pipeline, `pre_processing.py`, which is used together with the notebooks to prepare the data for plotting and calculations.

## 5. Final Data Structure

{/* TODO: describe the files the notebooks expect (names, folder tree) and where their paths are set. */}
