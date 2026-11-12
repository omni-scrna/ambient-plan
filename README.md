# ambient-rna-benchmark

This repo contains a benchmark plan for ambient RNA removal methods, covering three dataset types, each with its own `demultiplexing` module:

- **mixed-species
- **HTO-hashed** 
- **Flex (probe-based)**

**Experimental:** this plan uses the `foreach`/`over` features that exist on omnibenchmark's unreleased `feat/foreach-outputs` branch (requires `api_version: "0.8.0"`, installed via `pip install git+https://github.com/omnibenchmark/omnibenchmark.git@feat/foreach-outputs`).

Stages: `data_download → cell_calling → demultiplexing → cleaning → concatenate_dataset → doublets`. 

## `data_download` stage

Three modules, one per dataset type, each deciding `pool_id` values at runtime via `foreach: pool_id` on the shared `rawdata_h5ad` output (`path: "{pool_id}/raw.h5ad"`):

- `d_mixed_species` (`hgmm_10x`) — downloads 4 HGMM pools. Adds `rowData$species` (from the `GRCh*_`/`GRCm*_`genome prefix on each gene name)
- `d_hto` (`human_pbmcs_hto_10x`) — downloads the TotalSeq-hashed PBMC pool, parses the cellranger-multi config CSV into a `sample_id, hto_id` lookup, and attaches it to an `sce` object as `metadata(sce)$pool_lookup`.
- `d_flex`(`mouse_cell_lines_flex_10x`) — downloads the Flex mouse cell-line pool. Each column in the raw matrix is `{16bp GEM barcode}{8bp probe barcode}`; probe barcode is mapped to `probe_id` using 10x's fixed probe barcodes lookup file; parses the cellranger-multi config CSV into a `sample_id, probe_id` lookup, and attaches it to an `sce` object as `metadata(sce)$pool_lookup`.

`properties_info` (`batch_var`/`sample_var`/`labels_var`/`tenx_assay`) is a single output shared
across all pools of a dataset, since those values don't vary by pool.

## `cell_calling` stage

One module, `emptydrops`, runs `EmptyDrops` per pool and writes a TSV with all barcode called as cells. Exception if Flex data - `emptyDrops` runs separately for each `probe_id` in a pool.

## `demultiplexing` stage

Contains 3 modules, one for each data type/module in the `data_download` stage:

- `dm_mixed_species` — cellranger's own barnyard algorithm: the Nth percentile of same-species-dominant counts sets a per-species threshold; a barcode clearing
  *both* the human and mouse thresholds is a multiplet. Output columns: `pool_id, cell_id, species, sample_id, multiplet` (`sample_id` is `"{pool_id}_{species}"`, `NA` for multiplets).
- `dm_hto` — Seurat's standard `HTODemux` workflow. Output columns: `pool_id, cell_id, hto_id, class, multiplet, sample_id` (`sample_id` is `NA` for both Doublet and Negative).
- `dm_flex` — `probe_id` (already in `colData`) is joined with `pool_lookup` so thet every probe is assigned to the original `sample_id`; if the same GEM was called as a cell for more than one `probe_id`, every `cell_id` sharing GEM barcode is flagged as a multiplet.

## `cleaning` stage

Two modules, both in `ambient-r`:
- `decontx` — runs DecontX, either in default mode or using the raw matrix as background(`use_background: true|false`), on the called cells.
- `no_cleaning` — subsets the raw counts to the called-cell barcodes with no correction applied.

At the moment `run_decontx.R` does two extra dataset-specific transforms to the raw matrix:

- **Ortholog merging**: finds human-mouse orthologs using the `babelgene` package and sums ortholog counts (the matrix only contains merged features, species-specific features are removed).
- **GEM merging**: collapses the raw matrix with GEM × `probe_id` pseudo-cells down to one column per physical GEM barcode before.

`no_cleaning` does neither of these at the moment so the output isn't directly comparable in shape to `decontx`'s output.

## `concatenate_dataset` stage

An `over: pool_id` stage that collects every pool's `filtered_h5ad` and `demultiplexed_tsv` for one dataset.

## `doublets` stage

Runs after `concatenate_dataset`, on the whole merged dataset at once (not per pool). A single `scDblFinder()` call uses its own `samples = "pool_id"` argument so doublets are searched for independently within each pool.


