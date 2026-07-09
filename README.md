# epi-clusters

Unsupervised tumor/normal classification of **Kidney Renal Clear Cell Carcinoma (KIRC)** from DNA methylation data. Instead of training on ground-truth tumor/normal labels, this pipeline reduces the dimensionality of methylation profiles, clusters the reduced representation into two groups, treats cluster membership as a *pseudo-label*, and trains a standard classifier (SVM / Random Forest) on those pseudo-labels. Ground truth is used only to *evaluate* the pipeline, never to train it.

Full write-up: [`results/paper/main-acm-format-draft.tex`](results/paper/main-acm-format-draft.tex). Slide deck: [`results/presentation/slides.tex`](results/presentation/slides.tex).

## Key results

- **PCA decisively outperforms UMAP** as the dimensionality-reduction front end (mean ARI 0.79 vs. 0.07 across nine clustering configurations).
- The automatically selected configuration (**KMeans on 112 principal components**, ARI = 0.876) yields a classifier trained on pseudo-labels within **0.005 AUC** of one trained on ground truth (SVM: 0.995 vs. 1.000; RF: 0.997 vs. 0.999) — confirmed under 5-fold patient-grouped cross-validation.
- **Clustering quality (ARI) strongly predicts** the pseudo-label/ground-truth AUC gap (r = -0.93 SVM, r = -0.94 RF), but fully label-free internal metrics (silhouette, Davies–Bouldin) do **not** substitute for ARI in this role — tested directly, not assumed.
- The fitted pipeline generalizes to a patient-disjoint holdout split of the same cohort (train ARI 0.876 → holdout ARI 0.825) and is consistent with published pan-cancer methylation diagnostic benchmarks.

## Repository structure

```
.
├── .githooks/               # pre-commit hook blocking >100MB commits (see Setup)
├── notebooks/              # the pipeline, run in order (see "Reproducing the pipeline" below)
├── data/
│   ├── raw/                # reference files (probe manifest, subtype annotations) — gitignored
│   ├── others/              # source TCGA TSVs (KIRC + other cancers) — gitignored, download separately
│   └── processed/          # pipeline outputs: train/holdout splits, cluster assignments, CV results
│                            #   (large .parquet files are gitignored; small .csv/.png outputs are tracked)
├── models/                 # fitted PCA + clustering models (.joblib files are gitignored, regenerate via notebooks)
├── results/
│   ├── paper/               # ACM sigconf LaTeX paper draft + figures
│   └── presentation/        # Beamer slide deck + figures
└── requirements.txt
```

## Setup

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
git config core.hooksPath .githooks   # enables the pre-commit size guard below
```

A pre-commit hook (`.githooks/pre-commit`) blocks any commit that stages a file over 100MB (GitHub rejects pushes over that limit anyway). It isn't enabled automatically on clone — run the `git config` line above once per clone to turn it on.

## Data

This repository does not include the underlying TCGA data (multi-GB methylation matrices). To reproduce the pipeline from scratch, download from [Xena Hub](https://xenabrowser.net/):

- `TCGA-KIRC.clinical.tsv` and `TCGA-KIRC.methylation450.tsv` → place in `data/others/kirc/`
- Illumina 450K probe manifest (`HM450.hg38.manifest.gencode.v36.probeMap`) → place in `data/raw/`

`notebooks/01_preprocessing.ipynb` reads directly from these paths.

## Reproducing the pipeline

Run the notebooks in order:

1. **`01_preprocessing.ipynb`** — loads raw methylation + clinical data, drops probes with >1% missing values (371,082 probes remain), and creates a **patient-grouped** train/holdout split (285 / 198 samples) so a patient's matched tumor/normal pair never crosses the split.
2. **`02-1_clustering_pca.ipynb`** / **`02-2_clustering_umap.ipynb`** — reduce dimensionality (PCA and UMAP respectively, at 2, 3, and a systematically chosen "optimal" number of components), cluster with KMeans/Hierarchical/GMM, score each configuration's ARI against ground truth, and automatically save the best-ARI deployable configuration to `models/`.
3. **`03-1_dim_red_pca.ipynb`** / **`03-2_dim_red_umap.ipynb`** *(optional, exploratory)* — 2D/3D visualizations of the clustered PCA/UMAP embeddings.
4. **`04_classification.ipynb`** — trains SVM and Random Forest on the selected pseudo-labels and, separately, on ground truth; evaluates both on the untouched holdout set and plots ROC curves.
5. **`05_holdout.ipynb`** — validates that the fitted clustering pipeline generalizes to the holdout split (ARI comparison, train vs. holdout).
6. **`06_cv_robustness.ipynb`** — 5-fold patient-grouped nested cross-validation: robustness of ARI/AUC across configurations, plus the central ARI-vs-AUC-gap analysis (using a deliberately poor UMAP×Hierarchical configuration to widen the tested ARI range) and the silhouette/Davies–Bouldin label-free-metric check.

All notebooks are patient-grouped throughout (`GroupShuffleSplit` / `GroupKFold`) to prevent a patient's matched tumor/normal samples from leaking across any train/test boundary.

## Reference

Mariano, T. C. and Clemente, J. B. *Clustering-Derived Pseudo-Labels for Unsupervised Tumor/Normal Classification of Kidney Renal Clear Cell Carcinoma from DNA Methylation Data.* (ICBBE submission, draft — see `results/paper/`.)
