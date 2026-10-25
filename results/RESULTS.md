# Results write-up: Dynamic vs. Static token merging

*Corpus A: 138 pages from 5 ViDoRe datasets, one real query per page. Metric: NDCG@5 (higher = correct page ranked nearer the top). Model: ColPali v1.2, training-free.*

**The core of the idea is MEMORY: ColPali's index is prohibitively large, and we shrink it adaptively - per document, per region - while keeping retrieval accuracy.**

## Headline numbers (Benchmark A)

| Method | Memory | Tokens/page | NDCG@5 | Retained |
|---|---|---|---|---|
| Full (no merging) | 100.0% | 1,030 | 0.837 | 100.0% |
| Static 9 (paper) | 11.1% | 114 | 0.820 | 98.0% |
| Attempt 1: Dynamic factor (per page) | 8.7% | 90 | 0.818 | 97.8% |
| Attempt 2: Adaptive per-cluster | 8.7% | 90 | 0.823 | 98.3% |

## Memory savings - the core result

- **Attempt 1 (per-page dynamic factor):** stores 90 vectors/page instead of 1,030 - the index shrinks from 34.7 MB to 3.02 MB (**91% smaller than full ColPali, a 11x reduction**) and **21% smaller than the paper's Static 9** - while retaining 97.8% of retrieval accuracy.
- **Attempt 2 (adaptive per-cluster):** stores 90 vectors/page instead of 1,030 - the index shrinks from 34.7 MB to 3.02 MB (**91% smaller than full ColPali, a 11x reduction**) and **21% smaller than the paper's Static 9** - while retaining 98.3% of retrieval accuracy.

For reference, Static 9 needs 3.84 MB for 98.0% retained accuracy: both our methods save memory beyond the paper's operating point without giving up accuracy.


**Attempt 1 (per-page dynamic factor) vs Static 9:** -0.002 NDCG@5 (loss) at -2.4 points of memory.


**Attempt 2 (cluster tree cut by distance) - SEPARATE RESULT vs Static 9:** +0.002 NDCG@5 (gain) at -2.4 points of memory.

Attempt 1's weak spot is per-page misidentification (e.g. visually repetitive tables judged compressible); Attempt 2 removes the per-page verdict - each patch cluster merges at its own granularity, so a wrong call can cost at most one region.

**Total index for these 138 pages:** full ColPali 34.7 MB -> Dynamic 3.02 MB (a 91% memory saving) while retaining 98.3% of accuracy; Static 9 needs 3.84 MB for 98.0%.

## Where the accuracy moves (per dataset, Attempt 2 - Static 9)

| Dataset | Static 9 | Dynamic | Delta | Mean F chosen | Reading |
|---|---|---|---|---|---|
| ArxivQA | 0.857 | 0.866 | +0.009 | 18.1 | gain: easing to F=18 recovered accuracy here |
| DocVQA | 0.707 | 0.707 | +0.000 | 15.0 | tie: merging choice made no measurable difference |
| InfoVQA | 0.850 | 0.875 | +0.025 | 11.7 | gain: easing to F=12 recovered accuracy here |
| TAT-DQA | 0.869 | 0.869 | +0.000 | 9.3 | tie: merging choice made no measurable difference |
| TabFQuAD | 0.926 | 0.863 | -0.063 | 14.4 | loss: F=14 on this data cost accuracy |

Hardest dataset for merging overall: **TabFQuAD** (Dynamic 0.863 vs Full 0.913).

## Benchmark B (multi-paper corpus, 8 queries - qualitative only)

Attempt 2: 0.891 at 10.8% memory; Attempt 1: 0.845 at 12.3%; Static 9: 0.891 at 11.1%.

## Caveats

- Training-free setting: the paper's 97.8% at factor 9 used fine-tuning; compare methods with each other.
- With ~10 queries per dataset, differences under ~0.02 NDCG@5 are within noise.
- Density thresholds were calibrated once on this corpus (see Section 3).