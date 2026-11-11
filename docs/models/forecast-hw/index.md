---
title: Data-Driven Seasonal Forecasts of European Heatwaves
sidebar_label: Overview
sidebar_position: 1
---

**Authors:** [Stefano Materia](https://orcid.org/0000-0001-5635-2847), [Martin Jung](https://www.bgc-jena.mpg.de/person/mjung/4777761), [Lluís Palma](https://orcid.org/0000-0002-3284-2152), [Amanda Duarte](https://orcid.org/0000-0002-9340-958X), Ullrich Weber, [Margarida Samso-Cabré](https://orcid.org/0000-0003-2868-2755), [Pere Bancells](https://orcid.org/0009-0009-9920-5084), [Markus Donat](https://orcid.org/0000-0002-0608-7288).

**Links:** [Paper](https://www2.cloud.editorialmanager.com/aies/download.aspx?id=308962&guid=de3bcf4e-f431-4562-84bf-00019373e3a8&scheme=1) | Code and Data: available upon request to the corresponding author

{/* TODO: replace the Editorial Manager link with the DOI once the paper is published, and add | [Code](...) | [Data](...) once they are public. */}

---

### At a Glance

| | |
| :--- | :--- |
| **Task** | Seasonal forecasting of heatwave indices |
| **Target variables** | Tx90n, HWP |
| **Spatial domain** | Ten European regions |
| **Time period** | 1950–2022; MAM, JJA, SON, DJF |
| **ML method** | Guided Hybrid Genetic Algorithm (GHGA) + Random Forest |
| **Training data** | ERA5, ERA5-Land |
| **Validation data** | Expanding-window test set (2003–2022), benchmarked against SEAS5 |
| **Compute** | TODO |

### Scientific Objective
While artificial intelligence has significantly improved short-term weather forecasting, long-range predictions of climate extremes remain a complex challenge. This model addresses that gap by forecasting European heatwaves at seasonal timescales (months in advance) using machine learning techniques.

The model leverages a combination of localized and global predictors (specifically land surface states, ocean temperatures, sea ice concentrations, and atmospheric CO2 levels) to estimate two distinct heatwave indices, defined in [Target Variables](#target-variables).

### Study Design

#### Spatial and Temporal Domain
Forecasts are generated for **ten distinct European regions**, grouped according to their shared climate characteristics. The data domain and the ten specific European target regions can be seen below:

<figure>
![Large-scale data domain and the ten European target regions](/img/forecast-hw/forecast-regions.png)
<figcaption>*Large-scale data domain and the ten European target regions.*</figcaption>
</figure>

#### Target Variables
* **Number of Extremely Hot Days (Tx90n):** The total number of days in a season where the maximum temperature exceeds the smoothed 90th percentile of the daily maximum temperature (calculated over a historical reference period).
* **Heatwave Propensity Index (HWP):** A composite index representing a season's overall vulnerability to heatwaves. It integrates the frequency, intensity, and duration of heat events into a single metric.

#### Experimental Setup
Each of the ten regions is tested independently across the four main meteorological seasons:
* Spring (MAM)
* Summer (JJA)
* Autumn (SON)
* Winter (DJF)

This results in **40 parallel experiments**, producing 40 highly specialized models (one for every region-season combination).

### Data Summary
The model is trained on ERA5 and ERA5-Land reanalysis datasets spanning from 1950 to 2022.

Full data acquisition and preprocessing steps are described in [Data](./data.md).

### Methodology

#### Model Architecture
The model's architecture is designed to sift through different climate data to find the most predictive signals. It achieves this using a two-part system:

1. **Feature Selection (GHGA):** We use a **Guided Hybrid Genetic Algorithm (GHGA)** to select the most relevant predictors. This algorithm uses a genetic algorithm as its global search strategy to avoid premature convergence, while simultaneously performing local search operations to speed up the process.
2. **Prediction (Random Forest):** The GHGA is wrapped around a **Random Forest (RF)** regressor, which acts as the base learner. The RF consists of 200 decision trees generated via bootstrap sampling.

#### Training Strategy
During training, the system seeks to minimize a specific cost function: the Mean Squared Error (MSE) calculated from the samples omitted during the bootstrap process (out-of-bag samples). The GHGA runs separately for the HWP and Tx90n indices. Ultimately, the system selects an ensemble of "best models" whose specific combination of predictors yields the lowest MSE.

#### Explainability
To ensure our data-driven forecasts are transparent and interpretable, the importance of each predictor is quantified using **SHAP (SHapley Additive exPlanations) values**.

* **Positive SHAP values** indicate that a specific climate feature (e.g., a regional sea surface temperature anomaly) increases the predicted heatwave index relative to the baseline.
* **Negative SHAP values** indicate a decrease in the prediction.
* **Magnitude** reflects the overall strength of that feature's influence on the forecast.

### Key Results
{/* TODO: add the main metrics and figures from the paper. */}

To reproduce these results, follow [How to Use](./usage.md).

### Citation

```bibtex
@article{materia_heatwaves,
  title   = {Data-driven seasonal forecast of European heatwaves},
  author  = {Materia, Stefano and Jung, Martin and Palma, Llu{\'i}s and Duarte, Amanda and Weber, Ullrich and Samso-Cabr{\'e}, Margarida and Bancells, Pere and Donat, Markus G.},
  journal = {Artificial Intelligence for the Earth Systems (submitted)},
  year    = {TODO},
  doi     = {TODO}
}
```

**License:** TODO
