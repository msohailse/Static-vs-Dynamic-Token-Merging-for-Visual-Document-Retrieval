# Results write-up: Dynamic vs. Static token merging

*Corpus A: 68 pages from 5 ViDoRe datasets, one real query per page. Metric: NDCG@5 (higher = correct page ranked nearer the top). Model: ColPali v1.2, training-free.*

## Headline numbers (Benchmark A)

| Method | Memory vs full | NDCG@5 | Retained |
|---|---|---|---|
| Full (no merging) | 100.0% | 0.924 | 100.0% |
| Static 9 (paper) | 11.1% | 0.899 | 97.3% |
| Thesis Dynamic | 8.3% | 0.902 | 97.7% |

**Overall:** Dynamic vs Static 9: +0.003 NDCG@5 (gain) at -2.8 points of memory.

**Total index for these 68 pages:** full ColPali 17.1 MB -> Dynamic 1.42 MB (a 92% memory saving) while retaining 97.7% of accuracy; Static 9 needs 1.89 MB for 97.3%.

## Where the accuracy moves (per dataset, Dynamic - Static 9)

| Dataset | Static 9 | Dynamic | Delta | Mean F chosen | Reading |
|---|---|---|---|---|---|
| ArxivQA | 0.857 | 0.857 | +0.000 | 18.2 | tie: merging choice made no measurable difference |
| DocVQA | 0.892 | 0.892 | +0.000 | 13.7 | tie: merging choice made no measurable difference |
| InfoVQA | 0.862 | 0.894 | +0.033 | 11.1 | gain: easing to F=11 recovered accuracy here |
| TAT-DQA | 0.962 | 0.962 | +0.000 | 9.4 | tie: merging choice made no measurable difference |
| TabFQuAD | 0.926 | 0.902 | -0.024 | 16.6 | loss: F=17 on this data cost accuracy |

Hardest dataset for merging overall: **InfoVQA** (Dynamic 0.894 vs Full 0.974).

## Benchmark B (multi-paper corpus, 8 queries - qualitative only)

Dynamic: 0.845 at 12.3% memory; Static 9: 0.891 at 11.1%.

## Caveats

- Training-free setting: the paper's 97.8% at factor 9 used fine-tuning; compare methods with each other.
- With ~10 queries per dataset, differences under ~0.02 NDCG@5 are within noise.
- Density thresholds were calibrated once on this corpus (see Section 3).