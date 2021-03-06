<div align=center><img width="300" height="320" src="https://github.com/Gavin-Yinld/coMethly/blob/master/figures/co-methy.gif"/></div>

# coMethy
# Introduction
coMethy is an R package for grouping genomic loci with similar methylation pattern (or called co-methylated loci). According to the methylation profile of genomic loci in target methylomes, co-methylation analysis was performed in combination with k-means clustering and WGCNA analysis. For each co-methylation module, PCA analysis was performed to select a subset of loci as the eigen loci representing the methylation trend of the module.
# Installation
In R console,
```R
source("http://bioconductor.org/biocLite.R")
biocLite("WGCNA")
library("devtools")
devtools::install_github("Gavin-Yinld/coMethy")
```
# How to Use

## Step 1. K-means clustering to divide genome with distinct methylation levels
`coMethy` takes the methylation profile of input genomic loci in target methylomes. A numeric matrix is needed as input with each row denoting a genomic locus, each column denoting a sample, and the corresponding entry containing the methylation level.
```R
library("coMethy")

# get the demo dataset
file=paste(system.file(package="coMethy"),"extdata/co_methy.test.data.txt",sep='/')
meth_data <- read.table(file,sep='\t',header=T,row.names=1)

# a typical input data looks like this:
head(meth_data)
                         sample1 sample2 sample3 sample4 sample5 sample6
chr7_82243987_82244107      0.39    0.77    0.21    0.73    0.81    0.89
chr2_166158817_166158941    0.64    0.90    0.59    0.92    0.90    0.46
chr5_30670219_30670424      0.69    0.84    0.64    0.88    0.85    0.22
chr7_66421727_66421857      0.66    0.76    0.68    0.80    0.74    0.80
chr4_131771412_131771552    0.47    0.91    0.33    0.93    0.94    0.93
chr3_135529338_135529388    0.92    1.00    1.00    0.97    0.76    0.73
                         sample7 sample8 sample9
chr7_82243987_82244107      0.65    0.72    0.73
chr2_166158817_166158941    0.62    0.81    0.59
chr5_30670219_30670424      0.90    0.93    0.83
chr7_66421727_66421857      0.84    0.64    0.71
chr4_131771412_131771552    0.10    0.36    0.30
chr3_135529338_135529388    0.90    0.91    0.92
```
Firstly, K-means clustering analysis is adopted to divide pCSM loci into hypo/mid/hyper-methylated groups. In addition, the function `pickSoftThreshold` in `WGCNA` package is called to show the topological properties of the network inferred from each k-means group.

```R
kmeans_cluster <- co_methylation_step1(meth_data)
# A file named "parameter.pdf" will be generated to show the topological properties.
# The two figures in each row represent the topological properties of the network in one k-means group.
# The X-axis represents the soft-thresholding power.
# For each k-means group, a proper soft-thresholding power is need to be choosen, to ensure the scale-free topology property of the network inferred from each k-means group and the connectivity of the genomic loci in the network.
```
<div align=center><img width="700" height="525" src="https://github.com/Gavin-Yinld/coMethy/blob/master/figures/parameter.png"/></div>

## Step 2. WGCNA analysis to detect co-methylation module
Network construction is performed by using the `blockwiseModules` function in the `WGCNA` package. For each k-means group, a pair-wise correlation matrix is computed, which is converted to an adjacency matrix by raising the correlations to a power. The proper power parameter needs to be chosen by using the scale-free topology criterion in step 1. For example, the power of 16, 18, and 20 are chosen for the networks built for each k-means group, to balance the scale-free topology property and the connectivity of the genomic loci in the network.
```R
module <- co_methylation_step2(data=meth_data,
                               kmeans_result=kmeans_cluster,
                               softPower_list=c(16,18,20),plot=T)
# By setting "plot=T", a file named "wgcna.module.pdf" will be generated to show the methylation level of the loci in each co-methylation module.
# Each grey line represents the methylation level of one genomic loci.
# The red line in each co-methylation module represents the average methylation level of this module.
```
<div align=center><img width="700" height="525" src="https://github.com/Gavin-Yinld/coMethy/blob/master/figures/wgcna.module.png"/></div>

## Step 3. PCA analysis to extract eigen-loci from each co-methylation module
PCA analysis is performed to pick a set of pCSM loci with the largest loading in PC1 as eigen-loci in each co-methylation module to represent the methylation trend in the module.
```R
eigen_loci <- extract_eigen(methy_data=module$profile,
                            all_label=module$module_id,
                            number_of_eig=100,plot=T)
#By setting "plot=T", a file named "eigen_loci.pdf" will be generated to show the methylation level of the eigen-loci in each co-methylation module.
#Red lines in each co-methylation module represent the eigen-loci picked by PCA analysis.
```
<div align=center><img width="700" height="525" src="https://github.com/Gavin-Yinld/coMethy/blob/master/figures/eigen_loci.png"/></div>

coMethy is a major step in our pipeline to perform virtual methylome dissection, and the final step to dissect the methylomes is described below.

## Non-negative matrix factorization (NMF) analysis to decompose methylomes based on the methylation profile of eigen-pCSM loci
NMF analysis is used to explore the composition of the target methylomes. The methylation matrix of eigen-loci in all samples will be decomposed into a product of two matrices: one for the methylation profiles of the estimated cell types and the other for the cell-type proportions across all samples. 'MeDeCom' package is adopted to perform the NMF analysis. For details, please check (https://github.com/lutsik/MeDeCom/blob/master/vignettes/MeDeCom.md).
```R
library(MeDeCom)

#MeDeCom requires intensive computations. The processing of this simple data matrix may take several minutes.
medecom.result<-runMeDeCom(as.matrix(eigen_loci$methy_prof), 2:5, 10^(-5:-1), NINIT=10, NFOLDS=10, ITERMAX=300, NCORES=9)

#The methylation profile of the estimated cell types and their proportions across all samples can be obtained:
profile<-getLMCs(medecom.result, K=5, lambda=0.01)
proportion<-getProportions(medecom.result, K=5, lambda=0.01)

#The number of cell compositions K could be determined by prior knowledge and regularizer shifts parameter λ could be selected via cross-validation provided in the MeDeCom package. In case that no prior knowledge of cell composition is available for the input methylomes, both k and λ may be selected via cross-validation as suggested in the MeDeCom package.
plotParameters(medecom.result)

```

