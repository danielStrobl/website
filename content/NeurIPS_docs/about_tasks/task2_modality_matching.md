---
title: "Task 2: Modality Matching"
type: book
date: "2021-08-02T00:00:00+01:00"
# Prev/next pager order (if `docs_section_pager` enabled in `params.toml`)
weight: 3
---
## Matching profiles of each cell from different modalities.
While joint profiling of two modalities in the same single cell is now possible, most single-cell datasets that exist measure only a *single* modality. These modalities complement each other in their description of cellular state. Yet, it is challenging to analyse uni-modal datasets together when they do not share observations (cells) or a common feature space (genes, proteins, or open chromatin peaks). If we could map observations to one another across modalities, it would be possible to treat separately profiled datasets in the same manner as new multi-modal sequencing data. Mapping these modalities to one another opens up the vast amount of uni-modal single-cell datasets generated in the past years to multi-modal data analysis methods.

Unlike in task 1, where the goal was to predict all values of RNA or protein abundances from ATAC or RNA (respectively) in each cell, the goal of this task is to identify the correspondence between single-cell profiles. Because we are only interested in matching observations, the competitors are encouraged to consider feature selection to identify the representation of the input most important for matching observations.

<figure>
  <img src="/media/tasks/match.svg">
  <figcaption>
    <h3>
      Task 2: Matching
    </h3>
    <p style="font-size: medium;">
      In this task, competitors are given a jointly measured multimodal dataset (only ATAC + RNA shown) where the cell barcodes linking the two measures are obscured for the purposes of the competition. Competitors must estimate which profiles match across modalities and provide the probability distribution of these predictions. The sum of weights in the correct correspondences is used to score submissions.
    </p>
  </figcaption>
</figure>

## Task API

