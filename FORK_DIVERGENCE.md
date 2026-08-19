# Fork Divergence Summary: `Springbok-LLC/nlm-ckn` vs `NIH-NLM/nlm-ckn`

_Generated 2026-07-17, revised 2026-08-19 after the respiratory-system UBERON root
reclassification. Compares `origin/main` (Springbok-LLC fork) against `upstream/main` (NIH-NLM)._

## Headline

The fork has **not** structurally diverged. `origin/main` contains all of `upstream/main`'s
history (0 commits behind) and sits **64 commits ahead** — 31 functional commits plus 33 merge
commits from regular upstream syncs. The merge base is current (`ec287fb3`, 2026-08-05).

**Nearly every divergent change is confined to `data/prod/`.** The exceptions are
`.github/pull_request_template.md` (added 2026-07-11), this file, the cell-count reconciliation
analysis (`CELL_COUNT_RECONCILIATION.md`, `cell_count_reconciliation.csv`,
`tools/reconcile_cell_counts.py`), and three CI workflow commits from 2026-08-11/12 (`docs.yml`,
`package-prod-data.yml`, `promote-to-upstream.yml`; see §10). Otherwise no code or tooling differs
on `main`:

```
257 files changed, 1751 insertions(+), 2186140 deletions(-)
  151 deleted   15 modified   11 added   80 renamed
  249 under data/prod/  +  8 process, analysis and workflow files
```

The line counts are almost entirely deletions, and 2,182,400 of the 2,186,140 come from the single
superseded retina Li subset run (`results/2026-aug-01-a56002/`, see §3 and §5) — four SVG plots and
one per-cell silhouette CSV account for most of it. Insertions total 1,751. The curation surface is
much smaller than the raw numbers suggest.

So this is a *data-hygiene* divergence (plus process and CI files), not an architectural one.

## The one point of real friction

Upstream keeps **reintroducing files the fork deletes.** Concretely:

- The fork removed all **non-reference `cluster_cid_mapping` files** (placeholder copies under
  `results/<dataset>/` with a blank `manual_mapped_cid`).
- **Today: fork has 0, upstream still has 82** of these in `results/` (verified 2026-08-19 against
  `origin/main` and `upstream/main` at `ec287fb3`). Both sides carry the same 11 `reference/` ones.
- They keep coming back. The PR #32 upstream merge included a *revert* of an upstream
  "Remove redundant results" change, which resurfaced 9 under `heart_plus_pericardium/`; the fork
  had to re-delete them (commit `71b44807`). Upstream PR #261 then shipped an 82nd for retina Li,
  re-deleted in `67740ac6`; the 2026-08-05 retina publish shipped an 83rd. Every new pipeline
  publish adds one.

**This is the thing to settle with the maintainer:** whether non-reference `cluster_cid_mapping`
files should exist at all. If upstream agrees they're redundant, the fork's deletions should be
adopted upstream so each sync stops re-litigating them. Both sides agree on the 11 `reference/`
mapping files — those are the canonical ones.

## The functional changes, grouped

**0. Process files** (`93c0ef3c`, `760bad07`) — see also §10 for the CI workflow changes
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

**Upstream has since converged on the fork's position.** PR #267 (`bded3d4c`, 2026-08-05) removed
`results/retina-Xu-Cell-2023-5188c1/` upstream — the same deletion the fork made in `61156d26`. PR
#265 (`d93b140d`) replaced the bogus `homo_sapiens_retina_li*.csv` rows with
`homo_sapiens_retina_v2_harvester_li.csv`, carrying correct Li provenance under
`dataset_version_id` `25f1558d-…-b778d8cbd4aa`. This was the largest open item in this document and
it is now settled in both repositories.

