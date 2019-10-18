# scDblFinder

## Introduction

scDblFinder  identifies doublets in single-cell RNAseq directly by creating artificial doublets and looking at their prevalence in the neighborhood of each cell. The rough logic is very similar to *[DoubletFinder](https://github.com/chris-mcginnis-ucsf/DoubletFinder)*, but it is much simpler and more efficient. In a nutshell:

* scDblFinder works directly on a reduced count matrix (using rank correlation on top expressed genes), making the detection independent of downstream processing. This avoids the need for all the Seurat pre-processing, making the detection faster and compatible with other analytic choices.
* instead of creating doublets from random pairs of cells, scDblFinder first overclusters the cells and create cross-cluster doublets. It also uses meta-cells from each cluster to create triplets. This strategy avoids creating homotypic doublets and enables the detection of most doublets with much fewer artificial doublets.
* while we also rely on the expected proportion of doublets to threshold the scores, we include a variability in the estimate of the doublet proportion (`dbr.sd`), and use the error rate of the real/artificial predicition in conjunction with the deviation in global doublet rate to set the threshold.

## Installation

scDblFinder was developed under R 3.6. Install with:

```r
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install("scDblFinder")
```

Or, until the new bioconductor release:
```r
BiocManager::install("plger/scDblFinder")
```

## Usage

Given an object `sce` of class `SingleCellExperiment`:

```r
library(scDblFinder)
sce <- scDblFinder(sce)
```

This will add the following columns to the colData of `sce`:

* `sce$scDblFinder.neighbors` : the number of neighbors considered
* `sce$scDblFinder.ratio` :  the proportion of artificial doublets among the neighborhood (the higher, the more chances that the cell is a doublet)
* `sce$scDblFinder.score` :  a doublet score integrating the ratio in a probability of the cell being a doublet
* `sce$scDblFinder.class` : the classification (doublet or singlet)

### Multiple samples

If you have multiple samples (understood as different cell captures), then it is
preferable to look for doublets separately for each sample. You can do this by 
simply providing a vector of the sample ids to the `samples` parameter of scDblFinder or,
if these are stored in a column of `colData`, the name of the column. In this case,
you might also consider multithreading it using the `BPPARAM` parameter. For example:

```{r, eval=FALSE}
library(BiocParallel)
sce <- scDblFinder(sce, samples="sample_id", BPPARAM=MulticoreParam(3))
table(sce$scDblFinder.class)
```

### Parameters

The important sets of parameters in `scDblFinder` refer respectively to the expected proportion of doublets, to the clustering, and to the number of artificial doublets used.

#### Expected proportion of doublets

The expected proportion of doublets has no impact on the score (the `ratio` above), but a very strong impact on where the threshold will be placed. It is specified through the `dbr` parameter and the `dbr.sd` parameter (the latter specifies the standard deviation of `dbr`, i.e. the uncertainty in the expected doublet rate). For 10x data, the more cells you capture the higher the chance of creating a doublet, and Chromium documentation indicates a doublet rate of roughly 1\% per 1000 cells captures (so with 5000 cells, (0.01\*5)\*5000 = 250 doublets), and the default expected doublet rate will be set to this value (with a default standard deviation of 0.015). Note however that different protocols may create considerably more doublets, and that this should be updated accordingly.

#### Clustering

Since doublets are created across clusters, it is important that subpopulations are not misrepresented as belonging to the same cluster. For this reason, we rely on an over-clustering approach which is similar to *[scran](https://bioconductor.org/packages/3.9/scran)*'s `quickCluster`, but splits clusters above a certain size. This is implemented by scDblFinder's `overcluster` function. The default maximum cluster size is estimated from the population size and number of cells. While this estimate is reasonable for most datasets (i.e. all those used for benchmark), so many clusters might be unnecessary when the cell population has a simple structure, and more clusters might be needed in a very complex population.

#### Number of artificial doublets

`scDblFinder` itself determines a reasonable number of artificial doublets to create on the basis of the size of the population and the number of clusters, but increasing this number can only increase the accuracy. If you increase the number above default settings, you might also consider increasing parameter `k` - i.e. the number of neighbors considered.

<br/><br/>

# Comparison with other tools

To benchmark scDblFinder against alternatives, we used datasets in which cells from multiple individuals were mixed and their identity deconvoluted using SNPs (via *[demuxlet](https://github.com/statgen/demuxlet)*), which also enables the identification of doublets from different individuals.

The method is compared to:

* *[DoubletFinder](https://github.com/chris-mcginnis-ucsf/DoubletFinder)*
* *[scran](https://bioconductor.org/packages/3.9/scran)*'s `doubletCells` function
* *[scds](https://bioconductor.org/packages/3.9/scds)* (hybrid method)

## Mixology10x3cl

![Accuracy of the doublet detection in the mixology10x3cl dataset (a mixture of 3 cancer cell lines). All methods perform very well.](scDblFinder_files/figure-html/ds1-1.png)

(DoubletFinder failed on the Mixology10x3cl dataset)

## Mixology10x5cl

![Accuracy of the doublet detection in the mixology10x5cl dataset (a mixture of 5 cancer cell lines).](scDblFinder_files/figure-html/ds2-1.png)

## Demuxlet controls

![Accuracy of the doublet detection in the demuxlet control (Batch 2) dataset (GSM2560248).](scDblFinder_files/figure-html/ds3-1.png)

## Running time


```
## Warning: Removed 1 rows containing missing values (position_stack).
```

![Running time for each method/dataset](scDblFinder_files/figure-html/runtime-1.png)

Note that by far most of the running time of `scDblFinder` is actually the clustering.
