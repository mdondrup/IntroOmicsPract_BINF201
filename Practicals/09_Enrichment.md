# Practical 9 - Enrichment analysis

In the last practical we determined which genes are differentially expressed in an experiment.
The biggest problem with these kind of analyses is that sometimes you can get hundreds of differentially expressed genes.
Going through all of them is a labourious task, so we need some help.

We will use enrichment analysis to make sense of a set of genes.
In this tutorial we will perform 3 different analyses:

- Gene set enrichment by unranked set over-representation: GO set analysis using the GO database
- Gene set enrichment by ranked set over-representation: GSE analysis using the GO database
- Gene set enrichment by ranked set over-representation: GSE analysis using the KEGG database

## Data & packages

We will perform the analyses on the results from a DESeq2 analysis on _Drosophila melanogaster_.
We can directly import the data from the download link:

```
df <- read.csv("https://raw.githubusercontent.com/mdondrup/IntroOmicsPract_BINF201/refs/heads/main/Other/drosphila_example_de.csv", header=TRUE)
head(df)
```

<details>
<summary>What data does the dataframe contain?</summary>

_It contains the results from the differential gene expression analysis,_
_showing the expression, log2fold change, pvalue, etc._
</details>

We also need to download the reference data from the organism.
Since _D. melanogaster_ is a commonly used model species, there are pre-made packages/databases containing all information we need.
Below we will download that data and install the necessary packages.

```
organism = "org.Dm.eg.db"
packages_we_need <- c("clusterProfiler", "pathview", "enrichplot", "ggplot2", organism)
if (!requireNamespace(packages_we_need, quietly = T)) {
  BiocManager::install(packages_we_need)
} else print("You already have packages installed")
for(package in packages_we_need) {
  library(package, character.only = TRUE)
}
```
> We use a for loop to load all installed packages (using the `library()` function).

## Overrepresentation analysis

We'll start by a classic overrepresentation analysis.
In this analysis, we check if any function is overrepresented in our gene set.
If a function is enriched in our gene set, 
that means that we have proportionally more genes of that function in our geneset than in our whole genome.

We will do this analysis using the GO-ontology.
This ontology assigns GO-terms to genes, and these terms says something about their function.
The terms are subdivided in 3 categories: Biological process, Molecular function, and Cellular component.

In the code below we will filter out the differentially expressed genes, and perform the enrichment analysis on our set.
We will only work on genes with a log2FC < -2 or > 2.

```
# First, extract significant results (padj < 0.05)
sig_genes_df = subset(df, padj < 0.05)

# Remove NA values
sig_genes_df <- na.omit(sig_genes_df)

# Filter on log2fold change (|log2FoldChange| > 2)
sig_genes_df <- subset(sig_genes_df, abs(log2FoldChange) > 2)

# Lastly, we need the list of DE genes, and the list of all genes:
allgenes = df$X
siggenes = sig_genes_df$X

# GO analysis, for all 3 GO-categories (Biological Process; Molecular Function; Cellular Component)
GO_BP <- enrichGO(gene = siggenes,
                      universe = allgenes,
                      OrgDb = organism, 
                      keyType = 'ENSEMBL',
                      readable = T,
                      ont = "BP",
                      pvalueCutoff = 0.05, 
                      qvalueCutoff = 0.10)

GO_MF <- enrichGO(gene = siggenes,
                      universe = allgenes,
                      OrgDb = organism, 
                      keyType = 'ENSEMBL',
                      readable = T,
                      ont = "MF",
                      pvalueCutoff = 0.05, 
                      qvalueCutoff = 0.10)

GO_CC <- enrichGO(gene = siggenes,
                      universe = allgenes,
                      OrgDb = organism, 
                      keyType = 'ENSEMBL',
                      readable = T,
                      ont = "CC",
                      pvalueCutoff = 0.05, 
                      qvalueCutoff = 0.10)

# Visualise the results 
dotplot(GO_BP)
dotplot(GO_MF)
dotplot(GO_CC)
```

