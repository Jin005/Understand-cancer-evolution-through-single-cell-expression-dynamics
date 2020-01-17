---
output:
  rmarkdown::html_document:
    highlight: pygments
    toc: false
    toc_depth: 3
    fig_width: 5
vignette: >
  %\VignetteIndexEntry{monocle3}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding[utf8]{inputenc}  
---

# Application of Monocle3 (beta) to cancer scRNA-seq datasets

```{r, message = FALSE, warning = FALSE}
library(GEOquery)
library(SingleCellExperiment)
library(SingleR)
library(monocle3) # https://cole-trapnell-lab.github.io/monocle3/
library(scater)
```

## GSE118828 (ovarian cancer scRNA-seq dataset)

### Obtain GSE118828

Download [GSE118828_RAW.tar](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE118828)
from [GEO](https://www.ncbi.nlm.nih.gov/geo/) and `untar -xvf GSE118828_RAW.tar`.

```{r}
untar("../datasets/GSE118828_RAW.tar")
gsm.files <- list.files(".", pattern = "^GSM")
```

Helper function that reads a GSM file and returns a `SingleCellExperiment`.
```{r} 
readGSM <- function(gsm.file)
{
    gsm <- read.csv(gsm.file, as.is = TRUE, row.names = 1L)
    gsm <- t(gsm)
    SingleCellExperiment(assays = list(counts = gsm))    
}
```

Store scRNA-seq data for individual tumors as a `SingleCellExperiment`:
```{r}
sces <- lapply(gsm.files[5:6], readGSM)
for(f in gsm.files) file.remove(f)
names(sces) <- vapply(gsm.files[5:6], 
                        function(n) unlist(strsplit(n, "_"))[[1]], 
                        character(1))
sces[[1]]
```

Information about the individual tumors:
```{r, message = FALSE}
gse <- getGEO("GSE118828")[[1]]
gse <- as(gse, "SummarizedExperiment")
colData(gse)[,10:12]
```

Combine tumor and normal sample for one patient:
```{r}
sce <- cbind(sces[[1]], sces[[2]])
sce
colData(sce)$tissue <- c(rep("normal", ncol(sces[[1]])), 
                            rep("tumor", ncol(sces[[2]])))
```

### Cell type annotation

Preprocessing:
```{r}
sce <- logNormCounts(sce)
```

Get built-in reference data from the Human Primary Cell Atlas:
```{r, message = FALSE}
hpca.se <- HumanPrimaryCellAtlasData()
```

```{r}
pred <- SingleR(test = sce, ref = hpca.se, labels = hpca.se$label.main)
sce$cell.type <- pred$labels
table(sce$cell.type)
```

### Trajectory inference

Creating a `cell_data_set` (aka `Monocle3`'s SingleCellExperiment)
```{r}
rowData(sce)$gene_short_name <- rownames(sce)
cds <- new_cell_data_set(assays(sce)$counts,
     cell_metadata = as.data.frame(colData(sce)),
     gene_metadata = as.data.frame(rowData(sce)))
```

Preprocessing + Dimensionality reduction:
```{r, message = FALSE}
cds <- preprocess_cds(cds)
cds <- reduce_dimension(cds)
plot_pc_variance_explained(cds)
plot_cells(cds, label_groups_by_cluster=FALSE,  color_cells_by = "cell.type")
```

Clustering + learning trajectory graph
```{r, message = FALSE}
cds <- cluster_cells(cds)
cds <- learn_graph(cds)
plot_cells(cds,
           color_cells_by = "cell.type",
           label_groups_by_cluster=FALSE,
           label_leaves=FALSE)
plot_cells(cds,
           color_cells_by = "tissue",
           label_groups_by_cluster=FALSE,
           label_leaves=FALSE)
```

Identify root cell as the one with highest CD34 expression 
```{r, message = FALSE}
cds <- order_cells(cds, root_cells = "cgagcagccagattcgca")
plot_cells(cds,
           color_cells_by = "pseudotime",
           label_groups_by_cluster=FALSE,
           label_leaves=FALSE)
pt <- pseudotime(cds)
head(pt)
```

## GSE103322 (head and neck cancer scRNA-seq dataset)

### Obtain GSE103322

```{r, eval = FALSE}
gse <- read.delim("../datasets/GSE103322_HNSCC_all_data.txt.gz", as.is = TRUE, row.names = 1L)
rownames(gse) <- gsub("\'", "", rownames(gse))
```

Store in a `SingleCellExperiment`:
```{r, eval=FALSE} 
coldat <- DataFrame(t(gse[1:5,]))
coldat
for(i in 1:4) coldat[,i] <- as.logical(as.integer(coldat[,i]))
sce <- SingleCellExperiment(assays = list(TPM = gse[6:nrow(gse),]),
                            colData = coldat)
sce
```

Information about tumors
```{r, eval=FALSE}
gse.info <- getGEO("GSE103322")[[1]]
gse.info <- as(gse.info, "SummarizedExperiment")
head(colData(gse.info)[,11])
```

Annotate tumor site:
```{r, eval=FALSE}
sce <- sce[,as.character(colData(gse.info)[,1])]
colData(sce)$tumor_site <- sub("^tumor site: ", "", as.character(colData(gse.info)[,11]))
table(colData(sce)$tumor_site)
```

### Cell type annotation


### Trajectory inference
