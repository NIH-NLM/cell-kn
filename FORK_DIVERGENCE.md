# Fork Divergence Summary: `Springbok-LLC/nlm-ckn` vs `NIH-NLM/nlm-ckn`

_Generated 2026-07-17, revised 2026-08-03 after merging upstream PR #261. Compares `origin/main`
(Springbok-LLC fork) against `upstream/main` (NIH-NLM)._

## Headline

The fork has **not** structurally diverged. `origin/main` contains all of `upstream/main`'s
history (0 commits behind) and sits **46 commits ahead** — 20 functional commits plus 26 merge
commits from regular upstream syncs (most recently `ab2cf808`, upstream PR #261). The merge base
is current (`ab2cf808`, 2026-08-02).

**Nearly every divergent change is confined to `data/prod/`.** The two exceptions are
`.github/pull_request_template.md` (added 2026-07-11) and this file. Otherwise no code, workflow,
CI, or tooling differs on `main`. The net effect is a data-curation/cleanup layer plus two process
files:

```
313 files changed, 2183490 insertions(+), 4368324 deletions(-)
  224 deleted   10 modified   79 added
  311 under data/prod/  +  .github/pull_request_template.md  +  FORK_DIVERGENCE.md
```

The line counts are dominated by one 351 MB duplicate results folder deleted on each side of a
rename (see §3); the curation surface is much smaller than the raw numbers suggest.

So this is a *data-hygiene* divergence (plus process files), not an architectural one.

## The one point of real friction

Upstream keeps **reintroducing files the fork deletes.** Concretely:

- The fork removed all **non-reference `cluster_cid_mapping` files** (placeholder copies under
  `results/<dataset>/` with a blank `manual_mapped_cid`).
- **Today: fork has 0, upstream still has 82** of these in `results/` (verified against current
  `origin/main` and `upstream/main`).
- They keep coming back. The PR #32 upstream merge included a *revert* of an upstream
  "Remove redundant results" change, which resurfaced 9 under `heart_plus_pericardium/`; the fork
  had to re-delete them (commit `71b44807`). Upstream PR #261 then shipped an 82nd for retina Li,
  re-deleted in `67740ac6`. Every new pipeline publish adds one.

**This is the thing to settle with the maintainer:** whether non-reference `cluster_cid_mapping`
files should exist at all. If upstream agrees they're redundant, the fork's deletions should be
adopted upstream so each sync stops re-litigating them. Both sides agree on the 11 `reference/`
mapping files — those are the canonical ones.

## The 20 functional changes, grouped

**0. Process files** (`93c0ef3c`, `760bad07`) — the only non-`data/prod/` changes
- Added `.github/pull_request_template.md` with a `Closes #` line (auto-closes the linked issue on
  merge, moving the board card to Done) plus "What & why" / "Notes for review" sections.
- Added this file (`FORK_DIVERGENCE.md`), the running record of what the fork changes and why.

**1. Schema normalization of reference mappings** (`a0d35e3b`, `598b1d2d`)
- Normalized all 11 `reference/cluster_cid_mapping` files to a uniform schema:
  `dataset_version_id,cluster_name,skos,manual_mapped_cid,cell_ontology_id`.
- Fixed a header typo (`manual:mapped:cid` → `manual_mapped_cid`, kidney Lake), added the
  `dataset_version_id` column (sourced from `cellxgene.json`, `external-1.0.0-rc.4`), and moved
  populated `cell_ontology_id` values into `manual_mapped_cid` for retina (140 rows — labelled Xu
  at the time, in fact Li; see §3) and neocortex Jorstad (153 rows).
- The 8 already-standard files are byte-identical apart from the new column.

**2. Removal of redundant non-reference mappings** (`cc30b3ba`, `71b44807`, `67740ac6`) — the
friction point above. 81 + 9 + 1 placeholder files removed; only `reference/` mappings retained.

**3. Retina identity correction** (`13dc6222`, then reversed by `61156d26`, `a69001b5`)

This one was called wrong the first time, and the correction is the most substantive thing in the
fork. **The retina results are Li's, not Xu's.**

- `13dc6222` read the retina folder as "Li superseded by Xu", deleted the Li harvester rows, and
  renamed the curated reference mapping from the Li token to the Xu token (`5188c1`). That had the
  direction backwards.
- Upstream PR #261 (`ab2cf808`, 2026-08-02) published `retina-Li-Nat Genet-2026.0-cbd4aa/` — the
  *same analysis* as the existing `retina-Xu-Cell-2023-5188c1/`, carrying correct Li metadata.
  Same source h5ad (`li_et_al_2023_BC_RGC_RPE.subset5000.h5ad`), 383,129 cells × 35,475 genes, 135
  clusters, byte-identical `cluster_order`/`cluster_sizes` before and after filter, identical
  `results_ensg`/`results_symbols`/`markers_ensg` once sorted, and identical silhouette and f-score
  values to full precision. Of the 69 shared files, 19 are byte-identical outright; the rest differ
  only by the embedded dataset token and plot-rendering nondeterminism.
- The Xu copy's metadata was **cross-joined from the bone-marrow dataset**: `dataset_title`
  `Bone_marrow`, tissue `UBERON:0002371`, bone-marrow cell types, DOI `10.1016/j.cell.2023.11.026`.
  That dataset already lives correctly at
  `data/prod/bone_marrow/sc-nsforest-qc-nf/results/bone_marrow-Xu-Cell-2023-5188c1/` — a genuinely
  different analysis (66,613 cells, 45 `Curated_annotation` clusters) that happens to share the
  `5188c1` token. The retina folder inherited its metadata from that row.
- **Decisive evidence:** the curated reference mapping's 140 `cluster_name` values are an *exact*
  match for the Li run's 140 pre-filter clusters — including five `RPE_*` clusters that the run's
  filter drops and that no other dataset in the repo has. The mapping always described Li.
- `61156d26` deletes the mislabeled `retina-Xu-Cell-2023-5188c1/` (69 files, 351 MB).
  `a69001b5` renames the reference mapping to the `retina_Li_Nat_Genet_2026.0_..._cbd4aa` token and
  updates `dataset_version_id` on all 140 rows from `ad9529a3-…-31dee15188c1` (Xu/bone marrow) to
  `25f1558d-…-b778d8cbd4aa` (Li). The §1 schema normalization is preserved; only identity moves.
- The Li harvester rows deleted in `13dc6222` stay deleted: upstream's
  `homo_sapiens_retina_li{,_subset5000}.csv` describe the Xu/`Bone_marrow` dataset under Li
  filenames and are the likely origin of the bad join.

**4. Recovery & layout fix — neocortex Jorstad** (`a5d902cd`, `30f20ee2`, `cb73642b`)
- Fixed a misspelled flat path (`neocortext/`) to the standard
  `neocortex/sc-nsforest-qc-nf/{reference,results}/` layout.
- Recovered per-cluster NSForest results (154 rows) from tag `v1.0.0-rc.4`; added a
  `master_dataset_summary` (5 superclusters) with `dataset_version_id`.
- Backfilled the missing `doi` column in that `master_dataset_summary` with
  `10.1126/science.adf6812` (Jorstad et al. 2023, *Science*). This file was the only summary using
  a reduced schema; all other summaries already carry `doi`. Placed before `collection_name` to
  match the canonical column order.

**5. Run-folder lifecycle** (`56b64cb1`, `f1fcb75c`)
- Promoted 9 heart results out of a dated run folder (`2026-jun-05-d97160/`) to sit directly under
  `results/`; pruned older dated runs (`heart/2026-may-05-ef5103`, `pancreas/2026-jun-05-2c08cf`).
- Upstream PR #261 reintroduced the pattern (`results/2026-aug-01-a56002/`); `f1fcb75c` promotes it
  out again. No dated run folder exists on either `origin/main` or `upstream/main` otherwise, so
  the pipeline appears to emit them intermittently rather than by design.

Note on naming: `retina-Li-Nat Genet-2026.0-cbd4aa` keeps upstream's literal directory name,
spaces and `.0` included. Both are **existing repo convention**, not slips — 44 of 85 results
directories contain spaces, and `respiratory_system-Guo-Nat Commun-2023.0-3058a2` sets the `.0`
precedent. Normalizing them here would make retina the outlier and re-litigate at every sync. The
space in paths is still a hazard for unquoted shell/Nextflow glue; that is worth raising upstream
as a repo-wide change, not a fork-local one.

**6. Smaller fixes** (`42b4d911`, `613751b7`, `6b36ebcd`)
- Corrected `h5ad_url` column value; added original mapping files; added missing
  respiratory-system files.

## Suggested talking points for the maintainer

1. **Adopt the reference-mapping schema normalization upstream** so all 11 files share one schema
   (incl. `dataset_version_id`) — low-risk, mostly additive.
2. **Decide the fate of non-reference `cluster_cid_mapping` files.** If redundant (the fork's
   position), delete upstream and stop reverting their removal — this is the only recurring merge
   friction.
3. **Fix the retina metadata cross-join at the source** (highest value). Upstream currently carries
   two copies of one retina analysis under two dataset identities, one of them wearing bone-marrow
   metadata. Ask upstream to delete `results/retina-Xu-Cell-2023-5188c1/` and the two
   `homo_sapiens_retina_li*.csv` harvester rows that describe the Xu/`Bone_marrow` dataset under Li
   filenames — those rows are the probable origin of the bad join, so leaving them invites a
   repeat. Worth checking whether other organs share a token the same way `5188c1` does.
4. **Confirm the remaining curation calls**: the `neocortext`→`neocortex` path fix + Jorstad
   recovery, and the dated-run promotion convention.
5. **Consider adopting the PR template upstream** so contributions across both repos use the same
   `Closes #` convention.
6. **Raise Git LFS before more organs land.** No `.gitattributes` exists anywhere. The retina Li
   publish alone shipped two 62 MB `.svg.tar.gz` files and four 3.7–7.4 MB pickles; `.pkl` is also
   version-fragile as an archival format. Cheaper to settle now than after the next ten datasets.

---

These are all upstreamable as PRs; nothing here is a hard fork.

## Appendix: divergent functional commits

| Commit | Date | Summary |
|---|---|---|
| `a69001b5` | 2026-08-03 | Re-token retina reference mapping to the Li cbd4aa dataset |
| `61156d26` | 2026-08-03 | Remove mislabeled retina Xu results (duplicate of retina Li cbd4aa) |
| `67740ac6` | 2026-08-03 | Remove non-reference cluster_cid_mapping for retina Li |
| `f1fcb75c` | 2026-08-03 | Promote retina Li results out of dated run folder |
| `1f74b5cc` | 2026-07-17 | Backfill missing doi in neocortex Jorstad master_dataset_summary |
| `760bad07` | 2026-07-17 | Add fork divergence summary (origin vs upstream) |
| `93c0ef3c` | 2026-07-11 | Add a PR template with a Closes line |
| `71b44807` | 2026-06-14 | Remove heart non-reference cluster_cid_mapping files (reintroduced upstream) |
| `598b1d2d` | 2026-06-13 | Move cell_ontology_id into manual_mapped_cid for retina Xu and neocortex Jorstad |
| `a0d35e3b` | 2026-06-13 | Normalize all 11 reference cluster_cid_mapping files to a uniform schema |
| `cb73642b` | 2026-06-13 | Add dataset_version_id column to neocortex Jorstad master_dataset_summary |
| `30f20ee2` | 2026-06-13 | Add master_dataset_summary for neocortex Jorstad (5 superclusters) |
| `a5d902cd` | 2026-06-13 | Recover neocortex Jorstad results_ensg from v1.0.0-rc.4; normalize layout |
| `13dc6222` | 2026-06-13 | Remove retina Li data (superseded by Xu); rename reference mapping to Xu |
| `cc30b3ba` | 2026-06-13 | Remove non-reference cluster_cid_mapping files |
| `56b64cb1` | 2026-06-13 | Promote heart results from dated run folder; prune older dated runs |
| `42b4d911` | 2026-06-12 | Use correct h5ad_url column value |
| `613751b7` | 2026-06-12 | Add original mapping files |
| `6b36ebcd` | 2026-06-12 | Add missing respiratory system files |