<details>
<summary>The dotplot only shows the top 10 enriched categories. How many enriched categories do we have in each category?</summary>

_BF: 224; MF: 58; 27. You can see this if you call the object in the terminal, or by browsing the object in the environment pane._
</details>

<details>
<summary>Based on the top 10 enriched terms in each category, what kind of tissue/organ is most likely influenced?</summary>

_The brain (neuron & axon development and signalling)_
</details>

Try redoing the analysis, but using only up- and/or down-regulated genes.
Also try running it on all the DE genes, not only the ones with a |log2FoldChange| > 2.

## Gene set enrichment analysis

In the gene set enrichment we do something like the opposite of classical enrichment.
Instead of looking if certain functions are present more than expected in our set of genes,
we are going to look if functions are enriched in genes that are differentially expressed.
This means that we will look at all genes that are associated to a function
and check if more of these genes are differentially expressed than what we would expect by chance.

We will perform this first using GO-terms. 
The function to do the enrichment analysis will do the filtering for us, so we don't need to do it here.
This does however mean we have to both give the genenames and their log2FC to the function.

```
# Extract only the log2FC
foldchange <- df$log2FoldChange

# Attach the names to the log2FC
names(foldchange) <- df$X

# Remove NA values
foldchange <- na.omit(foldchange)

# Sort the list in decreasing Log2FC (required for the analysis)
foldchange = sort(foldchange, decreasing = TRUE)

gse <- gseGO(geneList=foldchange,
             ont ="ALL",
             keyType = "ENSEMBL",
			 minGSSize = 3,
             maxGSSize = 800,
             pvalueCutoff = 0.05,
             verbose = TRUE,
             OrgDb = get(organism),
             pAdjustMethod = "none")

dotplot(gse, showCategory=10, split=".sign") + facet_grid(.~.sign)
```

<details>
<summary>What is the difference between "activated" and "repressed" categories?</summary>

_The "activated" terms are enriched in genes with a positive log2FC,_ 
_the "repressed" terms are enriched in genes with a negative log2FC._
</details>

We didn't adjust the _p_-values in our analysis here.
Try rerunning the analysis with the Benajmini-Hochberg correction, and see what changes.

### GSEA using KEGG

In the previous examples we have used the GO-terms as proxies for the gene functions.
However, we can allso use other databases.
Here, we will perform a GSEA similar using the KEGG database instead of the GO database.
The KEGG database has different kind of mappings, but the most commonly used is the KEGG pathway mapping,
assigning each gene to a certain biological pathway.
> We will continue working on the `foldchange` object from the previous analysis.

```
# First, we need to convert our gene IDs for gseKEGG function.
# This is because the gseKEGG function does not accept ENSEMBL IDs.
# Thus, we will convert them to ENTREZIDs, which can be recognized by the function.
ids <- bitr(names(foldchange), fromType = "ENSEMBL", toType = "ENTREZID", OrgDb=organism)

# You will see that some genes have multiple mappings, and that some genes have no mappings.
# This means we have some loss in our genes, and that we need to remove double mappings

# Removing duplicates
dedup_ids = ids[!duplicated(ids[c("ENSEMBL")]),]

# Filtering genes without mapping from the original dataframe
dfmapped = df[df$X %in% dedup_ids$ENSEMBL,]

# Now we do the same as before
# Extracting the log2FC
keggFC <- dfmapped$log2FoldChange

# Name vector with ENTREZIDs
names(keggFC) <- dedup_ids$ENTREZID

# Remove NA values
keggFC <- na.omit(keggFC)

# Sort in decreasing order
keggFC = sort(keggFC, decreasing = TRUE)

# Run the GSE
kegg <- gseKEGG(geneList     = keggFC,
               organism     = "dme",
               minGSSize    = 3,
               maxGSSize    = 800,
               pvalueCutoff = 0.05,
               pAdjustMethod = "none",
               keyType       = "ncbi-geneid")

dotplot(kegg, showCategory=10, split=".sign") + facet_grid(.~.sign)
```