**The dataset has also been re-run on the full object.** The 2026-08-05 publish analyzes
`li_et_al_2023_BC_RGC_RPE.combined.h5ad` (3,144,969 cells) where the 2026-08-01 publish used a
5,000-cells-per-cluster subset (383,129 cells) — same `dataset_version_id`, same 135 post-filter and
140 pre-filter clusters. The subset run is removed as superseded, and the reference mapping is
re-tokened from `2026.0` to `2026` to follow. The `.0` year artifact is fixed upstream this round.

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
  out again. The 2026-08-05 publish did it a third time (`results/2026-aug-05-9e4c71/`). No dated
  run folder exists on either `origin/main` or `upstream/main` otherwise, so the pipeline appears
  to emit them intermittently rather than by design.

Note on naming: `retina-Li-Nat Genet-2026-cbd4aa` keeps upstream's literal directory name, space
included. Spaces are **existing repo convention**, not slips — 44 of 85 results directories contain
them. The `.0` year artifact that the 2026-08-01 publish carried was fixed upstream in the
2026-08-05 publish; `respiratory_system-Guo-Nat Commun-2023.0-3058a2` is now the only folder left
with one. Note the fix is downstream of the harvester, not at the source: the v2 harvester rows
still record `year = 2026.0`, so the artifact can reappear on the next publish.
Normalizing the space here would make retina the outlier and re-litigate at every sync. The
space in paths is still a hazard for unquoted shell/Nextflow glue; that is worth raising upstream
as a repo-wide change, not a fork-local one.

**6. Smaller fixes** (`42b4d911`, `613751b7`, `6b36ebcd`)
- Corrected `h5ad_url` column value; added original mapping files; added missing
  respiratory-system files.

**7. Path consolidation — `neocortext/` → `neocortex/`** (`a5d902cd`, and the 2026-08-05 merge)
- `a5d902cd` originally corrected the misspelled `neocortext/` path. Upstream PR #265 re-created it,
  adding `neocortext/cellxgene-harvester/` and a second copy of the neocortex reference mapping, so
  `origin/main` briefly carried that mapping twice.
- Consolidated back under `neocortex/`: the harvester directory moved, the duplicate mapping removed,
  and the six `homo_sapiens_neocortext_*` basenames renamed to match the `homo_sapiens_<organ>_*`
  pattern every other organ uses (`uberon_neocortex.*` was already correct).
- Dropping the duplicate is safe — across all 153 rows the two files agree on `cluster_name`,
  `dataset_version_id` and `skos`, and the fork's `manual_mapped_cid` equals upstream's
  `cell_ontology_id` exactly. Upstream's copy carries one column the fork's does not,
  `cell_ontology_term` (human-readable labels); the uniform 5-column schema from §1 is kept and
  adopting labels repo-wide is listed as a talking point below.

**8. Cell count reconciliation** (`06a530ea`) — analysis, not curation
- Added `tools/reconcile_cell_counts.py`, `cell_count_reconciliation.csv` and
  `CELL_COUNT_RECONCILIATION.md`, comparing harvester `normal_cell_count` against summary
  `filtered_cell_count` across every organ. See that file; two findings are upstream-facing and
  appear as talking points below.

**9. UBERON root/descendant classification — respiratory system** (`523659f3`) — 2026-08-19
- The respiratory-system harvester is the only one of the ten organs run with **two** UBERON
  queries (`respiratory system`, `nose`); every other organ has a single query. Upstream's output
  records both as roots, so `uberon_respiratory_system.{csv,json}` carries `UBERON:0000004` (nose)
  at `level = root` and lists two entries in `root_terms`.
- The fork reclassifies nose as a **descendant** of `UBERON:0001004` (respiratory system):
  `root_terms` drops to the single respiratory-system entry, and the nose row's `level` changes
  from `root` to `descendant` in both files. This matches UBERON, where nose is part of the
  respiratory system, and makes all ten organ files single-rooted.
- **No terms are added or removed.** The `obo_ids` list, `terms` list and `total` (660) are
  unchanged, as is the CSV's row set; the entire diff is one CSV field, five deleted JSON lines,
  one JSON field, and a trailing newline. The nose subtree was already present in upstream's
  output, so nothing downstream of the term list changes.
