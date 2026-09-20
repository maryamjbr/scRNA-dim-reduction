# Dimensionality reduction for selected scRNA-seq datasets

This project compares PCA, t-SNE, UMAP, and a small variational autoencoder (VAE) on selected single-cell RNA-seq datasets. After each method produces a reduced representation, K-means is applied and the resulting clusters are compared with the available cell labels using permutation-invariant clustering metrics.

The main artifact is [`scrna_dimensionality_reduction.ipynb`](scrna_dimensionality_reduction.ipynb). It is a **partial reproduction**, not a reproduction of the complete benchmark by Xiang et al. (2021). In particular, the reference study evaluates ten methods, 30 simulated datasets, and five real datasets; this repository evaluates four methods on three selected data subsets.

## What is implemented

- Dataset retrieval and parsing for Deng (GSE45719), Chu (GSE75748), and the scVelo PBMC68k object.
- Dataset-specific preprocessing documented below.
- Direct application of PCA, t-SNE, UMAP, and a project-specific PyTorch VAE. PCA is not used before the other methods.
- K-means clustering in each reduced embedding, with `k` equal to the number of known labels.
- Adjusted Rand index (ARI), adjusted mutual information (AMI), and silhouette score.
- Two-dimensional visualizations and higher-dimensional clustering evaluations.

The implementation reports **AMI** rather than NMI, does not measure memory usage, and uses dataset-specific preprocessing. The reported values are single-run results for the datasets used here.

## Datasets

The repository does not redistribute any dataset. The notebook downloads source files into the ignored, repository-relative `data/` directory.

