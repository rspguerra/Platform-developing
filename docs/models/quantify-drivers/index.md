---
title: Quantifying Atmospheric and Land Drivers of hot tempERAture extremes
sidebar_label: Overview
---


**Authors:** [Arnau Garcia Mesa](https://orcid.org/0009-0004-5738-8867), [Lluís Palma](https://orcid.org/0000-0002-3284-2152), [Markus Donat](https://orcid.org/0000-0002-0608-7288), [Stefano Materia](https://orcid.org/0000-0001-5635-2847), [Raül Marcos Matamoros](https://orcid.org/0000-0002-3610-3445).

**Links:** [Original Paper](https://egusphere.copernicus.org/preprints/2025/egusphere-2025-5392/) | [Code Repository](https://zenodo.org/records/22143255) | [Data repository](https://zenodo.org/records/21337082)  

---

### Scientific Objective
Although different physical drivers play a central role in modulating the intensity and frequency of summer temperature extremes, quantifying their individual contributions remains a complex challenge. This project provides a diagnostic framework designed to analyse the influence of three components on boreal summer heat extremes: large-scale atmospheric circulation, local land surface conditions and anthropogenic forcing. 

To achieve this, certain variables from available climatological datasets were selected and used to train a machine learning model, and the results were interpreted using explainable AI techniques. A daily extreme event is classified using the 90th percentile threshold of the climatological maximum temperature distribution; however, the experiments can also be performed using an 80th percentile threshold. A brief overview of this process is provided below; for a more detailed explanation, refer to the paper linked at the beginning of this page.

### Dataset and variables
This model uses data from the **ERA5** and **ERA5-Land** datasets, which are part of the fifth-generation atmospheric reanalysis from the **European Centre for Medium-Range Weather Forecasts (ECMWF)**. It also uses data on seasonal atmospheric concentrations of CO₂ collected by the **US National Oceanic and Atmospheric Administration (NOAA)**. 

The **ERA5**  dataset uses daily gridded data from 1940 to the present. More specifically, the following variables are used: Geopotential at 500 hPa (`g500`), Geopotential at 200 hPa (`g200`) and Sea Level Pressure (`psl`).

From the **ERA5-Land dataset**, which provides a better representation of land surface variables, the following variables are used: daily maximum 2-meter temperature (`TX`) and the soil moisture at three depth levels (`swl1`, `swl2` and `swl3`).

To avoid including unrelated atmospheric variability, the selected domain is the area surrounded by the red line in the image below. The roles of circulation, soil moisture and anthropogenic climate change during extreme temperature events are specifically studied for the following regions:  **Córdoba (Spain)**, **Marrakech (Morocco)**, **Lyon (France)**, **Belgrade (Serbia)**, **Hannover (Germany)** and **Stockholm (Sweden)**.



<figure>
![Large-scale data domain and locations selected for the local-scale data in the prediction target](/img/quantify-drivers/sites-screenshot.png)
<figcaption>*Large-scale data domain and locations selected for the local-scale data in the prediction target.*</figcaption>
</figure>

### Methodology
To capture land-atmosphere coupling, the model architecture consists of a **Multi-Layer Perceptron (MLP)** for land and CO₂, and a **COnvNeXt Convolutional Neural Network (CNN)** for the spatial features of circulation. The AdamW optimiser is used for optimisation, and the Optuna Python package is used for hyperparameter tuning, with balanced accuracy minus final validation loss as the score. The model is then trained for 75 epochs, unless the validation loss increases for a total of five epochs.

<figure>
![Model architecture](/img/quantify-drivers/model.png)
<figcaption>*Model architecture.*</figcaption>
</figure>

To quantify the contribution of each input feature to the model's predictions, **Explainable AI (XAI)** techniques are employed. This technique is called **SHapley Additive exPlanations (SHAP)** and is a game-theoretic approach that explains the output of any machine learning model by attributing it to the input features. This allows driver dominance to be quantified, spatial patterns to be identified, and long-term trends to be recognised, among other things.

To create a robust explanation, the model is trained by setting twenty different deterministic seeds, and the results are then extracted from an avergaged combination of those.

### Evaluation 
The model performance is assessed by the **balanced accuracy (BA)** and the **Area Under the Receiver Operating Characteristic Curve (AUC)**, which are a good choice for imbalanced classification tasks.

The results using the 90th percentile definition and 80th percentile definition for extreme event are the following. This results can be replicated by running the model for a concrete seed ensamble and analysing them with the jupyter notebooks provided. More detail can be found in [How to use](./usage.md).

**90th percentile defintion:**
| Site |  Balanced Accuracy |  AUC  |
| :--- | :--- |  :--- |
| Cordoba | 0.8367 | 0.9353 |
| Hannover | 0.8314 | 0.9306 |
| Lyon | 0.8421 | 0.9371 |
| Belgrado | 0.8486 | 0.9262 |
| Marrakech | 0.8600 | 0.9415 |
| Stockholm | 0.8567 | 0.9367 |

**80th percentile defintion:**
| Site |  Balanced Accuracy |  AUC  |
| :--- | :--- |  :--- |
| Cordoba | 0.8288 | 0.93189 |
| Hannover | 0.8116 | 0.9070 |
| Lyon | 0.8573 | 0.9343 |
| Belgrado | 0.8496 | 0.9365 |
| Marrakech | 0.8479 | 0.9326 |
| Stockholm | 0.8545 | 0.9374 |

### Plots and analysis
To show the results, the most relevant plots generated are the following:
- `Extremes yearly count for the six locations for both ERA5-Land and E-OBS data`. This plot shows for each year the number of days classified as temperature extremes, given a percentile for the definition of the extreme event.
- `Extreme class prediction mean SHAP value percentage for all locations using the CombinedModel taking the 50% most confident predictions`. This plot uses SHAPto assign a percentage score to each environmental driver.
- `Change in ROC curves for the Combined Model for different percentages of confidence`.  This plot evaluates how well the model can distinguish between an actual extreme heat day and a normal day.

<figure>
![`Extreme class prediction mean SHAP value percentage](/img/quantify-drivers/image.png)
<figcaption>*Extreme class prediction mean SHAP value percentage.*</figcaption>
</figure>