- This will be reintroduced on the next harvester run unless the query configuration or the
  harvester's root-assignment logic is changed upstream.

**10. CI workflow changes** (`c01fec8b`, `00ca7a55`, `651ea296`) — 2026-08-11/12

The only divergence outside `data/prod/` and the process/analysis files. All three modify workflows
that already exist upstream; none adds or removes a workflow file.

*a. `docs.yml` — gate GitHub Pages to the upstream* (`c01fec8b`)
- The `Setup Pages` and `Upload artifact` steps and the whole `deploy` job now carry
  `if: github.repository_owner == 'NIH-NLM'`. `configure-pages` hits the Pages REST API, which
  404s on a fork that has Pages disabled, so every push to fork `main` failed the docs run.
- **Upstream behavior is unchanged** — NIH-NLM still builds and publishes the site. This is purely
  additive gating and is the safest of the three to adopt upstream.

*b. `package-prod-data.yml` — drop the per-author "small" packages* (`c01fec8b`)
- Removes the `discover-authors` and `package-small` jobs (~55 lines). `discover-authors` scanned
  every `*master_dataset_summary.csv` for distinct `first_author` values; `package-small` fanned out
  over that matrix, tarring each `data/prod` directory matching an author name with `.html`, `.svg`
  and `.pkl` excluded, and attaching each tarball to the release.
- Only the single full `prod-data-<tag>.tar.gz` remains. The workflow fires on `release: published`.
- **This is the one with real upstream consequence.** Unlike (a) it is not gated by owner, so
  adopting it means upstream releases stop carrying per-author tarballs. Worth confirming nobody
  downstream consumes them before this is upstreamed.

*c. Action version bumps* (`c01fec8b`) — `checkout` v4→v7, `setup-python` v5→v7, `configure-pages`
v4→v6, `upload-pages-artifact` v3→v5, `deploy-pages` v4→v5, `softprops/action-gh-release` v2→v3,
to clear a Node 20 deprecation warning.

*d. `promote-to-upstream.yml` — two robustness fixes* (`00ca7a55`, `651ea296`)

This workflow exists in both repos but is gated `if: github.repository_owner == 'Springbok-LLC'`, so
it is inert upstream and only ever runs on the fork. Both fixes therefore change fork behavior only,
even though the file itself now differs in the two repositories.
- `00ca7a55` replaces the GitHub compare API (`repos/NIH-NLM/nlm-ckn/compare/main...Springbok-LLC:main`)
  with a local divergence count. The compare endpoint always materializes a diff and, on a repo
  carrying `data/prod`, returns HTTP 422 "Sorry, this diff is taking too long to generate". The
  replacement does a bare `git init` plus two treeless partial fetches (`--filter=tree:0 --no-tags`)
  of the two tips and reads `git rev-list --left-right --count` — a few hundred KB, no working tree.
- `651ea296` makes PR creation idempotent. A 502 from the API gateway (run `31621460361`) let a
  create succeed server-side while reporting failure to the client; the retries then hit 422
  "already exists" and failed the job over a PR that had in fact been opened. The lookup is factored
  into a `find_existing_pr()` helper and re-checked after a failed attempt, exiting 0 if the PR
  exists; the retry loop also breaks on attempt 3 rather than sleeping before a retry it won't make.

## Suggested talking points for the maintainer

1. **Adopt the reference-mapping schema normalization upstream** so all 11 files share one schema
   (incl. `dataset_version_id`) — low-risk, mostly additive.
2. **Decide the fate of non-reference `cluster_cid_mapping` files.** If redundant (the fork's
   position), delete upstream and stop reverting their removal — this is the only recurring merge
   friction.
3. ~~**Fix the retina metadata cross-join at the source.**~~ **Done** — upstream PRs #265 and #267
   removed the mislabeled Xu folder and replaced the bogus Li harvester rows. Still worth checking
   whether other organs share a token the way `5188c1` did.
