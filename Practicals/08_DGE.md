# Practical 8 - Differential Gene Expression

In this practical we will perform a differential gene expression analysis to determine differentially expressed genes in a plant under certain conditions.

## Introduction

Our goal for this experiment is to determine which *Arabidopsis thaliana* (a small flower) genes respond to nitrate. 
The dataset is a simple experiment where RNA is extracted from roots of independent plants and then sequenced. 
Two plants were treated with a control solution (KCl) and two plants were treated with Nitrate (KNO3).

## Input data

The sequences were pre-processed to remove all low quality sequences, trim all low quality nucleotides, and finally aligned against the *Arabidopsis* reference genome using TopHat. 
Our analysis starts from the `.bam` files created by the alignment/mapping program.

We have been provided the following files:

- 4 bam files: alignment files, one for each sample
- `Arabidopsis.gtf` file: Arabidopsis thaliana reference genome annotation. This contains information about the genes in Arabidopsis and where they are located in the genome.
- `expdesing.txt`: experimental design, a comma separated file containing meta data of the samples (which sample is wich condition)
- `gene_description.txt`: description of the functions of the genes in Arabidopsis.

The data is available for on [Zenodo](https://zenodo.org/uploads/13120340) (08_DGE.zip).
Download the data, and extract the `zip` archive to a folder of your choice.

## Working in Rstudio

Open Rstudio, and create a new document (Rscript or Rmarkdown).
Set the working directory to the folder containing the downloaded data by pressing "Session" on the top, and then "Set Working Directory" and "Choose Directory...".
This will print a command (starting with `setwd` to the console. 
You can copy that command into your script so you are sure to set the working directory correctly each time you run the script.

## Installing required packages

The code below will install the correct packages necessary for this practical. 
We use [Bioconductor](https://www.bioconductor.org/) to download the necessary bioinformatic packages in R.

```
# If we don't have BiocManager
if (!requireNamespace("BiocManager", quietly = TRUE)) {
  install.packages("BiocManager")
  library(BiocManager)
}

# Installs all packages you don't have in this list
package.list <- c("Rsamtools", "GenomicFeatures","GenomicAlignments","DESeq2","GOstats","GO.db","Category","org.At.tair.db","txdbmaker")
for (package in package.list) { 
  if (!requireNamespace(package, quietly = TRUE))
    BiocManager::install(package, ask = FALSE)
}
```

## Generate counts from bam files

To perform differential gene expression (DE) analysis, we need to compare gene expression between genes. 
We approximate the expression of a gene by the number of reads that map to it (and normalize to avoid bias).
To do this , we first need to load all the bam files. 
The normal way to do this is to create a BamFileList object, that contains the location of the bam files, but does not load them into R. 
This is because bam files can be huge (often more than 5GB) , and can make your computer run out of memory if you load them all at the same time.
> The code assumes that your working directory contains the bam files.
> If this is not the case, you can set the `data_folder` variable to the folder containing the bam files.

```
library(Rsamtools, quietly = TRUE)
data_folder <- "./"
bamfiles <- BamFileList(dir(path = data_folder, pattern="sorted.bam", full.names = TRUE))
print(bamfiles)
```

<details>
<summary>Why do we use pattern = "sorted.bam" above, what if you set pattern = "" ?</summary>

_The pattern tells the function to only include files ending in `sorted.bam`._
_If we set the pattern to "", it include all files._
</details>

We now have a list of bam files, which contain information about the read mapping to the genome.

## Loading the annotation file

The bam files contain information about where the reads map in the genome. 
It does not contain any information about where the genes are.
We thus need to have the annotation, and load it into R before we can count our reads per gene.
The annotation is here stored in a `.gtf` file.
The purpose of a `gtf` file is to store the location of genome features, such as genes, exons, and CDS. 
It is a tab delimited text file which contains information about all annotated features of the genome (location, length, strand, ...).

We will load the `gtf` as a database object (called `txdb`). 

```
library(GenomicFeatures, quietly = TRUE)
library(txdbmaker, quietly = TRUE)
txdb = makeTxDbFromGFF(paste0(data_folder, "Arabidopsis.gtf"), format="gtf")
```

We now know where our genes are.
Now, we need to determine which annotated regions we will include for our counting. 
Since we are dealing with RNA-seq data, we want to count all exons (5' UTR + CDS + 3' UTR), both for protein-coding as non-protein-coding (e.g. ncRNAs) genes. 
We can do this using the following code:

```
ebg = exonsBy(txdb, by="gene")
ebg
```
> The `by="gene"` option tells the function to use the real gene names instead of the transcript names.
> This makes it easier for us to understand what the genes do.

<details>
<summary>How many genes are present in the .gtf file?</summary>

_33610, the number of elements in the `ebg` list_
</details>

<details>
<summary>If we would want to check reads mapping to the CDS (coding sequence), and not to the UTRs, what function would we need to use? (hint: run ?exonsBy)</summary>

_`cdsBy(txdb, by="gene")`_
</details>

<details>
<summary>There are many functions that we can apply to the 'ebg' object. What do the functions start(ebg), end(ebg), seqnames(ebg), and strand(ebg) do?</summary>

- _`start()`: shows the start positions of each exon in each gene._
- _`end()`: shows the end positions of each exon in each gene._
- _`seqnames()`: shows the name of the sequence on which each gene is found (i.e. which chromosome)._
- _`strand()`: shows the strand on which each gene is located._

</details>

## Counting the reads

Now we will use the GenomicAlignments package to determine on which feature (if any) the reads map.
The function `summarizeOverlaps` uses the bam files and the genome annotation to count the number of reads that map to a region that is annotated as a gene. 
We use the "union" mode, which means that we will consider all exons of a gene as part of the transcript. 
We also state that the reads that were mapped to the genome were single-end reads, and not paired-end reads.

```
library("GenomicAlignments", quietly = TRUE)
se = summarizeOverlaps(features=ebg, 
    reads=bamfiles, mode="Union", singleEnd=TRUE, 
    ignore.strand=TRUE ) 
```

<details>
<summary>Look at the "se" object we just created. What are the dimensions? What do they correspond to?</summary>

_33610 by 4. The 33610 rows are the genes, the 4 columns are the four samples we are analyzing._
</details>

Now that we know what reads map to which genes, we can count how many map to each gene/feature.
The results will be stored in a count table, showing how many reads are mapped to each gene in every condition.

```
counts = assay(se) 
countgenenames = gsub("[.][1234567890]", "", row.names(counts)) 
rownames(counts)=countgenenames 
```
> The second command removes the ".1", ".2", etc. that is often found after transcripts to show which isoform it is.

Not all genes are expressed in every condition. Many genes will have no or only very few reads mapped to them.
This makes it more difficult to do statistical analyses on them, so we generally filter them out.
We will filter out genes which have an average count of fewer than 10 genes.

```
row_means <- rowMeans(counts)
counts_filtered = counts[row_means >= 10,]
dim(counts); dim(counts_filtered)
```

<details>
<summary>How many genes were filtered out?</summary>

_We had originally 33610 genes, and have only 10193 genes left. We thus have filtered out 23417 genes._
</details>

## Implementing the experimental design

In order for the differential gene expression to work, we need to tell the program how our experiment is designed. 
This means we need to label the columns (= experiments), and tell it which experiments to compare. 
Here, we use a simple comma separated file created using a text editor (`expdesign.txt`). 
In that file, the first column corresponds to the bam filename, and the second column to the treatment.

We will load in the experimental design using `read.csv` (read.table, read.csv, and read.delim are all very the same function but with different default parameters). 
This will create a `data.frame` object we can use later on. 
Additionally it will convert all text (i.e. non-numbers) to factors, which is useful for our downstream analysis. 
A factor is basically a label that tells us which files/objects/rows belong together (e.g. underwent the same treatment).

```
expdesign = read.csv(paste0(data_folder, "expdesign.txt"), row.names=1, sep=",")
expdesign
```

## Calculating differentially expressed genes

Using the filtered count table, and the experimental design, we can now perform differential gene expression analysis. 
We will use [DESeq2](https://bioconductor.org/packages/release/bioc/html/DESeq2.html) for this. 
DESeq2 performs differential gene expression analysis using the negative binomial model.

We start by creating an object that serves as input for DESeq2 using the function `newCountDataSet`. 
In order to create this dataset, we need the filtered data frame of read counts and the factor that corresponds to the treatment groups (in our case "condition").

```
library(DESeq2, quietly = TRUE)

ds <- DESeqDataSetFromMatrix(countData=counts_filtered,
colData=expdesign,
design= ~ condition)
```

And then we run the analysis using the prepared object:

```
analysis <- DESeq(ds)
```

This will create an analysis object, containing all information about the analysis. 
We can get the actual results using the following command:

```
res <- results(analysis)
head(res)
```

You should see the results of the first few genes.

<details>
<summary>What is the meaning of the different columns?</summary>

- _baseMean: the mean of normalized counts over all samples._
- _log2FoldChange: the log2 of the fold change from untreated to treated._
- _lfcSE: the standard error on the log2 fold change._
- _stat: the value of the calculated statistics._
- _pvalue: the associated p-value of the statistic._
- _padj: the adjusted p-value._
</details>

<details>
<summary>What does the p-value mean in the case of differential gene expression?</summary>

_In general, a p-value tells you how likely it is to observe a certain value given a certain (background) distribution._
_Here, the p-value tells us how likely it is to observe the values of the treated samples, given the distribution of the untreated samples._
_In other words, the lower the p-value, the lower chance we have to observe the values from the treated samples just by chanche,_
_and the more likely it is that there is a real difference between the samples._
</details>

<details>
<summary>Why are some log2FoldChange values positive, and some negative?</summary>

_It depends on wether the counts for the treated sample are higher or lower than the values for the untreated sample._
_The fold change is calculated as the ratio of the mean of the treated samples to the untreated samples._
_If the treated samples have higher counts, the ratio (fold change) will be >1, leading to a positive log2 value._
_If the treaded samples have lower counts, the ratio (fold change) will be between 0 and 1, and the log2 value will be negative._
</details>

<details>
<summary>Why do we need to adjust the p-values?</summary>

_The p-values need to be adjusted for multiple testing._
_Since we do a statistical test on every gene, and we have a lot of genes, we are more likely to find a significant difference between counts just by chance._
_By adjusting the p-value, we take this into account, and try to better control the amount of false positives we might find._
</details>

<details>
<summary>What type of multiple testing adjustment does DESeq2 use by default?</summary>

_By default, DESeq2 uses the Benjamini-Hochberg FDR correction (https://hbctraining.github.io/DGE_workshop/lessons/05_DGE_DESeq2_analysis2.html)_
</details>

Now the analysis is done, we can count the number of significantly differentialy expressed genes between the two treatments.

```
sum(res$padj < 0.05, na.rm=T)
```
> The sum command is normally used to calculate the sum of numeric values. 
> However here we use it on a logic vector containing TRUE and FALSE values.
> When counting, TRUE will be counted as 1, and FALSE as 0.
> As such, we can count the number of TRUE values to get the amount of genes with an adjusted p-value below 0.05.

## Plotting differential expression

We can graphically summarize the results in an MA-plot.

```
plotMA(res, ylim=c(-5,5))
```
> This plot shows us for every gene (dot) the average expression over all samples (x-axis), and the log2FoldChange (y-axis).

Now using the data you created during this tutorial, try answering the following questions:

<details>
<summary>Which gene is the most upregulated (highest Log2FC) in the treated condition? Which one is the most down-regulated (lowest log2FC)?</summary>

_Most upregulated: AT1G67810 (Log2FC: 3.10781); Most downregulated: AT1G68238 (Log2FC: -5.87246)._
_One way of getting this is to sort the results by Log2FC: `res[order(res$log2FoldChange),]`_
</details>

To further narrow down the list of differentially expressed genes, we can also put a cut-off on the log2FC.
An often used log2FC cut-off is 1/-1 or 2/-2.

<details>
<summary>What does a log2FC of 2 mean in terms of fold change?</summary>

_A log2FC of 2 means a normal fold change of 4_ ($\log2(4)=2$).
_This means that the gene in a treated condition is expressed 4x more (have 4x more normalized reads mapped to them) than in an untreated condition._
</details>

<details>
<summary>How many of the significantly differentially expressed genes (padj < 0.05) have a log2FC > 2?</summary>

_6 - This can be calculated in different ways._
_One way is to filter the results by log2FC first (`resFC = res[(res$log2FoldChange >2),]`),_
_which gives us 13 results._
_Then we can check how many of the padj values are below 0.05: `sum(resFC$padj < 0.05, na.rm=T)`_
</details>

## References:

-   [DESeq2 tutorial](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html).
-   [Inspiration for this tutorial](https://learn.gencore.bio.nyu.edu/rna-seq-analysis/deseq-2/).
