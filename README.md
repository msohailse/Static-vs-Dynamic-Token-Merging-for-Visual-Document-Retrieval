# Static vs. Dynamic Token Merging for Visual Document Retrieval

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/msohailse/Static-vs-Dynamic-Token-Merging-for-Visual-Document-Retrieval/blob/main/Static_vs_Dynamic_Token_Merging_for_Visual_Document_Retrieval.ipynb)

**One-click run:** the badge above opens the notebook in Google Colab. Pick a GPU runtime
(Runtime → Change runtime type → T4), then Runtime → Run all — the first cell clones this repo,
so the evaluation documents come along automatically.

Thesis experiment built on **ColPali v1.2** and the paper *"Towards Storage-Efficient Visual Document
Retrieval: An Empirical Study on Reducing Patch-Level Embeddings"* (Ma et al., ACL Findings 2025 —
`2025.findings-acl.1003.pdf` in this folder).

## The idea in one paragraph

ColPali turns every document page into ~1,030 patch embeddings, which makes the index huge. The paper's
Light-ColPali merges similar patches with one **fixed** merging factor (9 ⇒ ~11% of the memory) and notes
that dense text pages lose the most accuracy — choosing the factor **per page** is left as future work.
This project does exactly that: each page's **own patch embeddings** reveal how compressible it is
(`S = 1 − mean nearest-neighbour cosine similarity`; validated AUC 0.70 for separating text-rich from
low-density pages, where pixel heuristics fail). Around a default factor (D=13, chosen by a sweep),
information-dense pages are merged gently (F≈5–9) and redundant pages harder (F≈13–26). Result on 54 real
PDF documents (68 pages) with real queries: **97.7% of full ColPali's NDCG@5 at 8.3% of the memory — more
accurate and ~25% smaller than the paper's Static 9 baseline** (97.3% at 11.1%). See `results/RESULTS.md`.

## Files

| Path | What it is |
|---|---|
| `Static_vs_Dynamic_Token_Merging_for_Visual_Document_Retrieval.ipynb` | The whole experiment (fully commented, run top to bottom) |
| `METHOD.md` | Plain-language explanation of the algorithms: the density analysis, both attempts, scoring, and efficiency |
| `evaluation_docs/` | 54 real PDF documents (50 single-page + 4 multi-page, 68 pages) from five ViDoRe test sets |
| `evaluation_docs/manifest.csv` | Ground truth: one row per (document, page) with the real benchmark query for that page |
| `2025.findings-acl.1003.pdf` | The paper; also used as the second benchmark corpus (13 pages, 8 hand-labelled queries) |
| `requirements.txt` | Python dependencies |
| `results/` | Created by the run: summary tables (CSV) and plots (PNG) |

## Install

```bash
pip install -r requirements.txt

# PDF rendering needs the poppler system package:
#   Linux / Colab:  apt-get install -y poppler-utils
#   macOS:          brew install poppler
```

A GPU is strongly recommended (Colab T4 is fine). The first run downloads the ColPali v1.2 weights (~6 GB).

## Run

Open the notebook and run all cells top to bottom. Section 6 prints the final "ours vs. paper baseline"
comparison; all tables and figures are also written to `results/`.

## How to read the results

- Always read **accuracy and memory together** — Dynamic and Static 9 use different budgets.
- The fair test of the thesis is **Thesis Dynamic vs. the memory-matched static control** (same index size).
- The paper's 97.8% retention at factor 9 comes from a model *fine-tuned with merging*; everything here is
  training-free, so compare the methods with each other, not with the paper's absolute numbers.
