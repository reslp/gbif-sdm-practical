# Practical session: GBIF data cleaning, mapping and a simple SDM in R

This practical introduces a compact version of a common biodiversity-informatics workflow: download occurrence records, clean them, inspect them on a map, and use them to fit a simple species distribution model.

By the end of the session, you should understand what GBIF occurrence data represent, why raw biodiversity records need careful checking, how to make a basic occurrence map in R, and how a simple climate-based SDM can be fitted from presence records and background points. The model used here is deliberately simple. It is good for teaching the conceptual workflow, but it is not meant to be publication-ready.

Despite that, this practical mirrors the broader logic of a project using publicly available species occurence data. Species distribution models can be powerful tools for ecological explanation and spatial prediction, but their results depend strongly on data quality, predictor choice, sampling design and evaluation strategy (Elith & Leathwick 2009).

## Software and R packages

You will work in [Jupyterhub](https://jupyter.org/hub) running on a cluster. The R code performing most of the work is provided as a Juypter Notebook. 

Follow the instructions given in the course to log in.


You can get everything you need like this. We will see how to get to this data inside the Jupyterhub.

```
git clone https://github.com/reslp/gbif-sdm-practical.git
```

The `data/` folder is where the downloaded GBIF file should be placed. The `results/` folder is where the cleaned table, maps, rasters and model summary will be written.


The notebook `gbif_sdm_practical.ipynb` walks through the full practical workflow: loading a GBIF occurrence file, applying basic and optional `CoordinateCleaner` filters, mapping cleaned records in Austria, extracting climate predictors, fitting a simple logistic-regression SDM with presence and background points, and saving/plotting continuous and binary suitability maps.

## Hint:

This has been tested with this:

```
https://github.com/reslp/jupyterhub-singularity
```

## What is GBIF?

GBIF, the **Global Biodiversity Information Facility**, is a large international infrastructure for sharing biodiversity occurrence records. A single GBIF occurrence record usually represents a statement that a particular organism was observed or collected at a particular place and time. These records may come from museum collections, herbaria, monitoring schemes, research projects, citizen-science platforms or institutional databases.

The GBIF website is here: [https://www.gbif.org/](https://www.gbif.org/).

A typical GBIF table contains the scientific name, latitude and longitude, country, year, source dataset, basis of record, coordinate uncertainty and many additional fields. These columns make GBIF extremely useful for ecological and biogeographical work, because they allow researchers to combine individual records and analyze spatial patterns. 

At the same time, GBIF data are heterogeneous. Records may have missing coordinates, duplicated coordinates, imprecise localities, taxonomic problems, old names, geographic errors or uneven sampling effort. These problems do not make GBIF unsuitable; they simply mean that cleaning and critical inspection are part of the analysis, *not* optional extras.

This is especially important for species distribution modelling. Presence-only data are often **spatially biased** because people collect and observe organisms more often near roads, cities, protected areas, universities or well-studied regions. Phillips et al. (2009) showed that this kind of sample selection bias can strongly affect presence-only models because the model may *learn* the geography of sampling effort rather than the ecology of the species.


## Part 1: Download occurrence data from GBIF

Open [https://www.gbif.org](https://www.gbif.org), search for a species of interest, switch to the occurrence records, and filter the records to Austria. For this practical, the easiest download format is **Simple CSV**. Save the downloaded file into `gbif-sdm-lecture/data/`.

>[!CAUTION]
>To download data from Gbif you need to have an account. You can also connect some (e.g. Github, ORCID, Google) other accounts if you already have one. If you don't to create an account now, we already provide occurence data for some species for you (see blow).

A good teaching species is one with enough Austrian records to survive cleaning. The SDM script requires at least 20 cleaned records, but more records are better because very small datasets make the model unstable and difficult to interpret. 

### Three examples
1) A widespread lichen such as [*Hypogymnia physodes*](https://en.wikipedia.org/wiki/Hypogymnia_physodes) is a reasonable default example. In case you want to follow, the example with *Hypogymnia physodes*, we provide occurence data for you in `data/`.

2) A second example is [*Lasius niger*](https://en.wikipedia.org/wiki/Black_garden_ant), the black garden ant. Also the occurence data for Austria for this species is given in `data/`.

3) The third provided species is [*Pinus cembra*](https://en.wikipedia.org/wiki/Pinus_cembra), Austrian stone pine. You can also find Gbif occurence data in `data/`

>[!TIP] 
>For the purpose of this session, and if you want to download data, it is better to keep the first download small. Filtering for occurence only from Austria before downloading makes everything run faster and reduces the chance you spend most of the session waiting for files to import.


## Part 2: Evaluate and clean GBIF occurrence data

The first part when working with GBIF data is to perform some basic data cleaning. This is necessary because GBIF records can be incomplete, inaccurate or simply wrong. One could, of course, inspect the occurrence table manually and remove problematic entries by hand, but as soon as you have more than about 100 records this becomes unfeasible very quickly and it is also not reproducible.

In the exercise you can compare two cleaning approaches. Both approaches first apply the same basic filters: they keep the focal species, keep Austrian records, require valid longitude and latitude columns, remove records with very large coordinate uncertainty, keep common occurrence record types, and optionally remove very old records.

The first cleaning approch is simple. It removes coordinates at `0, 0`, keeps only records inside a simple Austria bounding box, and removes duplicated coordinate pairs for the same species. The central standard-cleaning operation is:

A more realistic approach uses the rOpenSci package [CoordinateCleaner](https://ropensci.github.io/CoordinateCleaner/). CoordinateCleaner runs a set of spatial tests that flag common problems in biological collection data, including country centroids, capital coordinates, biodiversity institutions, the GBIF headquarters (you would be surprised how many samples have these coordinates), coordinates in the the middle of the ocean, plain zero coordinates, equal longitude and latitude, duplicated coordinates and spatial outliers. 

The output of CoordinateCleaner contains one flag column per test. In these columns, `TRUE` means that a record passed the test and `FALSE` means that it was flagged as potentially problematic. 

Change the respective setting to `"coordinatecleaner"` if you want the map and SDM scripts to use the CoordinateCleaner result instead of the basic cleaning method.

The simple bounding-box method is easy to understand, while CoordinateCleaner is more systematic and catches common coordinate artefacts that are easy to miss manually. Neither approach removes the need for biological judgement.


## Part 3: Plot occurrence data in R with ggplot2

A map is more than just a pretty figure. It is a quality-control step. You should check whether the records are actually in Austria, whether they form suspicious clusters, whether there are still outliers, and whether the distribution reflects plausible biology or obvious sampling structure. Dense clusters around cities or well-sampled regions may indicate observation effort rather than species ecology. This is one reason why occurrence maps should be inspected before any model is fitted.

Of course what you will see will depend on the species you choose to analyze.


## Part 4: Build a simple species distribution model

After we have verified the occurence records, we can start to build a simple species distribution model (SDM).

A species distribution model links observed occurrences to environmental conditions and then predicts where similar conditions occur in geographic space. In this practical, the model is kept simple so that you can understand every component. 

Our model uses cleaned GBIF records as presences, samples random background points within Austria, downloads current WorldClim bioclimatic variables through `geodata`, keeps annual mean temperature (`bio1`) and annual precipitation (`bio12`), and fits a logistic regression with `glm`.

The model formula is:

```r
sdm <- glm(pa ~ bio1 + bio12, data = model_dat, family = binomial())
```

Here, `pa = 1` marks the cleaned GBIF occurrence records and `pa = 0` marks the background points. It is important to be precise about this: the background points are not confirmed absences. They are random locations used to describe the available environment in Austria. 

Phillips et al. (2009) explains that background or pseudo-absence choices strongly influence presence-only models, especially when occurrence data are spatially biased.

Here, random background points are acceptable because they make the idea clear. In a research analysis, one would think much harder about the accessible area, sampling bias, target-group background, spatial thinning and model evaluation.

>[!CAUTION] 
>Think about the species you chose, can you think of any reasons how the biologoy of the species could violate the assumptions about the model?

The two predictors are also chosen for simplicity rather than completeness. Annual mean temperature (`bio1`) and annual precipitation (`bio12`) are easy to interpret and often ecologically relevant, but real species distributions may depend on seasonality, extremes, substrate, land use, forest structure, dispersal limitation, biotic interactions and historical factors and more. Elith & Leathwick (2009) write that SDMs are strongest when predictors have **a plausible ecological link** to the species, not merely when they improve statistical fit.

After fitting the model, we can predict climatic suitability across Austria and create both a continuous probability map and a binary map. The binary map uses the 10th percentile of the predicted values at presence points as a simple threshold.

Such thresholds are useful because they turn a continuous prediction into a simple map of relatively suitable and less suitable areas. They are also dangerous if interpreted too strongly. A threshold does not magically separate true presence from true absence, and different thresholds can produce different maps. For research, threshold choice should be justified, sensitivity should be checked, and the continuous prediction should usually be inspected alongside the binary result.

## How to interpret the SDM output

The probability map shows the modelled relationship between the occurrence records and the two climate predictors. Areas with high predicted values are places where the climate resembles the climate at the cleaned occurrence points, relative to the background points sampled across Austria. This should be described as **modelled climatic suitability**, not as a guaranteed distribution map.

Several caveats matter. GBIF records are not collected with a balanced sampling design, so dense clusters of records may reflect observer activity.

The model uses only two predictors, so it ignores many ecological drivers. These drivers may also be different for different species. Random background points are a simple teaching device, not a carefully designed sampling-bias correction. The model has no robust validation step. It does not evaluate transferability, uncertainty, or alternative algorithms. It also assumes that the cleaned occurrence records and the selected climate variables are enough to approximate the ecological niche, which is rarely fully true.

Roberts et al. (2017) explain why evaluation is especially difficult for spatial ecological data. Random train-test splits can give over-optimistic performance estimates when nearby points are environmentally and spatially similar. Spatial or otherwise structured cross-validation is often more appropriate when the goal is to estimate how well a model transfers to new places, new times or new sampling situations.

Recent work also warns against judging SDMs only by numerical performance metrics. Fiorentino et al. (2025) show that models can perform well according to common metrics while still producing ecologically implausible responses or poor climate-change extrapolations. For this reason, students should look at the map, the model summary and the ecological plausibility of the response, not only at whether the script runs successfully.

## References cited

Elith, J. & Leathwick, J. R. 2009. Species distribution models: ecological explanation and prediction across space and time. *Annual Review of Ecology, Evolution, and Systematics* 40: 677–697. https://doi.org/10.1146/annurev.ecolsys.110308.120159

Fiorentino, D., Núñez-Riboni, I., Pierce, M. E., Oesterwind, D., Ammar, I. A., Bastardie, F., Chust, G., Janßen, H., Petza, D., Ter Hofstede, R., Tserpes, G., Virtanen, E., Wisz, M. S., Wright, P. J. & Palialexis, A. 2025. Improving species distribution models for climate change studies: ecological plausibility and performance metrics. *Ecological Modelling* 508: 111207. https://doi.org/10.1016/j.ecolmodel.2025.111207

Phillips, S. J., Dudík, M., Elith, J., Graham, C. H., Lehmann, A., Leathwick, J. & Ferrier, S. 2009. Sample selection bias and presence-only distribution models: implications for background and pseudo-absence data. *Ecological Applications* 19: 181–197. https://doi.org/10.1890/07-2153.1

Roberts, D. R., Bahn, V., Ciuti, S., Boyce, M. S., Elith, J., Guillera-Arroita, G., Hauenstein, S., Lahoz-Monfort, J. J., Schröder, B., Thuiller, W., Warton, D. I., Wintle, B. A., Hartig, F. & Dormann, C. F. 2017. Cross-validation strategies for data with temporal, spatial, hierarchical, or phylogenetic structure. *Ecography* 40: 913–929. https://doi.org/10.1111/ecog.02881