You can see that a lot less functions are displayed.
The main reason is that a majority of genes do not have a KEGG mapping, and thus they will not contribute to the analysis.
This is because KEGG contains only biochemical pathways that are well-studied.
The benefit is, however, that we can visualise certain pathways and map our fold changes to them.
Let's have a look at one of the pathways from our GSE analysis.
You can see the significant pathways by calling the object, or looking at the summary (run `summary(kegg)`).
We will look at dme00480 - Glutathione metabolism here.

```
dme <- pathview(gene.data=keggFC, pathway.id="dme00480", species = "dme")
```
> This will create a figure and save it in your working directory

Open the created figure and have a look at it.

<details>
<summary>What are the different boxes?</summary>

_They represent enzymes/genes that are part of the pathway._
</details>

<details>
<summary>What do the colours represent?</summary>

_ They represent the log2FC of the gene coding for that enzyme._
</details>

## Enrichment analysis using browser 

Enrichment analysis can also be done using some web tools, especially for model organisms.
Many online tools exist to perform overrepresentation and/or gene set enrichment analysis.

Eg.:
 - [ShinyGO](https://bioinformatics.sdstate.edu/go/) 
 - [GOrilla](http://cbl-gorilla.cs.technion.ac.il/). As the name implies, GOrilla works only with GO annotations.
 - [DAVID](https://davidbioinformatics.nih.gov/)
 - [g:Profiler](https://biit.cs.ut.ee/gprofiler/gost)
 - [Gene Ontology](https://geneontology.org/)

Many of these tools can do multiple analyses and visualizations, and they have different organisms they support. 

Let's try doing the gene set enrichment using some of these tools.

Try some of the tools above, upload the created file, and run it using the following settings (if required):

- Organism: the correct organism we used in this tutorial
- Background set: Think about what to use as your background set
- Running mode: The appropriate one for our data (did we store the data in an ordered list, or two lists?)

Run the analysis and inspect the results, and compare to the results from the analysis in R.
Look at the different analyses and visualizations and compare the focus and strengths of the different tools.

 - Which tool is most versatile and user-friendly?
 - Do all tools accept our gene list?

You can also try doing the overrepresentation analysis.
To do that, you'll have to create a list of all your genes (background) and a list of the differentially expressed genes (target).
This means that you need to sort your genes by log2FC first, then export the gene name column. 

Some tools need a **ranked** list of genes, but all need some sort of gene list.
Try using the `write.table` function to export a ranked list of DEGs (define a sensible cutoff). 
> Hint: you can work from one of the dataframes we have used in the analyses above.

Go back to the gene expression data from Practical 8. Attempt to export a ranked gene list as well.
Do the tools accept Arabidopsis thaliana genes?

## Working with non-model species

As you might have noticed, all the above analyses use pre-existing databases of our model species.
If you are working on a species that does not have one of these databases, things are a bit trickier.

First, you'll need to annotate the genes of your species.
This means you need to attach GO-terms or KEGG pathways to your genes.
Some good annotation tools to do this are [InterProScan](https://www.ebi.ac.uk/interpro/about/interproscan/)
and the [EggNOG mapper](http://eggnog-mapper.embl.de/).

Then, you can create your own EGGNOG or KEGG mappings.
Not all packages can work with these custom mappings, 
but one example of a package that can is [TopGO](https://www.bioconductor.org/packages/release/bioc/html/topGO.html).

## References:

- https://learn.gencore.bio.nyu.edu/rna-seq-analysis/gene-set-enrichment-analysis/ 
- https://learn.gencore.bio.nyu.edu/rna-seq-analysis/over-representation-analysis/
