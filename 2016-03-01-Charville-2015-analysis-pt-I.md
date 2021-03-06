# Charville <i>et al</i> (Stem Cell Reports, 2015) RNA-Seq Data Re-analysis

I am in the process of re-analysing the data published by Charville *et al* in Stem Cell Reports: [*Ex Vivo* Expansion and *In Vivo* Self-Renewal of Human Muscle Stem Cells (2015)](http://www.ncbi.nlm.nih.gov/pubmed/26344908). 

# Background & Preliminary Steps
Adult skeletal muscle stem cells (SCs) were prospectively isolated from human donors with fluorescence activated cell sorting (CD31-,CD45-,CD34-,ITGB1+,EGFR+). The cells were then either directly sequenced after isolation (the Quiescent group), or after a seven day period where they were cultured *in vitro* (the Activated group). Cultured cells were thought to be in a fully activated state at seven days post-isolation, undergoing an exponential growth phase. These activated cells show a sharp decrease in transplantation efficiency and engraftment. Additionally, a third group received a p38 MAPK inhibitor treatment (the Activated+p38i group). This third group of cells showed higher engraftment efficiency than their non-treated counterparts. The authors suggest that the p38 pathway signaling is a major driver of SC activation and proliferation.

![visual-abstract](http://joannacodes.github.io/images/abstract.jpg)

This analysis picks up after the reads have been aligned to the genome, and the features counted. Briefly, the sequencing files were obtained from the European Nucleotide Archive (ENA), reference PRJEB10091. Following FastQC assesment of read quality, the paired reads were aligned to the hg38 assembly of the human genome using the STAR aligner. The features were counted using featureCount. More details on this RNA-Seq pipeline can be found in a dedicated git repository [here](https://github.com/Joannacodes/RNA-Seq-pipeline-SGE-cluster), or in the previous post on this blog.

Each group contains 2 replicates corresponding to one donor, e.g.: Q1, A1 and p38i-1 come from the same individual. 

# Dependencies
```{r}
library(edgeR)
library(preprocessCore)
```

***

# Setting up the data
Each sample .txt file contains a count for each feature, the following function puts each file into the same table:

```{r eval=TRUE}
samples <- c("A1","A2","Q1","Q2","p38i_1","p38i_2")

# a function to import each .txt file with the "htseq_counts" suffix into R:
read.sample <- function(sample.name) {
  file.name <- paste("./htseq_counts/", sample.name, "_htseq_counts.txt", sep="")
  result <- read.delim(file.name, col.names=c("gene", "count"), sep="\t", colClasses=c("character", "numeric"), header = TRUE)
}

all.data <- read.sample(samples[1])
head(all.data)

for (c in 2:length(samples)) {
  temp.data <- read.sample(samples[c])
  all.data <- cbind(all.data, temp.data$count)
}
colnames(all.data)[2:ncol(all.data)] <- samples 
rownames(all.data) <- all.data$gene 
all.data <- all.data[,-1]

head(all.data)
```

Now let's get the phenotype data from a tab-delimited file:
```{r eval=TRUE}
pData <- read.table("pData.txt", row.names=1, header=TRUE, sep="\t")
pData
```
***
# Data transforms and Filtering
If you look at the overall distributions, you see there are a lot of outliers:
```{r}
boxplot(log2(all.data+1),col="blue", main="Raw distribution of counts within samples")
```

Looking at this sample by sample with histograms
```{r}
par(mfrow=c(2,3))
hist(log2(all.data[,1]+1),col="blue", main="Activated 1")
hist(log2(all.data[,2]+1),col="blue", main="Activated 2")
hist(log2(all.data[,3]+1),col="blue", main="Quiescent 1")
hist(log2(all.data[,4]+1),col="blue", main="Quiescent 2")
hist(log2(all.data[,5]+1),col="blue", main="p38 inhibitor 1")
hist(log2(all.data[,5]+1),col="blue", main="p38 inhibitor 2")
```


## Discarding uninteresting features
We discard any genes with a CPM greater than 2 in at least two samples (since we have two replicates):
```{r}
filt_edata <- all.data
summary(cpm(filt_edata))
dim(cpm(filt_edata)) 
```
We have ~58K genes, we need to narrow this down to 9-12K.

```{r}
countCheck <- cpm(filt_edata) > 2
keep <- which(rowSums(countCheck) >= 2) 
filt_edata <- filt_edata[keep,]
dim(filt_edata) 
```
Now down to 13K. Looking at the boxplot and histograms again:
```{r}
par(mfrow=c(1,1))
boxplot(as.matrix(log2(filt_edata+1)),col="orange", main="Distribution after removing low expression genes")

par(mfrow=c(2,3))
hist(log2(filt_edata[,1]+1),col="orange", main="Activated 1")
hist(log2(filt_edata[,2]+1),col="orange", main="Activated 2")
hist(log2(filt_edata[,3]+1),col="orange", main="Quiescent 1")
hist(log2(filt_edata[,4]+1),col="orange", main="Quiescent 2")
hist(log2(filt_edata[,5]+1),col="orange", main="p38 inhibitor 1")
hist(log2(filt_edata[,6]+1),col="orange", main="p38 inhibitor 2")
```
Much better!

To visualize the reads excluded: 
```{r}
cpm_log <- cpm(all.data, log = TRUE)
median_log2_cpm <- apply(cpm(all.data, log=TRUE), 1, median)
par(mfrow=c(1,1))
hist(median_log2_cpm, col="blue", main="Discarded genes to the left of the line", xlim=c(-10,15))
expr_cutoff <- -1
abline(v = expr_cutoff, col = "red", lwd = 3)
```


## Quantile Normalization
```{r}
norm_edata <- normalize.quantiles(as.matrix(log2(filt_edata+1)))
rownames(norm_edata) <- rownames(filt_edata)
colnames(norm_edata) <- colnames(filt_edata)
head(norm_edata)

par(mfrow=c(1,1))
boxplot(norm_edata,col="hotpink", main="With quantile normalization")

par(mfrow=c(2,3))
hist(norm_edata[,1],col="hotpink", main="A1")
hist(norm_edata[,2],col="hotpink", main="A2")
hist(norm_edata[,3],col="hotpink", main="Q1")
hist(norm_edata[,4],col="hotpink", main="Q2")
hist(norm_edata[,5],col="hotpink", main="p38i_1")
hist(norm_edata[,6],col="hotpink", main="p38i_2")
```

***

# Exploratory Analysis

## Data clustering

Calculate euclidian distances and cluster the data
```{r}
dist1 = dist(t(norm_edata))
hclust1 = hclust(dist1)
par(mfrow=c(1,1))
plot(hclust1, xlab=NULL)
```

## Everyone likes a heatmap
```{r}
heatmap(cor(norm_edata), main="Heatmap of Log CPM correlation", Colv = NA, margins=c(5,5))
```

## Dimension reduction
Remove the mean variation for PC analysis
```{r}
svd1 = svd(norm_edata - rowMeans(norm_edata))
par(mfrow=c(1,1))
plot(svd1$v[,1],svd1$v[,2],xlab="PC1",ylab="PC2",col=c(1,1,2,2,3,3),
     main="PC1 vs PC2", ylim=c(-0.7,0.6))
legend("top", legend=c("Activated", "Quiescent", "p38i Treated"), col=c(1,2,3), pch=1)
```
Looking closer at how these break down compared to the pheno data

### PC1
```{r eval=FALSE}
boxplot(svd1$v[,1] ~ pData$Type, border=c(4,2), main="PC1 as a function of group") 
points(svd1$v[,1] ~ jitter(as.numeric(pData$Type)),col=c(4,4,2,2,4,4))
```

![PC1-group](http://joannacodes.github.io/figs/PC1_group.svg)

### PC2
```{r eval=FALSE}
boxplot(svd1$v[,2] ~ pData$Donor,border=c(4,2), main="PC2 as a function of Donor")
points(svd1$v[,2] ~ jitter(as.numeric(pData$Donor)),col=as.numeric(pData$Donor))
```

![PC2-donor](http://joannacodes.github.io/figs/PC2_donor.svg)

### Finally, a variation on PCA with plotMDS
```{r}
plotMDS(norm_edata, top=1000, main="plotMDS of top 1000 genes", col=c(1,1,2,2,1,1), pch=24)
legend("bottomleft", as.character(unique(pData$Type)), col=1:2, pch=24)
```

***

# Session Info
```{r}
sessionInfo()
```
