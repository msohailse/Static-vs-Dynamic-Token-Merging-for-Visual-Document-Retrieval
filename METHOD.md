# Method documentation: how the code decides and applies the merging factor

This document explains the algorithms implemented in
`Static_vs_Dynamic_Token_Merging_for_Visual_Document_Retrieval.ipynb`, function by function.
The notebook's Section 3 contains the code itself; this file explains *why* each piece looks the way it does.

## 1. The pipeline at a glance

```
PDF document ──render──▶ page image ──ColPali──▶ ~1,030 patch embeddings (128-d each)
                                                    │
                                        [our analysis: ~ms per page]
                                                    │
                              ┌─────────────────────┴─────────────────────┐
                              ▼                                           ▼
                 Attempt 1: per-page factor               Attempt 2: per-cluster granularity
                 (density S → factor F)                   (cut cluster tree by distance,
                                                           dense-page safety floor)
                              └─────────────────────┬─────────────────────┘
                                                    ▼
                                     merged vectors stored in the index
                                     (85–90% smaller than unmerged)
```

Everything happens **once, offline, at indexing time**. At query time there is no analysis at all —
queries score against a smaller index, so search gets *faster*.

## 2. The analysis signal: `get_density_score(embedding)`

**Question it answers:** *how much distinct information does this page carry?*

```python
S = 1 − mean( nearest-neighbour cosine similarity of the page's patch embeddings )
```

For every one of the ~1,030 patch vectors, find its most similar *other* patch on the same page.
If patches mostly duplicate each other (title slides, posters, white space, repeated texture),
nearest-neighbour similarity is high and **S is low** — the page is *redundant*. If patches are
diverse (dense multi-column text, financial reports), **S is high** — the page is *information-dense*.

Why embeddings and not pixels? We validated both. Pixel heuristics (grayscale variance, edge
density, ink coverage) rank text-rich pages **below** infographics (AUC < 0.5 — they measure visual
busyness, not information). Embedding redundancy ranks them correctly (AUC 0.70), because ColPali's
embeddings already encode *content*, not appearance.

**Cost:** one 1,030×1,030 similarity product ≈ 2–5 ms per page — under 1% of the ~1 s ColPali
forward pass that produced the embeddings.

## 3. Attempt 1: per-page dynamic factor — `get_thesis_factor(S)`

One factor per page, chosen from S around a default:

| Density verdict | Condition | Factor |
|---|---|---|
| **Dense** — protect detail | S > 0.090 | eased from the default toward ~default/2 (e.g. 13 → 5) |
| **Unsure** — don't gamble | 0.080 ≤ S ≤ 0.090 | the default (13) |
| **Clearly redundant** — harvest memory | S < 0.080 | compressed toward 2×default (e.g. 13 → 26) |

Design decisions worth knowing:

- **The default (13) was not hand-picked** — Section 4b sweeps it and plots the full
  accuracy-vs-memory curve; 13 is where the curve beats the static baseline on both axes.
- **The "unsure band" exists because of a measured failure:** TabFQuAD table pages sit at S ≈ 0.084.
  They *look* redundant (repeated table cells) but the cells carry distinct numbers; compressing
  them cost accuracy for almost no memory (they are a small share of the corpus). Rule: compression
  must be earned by a *clear* redundancy verdict; uncertainty defaults to the safe factor.
- **Known limitation (kept honestly in the write-up):** any per-page verdict can misidentify a page,
  and then the *whole page* pays. That motivates Attempt 2.

## 4. Attempt 2: adaptive per-cluster merging — `thesis_pool(embedding)`

Instead of one factor per page, let **each region of the page choose its own granularity**:

1. Build the Ward cluster tree over the page's patch embeddings — *the identical tree the official
   Light-ColPali pooler already builds*, so this step adds nothing over the paper's own pipeline.
2. Cut the tree at a merge **distance** (9.3, swept in Section 4b) instead of at a fixed cluster
   *count*. Patches closer than the cut merge into one vector; patches farther apart stay separate.
   Redundant regions collapse to a handful of vectors; dense text regions keep many.
3. **Safety floor** ("keep the default where needed"): if the density analysis flags the page as
   dense (S > 0.090) and the cut produced fewer than n/9 vectors, re-cut at exactly n/9 clusters —
   a dense page is never merged harder than the paper's own operating point.

Why this fixes Attempt 1's weakness: there is no per-page verdict to get wrong. A misjudged
*cluster* costs one region of one page, not the page. In our measurements this restored the densest
dataset (TAT-DQA) to full-model accuracy.

## 5. Scoring and evaluation (shared by every method)

- **`pool_page(embedding, factor)`** — the official `HierarchicalTokenPooler` from the paper
  authors' repo (illuin-tech/colpali). Static baselines use it unmodified, so "Static 9" in our
  tables is a faithful reproduction of Light-ColPali's merging, not a reimplementation.
- **`maxsim_matrix(q_embs, d_embs)`** — ColPali's late interaction: each query token takes its best
  match among the page's stored vectors; the sum over query tokens is the page score (paper, §3).
- **`retrieval_metrics(...)`** — ranks the correct page among **all** pages in the corpus and
  reports NDCG@5 / Recall@1 / MRR. The corpus mixes all documents, including multi-page and huge
  documents, so a method that blurs the differences between sibling pages is punished.
- **`run_benchmark(...)`** — evaluates Full / Static 9 / Attempt 1 / Attempt 2 on the *same*
  embeddings (one model run for everyone), and reports per-dataset accuracy, per-page factors,
  tokens stored and index size.

## 6. Efficiency summary

| Step | Cost per page | Notes |
|---|---|---|
| ColPali forward pass | ~1 s (GPU) | required by every method, including the paper's |
| Density analysis (S) | ~2–5 ms | one matrix product; <1% overhead |
| Attempt 1 factor choice | ~0 | a two-branch formula |
| Attempt 2 tree + cut | ~0.12 s | same tree the paper's pooler builds anyway |
| Query time | **lower than baseline** | no analysis at search time; smaller index to score |

## 7. Reproducing

Run the notebook top to bottom (see `README.md`). Every number in `results/` — tables, charts,
`RESULTS.md` — is regenerated from scratch by the run; nothing is hand-entered.
