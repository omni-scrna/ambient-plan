# ambient-rna-benchmark

This repo contains a benchmark plan which combines modules for ambient RNA removal with some
existing `omni-scrna` modules, to test whether ambient RNA removal can be merged with the existing
benchmark. For now it only covers the mixed human/mouse HGMM datasets (no `be1`/non-ambient
datasets) and only `decontx`/no-cleaning (`CellBender` is wired in but disabled, see below).

**Experimental:** this plan uses two unreleased omnibenchmark features (both need
`api_version: "0.8.0"`): `foreach`/`over`, from the `feat/foreach-outputs` branch, and
`gather.split_by`, from `feat/gather-split-by`. The branch `integ/split-by-foreach` has both.

Stages: `DATA_RAW → CELLCALL → (CLUSTER, DEMUX, PREP) → AMBIENT → (AMBIENT_M, MERGE_M, CONCAT) → DOUBLET`.
The sections below still use the earlier names for some of them: `data_download` = `DATA_RAW`,
`cell_calling` = `CELLCALL`, `demultiplexing` = `DEMUX`, `concatenate_dataset` = `CONCAT`.
There is currently no `FILT` stage (see [Known issues](#known-issues)).

## `data_download` stage

One module, `d_hgmm`, downloads all 4 HGMM pools and decides the `pool_id` values at runtime via `foreach: pool_id` on its `rawdata_h5ad` output (`path: "{pool_id}/raw.h5ad"`) — omnibenchmark discovers the pool directories by globbing after the module runs, and every stage below fans out automatically, one node per discovered pool. `properties_info` (`mixed_species`/`batch_var`/`sample_var`/`labels_var`) is a single output shared across all pools, since those values don't vary by pool for this dataset.

## `cell_calling` stage

One module, `emptydrops`, runs `EmptyDrops` per pool and writes the called cell barcodes. Species
assignment does *not* happen here (see `demultiplexing` below) — this stage is pool-count-agnostic
and dataset-property-agnostic by design.

## `PREP` stage

The count matrix the ambient removal methods run on. One entrypoint, `prep` in `ambient-r`, used
by three modules with a `merge` parameter:

- `as_is` (`merge: none`) — the raw matrix and the called cells, unchanged. Every dataset.
- `merge_orthologs` (`merge: orthologs`) — human-mouse 1:1 orthologs (`babelgene`) summed into
  one feature; every other gene is dropped. `requires: {multiplexing: mixed_species}`.
- `merge_gems` (`merge: gems`) — the GEM × `probe_id` pseudo-cells of one physical GEM barcode
  summed into one column; called cell ids become GEM barcodes. `requires: {multiplexing: flex}`.

It writes `prepared_h5ad` and `prepared_cells_tsv` (the called cells, in the ids of the matrix).
The stage provides the lineage label `counts` (`raw`, `orthologs`, `gems`), so a later module can
gate on the variant with `requires`.

Every stage below `PREP` runs once per variant: a mixed-species or Flex dataset gets two runs of
each method, an HTO dataset one.

## `AMBIENT` stage

All modules read the `PREP` outputs, subset the matrix to the called cells and write the
(corrected or uncorrected) matrix as `filtered_h5ad`:

- `decontx` — DecontX, in default mode or with the whole matrix as background
  (`use_background: true|false`).
- `no_cleaning` — no correction applied.
- `scar`, `cellclear` — in `ambient-py`.

The merges used to live inside `run_decontx.R` only, so `no_cleaning` was not comparable with
it. With `PREP`, every method of one variant sees the same matrix.

`CellBender` (calling and ambient removal in one step) is implemented but commented out in
`benchmark_conda.yaml` pending GPU testing (`requires_capabilities: [gpu]`).

## `AMBIENT_M` stage

`ambient-metrics`, once per pool, variant and method: count inflation against the matrix the
method was given (`prepared_h5ad`), and the barnyard metrics for mixed-species pools. On the
`orthologs` variant the barnyard keys are empty: a merged feature belongs to neither species.

## `MERGE_M` stage

Does removing ambient RNA and then merging give what merging first gives? A gather over
`filtered_h5ad`:

```yaml
gather:
  from: [filtered_h5ad]
  group_by: DATA_RAW
  split_by: [CELLCALL, AMBIENT]
```

One node per dataset, cell caller and method + parameters. `PREP` is not named, so both of its
variants land in the same node. `group_by` alone cannot say this: it merges only what lies below
the stage it names, and `PREP` sits above `AMBIENT`.

The module (`ambient-metrics`, entrypoint `compare`) reads the manifest, pairs the members per
pool, folds the unmerged output into the merged ids and reports the difference. HTO has no merged
variant and gives `n_pairs: 0`.

## `demultiplexing` stage

One module, `dm_mixed_species`, assigns each called cell to human/mouse/ambiguous based on the
ratio of `GRCh38_`/`GRCm39_`-prefixed gene counts — **in the cleaned matrix**, i.e. species
assignment runs after ambient removal, not on raw counts. Writes a `(pool_id, cell_id, species,
majority_ratio)` TSV with species assignments;

## `concatenate_dataset` stage

A `gather` stage (`group_by: data_download`) that collects every pool's `filtered_h5ad` and
`demultiplexed_tsv`, concatenates them into one h5ad per logical dataset, and joins species labels
onto `obs` by `(pool_id, cell_id)`.

## Known issues

- **`group_by` conflates downstream method choices.** `group_by: data_download` only partitions by
  the ancestor module of `data_download` — it does not keep the `cleaning` stage's method choice
  (`decontx` theta=true/false vs. `no_cleaning`) as separate groups, so `concatenate_dataset`
  currently receives all 3 cleaning variants of all 4 pools mixed together and the join blows up.
- **No `FILT` stage yet.** `FILT`'s script (`scrapper`) expects its h5ad input under the  id `rawdata_h5ad` (its schema    hardcodes `--rawdata_h5ad`), so re-adding it means renaming `concatenate_dataset`'s `dataset_h5ad` output to `rawdata_h5ad` (or adjusting `FILT`'s `inputs:`) to match.
