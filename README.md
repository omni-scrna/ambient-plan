# ambient-rna-benchmark

This repo contains a benchmark plan which combines modules for ambient RNA removal with some existing `omni-scrna` modules to test whether ambient-rna removal can be merged with the existing benchmark. The plan consists of four stages: `data_download → cell_calling → cleaning → FILT`.

## `data_download` stage

This stage is split into 2 modules: `datasets_standard` and `datasets_ambient`. The latter uses code stored in the `1-data` repo to download **raw** mixed species (human and mouse) datasets from 10x. Each downloaded dataset represents one 10x run/one pool. Data for all pools is combined into one big matrix and stored as `h5ad`. For pre-filtered datasets the `datasets_standard` module writes an empty `h5ad` file.

## `cell_calling` stage

This stage is split into 2 modules: `emptydrops` and `no_cell_calling`. The former loops over pools to call cells in each pool using `EmptyDrops` and assign each cell in a pool to a species. `sample_id` is then defined as `{pool_id}_{species}`. It excludes datasets from the `datasets_standard` module. The `no_cell_calling` module writes empty files for all `dataset_standard` outputs and exludes `datasets_ambient` outputs.

## `cleaning` stage

This stage is split into 2 modules: `decontx` and `no_cleaning`. The former runs decontx on `datasets_ambient` either in default mode (`--use-background false`) or using the raw matrix as background (`--use-background true`) and returns the same outputs as the `DATA` stage of the main benchmark (`sample_id` values are currently use as cluster truths). The `no_cleaning` module downloads datasets which have already been filtered. 

## `FILT` stage

Is the same as the `FILT` stage in the main benchmark.

