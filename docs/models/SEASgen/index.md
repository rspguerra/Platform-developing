---
title: Data-driven Seasonal Climate Predictions via Variational Inference and Transformers
sidebar_label: Overview
sidebar_position: 1
---

**Authors:** [Lluís Palma](https://orcid.org/0000-0002-3284-2152), [Alejandro Peraza](https://orcid.org/0009-0000-0164-6290), [David Civantos](https://orcid.org/0009-0002-6573-0066), [Amanda Duarte](https://orcid.org/0000-0002-9340-958X), [Stefano Materia](https://orcid.org/0000-0001-5635-2847), [Ángel G. Muñoz](https://orcid.org/0000-0002-2212-6654), [Jesús Peña-Izquierdo](https://orcid.org/0000-0002-9231-989X), [Laia Romero](https://orcid.org/0000-0001-6476-0875), [Albert Soret](https://orcid.org/0000-0002-1962-2972), [Markus Donat](https://orcid.org/0000-0002-0608-7288).

**Links:** [Paper](https://arxiv.org/abs/2503.20466) | [Code](https://gitlab.earth.bsc.es/es/seasgen)

---

### At a Glance

| | |
| :--- | :--- |
| **Task** | Probabilistic seasonal climate prediction |
| **Target variables** | Three-month seasonal anomalies of temperature and precipitation |
| **Spatial domain** | Global (configurations with different spatial resolutions and target domains) |
| **Time period** | Verification periods 1985–2021 and 2001–2021 |
| **ML method** | Conditional Variational Autoencoder (two Vision Transformer encoders + CNN decoder) with a LOESS trend |
| **Training data** | CMIP6 |
| **Validation data** | ERA5 |
| **Compute** | 1× GPU; ~56.4 GB peak RAM; 06:36:03 wallclock; 07:13:51 CPU time |

### Scientific Objective
The primary scientific objective of this work is to evaluate a novel, probabilistic deep learning approach for seasonal climate prediction. Unlike weather forecasts, which predict daily conditions, seasonal predictions focus on multi-month anomalies and face severe challenges with observational data scarcity and climate non-stationarity (regime shifts).

This project provides a framework designed to predict global three-month seasonal anomalies of temperature and precipitation throughout the year. To achieve this while avoiding the overfitting that plagues large machine learning models trained on limited observational records, the model is trained on vast climate model outputs. Crucially, the approach explicitly disentangles the predictability derived from long-term climate change trends from the predictability of natural, year-to-year (interannual) fluctuations during both training and validation.

### Study Design

#### Spatial and Temporal Domain
The model predicts global three-month seasonal anomalies throughout the year. The tested model configurations combine different spatial resolutions and target domains (see panel D of the [methodology overview](#model-architecture)). Skill is verified over the periods 1985–2021 and 2001–2021.

#### Target Variables
Three-month seasonal anomalies of temperature and precipitation.

### Data Summary
To overcome the limited temporal extent of observational datasets and expose the model to diverse climate regimes, the pipeline leverages a combination of simulated and real-world Earth system data:

* *Training Data (CMIP6)*: The model is trained on extensive General Circulation Model (GCM) outputs from the Coupled Model Intercomparison Project Phase 6. This provides thousands of simulated years, effectively increasing the sample size and preventing overfitting.
* *Validation Data (ERA5)*: The model's real-world predictive skill is validated against the European Centre for Medium-Range Weather Forecasts (ECMWF) 5th generation atmospheric reanalysis.

**Input features:** The model ingests both global and regional fields to capture broad teleconnections and local dynamics. The primary variables used are:

* `tas`: Surface Air Temperature
* `pr`: Precipitation
* `tos`: Sea Surface Temperature (ocean thermal status)
* `zg`: Geopotential Height (specifically at 500 hPa and 300 hPa)

Full data acquisition and preprocessing steps are described in [Data](./data.md).

### Methodology

#### Model Architecture
The core architecture introduces an intrinsically probabilistic method to seasonal forecasting, using a conditional Variational Autoencoder (cVAE) paired with spatial attention mechanisms.

* *Spatial Encoding*: Two Vision Transformers (ViTs) process the complex spatial relationships and teleconnections within the input climate fields, compressing this information into a latent space representation.
* *Decoding*: A Convolutional Neural Network (CNN) decoder takes this latent representation to reconstruct and predict the targeted climate anomalies.

<figure>
![seasgen-methodology](/img/seasgen/methodology-seasgen.png)
<figcaption>**Methodology overview**. *A, Illustration of the signal decomposition of the target variable Y. B, Schematic representation of the conditional Variational Autoencoder (cVAE) architecture. Two vision transformers (ViTs) encode the information from multiple climate fields into the latent space. The compressed latent space representation is then passed to the CNN decoder that reconstructs the predicted climate fields. C, Final model assembling, combining the interannual variability prediction from the cVAE model and the regressed LOESS trend. D, Tested model configurations, combining different spatial resolutions and target domains.*</figcaption>
</figure>

#### Training Strategy
To address climate non-stationarity, the methodology strictly separates the climate change signal from stochastic variability:

* *Trend Extraction*: A locally estimated scatterplot smoothing (LOESS) trend is regressed to isolate the long-term anthropogenic forcing.
* *Interannual Prediction*: The cVAE model is explicitly tasked with predicting only the detrended, interannual fluctuations.
* *Model Assembling*: The final probabilistic forecast is generated by recombining the cVAE's interannual prediction with the deterministic LOESS trend.

#### Ensemble and Uncertainty
The model generates an ensemble of `forecast_members=125` probabilistic predictions for each target year.

### Key Results
Verification metrics are computed for each variable, over the verification periods 1985–2021 and 2001–2021, with and without detrending.

<figure>
![seasgen-examples](/img/seasgen/examples-seasgen.png)
<figcaption>**Examples of the obtained images**. *Verification metric plots for precipitation, geopotential height and surface air temperature with detrending. The complete series of figures can be found in the paper.*</figcaption>
</figure>

To reproduce these results, follow [How to Use](./usage.md).

### Citation

```bibtex
@article{palma2025seasgen,
  title   = {Data-driven Seasonal Climate Predictions via Variational Inference and Transformers},
  author  = {Palma, Llu{\'i}s and Peraza, Alejandro and Civantos, David and Duarte, Amanda and Materia, Stefano and Mu{\~n}oz, {\'A}ngel G. and Pe{\~n}a-Izquierdo, Jes{\'u}s and Romero, Laia and Soret, Albert and Donat, Markus},
  journal = {arXiv preprint arXiv:2503.20466},
  year    = {2025}
}
```

**License:** TODO
