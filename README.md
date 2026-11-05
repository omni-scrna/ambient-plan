# ambient-rna-benchmark

This repo contains a benchmark plan which combines modules for ambient RNA removal with some
existing `omni-scrna` modules, to test whether ambient RNA removal can be merged with the existing
benchmark. For now it only covers the mixed human/mouse HGMM datasets (no `be1`/non-ambient
datasets) and only `decontx`/no-cleaning (`CellBender` is wired in but disabled, see below).

**Experimental:** this plan uses `foreach` and `over`, features that only exist on
omnibenchmark's unreleased `feat/foreach-outputs` branch (requires `api_version: "0.8.0"`,
installed via `pip install git+https://github.com/omnibenchmark/omnibenchmark.git@feat/foreach-outputs`)

Stages: `data_download → cell_calling → cleaning → demultiplexing → concatenate_dataset`.
There is currently no `FILT` stage (see [Known issues](#known-issues)).

## `data_download` stage

One module, `d_hgmm`, downloads all 4 HGMM pools and decides the `pool_id` values at runtime via `foreach: pool_id` on its `rawdata_h5ad` output (`path: "{pool_id}/raw.h5ad"`) — omnibenchmark discovers the pool directories by globbing after the module runs, and every stage below fans out automatically, one node per discovered pool. `properties_info` (`mixed_species`/`batch_var`/`sample_var`/`labels_var`) is a single output shared across all pools, since those values don't vary by pool for this dataset.

## `cell_calling` stage

One module, `emptydrops`, runs `EmptyDrops` per pool and writes the called cell barcodes. Species
assignment does *not* happen here (see `demultiplexing` below) — this stage is pool-count-agnostic
and dataset-property-agnostic by design.

## `cleaning` stage

Two modules:
- `decontx` — runs DecontX, either in default mode or using the raw matrix as background
  (`use_background: true|false`), on the called cells.
- `no_cleaning` — subsets the raw counts to the called-cell barcodes with
  no correction applied.

Both just output the (corrected or uncorrected) matrix as `filtered_h5ad` — no dataset-level
metadata, no species labels.

A third option, `cell_calling_and_cleaning` (`CellBender`, which does both calling and ambient
removal in one step), is implemented but commented out in `benchmark_conda.yaml` pending GPU
testing (`requires_capabilities: [gpu]`).

## `demultiplexing` stage

One module, `dm_mixed_species`, assigns each called cell to human/mouse/ambiguous based on the
ratio of `GRCh38_`/`GRCm39_`-prefixed gene counts — **in the cleaned matrix**, i.e. species
assignment runs after ambient removal, not on raw counts. Writes a `(pool_id, cell_id, species,
majority_ratio)` TSV with species assignments;

## `concatenate_dataset` stage

A stage with `over: pool_id` that closes the `foreach`: one node per upstream lineage (dataset
module × cell calling × cleaning variant × demultiplexing), each receiving every pool's
`filtered_h5ad` and `demultiplexed_tsv` (same sorted pool order for both flags), concatenating them
into one h5ad and joining species labels onto `obs` by `(pool_id, cell_id)`. With three cleaning
variants that is three `concatenate` nodes, four pools each.

## Known issues

- **`gather`/`group_by` was the wrong fan-in here** (fixed by `over: pool_id`). `group_by: <stage>`
  keeps that stage's module and merges every other axis, so `concatenate_dataset` received all 3
  cleaning variants of all 4 pools in one node and the `(pool_id, cell_id)` join blew up. `over`
  is the complement: it collapses only the captured pools. See omnibenchmark design 013 §3.3.
- **No `FILT` stage yet.** `FILT`'s script (`scrapper`) expects its h5ad input under the  id `rawdata_h5ad` (its schema    hardcodes `--rawdata_h5ad`), so re-adding it means renaming `concatenate_dataset`'s `dataset_h5ad` output to `rawdata_h5ad` (or adjusting `FILT`'s `inputs:`) to match.
