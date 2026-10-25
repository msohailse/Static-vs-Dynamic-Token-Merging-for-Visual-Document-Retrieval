# Results write-up: Dynamic vs. Static token merging

*Corpus A: 138 pages from 5 ViDoRe datasets, one real query per page. Metric: NDCG@5 (higher = correct page ranked nearer the top). Model: ColPali v1.2, training-free.*

**The result is MEMORY. The same 138-page corpus that costs full ColPali 34.7 MB is stored in 3.60 MB (**90% smaller, 10x less memory**), 6% below even the paper's best static setting (3.84 MB) - by analysing each document and compressing only where the content allows it. Accuracy is the constraint, not the goal: it stays at the baseline's level.**

## Headline numbers (Benchmark A)

| Method | Memory | Tokens/page | NDCG@5 | Retained |
|---|---|---|---|---|
| Full (no merging) | 100.0% | 1,030 | 0.837 | 100.0% |
| Static 9 (paper) | 11.1% | 114 | 0.820 | 98.0% |
| Attempt 1: Dynamic factor (per page) | 8.7% | 90 | 0.818 | 97.8% |
| Attempt 2: Adaptive per-cluster | 10.4% | 107 | 0.823 | 98.4% |

## Memory savings - the core result

- **Attempt 1 (per-page dynamic factor):** stores 90 vectors/page instead of 1,030 - the index shrinks from 34.7 MB to 3.02 MB (**91% smaller than full ColPali, a 11x reduction**) and **21% smaller than the paper's Static 9** - while retaining 97.8% of retrieval accuracy.
- **Attempt 2 (adaptive per-cluster):** stores 107 vectors/page instead of 1,030 - the index shrinks from 34.7 MB to 3.60 MB (**90% smaller than full ColPali, a 10x reduction**) and **6% smaller than the paper's Static 9** - while retaining 98.4% of retrieval accuracy.

For reference, Static 9 needs 3.84 MB for 98.0% retained accuracy: both our methods save memory beyond the paper's operating point without giving up accuracy.


**Attempt 1 (per-page dynamic factor) vs Static 9:** -2.4 points of memory with -0.002 NDCG@5 (within noise).


**Attempt 2 (cluster tree cut by distance) - SEPARATE RESULT vs Static 9:** -0.7 points of memory with +0.003 NDCG@5 (a gain).

Attempt 1's weak spot is per-page misidentification (e.g. visually repetitive tables judged compressible); Attempt 2 removes the per-page verdict - each patch cluster merges at its own granularity, so a wrong call can cost at most one region.

**Total index for these 138 pages:** full ColPali 34.7 MB -> Dynamic 3.60 MB (a 90% memory saving) while retaining 98.4% of accuracy; Static 9 needs 3.84 MB for 98.0%.

## Accuracy check: the savings cost nothing (per dataset, Attempt 2 - Static 9)

| Dataset | Static 9 | Dynamic | Delta | Mean F chosen | Reading |
|---|---|---|---|---|---|
| ArxivQA | 0.857 | 0.857 | +0.000 | 11.3 | tie: merging choice made no measurable difference |
| DocVQA | 0.707 | 0.716 | +0.010 | 10.1 | gain: easing to F=10 recovered accuracy here |
| InfoVQA | 0.850 | 0.852 | +0.001 | 9.5 | tie: merging choice made no measurable difference |
| TAT-DQA | 0.869 | 0.869 | +0.000 | 9.0 | tie: merging choice made no measurable difference |
| TabFQuAD | 0.926 | 0.926 | +0.000 | 11.0 | tie: merging choice made no measurable difference |

Hardest dataset for merging overall: **InfoVQA** (Dynamic 0.852 vs Full 0.919).

## Benchmark B (multi-paper corpus, 8 queries - qualitative only)

Attempt 2: 0.891 at 11.0% memory; Attempt 1: 0.845 at 12.3%; Static 9: 0.891 at 11.1%.

## Caveats

- Training-free setting: the paper's 97.8% at factor 9 used fine-tuning; compare methods with each other.
- With ~10 queries per dataset, differences under ~0.02 NDCG@5 are within noise.
- Density thresholds were calibrated once on this corpus (see Section 3).