4. **Investigate `respiratory_system-Guo`.** Its run analyzed exactly `total_cell_count` (347,970)
   rather than `normal_cell_count` (332,335) despite `filter_normal = True`, so 15,635 non-normal
   cells appear to be included in a run flagged as normal-filtered. The only such case in the repo.
   See [CELL_COUNT_RECONCILIATION.md](CELL_COUNT_RECONCILIATION.md).
5. **Reconcile the neocortex `dataset_version_id`s.** PR #265 added a neocortex harvester, but its
   five Jorstad rows (`ba88ab2b`, `7d67c77b`, `3bb9be9d`, `0b8a9eb3`, `8b7fdf9f`) match none of the
   five in the `master_dataset_summary` (`3280113b`, `c4959ded`, `0476ef54`, `8b09695a`, `f72b8175`).
   Neocortex is the last dataset that cannot be reconciled.
6. **Consider adopting `cell_ontology_term` repo-wide.** Upstream's neocortex mapping carries
   human-readable labels alongside the ontology IDs. Useful, but only one of the 11 reference files
   has them; adding them everywhere would be a schema change worth making deliberately.
7. **Stop the misspelled `neocortext/` path from returning.** The fork has now corrected it twice.
8. **Confirm the remaining curation calls**: the Jorstad recovery and the dated-run promotion
   convention.
9. **Consider adopting the PR template upstream** so contributions across both repos use the same
   `Closes #` convention.
10. **Make the respiratory-system harvester single-rooted at the source.** The `nose` query is
   recorded as a second root, but nose is part of the respiratory system in UBERON and every other
   organ file has exactly one root. Either drop the redundant query or have the harvester classify
   a query term that resolves under an existing root as a descendant — otherwise §9 has to be
   re-applied after every run.
11. **Upstream the `docs.yml` Pages gate** (§10a). It is a no-op on NIH-NLM and stops the docs
   workflow failing on every fork push — the cheapest of the CI items to settle.
12. **Decide whether per-author release tarballs are still wanted** (§10b). The fork has removed
   the `package-small` matrix; adopting that upstream changes what a release ships, so it needs a
   deliberate call rather than a silent sync.
13. **Upstream the two `promote-to-upstream.yml` fixes** (§10d). The workflow is inert on NIH-NLM,
   so adopting them costs nothing there and keeps the two copies of the file from drifting.
14. **Raise Git LFS before more organs land.** No `.gitattributes` exists anywhere. The retina Li
   publish alone shipped two 62 MB `.svg.tar.gz` files and four 3.7–7.4 MB pickles; `.pkl` is also
   version-fragile as an archival format. Cheaper to settle now than after the next ten datasets.

---

These are all upstreamable as PRs; nothing here is a hard fork.

## Appendix: divergent functional commits

| Commit | Date | Summary |
|---|---|---|
| `523659f3` | 2026-08-19 | Make nose a respiratory system descendant |
| `651ea296` | 2026-08-12 | promote-to-upstream: re-check for an existing PR before failing (#52) |
| `00ca7a55` | 2026-08-12 | promote-to-upstream: count divergence with git, not the compare API (#49) |
| `c01fec8b` | 2026-08-11 | Drop per-author packages; gate Pages to NIH-NLM; bump action versions (#48) |
| `06a530ea` | 2026-08-05 | Add cell count reconciliation analysis and refresh for #265/#267 |
| `b59dc212` | 2026-08-05 | Consolidate neocortext/ into neocortex/ |
| `36f022cd` | 2026-08-05 | Re-token retina reference mapping to the 2026 dataset token |
| `a9d97ca6` | 2026-08-05 | Remove retina Li subset run superseded by the full-dataset run |
| `f93180b8` | 2026-08-05 | Remove non-reference cluster_cid_mapping for retina Li full run |
| `a6e2f013` | 2026-08-05 | Promote retina Li full-dataset results out of dated run folder |
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