| Dataset used here | Biological context and source | Matrix loaded by the notebook | Subset used for evaluation |
|---|---|---|---|
| Deng, [GSE45719](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE45719) | Mouse preimplantation development; Deng et al. (2014) | Per-cell GEO supplementary files; the `reads` column is assembled into a cell-by-gene matrix | 313 cells × 21,815 nonzero genes after removing four zygote-labelled cells and 1,143 all-zero genes; labels are grouped into `2-cell`, `4-cell`, `8-cell`, `16-cell`, `blastocyst`, and `other` |
| Chu, [GSE75748](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE75748) | Human embryonic stem-cell differentiation; Chu et al. (2016) | GEO expected-count matrix `GSE75748_sc_cell_type_ec.csv.gz` (1,018 cells × 19,097 genes) | H1 and H9 cells only: 374 cells × 19,097 genes (212 H1, 162 H9) |
| PBMC68k, [scVelo loader](https://scvelo.readthedocs.io/en/stable/scvelo.datasets.pbmc68k.html) | Human peripheral blood mononuclear cells from Zheng et al. (2017) | A labelled AnnData object downloaded by `scvelo.datasets.pbmc68k` (65,877 cells × 33,939 genes) | 2,027 cells × 2,000 highly variable genes after this notebook's filters and preprocessing; 11 saved cell-type labels |

Two distinctions matter. First, Xiang et al. evaluate **H1 versus definitive endoderm** from the Chu study, while this notebook selects **H1 versus H9**. Second, the notebook's Deng group named `other` is a code-defined catch-all, not a documented developmental stage from Xiang et al. Those results therefore should not be treated as a direct replication of the paper's Deng or Chu experiments.

## Preprocessing implemented here

| Step | Deng | Chu H1/H9 | PBMC68k subset |
|---|---|---|---|
| Input values | Downloaded `reads` columns | Downloaded expected counts | Integer-valued counts in the scVelo AnnData object |
| Cell selection/filtering | Remove four zygote-labelled cells | Select labels H1 and H9 | Keep cells with at least 140 detected genes |
| Gene filtering | Remove genes that are zero in every retained cell | None | Keep genes detected in at least 3 cells |
| Library-size normalization | No | No | Normalize each cell to the median retained library size |
| Log transformation | No | No | `log1p` |
| Highly variable genes | No | No | 2,000 genes (`min_mean=0.0125`, `max_mean=3`, `min_disp=0.5`) |
| Feature scaling | No | No | `scanpy.pp.scale` |
| VAE-only transform | Per-feature min-max scaling to `[0, 1]` | Same | Same, after PBMC preprocessing |
| PCA before another method | No | No | No |

The cell-type labels determine dataset selection/grouping, the requested number of K-means clusters, and the external metrics. They are not supplied to PCA, t-SNE, UMAP, the VAE, or the K-means fit itself.

## Relation to Xiang et al. (2021)

Xiang et al. state that PCA, t-SNE, UMAP, and VAE receive count matrices, that the benchmark produces two-dimensional latent spaces, and that K-means is repeated 50 times with `k` set to the true cell-type count. The article reports ARI, NMI, silhouette, runtime, and peak memory. The comparison below separates those published methods from this repository's implementation.

| Component | Xiang et al. (2021) | This repository | Assessment |
|---|---|---|---|
| Benchmark scope | 10 methods, 30 simulations, five real datasets | 4 methods, 3 selected data subsets | Different |
| Cell/gene filtering | Dataset-specific filtering details cannot be fully reconstructed from the main article | Explicit filters shown above | Cannot be verified from the available article text |
| PCA/t-SNE/UMAP input | Original count matrix | Direct downloaded matrix for Deng/Chu; filtered, normalized, log-transformed, HVG-selected, and scaled PBMC matrix | Similar for Deng/Chu; different for PBMC |
| Normalization/log transform | Method-specific normalization is stated, but exact selected-method operations are not fully specified; the selected methods are listed as taking counts | None for Deng/Chu; normalize and `log1p` for PBMC | Cannot be verified exactly; PBMC differs from the stated count input |
| Highly variable genes | 1,000 HVGs for GrandPrix/DCA and PBMC scalability tests; not the stated input for PCA/t-SNE/UMAP/VAE accuracy tests | 2,000 HVGs for PBMC only | Different |
| Feature scaling | Not specified for these four methods in the main article | PBMC only | Cannot be verified |
| PCA before downstream DR | Only scvis is documented as receiving PCA-100 preprocessing | No PCA before t-SNE, UMAP, or VAE | Matches closely for the four methods implemented here |
| Output dimensions | 2D benchmark representations | 2D plots; evaluation uses 100 dimensions for Deng/Chu and 50 for PBMC | Different |
| Chu comparison | H1 versus definitive endoderm | H1 versus H9 | Different |
| VAE | Hu and Greene Keras/TensorFlow implementation, with zero to two hidden layers | Project-specific PyTorch network described below | Different |
| Clustering | K-means repeated 50 times; `k` is the true cell-type count | One K-means fit per representation; `k` is the known label count | Similar principle, not identical |
| External metrics | ARI and NMI | ARI and AMI | Different |
| Silhouette | Reported as a low-dimensional separation metric | Computed in the reduced embedding using predicted K-means labels | Similar, but exact implementation equivalence is not established |
| Computing cost | PBMC downsampling through 68,579 cells, 1,000 HVGs, `system.time`, and `pidstat` peak memory | A single saved wall-clock reduction time per dataset/method; no memory measurement | Different |

Accordingly, the numerical values below are results from this repository and must not be mixed with the paper's reported results.

## Method settings

| Method | Settings explicitly used |
|---|---|
| PCA | `n_components=100` for Deng/Chu or `50` for PBMC (`2` for plots); scikit-learn's other defaults; the notebook sets `random_state=0` |
| t-SNE | Same component counts; `init="random"`; `method="exact"` when dimensions are at least 4 and Barnes-Hut otherwise; `random_state=0`; perplexity and learning rate are not explicitly set and therefore use the installed scikit-learn defaults |
| UMAP | Same component counts; `n_neighbors=15`; `min_dist=0.1`; Euclidean metric; the notebook sets `random_state=0` |
| VAE | Same latent dimensions; architecture and training settings below |
| K-means | `k` equals known label count (6 Deng, 2 Chu, 11 PBMC); one fit; the notebook uses `n_init=10`, `random_state=0` |

The VAE is trained separately for every dataset and separately for its 2D visualization and higher-dimensional evaluation. Its encoder is `input → 400 ReLU → latent mean/log-variance`; its decoder is `latent → 400 ReLU → sigmoid output`. It minimizes summed binary cross-entropy plus the standard KL-divergence term with Adam (`learning_rate=0.001`), batch size 128, and 500 epochs. Inputs are min-max scaled because binary cross-entropy is used. The implementation adapts the structure of the [official PyTorch VAE example](https://github.com/pytorch/examples/blob/main/vae/main.py); it is not the VAE implementation evaluated by Xiang et al.

## Clustering metrics

- **ARI** compares the true and inferred partitions while correcting pairwise agreement for chance. It is invariant to arbitrary cluster identifiers.
- **AMI** measures shared information between the true and inferred partitions and adjusts for chance. AMI is what the notebook computes; it is not the NMI reported by Xiang et al.
- **Silhouette score** measures within-cluster cohesion versus separation from the nearest other cluster. Here it is calculated in the reduced embedding with the predicted K-means labels, not the biological labels or original feature space.

Raw classification accuracy is not used for K-means because cluster identifiers have no intrinsic correspondence to the encoded class identifiers.

## Results from this repository

These values come from the outputs saved in repository commit `51779b1` and are also stored in [`results/saved_notebook_metrics.csv`](results/saved_notebook_metrics.csv). They are **not newly regenerated results**. In that run, only t-SNE had a fixed seed; UMAP, K-means, and VAE were unseeded. The current notebook sets seeds for future runs, so exact reruns may differ.

“Reduction time” is one saved wall-clock observation around the reduction call only. It excludes data loading, preprocessing, K-means, and metric calculation. The saved notebook reports a CUDA device for VAE while the classical methods use CPU implementations, so these timings are neither a controlled hardware benchmark nor direct evidence of scalability. Memory was not measured.

### Deng subset (100-dimensional evaluation)

| Method | AMI | ARI | Silhouette | Reduction time (s) |
|---|---:|---:|---:|---:|
| PCA | 0.3322 | 0.1153 | 0.3051 | 1.165 |
| t-SNE | 0.3392 | 0.2253 | 0.1275 | 17.775 |
| UMAP | 0.6944 | 0.7652 | 0.6840 | 1.339 |
| VAE | 0.3741 | 0.2375 | 0.0414 | 13.180 |

For this specific saved run and grouping, UMAP has the largest AMI, ARI, and silhouette values among the four methods.

### PBMC68k filtered/HVG subset (50-dimensional evaluation)

| Method | AMI | ARI | Silhouette | Reduction time (s) |
|---|---:|---:|---:|---:|
| PCA | 0.4356 | 0.5383 | 0.0659 | 0.935 |
| t-SNE | 0.0702 | 0.0234 | 0.0487 | 426.290 |
| UMAP | 0.0216 | 0.0065 | 0.1975 | 9.514 |
| VAE | 0.2357 | 0.0707 | 0.0224 | 30.922 |

For this saved run, PCA has the largest AMI and ARI, while UMAP has the largest silhouette value. This is a 2,027-cell filtered subset, not the full PBMC68k scalability experiment from Xiang et al.

### Chu H1/H9 subset (100-dimensional evaluation)

| Method | AMI | ARI | Silhouette | Reduction time (s) |
|---|---:|---:|---:|---:|
| PCA | 0.0080 | 0.0194 | 0.3795 | 1.481 |
| t-SNE | 0.0759 | 0.1040 | 0.3076 | 12.533 |
| UMAP | 0.1040 | 0.0667 | 0.4345 | 1.544 |
| VAE | 0.0538 | 0.0808 | 0.1888 | 12.452 |

For this saved H1/H9 run, UMAP has the largest AMI and silhouette value, while t-SNE has the largest ARI. These results do not reproduce the paper's H1/definitive-endoderm comparison.

## Reproduce the notebook

Python 3.11 or newer is recommended.

```bash
git clone https://github.com/maryamjbr/scRNA-dim-reduction.git
cd scRNA-dim-reduction
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
jupyter lab
```

Open `scrna_dimensionality_reduction.ipynb` and run cells in order. Network access is required on the first run. Deng and Chu are downloaded from GEO, while PBMC68k is downloaded by scVelo. The notebook uses CPU by default and CUDA or Apple MPS when PyTorch reports either accelerator as available.

A full run trains six VAE models (2D and evaluation embeddings for three datasets), each for 500 epochs, and uses exact t-SNE for the 50/100-dimensional evaluations. Plan compute time accordingly. The saved timing table is not a prediction of runtime on another machine.

## Limitations

- The project covers fewer methods and datasets than Xiang et al. and does not reproduce the paper's simulated-data stability analysis.
- Preprocessing is inconsistent across datasets, and the Chu and Deng label constructions do not match the paper's experiments exactly.
- Evaluation dimensions differ from the paper's 2D benchmark.
- The project VAE differs from the paper's implementation and uses a generic Bernoulli reconstruction objective after min-max scaling rather than a count-specific likelihood.
- The historical results are single stochastic runs; there is no cross-validation, repeated-seed analysis, uncertainty interval, significance test, or hyperparameter search.
- Selecting `k` from the true label count provides the clustering algorithm with label-derived information.
- Silhouette values depend on the embedding geometry and predicted clustering, so they are not directly comparable to ARI/AMI or evidence of biological validity.
- Runtime values depend on software, hardware, accelerator synchronization, and method implementation; memory usage is not measured.

## Provenance and licensing

The VAE follows the structure of the official PyTorch example. Because the provenance of every code fragment in the original coursework notebook is not fully documented, this repository does not currently include a software license. Any future license should preserve the required upstream notices and be compatible with the remaining code.

## References

1. Xiang, R., Wang, W., Yang, L., Wang, S., Xu, C., & Chen, X. (2021). A comparison for dimensionality reduction methods of single-cell RNA-seq data. *Frontiers in Genetics, 12*, 646936. https://doi.org/10.3389/fgene.2021.646936
2. Deng, Q., Ramsköld, D., Reinius, B., & Sandberg, R. (2014). Single-cell RNA-seq reveals dynamic, random monoallelic gene expression in mammalian cells. *Science, 343*(6167), 193–196. https://doi.org/10.1126/science.1245316. Data: [GSE45719](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE45719).
3. Chu, L.-F., Leng, N., Zhang, J., Hou, Z., Mamott, D., Vereide, D. T., et al. (2016). Single-cell RNA-seq reveals novel regulators of human embryonic stem cell differentiation to definitive endoderm. *Genome Biology, 17*, 173. https://doi.org/10.1186/s13059-016-1033-x. Data: [GSE75748](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE75748).
4. Zheng, G. X. Y., Terry, J. M., Belgrader, P., Ryvkin, P., Bent, Z. W., Wilson, R., et al. (2017). Massively parallel digital transcriptional profiling of single cells. *Nature Communications, 8*, 14049. https://doi.org/10.1038/ncomms14049.
5. Hu, Q., & Greene, C. S. (2019). Parameter tuning is a key part of dimensionality reduction via deep variational autoencoders for single cell RNA transcriptomics. *Pacific Symposium on Biocomputing, 24*, 362–373. https://doi.org/10.1101/385534.
