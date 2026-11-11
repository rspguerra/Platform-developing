---
title: Quantifying Atmospheric and Land Drivers of hot tempERAture extremes
sidebar_label: Overview
sidebar_position: 1
---

**Authors:** [Arnau Garcia Mesa](https://orcid.org/0009-0004-5738-8867), [Lluís Palma](https://orcid.org/0000-0002-3284-2152), [Markus Donat](https://orcid.org/0000-0002-0608-7288), [Stefano Materia](https://orcid.org/0000-0001-5635-2847), [Raül Marcos Matamoros](https://orcid.org/0000-0002-3610-3445).

**Links:** [Paper](https://egusphere.copernicus.org/preprints/2025/egusphere-2025-5392/) | [Code](https://zenodo.org/records/22143255) | [Data](https://zenodo.org/records/21337082)

---

### At a Glance

| | |
| :--- | :--- |
| **Task** | Classification of daily heat extremes and driver attribution |
| **Target variables** | Daily extreme event (TX > 90th or 80th percentile) |
| **Spatial domain** | Córdoba, Marrakech, Lyon, Belgrade, Hannover, Stockholm |
| **Time period** | 1950–2024, boreal summer (JJA); training period 1950–2014 |
| **ML method** | MLP (land and CO₂) + ConvNeXt CNN (circulation), explained with SHAP |
| **Training data** | ERA5, ERA5-Land, NOAA CO₂ |
| **Validation data** | 80/20 random training/validation split; E-OBS as alternative observational data |
| **Compute** | TODO |

### Scientific Objective
Although different physical drivers play a central role in modulating the intensity and frequency of summer temperature extremes, quantifying their individual contributions remains a complex challenge. This project provides a diagnostic framework designed to analyse the influence of three components on boreal summer heat extremes: large-scale atmospheric circulation, local land surface conditions and anthropogenic forcing.

To achieve this, certain variables from available climatological datasets were selected and used to train a machine learning model, and the results were interpreted using explainable AI techniques. A brief overview of this process is provided below; for a more detailed explanation, refer to the paper linked at the beginning of this page.

### Study Design

#### Spatial and Temporal Domain
To avoid including unrelated atmospheric variability, the selected domain is the area surrounded by the red line in the image below. The roles of circulation, soil moisture and anthropogenic climate change during extreme temperature events are specifically studied for the following regions: **Córdoba (Spain)**, **Marrakech (Morocco)**, **Lyon (France)**, **Belgrade (Serbia)**, **Hannover (Germany)** and **Stockholm (Sweden)**.

<figure>
![Large-scale data domain and locations selected for the local-scale data in the prediction target](/img/quantify-drivers/sites-screenshot.png)
<figcaption>*Large-scale data domain and locations selected for the local-scale data in the prediction target.*</figcaption>
</figure>

#### Target Variables
A daily extreme event is classified using the 90th percentile threshold of the climatological maximum temperature distribution; however, the experiments can also be performed using an 80th percentile threshold.

### Data Summary
This model uses data from the **ERA5** and **ERA5-Land** datasets, which are part of the fifth-generation atmospheric reanalysis from the **European Centre for Medium-Range Weather Forecasts (ECMWF)**. It also uses data on seasonal atmospheric concentrations of CO₂ collected by the **US National Oceanic and Atmospheric Administration (NOAA)**.

The **ERA5** dataset provides daily gridded data from 1940 to the present. More specifically, the following variables are used: Geopotential at 500 hPa (`g500`), Geopotential at 200 hPa (`g200`) and Sea Level Pressure (`psl`).

From the **ERA5-Land** dataset, which provides a better representation of land surface variables, the following variables are used: daily maximum 2-meter temperature (`TX`) and the soil moisture at three depth levels (`swvl1`, `swvl2` and `swvl3`).

Full data acquisition and preprocessing steps are described in [Data](./data.md).

### Methodology

#### Model Architecture
To capture land-atmosphere coupling, the model architecture consists of a **Multi-Layer Perceptron (MLP)** for land and CO₂, and a **ConvNeXt Convolutional Neural Network (CNN)** for the spatial features of circulation.

<figure>
![Model architecture](/img/quantify-drivers/model.png)
<figcaption>*Model architecture.*</figcaption>
</figure>

#### Training Strategy
The AdamW optimiser is used for optimisation, and the Optuna Python package is used for hyperparameter tuning, with balanced accuracy minus final validation loss as the score. The model is then trained for 75 epochs, unless the validation loss increases for a total of five epochs.

#### Ensemble and Uncertainty
To create a robust explanation, the model is trained using twenty different deterministic seeds, and the results are then extracted from an averaged combination of those.

#### Explainability
To quantify the contribution of each input feature to the model's predictions, **Explainable AI (XAI)** techniques are employed. The technique used is **SHapley Additive exPlanations (SHAP)**, a game-theoretic approach that explains the output of any machine learning model by attributing it to the input features. This allows driver dominance to be quantified, spatial patterns to be identified, and long-term trends to be recognised, among other things.

### Key Results
The model performance is assessed by the **balanced accuracy (BA)** and the **Area Under the Receiver Operating Characteristic Curve (AUC)**, which are a good choice for imbalanced classification tasks.

The results using the 90th percentile and 80th percentile definitions of an extreme event are the following. These results can be replicated by running the model for a specific seed ensemble and analysing them with the Jupyter notebooks provided.

**90th percentile definition:**
| Site | Balanced Accuracy | AUC |
| :--- | :--- | :--- |
| Córdoba | 0.8367 | 0.9353 |
| Hannover | 0.8314 | 0.9306 |
| Lyon | 0.8421 | 0.9371 |
| Belgrade | 0.8486 | 0.9262 |
| Marrakech | 0.8600 | 0.9415 |
| Stockholm | 0.8567 | 0.9367 |

**80th percentile definition:**
| Site | Balanced Accuracy | AUC |
| :--- | :--- | :--- |
| Córdoba | 0.8288 | 0.93189 |
| Hannover | 0.8116 | 0.9070 |
| Lyon | 0.8573 | 0.9343 |
| Belgrade | 0.8496 | 0.9365 |
| Marrakech | 0.8479 | 0.9326 |
| Stockholm | 0.8545 | 0.9374 |

The most relevant plots generated are the following:
- `Extremes yearly count for the six locations for both ERA5-Land and E-OBS data`. This plot shows, for each year, the number of days classified as temperature extremes, given a percentile for the definition of the extreme event.
- `Extreme class prediction mean SHAP value percentage for all locations using the CombinedModel taking the 50% most confident predictions`. This plot uses SHAP to assign a percentage score to each environmental driver.
- `Change in ROC curves for the Combined Model for different percentages of confidence`. This plot evaluates how well the model can distinguish between an actual extreme heat day and a normal day.

<figure>
![Extreme class prediction mean SHAP value percentage](/img/quantify-drivers/image.png)
<figcaption>*Extreme class prediction mean SHAP value percentage.*</figcaption>
</figure>

To reproduce these results, follow [How to Use](./usage.md).

### Citation

```bibtex
@article{garciamesa2025drivers,
  title   = {Quantifying atmospheric and land drivers of hot temperature extremes through explainable Artificial Intelligence},
  author  = {Garcia Mesa, Arnau and Palma, Llu{\'i}s and Donat, Markus and Materia, Stefano and Marcos Matamoros, Ra{\"u}l},
  journal = {EGUsphere (preprint)},
  year    = {2025},
  doi     = {10.5194/egusphere-2025-5392}
}
```

**License:** TODO
