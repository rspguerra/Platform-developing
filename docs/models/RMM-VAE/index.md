---
title: Targeted weather regimes identify circulation patterns behind Western European summer heat extremes and trends
sidebar_label: Overview
---

**Authors:** [Julianna Carvalho-Oliveira](https://orcid.org/0000-0003-1222-0927), [Fiona Spuler](https://orcid.org/0009-0003-9358-0699), [Marlene Kretschmer](https://orcid.org/0000-0002-2756-9526).

**Links:** [Original Paper](https://iopscience.iop.org/article/10.1088/1748-9326/ae499b) | [Github Repository](https://github.com/jcarvoli/RMM-VAE_Euro_summer) | [Zenodo Repository](https://zenodo.org/records/18312651).

---

### Scientific Objective
As heat extremes keep rising across Western Europe, understanding the atmospheric circulation patterns behind them has become increasingly critical. The goal of this paper is to evaluate a novel approach for characterizing circulation variability and attributing regional temperature trends. Conventionally, the scientific community has mapped these patterns by combining *Principal Component Analysis (PCA)* to extract key dataset features with *k-means clustering* to group them into recurrent weather regimes. Because this approach is linear and looks at average domain-wide variability, it often blends together the specific circulation setups that trigger local heatwaves. While newer, machine-learning-based *targeted regimes* focus directly on extreme temperatures, they can lose sight of the broader atmospheric physics, sacrificing consistency and predictability. To overcome these limitations, the **Regression Mixture-Model VAE (RMM-VAE)** is applied. By integrating nonlinear dimensionality reduction, clustering, and a regression task into a single framework, RMM-VAE captures the full atmospheric phase space while remaining informative of temperature targets. This framework is used to: 

    - Compare RMM-VAE against k-means in capturing Western European heat extremes.

    - Explain and predict interannual temperature variability using seasonal regime frequencies.

    - Quantify the dynamical contribution of circulation changes to Europe's observed excess summer warming.


### Dataset and variables
This study utilizes daily atmospheric and surface data from the ERA5 reanalysis dataset for the period 1950–2022, focusing exclusively on the boreal summer (June–August, JJA).

The variables of interest are the following:

- **Target Variable (Surface Thermal Conditions)**: We use 2m surface air temperature (SAT) averaged over a targeted Western European domain (45°N–56°N, 5°W–16°E). This regional area-mean time series serves as the proxy for near-surface thermal conditions and is the primary target variable for the regression mixture-model variational autoencoder.
- **Large-Scale Atmospheric Circulation**: Mid-tropospheric flow patterns are evaluated using the 500 hPa streamfunction over the Euro-Atlantic sector (30°N–60°N, 30°W–20°E). This narrower domain (compared to classic North Atlantic–European configurations) improves cluster robustness and strengthens coupling to surface anomalies.
- **Land-Atmosphere & Hydrological Variables**: To assess coupling and hydrological feedback associated with the weather regimes, we include daily soil moisture in the uppermost layer (0–7 cm) and total precipitation.

To ensure computational efficiency and isolate the relevant physical signals, variables are regridded to a coarser, uniform spatial resolution. To remove the influence of the seasonal cycle and local spatial variability, all fields are standardized at each grid point by subtracting the daily climatological mean and dividing by the daily standard deviation. Finally, a centered 5-day rolling mean is applied to all daily fields to suppress transient synoptic weather disturbances and isolate persistent, large-scale circulation features.

### Methodology
To analyze extremes across different severities, we define temperature anomalies at three distinct levels:

    - **Extreme Heat Days**: A binary classification identifying days where the spatially averaged Western European SAT exceeds the 90th percentile of the full 1950–2022 climatology.

    - **Compound Heatwaves**: A stricter classification requiring simultaneous intensity, duration, and spatial criteria. A heatwave is identified when:

        - Intensity: Grid-point daily maximum temperatures exceed the local 90th percentile (calculated using a 15-day centered moving window to account for seasonality).

        - Duration: The intensity threshold is exceeded for a minimum of three consecutive days.

        - Spatial Extent: At least 30% of the Western European domain registers these local exceedances simultaneously.

    - **Seasonal Metric (SAT90​)**: To quantify interannual variability and long-term trends, we calculate the 90th percentile of the spatially averaged daily SAT for each individual summer.


We compare two distinct methodologies for identifying and classifying summertime weather regimes:

#### 1. Conventional Baseline (PCA and k-means)
As a non-targeted baseline, we apply principal component analysis (PCA) to the daily 500 hPa streamfunction anomalies over the Euro-Atlantic domain. We retain the first ten principal components (accounting for approximately 96.4% of the variance) and subsequently cluster their scores using the k-means algorithm. This represents the standard, unsupervised approach to identifying main modes of regional variability.

#### 2. Targeted Machine Learning (RMM-VAE)
To isolate regimes specifically tied to Western European heat extremes, we employ the RMM-VAE method. This approach learns a low-dimensional latent representation of the 500 hPa streamfunction fields while imposing a latent Gaussian-mixture prior to yield probabilistic daily weather regimes. Concurrently, a supervised regression term trains the encoder to predict the target variable (Western European SAT). This dual-optimization regularizes the latent space, generating regimes that are highly informative of surface temperature extremes while remaining constrained by physically consistent, dynamically plausible large-scale flow structures.


### Evaluation

