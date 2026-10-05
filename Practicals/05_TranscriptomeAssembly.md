# Practical 5 – Transcriptome Assembly

In this practical we will perform transcriptome assemblies on a dataset containing RNA-seq reads mapping to the human chromosome 15.

## Software installation and data retrieval

In this tutorial, we will have a look at the following software:

- [Trinity](https://github.com/trinityrnaseq/trinityrnaseq/wiki) - A _de novo_ RNA-seq transcript assembler
- [Stringtie2](https://github.com/skovaka/stringtie2) - A reference-based RNA-seq transcript assembler
- [Hisat2](https://daehwankimlab.github.io/hisat2/) - A fast and sensitive RNA alignment tool
- [Minimap2](https://github.com/lh3/minimap2) - A versatile aligner for sequences of any length, both DNA and RNA
- [samtools](https://github.com/samtools/samtools) - A very handy toolkit for handling alignment data (`.sam` and `.bam` files)
- [BUSCO](https://busco.ezlab.org/) - A tool to assess assembly completeness
- [GFFcompare](https://github.com/gpertea/gffcompare) - A tool for comparing and evaluating transcriptome assemblies

### For students using NREC

The data and software has been setup on the NREC server. 
Before starting the practical, make sure to activate the correct environment before each part of the tutorial 
(e.g. `QC` for the QC part, `Transcriptome` for the transcriptome part, `BUSCO` for running BUSCO)

Then navigate to your work folder.
We will not work on the home folder (`~` or `/home/{your_username}`) because there is only limited storage space (20Gb).
You will be working on a mounted drive (200Gb) which is located in `/storage`.
All students on NREC will have their own folder `/storage/{your_username}` (e.g. `/storage/brdan` if your username is "brdan").

First of all, go to your work folder:

```
cd /storage/{your_username}
```
> Replace `{your_username}` with your username on the server.

You can make a copy of the data you will be working on by running this command from your work directory:

```
mkdir -p Practical5
ln -s /storage/data/05_Transcriptome/* Practical5/
```
> `mkdir -p` creates a folder called Practical5. The "-p" options tells mkdir to create subdirectories if necessary, and to not give an error if the folder(s) already exist
> `ln -s` creates what we call a "symbolic link". This creates a small file that just says "Instead of this file, use the file that I'm liking to". 
> This allows you to "copy" files without actually having to make a physical copy.

Now, go to the newly created directory (by running `cd Practical5`), and you are ready to start!

### For students running on their own pc

You will first have to setup the correct environment with the necessary tools. 
See [the intro practical](00_IntroSetup.md) on how to install mamba, and how to create an enviroment for downloading the necessary data.

To create a new environment including the necessary tools for this practical, run the following command:

```
mamba create -n Transcriptome trinity hisat2 samtools busco minimap2 stringtie gffcompare ncbi-datasets-cli seqtk cutadapt
```
> This will create a new environment called "Transcriptome" with the necessary tools.
> The ncbi-datasets-cli will allow us to download our reference genome.

Create a new folder (e.g. `Practical5`), go into it (`cd Practical5`), and then download the necessary data.
You can either:

- Download directly from the [Zenodo repository](https://zenodo.org/uploads/13120340):

```
wget https://zenodo.org/records/13120340/files/05_Transcriptome.zip
unzip 05_Transcriptome.zip
pigz -d 05_Transcriptome/*gz
```

- Download and filter the data manually using the commands below:

> Remember to activate the download environment (see [here](00_IntroSetup.md)).
> This will download the data, map it to the human Chr15, and extract only the reads that map to that chromosome.
> Note: Total download size is +- 15 Gb; Size after mapping is +- 25Gb; Size after filtering is +- 3.5Gb

> Important: Part of the data-prep is mapping the reads to Chr15. It is not recommended to do this on a normal desktop computer.
> Running the commands as they are written (i.e. with only 2 cores) might take multiple hours to complete the mappings!
> If you don't have access to high-performance computing resources, we highly recommend downloading the prepared files from Zenodo (see above).

<details>
<summary>Click here to expand the command necessary for setting up the data yourself</summary>

```
datasets download genome accession GCF_000001405.40 --chromosomes 15 --include genome,gtf
unzip ncbi_dataset.zip
mv ncbi_dataset/data/GCF_000001405.40/chr15.fna ./Chr15.fasta
mv ncbi_dataset/data/GCF_000001405.40/genomic.gtf ./HomSap.gtf
rm -r ncbi_dataset* README*

prefetch SRR29044527
fasterq-dump -p --outdir ./ --split-files SRR29044527/SRR29044527.sra
mv SRR29044527_1.fastq RNA_PE_1.fastq
mv SRR29044527_2.fastq RNA_PE_2.fastq
rm -r SRR29044527*

prefetch SRR29744300
fasterq-dump -p --outdir ./ --split-files SRR29744300/SRR29744300.sra
mv SRR29744300.fastq RNA_Nano.fastq
rm -r SRR29744300*

prefetch SRR24153956
fasterq-dump -p --outdir ./ --split-files SRR24153956/SRR24153956.sra
mv SRR24153956.fastq RNA_HiFi.fastq
rm -r SRR24153956*

hisat2-build -p 2 Chr15.fasta Chr15.index
hisat2 -p 2 -x Chr15.index -1 RNA_PE_1.fastq -2 RNA_PE_2.fastq -S RNA_PE.sam
samtools view -b -F 4 RNA_PE.sam | samtools sort -n > RNA_PE.bam
samtools fastq -1 RNA_PE_F_Chr15.fastq -2 RNA_PE_R_Chr15.fastq -0 /dev/null -s /dev/null RNA_PE.bam

paftools.js gff2bed HomSap.gtf > HomSap.bed 
minimap2 -t 2 -ax splice -uf -k14 --junc-bed HomSap.bed Chr15.fasta RNA_Nano.fastq > RNA_Nano.sam
samtools view -b -F 4  RNA_Nano.sam | samtools sort > RNA_Nano.bam
samtools fastq RNA_Nano.bam > RNA_Nano_Chr15.fastq

minimap2 -t 2 -ax splice:hq -uf --junc-bed HomSap.bed Chr15.fasta RNA_HiFi.fastq > RNA_HiFi.sam
samtools view -b -F 4 RNA_HiFi.sam | samtools sort > RNA_HiFi.bam
samtools fastq RNA_HiFi.bam > RNA_HiFi_Chr15.fastq

rm RNA_Nano.fastq RNA_HiFi.fastq RNA_PE_1.fastq RNA_PE_2.fastq *bam *sam
```
</details>

## The Data

The data for this tutorial are paired Illumina RNA-seq reads, Nanopore RNA, and PacBio HiFi RNA reads from human cells. 
All read sets have been mapped to the Chr15 chromosome and the mapped reads were extracted.
For an overview of the data used in this practical, please see the [information sheet on Zenodo](https://zenodo.org/uploads/13120340).

## Quality control

As should be the habit by now, we will have a look at read QC first. 
Run FastQC on the four files, and look at the reports.

> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>How many reads are in each file?</summary>

_367 483 reads in the Nanopore dataset; 203 195 reads in each Illumina dataset; 133 885 reads in the HiFi dataset._
</details>

<details>
<summary>What are the longest Nanopore and HiFi reads? Is this expected?</summary>

_The longest Nanopore read is "only" 5015bp long, the longest HiFi read is 12288bp long._
_This is shorter than expected for long reads, but is normal since we are only sequencing transcripts, which have a limited length._
</details>

<details>
<summary>Do you think we need any trimming on these files?</summary>

_The Illumina files do not need trimming (based on the read lenghts we can see that trimming has likely already has been performed._
_The long reads have some Poly-A tails that could be trimmed (especially HiFi), and the Nanopore data has some low quality bases._
</details>

We will do some quick trimming using cutadapt on the long reads, and replace the original reads with the filtered ones.
> Remember to switch to the QC enviroment to have access to the cutadapt program!

```
cutadapt -q 20,20 -m 30 --poly-a -o temp.fastq RNA_HiFi_Chr15.fastq
mv temp.fastq RNA_HiFi_Chr15.fastq
cutadapt -q 20,20 -m 30 --poly-a -o temp.fastq RNA_Nano_Chr15.fastq
mv temp.fastq RNA_Nano_Chr15.fastq
```

Note that cutadapt can do some basic trimming, but it is recommended to use proper long read trimmers on long reads.
In addition is it recommended to perform read correction on Nanopore (e.g. using short reads) to improve their quality, and to prevent errors.

## Reference-based assembly

We will start by having a look at reference-based transcriptome assembly. 
Reference-based means we will use the reference genome to guide our transcriptome assembly.
To do this, we need to first map the reads to the genome, and then use a transcriptome assembler to assembly the actual transcripts.
Here we will use [Hisat2](https://daehwankimlab.github.io/hisat2/) and [Minimap2](https://github.com/lh3/minimap2) to map the reads to the genome,
and [Stringtie2](https://github.com/skovaka/stringtie2) to assemble the mapped reads.

However, first we need to get our reference genome (in our case human Chr15) and reference annotation (for the whole genome). 
We can download the correct files using [NCBI-datasets](https://www.ncbi.nlm.nih.gov/datasets/):
> If you downloaded and processed the data manually instead of copied from NREC or from Zenodo, you don't need to do this step

```
datasets download genome accession GCF_000001405.40 --chromosomes 15 --include genome,gtf
unzip ncbi_dataset.zip
mv ncbi_dataset/data/GCF_000001405.40/chr15.fna ./Chr15.fasta
mv ncbi_dataset/data/GCF_000001405.40/genomic.gtf ./HomSap.gtf
rm -r ncbi_dataset* README*
```

Let's take a look at the reference sequence. You can view a file on the command line using the `less` command:

 ```
 less Chr15.fasta
 ```
 > You can use the arrow keys to move the cursor, `PgUp` and `PgDwn` to go to the previous or next page.
 > You can press `g` to go the the beginning of the file, or `G` to go to the end of the page.
 > Press `Q` to exit the program.

<details>
<summary>The chromosome starts and ends with a lot of `N`s. Why could that be?</summary>

_This sequence is "hard-masked". Masking is a process where we indicate in a genome file where there are repeated sequences._
_There are two ways of masking: soft-masking and hard-masking._ 
_In softmasking, the repeats are marked in lowercase letters. In hardmasking the repeats are replaced by `N`s._
_The main reason to do repeatmasking is because eukaryotic sequences contain a lot of repeats._
_But for a lot of applications (e.g. RNA-seq mapping or genome annotation) we don't really care about the repeats._
_By masking the repeats we tell our programs to not look in these regions. This means they have to only consider a smaller part of the genom, increasing running time._
_The reason we have a lot of `N`'s at the beginning and end of our sequence, is because those are the telomeres which are very repetitive sequences._
</details>

Now open the annotation file (`HomSap.gtf`) using the same command.

<details>
<summary>What type of information is stored in a `gtf` file?</summary>

_Annotation information. This files tells you what kind of features (genes, introns, exons, repeats, ...) are found where in the genome._
_To get an overview of what the different columns mean, see [here](https://www.ensembl.org/info/website/upload/gff.html?redirect=no)._
</details>

Now that we had a look at the reference genome, we can start performing our reference-based transcriptome assembly.
We'll start by assembling the short reads. To do this, we'll map them first to the reference genome using HiSat2.
Similar to what we did in the [mapping practical](04_ReadMapping.md), we will identify the splice sites, and use that information to build the genome index.

```
hisat2_extract_splice_sites.py HomSap.gtf > HomSap.ss
hisat2_extract_exons.py HomSap.gtf > HomSap.exons
hisat2-build -p 2 --exon HomSap.exons --ss HomSap.ss Chr15.fasta Chr15.index
```

Now that we have our genome index, we can align the reads, and create a sorted `bam` file.

```
hisat2 -p 2 --dta -x Chr15.index --max-intronlen 5000 \
-1 RNA_PE_F_Chr15.fastq -2 RNA_PE_R_Chr15.fastq \
| samtools view -Sb \
| samtools sort > RNA_PE_Chr15.bam
```
> The `--dta` option of hisat2 is to tell the software to produce an output tailored for transcriptome assembly. 
> If you want to do just mapping, you can omit this option.

<details>
<summary>HiSat prints some mapping statistics to the screen. Based on those, did the mapping go well?</summary>

_Yes. Most reads could be mapped (97.85%), and only about 5% of reads did not map well ("aligned concordantly 0 times")_
</details>

So now we mapped the reads to the genome. But that doesn’t give us the transcriptome. 
We can use the mapping information to piece the reads together and infer the transcript structure. 
These assembled transcripts can then tell us what genes/transcripts were expressed in the tissue/condition from which they were derived. 
To piece the mapped reads together, we will use StringTie.

```
stringtie -p 2 -o PE_stringtie.gtf RNA_PE_Chr15.bam
```

This outputs a new `.gtf` file, similar to the one of the reference. This `.gtf` contains the predicted transcripts (including exons and splice-sites) based on the mapped reads.
We can use GFFcompare to compare some of the statistics between our transcripts, and the transcripts in the reference.

As the name suggests, this tool compares two gff files, and calculate some statistics that might help you assess the quality of the annotation.
Luckily for us, `gtf` files are just a special type of `gff` file, so we can also compare `gtf` files using GFFcompare.

```
gffcompare -r HomSap.gtf -o PE_gffcompare.txt PE_stringtie.gtf
```

You will likely get an error: `Error: no valid ID found for GFF record`.
This means there is something wrong with our files.
A quick google search of the error learns us that it is a problem with [the version of the reference `gtf` file](https://github.com/gpertea/stringtie/issues/361).
Luckily there is a solution mentioned in the last comment: we either get our hands on a GFF3 file, or we remove some lines from the `gtf`. 
Here we choose to download the GFF3 annotation from human:

```
datasets download genome accession GCF_000001405.40 --chromosomes 15 --include gff3
unzip ncbi_dataset.zip
mv ncbi_dataset/data/GCF_000001405.40/genomic.gff ./HomSap.gff3
rm -r ncbi_dataset* README.md
```

Now we can run gffcompare again using the `gff3` file instead:

```
gffcompare -r HomSap.gff3 -o PE_gffcompare.txt PE_stringtie.gtf
```
> We set the reference annotation (`-r`) as the human annotation, as that's the one we want to compare with.

This time it works. Since the file is quite small, we can print the content to the screen by runing `cat PE_gffcompare.txt`
This will give us some statistics about our transcripts.

There are two main parameters we're considering here: Sensitivity and Precision. 
To learn more about these, see the [gffcompare documentation](https://ccb.jhu.edu/software/stringtie/gffcompare.shtml#output-files).
To understand how these parameters work we need to know a bit about making predictions and errors. 
In short, when we predict a transcript, there are four possible possibilities:

1) We predict a transcript in a location, and there is a transcript on that location in the reference: Correct prediction -> True positive (TP)
2) We don't predict a transcript in a location, and there is no transcript in the reference: Correct prediction -> True negative (TN)
3) We predict a transcript in a location, but there is no transcript in the reference on that location: Wrong prediction -> False positive (FP)
4) We don't predict a transcript in a location, but there is a transcript in the reference on that location: Wrong prediction -> False negative (FN)

> Obviously, we don't really consider case 2 in this case, as it is difficult to assess.

In GFFcompare Sensitivity and Precision are calculated as follows:

- Sensitivity is calculated as $TP/(TP+FN)$: This tells us how many of the transcripts from the reference we have predicted in our Stringtie transcripts (i.e. how many reference transcripts can we recover).
- Precision is calculated as $TP/(TP+FP)$: This tells us how many of the transcripts we have predicted in our Stringtie transcripts are also present in the reference (i.e. what proportion of our predictions is correct).

GFFcompare calculates these statistics on different levels: base, exon, intron, intron chain, transcript, and locus. 
Based on the the output we have, it looks like we have very low sensitivity (i.e. we don't manage to predict a lot of the reference transcripts).
The precision is quite good on base, exon, and intron level, but not so great on intron chain, transcript, and locus level. 
If you want to know what these levels are exactly, have a look at the [documentation](https://ccb.jhu.edu/software/stringtie/gffcompare.shtml#output-files).
The statistics on the bottom also tell us how many exons/introns/loci we have missed, and how many novel features we have (False Negatives and False positives respectively).

<details>
<summary>Why do you think the predictions are so bad?</summary>

_Because we are comparing the predictions for Chr15 only, to the annotation of the whole genome. That is why we miss so many exons/introns/loci._
</details>

To do a better comparison, we have to compare only the reference annotation of Chr15 with our predictions. 
So we'll have to filter the reference annotation. This means we only need annotations that are located on Chr15.
We first need to find all the sequences that are part of Chr15. We can get these as follows:

```
grep ">" Chr15.fasta
```
>Since all sequence names in a fasta file start with `>`, we can use grep to give us all lines of the fasta which have that sybmol in them.

Luckily for us, Chr15 is fully assembled in one contig: `NC_000015.10`. Now we can easily filter our annotation:

```
grep "^NC_000015.10" HomSap.gff3 > Chr15.gff3
```
> Here we don't just ask grep to give us all lines containing NC_000015.10, but only the lines that *start* with NC_000015.10 (the `^` indicates the start of the line).
> The lines that start with `#` are considered comment lines, and only give human-readable information to us. We thus don't need them in our files.

Now we can use GFFcompare again, limiting the reference to Chr15.

```
gffcompare -r Chr15.gff3 -o PE_Chr15_gffcompare.txt PE_stringtie.gtf
cat PE_Chr15_gffcompare.txt
```

We see that the sensitivity has gone up by a lot, but is still not optimal. The precision is still the same, which makes sense.
Judging from the total number of loci (1616 in the reference, 912 in our stringtie), we likely don't have enough reads mapping to this chromosome to get the full transcriptome.

One cause for our sensitivity problem, is that the reference annotation contains a lot of alternative transcripts.
There are many splice variants known in humans, but they are not all expressed in the same conditions.
We can thus never really get to 100% sensitivity based on one sample.
To have a better idea of our sensitivity, we can compare to a reference annotation which contains only one transcript per locus.
Here we'll be using the [MANE](https://www.ncbi.nlm.nih.gov/refseq/MANE/) anntation.
This is an annotation set combining the NCBI RefSeq and Ensembl genome annotations, and picked the most "common" transcript for each gene/locus.
We can download the reference annotation in `gff` format using the following code:

```
wget https://ftp.ncbi.nlm.nih.gov/refseq/MANE/MANE_human/release_1.4/MANE.GRCh38.v1.4.ensembl_genomic.gff.gz
gunzip MANE.GRCh38.v1.4.ensembl_genomic.gff.gz && mv MANE.GRCh38.v1.4.ensembl_genomic.gff MANE.gff
grep "chr15" MANE.gff > Chr15_MANE.gff
sed -i 's/chr15/NC_000015.10/g' Chr15_MANE.gff
```
> We use the sed command to replace "chr15" by "NC_000015.10" to make sure that we're refering to the same reference sequence when comparing our GFF's.

Then we can use that as a reference:

```
gffcompare -r Chr15_MANE.gff -o PE_Chr15_MANE_gffcompare.txt PE_stringtie.gtf
cat PE_Chr15_MANE_gffcompare.txt
```

We see that our sensitivity has gone up to about 40-50%.
As mentioned before, it is almost impossible to get all possible transcripts from one RNA-seq experiment.
Many genes are only expressed in specific conditions and/or tissues, which are almost impossible to capture using only one sample.

<details>
<summary>Using the MANE transcripts has decreased precision. Why could that be?</summary>

_Because we only have 1 transcript per gene/locus, while in the full reference annotation we have multiple transcripts per locus._
_With the reduced transcript set it is thus less likely we have an exact hit of our assembled transcript, leading to decreased precision._
</details>

Let's see if we can do a better job using long reads. Since we're dealing with long reads we'll have to use a different mapper. 
We will be using minimap2 to do splice-aware alignment.
However, these mapping can easily take multiple hours to complete on 2 cores. 
In additon, Stringtie takes quite a bit of time as well to assemble the long transcripts we have here.
Since we don't have that much time nor computation power during the practicals, 
the resulting stringtie annotations (`.gtf`) have been provided if you are working on NREC or downloaded the data from Zenodo.
In case you want to run the mapping yourself, you can see the commands in the comment below.

>```
>paftools.js gff2bed HomSap.gtf > HomSap.bed
>minimap2 -t 2 -ax splice -uf -k14 --junc-bed HomSap.bed Chr15.fasta RNA_Nano_Chr15.fastq \
>| samtools view -b | samtools sort > RNA_Nano.bam
>stringtie -p 2 -L -o Nano_stringtie.gtf RNA_Nano.bam
>
>minimap2 -t 2 -ax splice:hq -uf --junc-bed HomSap.bed Chr15.fasta RNA_HiFi_Chr15.fastq \
>| samtools view -b | samtools sort > RNA_HiFi.bam
>stringtie -p 2 -L -o HiFi_stringtie.gtf RNA_HiFi.bam
>```
> Since PacBio HiFi reads are high quality, we use the `splice:hs` option instead of the `splice` option.
> We added the `-L` option to stringtie to indicate we're dealing with long reads

The files that you're looking for are `Nano_stringtie.gtf` and `HiFi_stringtie.gtf`.
With the Stringtie files, we can use GFFcompare again to compare the resulting transcripts to the reference:

```
gffcompare -r Chr15.gff3 -o Long_Chr15_gffcompare.txt Nano_stringtie.gtf HiFi_stringtie.gtf
cat Long_Chr15_gffcompare.txt

gffcompare -r Chr15.gff3 -o Long_Chr15_MANE_gffcompare.txt Nano_stringtie.gtf HiFi_stringtie.gtf
cat Long_Chr15_MANE_gffcompare.txt
```

<details>
<summary>How do the statistics compare to each other, and to the short read assembly?</summary>

_The HiFi-based transcriptome has better results than the Nanopore reads (higher sensitivity and precision)._
_However both long read assemblies have a lot lower precision compared to the short reads._
_When looking at the number of predicted transcripts, it looks like the long reads create a lot of transcripts (especially Nanopore)._
_The HiFi-based transcriptome however has a higher sensitivity than the other analyses. This could however be a results from a different experimental setup._
</details>

An important remark to make in this analysis is that we are dealing with a small dataset (one chromosome), and we are using most of the tools on default settings.
Additionally, we have not properly pre-processed our long reads (e.g. Nanopore polishing).
To generate beter results from this data, a lot more careful mapping and assembly has to be performed. This is however outside of the scope of this course.

## _De novo_ transcriptome assembly

In case no reference genome is available to guide the transcriptome assembly, we can still try assembling it _de novo_ (from scratch).
For this we will use a software called [Trinity](https://github.com/trinityrnaseq/trinityrnaseq/wiki).

Trinity is a quite resource-intensive program. 
To work optimally, Trinity requires high coverage of reads, which means it needs a lot of memory for the assembly.
In addition, Trinity tends to take some time to run. We have thus provided you with the Trinity transcript on NREC or through Zenodo.
We will run an assembly using only short reads, but Trinity can not assemble only long reads.
However, it can use long reads to help a short read assembly, which we'll try in a second assembly.
It preferes high-quality long reads, thus we will only use the HiFi long reads to help the trinity assembly.
We need to convert the long reads to fasta first (which we'll do using seqtk).
Below we provide the commands in case you are interested in running this yourself.

>```
>Trinity --left RNA_PE_F_Chr15.fastq --right RNA_PE_R_Chr15.fastq --seqType fq --CPU 2 --max_memory 20G --output PE_trinity
>seqtk seq -a RNA_HiFi_Chr15.fastq > RNA_HiFi_Chr15.fasta
>Trinity --left RNA_PE_F_Chr15.fastq --right RNA_PE_R_Chr15.fastq --long_reads RNA_HiFi_Chr15.fasta --seqType fq --CPU 2 --max_memory 20G --output PE_HiFi_trinity
>```
> The assembled transcripts will be called `Trinity.fasta` in the output folder

We will continue working with the transcript assembly files: `Trinity_PE.fasta`, and `Trinity_HiFi.fasta`.
These fasta files only contain transcripts: sequences derived from merging reads. 

<details>
<summary>How many transcripts are in each file?</summary>

_Trinity\_PE: 4465; Trinity\_HiFi: 4439_
</details>

Since we don't have a reference genome, we can't create a gtf or gff that we can compare to the reference. 
One way to test if our transcriptome is good, is to run BUSCO to see how complete the transcriptome is.
As the transcriptomes are quite large, it can take a bit of time (+- 30m-1h) to do the busco analysis with only 2 cores.
We show you the commands on how to run BUSCO (using the `-m trans` mode because we are working on transcriptomes), but also provide the output below.

```
busco -c 2 -m trans -i Trinity_PE.fasta --lineage primates
---------------------------------------------------
|Results from dataset primates_odb10               |
---------------------------------------------------
|C:1.3%[S:0.9%,D:0.4%],F:0.2%,M:98.5%,n:13780      |
|169    Complete BUSCOs (C)                        |
|120    Complete and single-copy BUSCOs (S)        |
|49    Complete and duplicated BUSCOs (D)          |
|30    Fragmented BUSCOs (F)                       |
|13581    Missing BUSCOs (M)                       |
|13780    Total BUSCO groups searched              |
---------------------------------------------------
busco -c 2 -m trans -i Trinity_HiFi.fasta --lineage primates
---------------------------------------------------
|Results from dataset primates_odb10               |
---------------------------------------------------
|C:1.2%[S:0.9%,D:0.3%],F:0.2%,M:98.6%,n:13780      |
|169    Complete BUSCOs (C)                        |
|123    Complete and single-copy BUSCOs (S)        |
|46    Complete and duplicated BUSCOs (D)          |
|29    Fragmented BUSCOs (F)                       |
|13582    Missing BUSCOs (M)                       |
|13780    Total BUSCO groups searched              |
---------------------------------------------------
```

<details>
<summary>Compare the BUSCO scores. Are they similar? Why are they so low?</summary>

_There is only a minor difference in BUSCO scores (a couple of genes)._
_The BUSCO scores are so low because we are only working on one chromosome. Chr15 is one of the smaller chromosomes, and thus has generally less genes._
_This means we will only pick up a very small fraction of the total BUSCO genes, as these are spread over the whole genome._
</details>

<details>
<summary>What would the BUSCO scores be if we used RNA-seq data for the whole genome?</summary>

_The score would be higher, but probably still lower than compared to running BUSCO on a whole genome._
_This is because not all genes are expressed at all times in a cell/tissue._
_So, when sampling cells for their RNA, this will only give you a snapshot of what is happening._
_A lot of genes are specific for certain cell types, tissues, times, etc., and will not show up in ever RNA-seq experiment._
</details>

Thus, BUSCO might not be the best tool to assess if our transcriptome is good, but it could give us a rough idea. 
Without reference genome (since you would only use Trinity if you don't have a reference),
we don't have anything to compare to, so there is no way to fully assess the quality of our transcriptome.

In our case we do have a reference genome. Instead of trying to generate a gff/gtf file, we will just map the transcripts to the genome and visualise the mapping in IGV (see later).
In Minimap2 there is no preset option to map transcripts to a genome, so we'll treat them as high-quality, long RNA reads.
Since we're only mapping +- 4500 transcripts to the genome, this mapping goes very fast.

```
paftools.js gff2bed HomSap.gtf > HomSap.bed
minimap2 -t 2 -ax splice:hq -uf --junc-bed HomSap.bed Chr15.fasta Trinity_PE.fasta | samtools view -b | samtools sort > TranscriptMap_PE.bam
minimap2 -t 2 -ax splice:hq -uf --junc-bed HomSap.bed Chr15.fasta Trinity_HiFi.fasta | samtools view -b | samtools sort > TranscriptMap_HiFi.bam
```

We will use the generated bam files in the next section for visualisation.

## Visualisation

We have now different sets of predicted transcripts. 
Some are from Stringtie (which gives `.gtf` files), others from Trinity (`fasta` files) which we mapped to the genome (`bam` files).
We will download these annotations and mappings to have a look at how they compare to the reference genome annotation.

First of all, we need to generate the bam indexes (`.bai`) for our bam files:

```
for bam in *bam; do samtools index $bam; done
```

We also need our reference. Technically, the Human reference genome is included in IGV, but since we have worked with only Chr15, we will have to download our fasta and annotation. 
We have the `.fasta` file already, and we have a `.gff3` file as well. 
However, IGV doesn't really like `gff3` and prefers `gtf`. Thus, we need to first create a Chr15-only `gtf`:

```
grep "^NC_000015.10" HomSap.gtf > Chr15.gtf
```

We need to download the necessary files: the reference data, the Stringtie annotations and the Trinity mappings.
The relevant files are: `Chr15.fasta`, `Chr15.gtf`, all `.bam` and `.bai` files, and all stringtie `.gtf` files.

> If you have problems downloading the files, you can find the relevant files on [Zenodo](https://zenodo.org/records/17119891).

Now open IGV. Go to "Genomes" -> "Load genome from file", and load the `Chr15.fasta` file.
Then, go to "File" -> "Load from file" and load the `Chr15.gtf`, and the 3 stringtie files (`gtf`s).
Zoom in until you can properly see the reference genes (about halfway the zoom bar should be fine).
Go through the chromosome and compare our assembled transcripts to the reference transcripts.
To get a better overview of the transcripts, right click on every track, and pick the "Squished" mode, to display all transcripts instead of having them overlap.
>Since the reference track is the widest, you can put it on the bottom of the track list for having a better overview.

<details>
<summary>Are the predictions good? Which of the three assemblies is the best? If you want to have a good example, go to the "TJP1" gene.</summary>

_The predictions are not great (which we already could gather from GFFcompare)._
_Some genes are not annotated, and some of the annotations do not match with the reference._
_The Nanopore predictions are the worst. They are very fragmented, and often don't capture full genes._
_The PE predictions are quite good, and often match with the reference, but many genes are not covered._
_The HiFi predictions seem to be the best, with most transcripts covered, but not always correctly._
</details>

Remove the stringtie tracks, and load the Trinity mappings instead. Browse the mappings.

<details>
<summary>How do the generated transcripts compare to the reference transcripts?</summary>

_Not all genes are covered, some genes are covered partially, some genes have a nice transcript that covers it._
_This is normal, as we don't have so many reads here and Trinity normally requires high-coverage data to assemble high-quality transcripts._
</details>

<details>
<summary>Are there many differences between PE-only and PE+HiFi trinity assemblies?</summary>

_Not really. The differences are very minor._
_It could be that there is not much extra information that the long reads add that is not contained in the short-reads._
</details>

## Final remarks

As you might have experienced from this practical, transcriptome assembly can get messy. 
The transcriptomes that we assembled in this practical were often of mediocre or low quality.
There are multiple reasons why this is the case:

- We are only looking at partial data (one chromosome), instead of a whole genome.
- Our long reads have not been properly pre-processed (we only used a quick trimming using cutadapt, which is not designed to deal with long reads).
- Some of the methods we have used are not optimized or designed with long reads in mind.
- We only used the default settings on the tools we have used.
- Assembling a transcriptome from one read set will never capture the full transcriptome (not all genes are expressed at a certain point).

This practical is mainly to show you about the tools you can use, and to familiarize with some of the output. If you want to read more about transcriptome assembly, a good review can be found [here](https://www.ncbi.nlm.nih.gov/books/NBK569566/).

## Cleanup

Once you have performed all the analyses, it is time to do some cleanup. We will remove some files that we don't need anmyore, and will compress files to save space.

Remove the `.zip` archives that FastQC creates (Once you have the multiqc report, they are not needed anymore):
```
rm *zip
```

Remove the genome indexes, and bam indexes (after you have downloaded them for visualisation):
```
rm *index* rm *bai
```

Compress the filtered reads, reference data, and transcriptomes so they take up less space on the disk:
```
gzip *fastq Chr15* HomSap* *gtf
```

Remove some gffcompare temporary files:

```
rm *tmap *refmap *loci *tracking
```

## Analysis on full datasets

In this tutorial we only created the transcriptome of Chr15.
If you want to have a look on what the analysis would look like if we included all reads, and mapped to the full human genome,
you can have a look [here](../Other/Transcriptome_FullAnalysis.md).