The following section describes the task API for the Modality Matching task. Competitors must submit their code as a Viash component. To facilitate creation of these components, [starter kits](//neurips_docs/submission/starter_kit_contents) have been provided.

### Input data formats

{{% callout note  %}}
The data format and attributes provided in the input data is tailored to each task and may differ from the publicly released [benchmarking dataset](/neurips_docs/data/dataset/). **Only the attributes listed in the following section will be accessible to methods submitted to the competition.**
{{% /callout  %}}

#### Inputs to methods

Method components should expect **five** h5ad files, `--input_train_mod1`, `--input_train_mod2`, `--input_train_sol`, `--input_test_mod1`, and `--input_test_mod2`.

The `*_mod1` and `*_mod2` arguments are paths to [AnnData](https://anndata.readthedocs.io/en/latest/) objects containing the modality 1 and modality 2 data for which the rows are shuffled and anonymized. These files have the attributes below. If the `feature_types` of one modality is ``"GEX"``, then that of the other must be either ``"ATAC"`` or ``"ADT"`` (antibody-derived tag, a measure of protein abundance). For the purposes of the competition, components should expect the following 4 combinations of modalities:

| `mod1`   | `mod2`   |
|----------|----------|
| `"GEX"`  | `"ATAC"` |
| `"ATAC"` | `"GEX"`  |
| `"GEX"`  | `"ADT"`  |
| `"ADT"`  | `"GEX"`  |

The `input_train_sol` file contains the solution pairing matrix of shape `(n_obs, n_obs)` indicating the true correpsondences from the joint profiling experiment in the input training data. A value of `1` in index `[i,j]` means that `input_train_mod1.X[i]` and `input_train_mod2.X[j]` were measured in the same cell. All other values are `0`.

  * `X`: The sparse pairing matrix. A value of 1 in this matrix means this modality 1 profile (row) corresponds to a modality 2 profile (column).
  * `.uns['dataset_id']`: The name of the dataset.

#### Objective for methods

The goal is to output a matching matrix that predicts the correspondences between profiles from modality 1 in modality 2.

{{% callout note  %}}
Note, you do not need to return predictions for all four combinations of inputs and outputs. We will be independently ranking and awarding prizes to each combination as described below in [Prizes](#prizes). For more details, see the [FAQs](/neurips_docs/submission/faq/#what-if-i-only-want-to-compete-for-one-of-the-prizes-in-a-task)
{{% /callout  %}}

#### Attributes of input data

The input `*_mod1` and `*_mod2` data objects have the following attributes:


```plaintext
adata
  Input AnnData object for task 2

  Attributes
  ----------
  adata.X : ndarray, shape=(n_obs, n_var)
    Sparse profile matrix of given modality. If .var['feature_types'] == "GEX" or "ADT",
    values in adata.X represent expression counts for each gene. If
    .var['feature_types'] == "ATAC", values represent counts of reads in peaks for
    chromatin accessibility
  adata.obs['batch']: ndarray, shape=(n_obs,)
    The batch from which the data was sequenced. Has format "s[1-4]d[1-9]" indicating the site and
    donor associated with the batch.
  adata.uns['dataset_id'] : str
    The name of the dataset.
  adata.var['feature_types']: ndarray, shape=(n_var,)
    The modality of this file, should be equal to "GEX", "ATAC" or "ADT".
  adata.var_names : ndarray, shape=(n_var,)
    Ids for the features.
  adata.obs_names : ndarray, shape=(n_obs,)
    Anonymised ids for the cells.
```

The input `input_train_sol` object has the following attributes:

```plaintext
adata
  Input solution AnnData object for task 2

  Attributes
  ----------
  adata.X : ndarray, shape=(n_obs)
    Sparse pairing matrix for input_train_mod[1|2]
  adata.uns['dataset_id'] : str
    The name of the dataset.
  adata.uns['method_id']: str
    The name of the prediction method.
```

Examples of how to load and process the data are contained in the [starter kits](/neurips_docs/submission/starter_kit_contents) for the respective programming language.

#### Normalization and transformation of data for the matching task

To make the task more straightforward, we have followed common practices for normalizing and transforming data of each modality. The raw data is also provided in `adata.layers` as described below.

For full details on preprocessing, see the [Data Preprocessing](/neurips_docs/data/dataset/#preprocessing) notes.

**GEX**  

For this task, gene expression data stored in `adata.X` for the training and test data has been size-factor normalized and log1p transformed.  Raw UMI counts are available in `adata.layers["counts"]`. Size factors are accessible in `adata.obs["size_factors"]`

**ATAC**

For this task, ATAC data stored in `adata.X` for the training and test data has been binarized. The raw UMI counts for each peak can be found in `adata.layers["counts"]`.

**ADT**

For this task, ADT derived protein abundance measures have been centered log-ration (CLR) normalized. Raw ADT counts can be found in `adata.layers["counts"]`.


### Output data formats

This component should output only *one* h5ad file, `--output`, containing the predicted probabilities of the matchings between the two input datasets.

```plaintext
adata
  Output AnnData object for task 2

  Attributes
  ----------
  adata.X : ndarray, shape=(n_obs)
    Sparse pairing matrix.
  adata.uns['dataset_id'] : str
    The name of the dataset.
  adata.uns['method_id']: str
    The name of the prediction method.
```

{{% callout note  %}}
**Restrictions on the pairing matrix**

There are two restrictions on the pairing matrix:
1. Rows sum to 1 - The sum of each row of the matrix should be 1. This limits the amount of weight that can be placed on each pairing $i,j$
2. Sparsity - If `input_mod1` has shape `(n_obs, n_var1)` and `input_mod2` has shape `(n_obs, n_var2)`, the pairing matrix must be a sparse matrix with shape `(n_obs, n_obs)` containing **at most `1000×N` non-zero values**. Predictions with more than `1000×N` non-zero values will be rejected by the metric component.
{{% /callout  %}}


### Metric

Performance in task 2 is measured using a weighted sum of the confidences placed on correct pairings `i,j`.
Specifically the score is calculated using the following equation:
$$ Score = \frac{1}{N} \sum_i \sum_j X_{i,j} * \delta_{i,j} $$
where $X_{i,j}$ is the entry in the sparse pairing matrix from the user's submission and $\delta_{i,j}$ is $1$ if profile $i$ and $j$ were measured in the same cell and $0$ otherwise. $N$ is the number of observations.


## Prizes

For this task, five $1000 prizes will be awarded to the submissions for each of the following criteria:
1. Best performance matching GEX → ATAC
2. Best performance matching ATAC → GEX
2. Best performance matching GEX → ADT
2. Best performance matching ADT → GEX
3. Best performance on average across modalities
