# SDM based on Gbif Data

This repository contains the materials for an exercise to create a species distribution model using Gbif data.

The notebook `gbif_sdm_practical.ipynb` walks through the full practical workflow: loading a GBIF occurrence file, applying basic and optional `CoordinateCleaner` filters, mapping cleaned records in Austria, extracting climate predictors, fitting a simple logistic-regression SDM with presence and background points, and saving/plotting continuous and binary suitability maps.

## Hint:

This has been tested with this:

```
https://github.com/reslp/jupyterhub-singularity
```
