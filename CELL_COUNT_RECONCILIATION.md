# Cell Count Reconciliation: harvester `normal_cell_count` vs summary `filtered_cell_count`

_Generated 2026-08-03, revised 2026-08-05 against `merge/upstream-2026-aug-05-retina-li` (post
upstream PRs #265, #267, and the retina Li full-dataset publish). Regenerate with
`python3 tools/reconcile_cell_counts.py` from the repository root; results land in
`cell_count_reconciliation.csv`._

## Question

Every analyzed dataset has two independent cell counts:

- **`normal_cell_count`** — from `data/prod/<organ>/cellxgene-harvester/*.csv`. What the harvester
  expected to find: cells in the queried tissue with `disease == normal`.
- **`filtered_cell_count`** — from `master_dataset_summary_*.csv` in each results folder. What the
  NSForest run actually analyzed.

They should agree. This documents how far they actually do, across every organ and dataset.

## Method

Join on `dataset_version_id`, scoped to the same organ — the same dataset is harvested separately
per tissue, so `dataset_version_id` alone is not unique (21 IDs appear under more than one organ,
with legitimately different normal counts — 46 IDs now, up from 21 before PR #265 added the
neocortex and retina v2 harvesters).

One caveat shapes the join: **`normal_cell_count = 0` is a placeholder, not a measurement.** 269
harvester rows carry it, all in the `*_with_normal_counts.csv` files (91 respiratory system, 70
neocortex, 48 digestive tract, 31 liver, 13 bone marrow, 11 skin, 3 pancreas, 2 heart). Where one collides with an
analyzed dataset — 8 cases — the matching `*_final.csv` row carries the real number. The script
therefore prefers the largest non-zero value and records every candidate in
`harvester_normal_values` so the choice stays auditable. A naive join that takes the
`_with_normal_counts` value reports a spurious 100% shortfall on those 8.

## Result

89 summary rows across 85 `master_dataset_summary` files; 84 join to a harvester row in the same
organ.

| Comparison | Rows |
|---|---|
| `filtered_cell_count` **==** `normal_cell_count` | 54 |
| short by 1–29 cells (median 6.5, worst 99.03% of normal) | 28 |
| **exceeds** `normal_cell_count` | 2 |
| no harvester row to join against | 5 |

The two sources agree exactly about two-thirds of the time. Where they disagree, the gap is almost
always a handful of cells. Two datasets overshoot, described below; only one of those looks like a
defect.

### The pipeline side is internally consistent

Before comparing across sources, the results files agree with each other perfectly:

- `filtered_cell_count` == `summary_normal.n_obs` — **84/84**
- `filtered_cell_count` == sum of post-filter cluster sizes — **84/84**

So `filtered_cell_count` is a faithful record of what each run analyzed, and any disagreement with
the harvester is a genuine cross-source difference rather than a bookkeeping error inside a run.

### Why the small shortfalls happen

Splitting on whether the run removed any cells at all — `summary_before_filter.n_obs` vs
`filtered_cell_count` — separates two populations:

| Population | Datasets | Exact | Short | Exceeds |
|---|---|---|---|---|
| Run removed nothing (already wholly normal upstream) | 30 | 29 | 0 | 1 (Guo) |
| Run filtered down to the normal subset | 54 | 25 | 28 | 1 (retina Li) |

Every shortfall is in the second group, and **every shortfall dataset has a small minimum
post-filter cluster size** (5–21 cells), while exact-match datasets range up to 1,696. The clearest
cases are the single-cell deltas:

| Dataset | delta | min cluster size after |
|---|---|---|
| `bone_marrow-Wells-Nat Immunol-2025-74e9e1` | −1 | 21 |
| `kidney-Stewart-Science-2019-de3575` | −1 | 10 |
| `kidney-Young-Science-2018-57f337` | −1 | 10 |
| `liver-Wells-Nat Immunol-2025-74e9e1` | −1 | 5 |

This is consistent with a **minimum-cells-per-cluster threshold** discarding clusters left with only
a few cells once the normal-disease filter is applied: the harvester counts cells, the pipeline
counts cells *in surviving clusters*.

**This explanation is inferred, not confirmed.** No configuration file in the repository records a
minimum cluster size, so the threshold value could not be read directly — the evidence is the
uniform association between shortfalls and small surviving clusters across all 28 cases. Confirming
it requires the `sc-nsforest-qc-nf` pipeline configuration.

## The two discrepancies

Two datasets analyze *more* cells than the harvester says exist in the normal subset. They have
different causes and only one looks like a defect.

### `respiratory_system-Guo-Nat Commun-2023.0-3058a2` — probable filter failure

| Field | Value |
|---|---|
| `normal_cell_count` (harvester) | 332,335 |
| `total_cell_count` (harvester) | 347,970 |
| `filtered_cell_count` (summary) | **347,970** |
| `filter_normal` (harvester) | `True` |
| `disease` (summary) | `normal` |
| `summary_before_filter.n_obs` | 347,970 — the run removed nothing |

The run analyzed exactly the *total* count, not the normal count. **15,635 non-normal cells appear
to have been included in a run flagged as normal-filtered**, and the resulting NSForest markers,
f-scores, and silhouette values are computed over that mixed population.

This is worth raising upstream. The same Guo dataset also carries the `2023.0` year artifact in its
folder and file tokens, so it may have taken an unusual path through the harvester.

### `retina-Li-Nat Genet-2026-cbd4aa` — probably benign

| Field | Value |
|---|---|
| `normal_cell_count` (harvester v2) | 3,137,294 |
| `total_cell_count` (harvester v2) | 3,177,310 |
| `filtered_cell_count` (summary) | **3,144,969** |
| `summary_before_filter.n_obs` | 3,214,213 — *more than the cellxgene total* |
| delta | **+7,675 (100.24%)** |

Unlike Guo, this run did remove cells (140 pre-filter clusters → 135), and it overshoots the normal
count by only 0.24%. The likely explanation is the input: the run reads a custom
`li_et_al_2023_BC_RGC_RPE.combined.h5ad` whose pre-filter population (3,214,213) *exceeds* the
cellxgene release's `total_cell_count` (3,177,310) by 36,903 cells. A custom object holding cells the
public release does not would explain a filtered count slightly above the harvester's normal count,
with no filter defect involved.

**This is a hypothesis, not a conclusion** — confirming it requires comparing the custom h5ad against
the cellxgene release, which is outside this repository. Worth asking the maintainer to confirm the
provenance of the combined object.

## Datasets that cannot be checked

**Retina is now resolved.** Upstream PR #265 replaced the bogus `homo_sapiens_retina_li*.csv` rows
— which described the Xu/`Bone_marrow` dataset — with `homo_sapiens_retina_v2_harvester_li.csv`,
carrying correct Li provenance under `dataset_version_id` `25f1558d-…-b778d8cbd4aa`. Retina joins and
appears above.

**Neocortex still does not join, for a new reason.** PR #265 added a neocortex harvester, so the
provenance file now exists — but its five Jorstad rows carry `dataset_version_id`s that match none of
the five in the `master_dataset_summary`:

| Source | `dataset_version_id` prefixes |
|---|---|
| `homo_sapiens_neocortex_harvester_jorstad_final.csv` | `ba88ab2b`, `7d67c77b`, `3bb9be9d`, `0b8a9eb3`, `8b7fdf9f` |
| `master_dataset_summary_neocortex_Jorstad_…csv` | `3280113b`, `c4959ded`, `0476ef54`, `8b09695a`, `f72b8175` |

The summary rows also carry no `filtered_cell_count`. This dataset was recovered from tag
`v1.0.0-rc.4` rather than produced by a normal pipeline run, which likely explains both. Reconciling
neocortex needs the maintainer to confirm which dataset versions the recovered results correspond
to — a concrete upstream ask.

## Other harvester data-quality issues surfaced

1. **Two kinds of placeholder.** 269 rows carry `normal_cell_count = 0` (all in
   `*_with_normal_counts.csv`) and a further 213 carry an *empty* value — the latter concentrated in
   the index files `homo_sapiens_neocortex_harvester.csv` and `homo_sapiens_retina_v2_harvester.csv`
   added by PR #265. The script treats both as absent.
2. **Five genuine conflicts between harvester files, all in respiratory system.** Setting placeholders
   aside, five datasets get materially different normal counts from `_final.csv` and
   `_with_normal_counts.csv`, and `_final` is larger every time:

   | `dataset_version_id` | dataset | `_final` | `_with_normal_counts` | summary follows |
   |---|---|---|---|---|
   | `4cb45d80` | Sikkema Nat Med 2023 | 579,285 | 461,098 | `_final` (exact) |
   | `018a8104` | Xu Cell 2023 | 318,426 | 260,254 | `_final` (exact) |
   | `7c98dadf` | Wells Nat Immunol 2025 | 213,154 | 186,415 | `_final` (exact) |
   | `769fff4f` | Madissoon Nat Genet 2023 | 193,108 | 157,352 | `_final` (exact) |
   | `170408c8` | Berg J Cyst Fibros 2025 | 80,624 | 24,820 | `_final` (short by 14) |

   The consistent direction suggests the two files were generated against different tissue or disease
   criteria rather than one being stale. Every analyzed run follows `_final`, so the reconciliation is
   unaffected — but the pair should not be treated as interchangeable, and only respiratory system is
   affected.
3. **Kidney has no `*_with_normal_counts.csv`** file, and carries two near-identical finals:
   `homo_sapiens_kidney_harvester_final.csv` and the misspelled
   `homo_sapiens_kidney_havester_final.csv`, 9 rows each.

## Output columns

`cell_count_reconciliation.csv` — 89 rows, 21 columns, sorted worst-discrepancy first. Filter on
`status != 'unjoined'` for the 84-row comparison proper.

| Column | Meaning |
|---|---|
| `organ`, `results_folder`, `first_author`, `dataset_version_id` | identity |
| `harvester_files`, `harvester_normal_values` | every candidate row and its `normal_cell_count`, in file order — audit trail for the placeholder rule |
| `normal_cell_count`, `total_cell_count` | harvester side (normal = largest non-zero candidate) |
| `filtered_cell_count` | summary side |
| `delta`, `pct_of_normal`, `status` | `filtered − normal`; `exact` / `short` / `exceeds` / `unjoined` |
| `run_removed_cells` | `no` if `summary_before_filter.n_obs == filtered_cell_count` |
| `summary_before_filter_n_obs`, `summary_normal_n_obs`, `sum_cluster_sizes_after` | pipeline-side cross-checks |
| `n_clusters_before`, `n_clusters_after`, `min_cluster_size_after` | evidence for the threshold hypothesis |
| `filter`, `disease` | filter flag and disease composition from the summary |
