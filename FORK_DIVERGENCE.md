# Fork Divergence Summary: `Springbok-LLC/nlm-ckn` vs `NIH-NLM/nlm-ckn`

_Generated 2026-07-17. Compares `origin/main` (Springbok-LLC fork) against `upstream/main` (NIH-NLM)._

## Headline

The fork has **not** structurally diverged. `origin/main` contains all of `upstream/main`'s
history (0 commits behind) and sits **37 commits ahead** — 13 functional commits plus 24 merge
commits from regular upstream syncs (most recently PRs #35 and #36, merging `NIH-NLM/main`). The
merge base is recent (`cd2041fd`, 2026-06-22).

**Nearly every divergent change is confined to `data/prod/`.** The one exception is a new
`.github/pull_request_template.md` (added 2026-07-11) — the first fork change outside `data/prod/`.
Otherwise no code, workflow, CI, or tooling differs on `main`. The net effect is a
data-curation/cleanup layer plus one process file:

```
105 files changed, 1062 insertions(+), 3811 deletions(-)
  85 deleted   11 modified   9 added
  104 under data/prod/  +  1 .github/pull_request_template.md
```

So this is a *data-hygiene* divergence (plus a PR template), not an architectural one.

## The one point of real friction

Upstream keeps **reintroducing files the fork deletes.** Concretely:

- The fork removed all **non-reference `cluster_cid_mapping` files** (placeholder copies under
  `results/<dataset>/` with a blank `manual_mapped_cid`).
- **Today: fork has 0, upstream still has 81** of these in `results/` (verified against current
  `origin/main` and `upstream/main`).
- They came back once already: the PR #32 upstream merge included a *revert* of an upstream
  "Remove redundant results" change, which resurfaced 9 under `heart_plus_pericardium/`. The fork
  had to re-delete them (commit `71b44807`).

**This is the thing to settle with the maintainer:** whether non-reference `cluster_cid_mapping`
files should exist at all. If upstream agrees they're redundant, the fork's deletions should be
adopted upstream so each sync stops re-litigating them. Both sides agree on the 11 `reference/`
mapping files — those are the canonical ones.

## The 13 functional changes, grouped

**0. Process: PR template** (`93c0ef3c`) — the only non-`data/prod/` change
- Added `.github/pull_request_template.md` with a `Closes #` line (auto-closes the linked issue on
  merge, moving the board card to Done) plus "What & why" / "Notes for review" sections.

**1. Schema normalization of reference mappings** (`a0d35e3b`, `598b1d2d`)
- Normalized all 11 `reference/cluster_cid_mapping` files to a uniform schema:
  `dataset_version_id,cluster_name,skos,manual_mapped_cid,cell_ontology_id`.
- Fixed a header typo (`manual:mapped:cid` → `manual_mapped_cid`, kidney Lake), added the
  `dataset_version_id` column (sourced from `cellxgene.json`, `external-1.0.0-rc.4`), and moved
  populated `cell_ontology_id` values into `manual_mapped_cid` for retina Xu (140 rows) and
  neocortex Jorstad (153 rows).
- The 8 already-standard files are byte-identical apart from the new column.

**2. Removal of redundant non-reference mappings** (`cc30b3ba`, `71b44807`) — the friction point
above. 81 + 9 placeholder files removed; only `reference/` mappings retained.

**3. Dataset supersession** (`13dc6222`)
- Retina **Li** data removed (superseded by **Xu**); reference mapping renamed from the Li token
  to the matching Xu token (hash `5188c1`).

**4. Recovery & layout fix — neocortex Jorstad** (`a5d902cd`, `30f20ee2`, `cb73642b`)
- Fixed a misspelled flat path (`neocortext/`) to the standard
  `neocortex/sc-nsforest-qc-nf/{reference,results}/` layout.
- Recovered per-cluster NSForest results (154 rows) from tag `v1.0.0-rc.4`; added a
  `master_dataset_summary` (5 superclusters) with `dataset_version_id`.
- Backfilled the missing `doi` column in that `master_dataset_summary` with
  `10.1126/science.adf6812` (Jorstad et al. 2023, *Science*). This file was the only summary using
  a reduced schema; all other summaries already carry `doi`. Placed before `collection_name` to
  match the canonical column order.

**5. Run-folder lifecycle** (`56b64cb1`)
- Promoted 9 heart results out of a dated run folder (`2026-jun-05-d97160/`) to sit directly under
  `results/`; pruned older dated runs (`heart/2026-may-05-ef5103`, `pancreas/2026-jun-05-2c08cf`).

**6. Smaller fixes** (`42b4d911`, `613751b7`, `6b36ebcd`)
- Corrected `h5ad_url` column value; added original mapping files; added missing
  respiratory-system files.

## Suggested talking points for the maintainer

1. **Adopt the reference-mapping schema normalization upstream** so all 11 files share one schema
   (incl. `dataset_version_id`) — low-risk, mostly additive.
2. **Decide the fate of non-reference `cluster_cid_mapping` files.** If redundant (the fork's
   position), delete upstream and stop reverting their removal — this is the only recurring merge
   friction.
3. **Confirm the curation calls**: retina Li→Xu supersession, the `neocortext`→`neocortex` path
   fix + Jorstad recovery, and the dated-run promotion convention.
4. **Consider adopting the PR template upstream** so contributions across both repos use the same
   `Closes #` convention.

---

These are all upstreamable as PRs; nothing here is a hard fork.

## Appendix: divergent functional commits

| Commit | Date | Summary |
|---|---|---|
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
