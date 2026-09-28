# Practical 4 – Read Mapping

In this practical we will map different kind of sequencing reads from different types of experiments to the _Arabidopsis thaliana_ (thale cress) reference genome using different mappers.

## Software installation and data retrieval

In this tutorial, we will have a look at the following mapping and related software:

- [Smalt](https://www.sanger.ac.uk/tool/smalt/) - A relatively fast short read aligner for DNA
- [BWA-Mem2](https://github.com/bwa-mem2/bwa-mem2) - An popular all-purpose aligner
- [HiSat2](https://daehwankimlab.github.io/hisat2/) - A fast and sensitive aligner for RNA (but can align DNA as well)
- [STAR](https://github.com/alexdobin/STAR) - A very popular RNA-seq aligner
- [Minimap2](https://github.com/lh3/minimap2) - A versatile aligner for sequences of any length, both DNA and RNA (but mainly used for long read alignment)
- [samtools](https://github.com/samtools/samtools) - A very handy toolkit for handling alignment data (`.sam` and `.bam` files) 

### For students using NREC

The data and software has been setup on the NREC server. 
Before starting the practical, make sure to activate the correct environment before each part of the tutorial. 
(e.g. `QC` for the QC part, `Mapping` for the mapping part)

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
mkdir -p Practical4
ln -s /storage/data/04_Mapping/* Practical4/
```
> `mkdir -p` creates a folder called Practical4. The "-p" options tells mkdir to create subdirectories if necessary, and to not give an error if the folder(s) already exist
> `ln -s` creates what we call a "symbolic link". This creates a small file that just says "Instead of this file, use the file that I'm liking to". This allows you to "copy" files without actually having to make a physical copy.

Now, go to the newly created directory (by running `cd Practical4`), and you are ready to start!

### For students running on their own pc

You will first have to setup the correct environment with the necessary tools. 
See [the intro practical](00_IntroSetup.md) on how to install mamba, and how to create an enviroment for downloading the necessary data.

To create a new environment including the necessary tools for this practical, run the following command:

```
mamba create -n Mapping hisat2 samtools smalt bwa-mem2 star minimap2 fastqc multiqc cutadapt
```
> This will create a new environment called "Mapping" with the necessary tools.

Create a new folder (e.g. `Practical4`), go into it (`cd Practical4`), and then download the necessary data. 
You can either:

- Download directly from the [Zenodo repository](https://zenodo.org/uploads/13120340):

```
wget https://zenodo.org/records/13120340/files/04_Mapping.zip
unzip 04_Mapping.zip
pigz -d 04_Mapping/*gz
```

- Download and process the data manually using the commands below:

> Remember to activate the download environment (see [here](00_IntroSetup.md)) - Total download size after subsampling: 5Gb, biggest downloaded file: 10Gb
> Note: Total download size after decompression is +- 5 Gb

<details>
<summary>Click here to expand the command necessary for setting up the data yourself</summary>

```
prefetch SRR8181715
fasterq-dump -p --outdir ./ --split-files SRR8181715/SRR8181715.sra
seqtk sample -s666 SRR8181715.fastq 1000000 > Atha_DNA_SE.fastq
rm -r SRR8181715*

prefetch SRR1945750
fasterq-dump -p --outdir ./ --split-files SRR1945750/SRR1945750.sra
seqtk sample -s666 SRR1945750_1.fastq 1000000 > Atha_DNA_PE_F.fastq
seqtk sample -s666 SRR1945750_2.fastq 1000000 > Atha_DNA_PE_R.fastq
rm -r SRR1945750*

prefetch SRR14765193
fasterq-dump -p --outdir ./ --split-files SRR14765193/SRR14765193.sra
seqtk sample -s666 SRR14765193.fastq 10000 > Atha_DNA_Nano.fastq
rm -r SRR14765193*

prefetch ERR11437332
fasterq-dump -p --outdir ./ --split-files ERR11437332/ERR11437332.sra
seqtk sample -s666 ERR11437332.fastq 10000 > Atha_DNA_HiFi.fastq
rm -r ERR11437332*

prefetch SRR11685307
fasterq-dump -p --outdir ./ --split-files SRR11685307/SRR11685307.sra
seqtk sample -s666 SRR11685307_1.fastq 1000000 > Atha_ChIP_PE_F.fastq
seqtk sample -s666 SRR11685307_2.fastq 1000000 > Atha_ChIP_PE_R.fastq
rm -r SRR11685307*

prefetch SRR28854394
fasterq-dump -p --outdir ./ --split-files SRR28854394/SRR28854394.sra
seqtk sample -s666 SRR28854394_1.fastq 1000000 > Atha_RNA_PE_F.fastq
seqtk sample -s666 SRR28854394_2.fastq 1000000 > Atha_RNA_PE_R.fastq
rm -r SRR28854394*

prefetch SRR21232854
fasterq-dump -p --outdir ./ --split-files SRR21232854/SRR21232854.sra
seqtk sample -s666 SRR21232854.fastq 100000 > Atha_RNA_Nano.fastq
rm -r SRR21232854*

prefetch SRR25410923
fasterq-dump -p --outdir ./ --split-files SRR25410923/SRR25410923.sra
seqtk sample -s666 SRR25410923_1.fastq 1000000 > Bnap_DNA_PE_F.fastq
seqtk sample -s666 SRR25410923_2.fastq 1000000 > Bnap_DNA_PE_R.fastq
rm -r SRR25410923*
```
</details>

## The Data

The data for this practical are different read sets for _Arabidopsis thaliana_ (thale cress), the most studied plant model, 
and _Brassica napes_ (rape seed), a crop plant related to _A. thaliana_.
The reads used here are only subsets of the total data, to make sure the analyses don't take too long to run.
For an overview of the data used in this practical, please see the [information sheet on Zenodo](https://zenodo.org/uploads/13120340).

##	Quality Control

We have many sets of reads, but thanks to [MultiQC](https://multiqc.info/) (see [Practical 2](02_QC.md)) we can have a quick overview of the data and figure out if we need any trimming.
Acivate the QC environment, run FastQC on all the files, and create the MultiQC reports. We'll create separate reports for the short and long reads.

```
for file in *fastq; do fastqc $file; done
mkdir shortReadsQC longReadsQC
mv Bnap* Atha_ChIP* Atha_DNA_SE* Atha_DNA_PE* Atha_RNA_PE* shortReadsQC/
mv Atha_DNA_HiFi* Atha_DNA_Nano* Atha_RNA_Nano* longReadsQC/
multiqc --outdir shortReadsQC/ shortReadsQC/
multiqc --outdir longReadsQC/ longReadsQC/
```
> Here we are using a bash for loop: for every file that ends in `fastq`, we will run fastqc on that file. 

Download and have a look at the reports.
> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>In the long reads, the RNA_Nano reads are suprisingly short. Why could that be the case?</summary>

_RNA-seq sequences transcripts. Transcripts only have a limited length._ 
_Thus, the reads can only be as long as the transcripts themselves, which is significantly shorter than the average length of genomic long reads._
</details>

There seems to be some adapter content, but the quality seems mostly ok (except the Nanopore sequences). 
To keep things simple, we will run the same trimming process on each sample (this is not recommended for real-life analysis!).
>Make sure to run the following command from the folder containing the reads.

```
for file in longReadsQC/*fastq shortReadsQC/*SE.fastq; do cutadapt --poly-a -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -q 20,20 -m 30 -o ${file}.filt $file; done
mv */*filt ./
cutadapt --poly-a -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -m 30 -q 20,20 -o Atha_ChIP_PE_F.fastq.filt -p Atha_ChIP_PE_R.fastq.filt shortReadsQC/Atha_ChIP_PE_F.fastq shortReadsQC/Atha_ChIP_PE_R.fastq
cutadapt --poly-a -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -m 30 -q 20,20 -o Atha_DNA_PE_F.fastq.filt -p Atha_DNA_PE_R.fastq.filt shortReadsQC/Atha_DNA_PE_F.fastq shortReadsQC/Atha_DNA_PE_R.fastq
cutadapt --poly-a -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -m 30 -q 20,20 -o Atha_RNA_PE_F.fastq.filt -p Atha_RNA_PE_R.fastq.filt shortReadsQC/Atha_RNA_PE_F.fastq shortReadsQC/Atha_RNA_PE_R.fastq
cutadapt --poly-a -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -m 30 -q 20,20 -o Bnap_DNA_PE_F.fastq.filt -p Bnap_DNA_PE_R.fastq.filt shortReadsQC/Bnap_DNA_PE_R.fastq shortReadsQC/Bnap_DNA_PE_R.fastq
```
>These commands will trim poly-A tails (typical in RNA-seq data), trim Illumina adapters, trim with a Q20 cutoff, and discard reads < 30bp. 
>They will create `.filt` files with the filtered reads in our directory. 
>If you want you can run FastQC on those again to check, but this is not required for the practical.

We will continue working on the `.filt` files for the rest of the practical.

## Reference genome

If we want to map reads, we need to map them to something. 
We will map the reads to the _Arabidopsis thaliana_ reference genome (TAIR10.1). To download the reference genome, run:

```
wget -O TAIR.zip "https://api.ncbi.nlm.nih.gov/datasets/v2alpha/genome/accession/GCF_000001735.4/download?include_annotation_type=GENOME_FASTA&include_annotation_type=GENOME_GTF"
unzip TAIR.zip
mv ncbi_dataset/data/GCF_000001735.4/GCF_000001735.4_TAIR10.1_genomic.fna ./TAIR.fasta
mv ncbi_dataset/data/GCF_000001735.4/genomic.gtf ./TAIR.gtf
rm -r TAIR.zip ncbi_dataset/ README.md
```

## Short read genomic mappers

First, we will have a look at short read genomic mappers. 
We will compare 3 mappers: [Smalt](https://www.sanger.ac.uk/tool/smalt/), [BWA-Mem2](https://github.com/bwa-mem2/bwa-mem2), and [HiSat2](https://daehwankimlab.github.io/hisat2/).
We will start by mapping the normal Illumina Paired-end reads using the three mappers, and see which one is the fastest.

Let's start by using Smalt. The first step of mapping is indexing the genome. 
This will make it easier to search the genome for positions where the reads map.
Make sure to activate the Mapping environment: `conda activate Mapping` before running the commands.

```
smalt index -k 15 -s 8 TAIR_smalt.index TAIR.fasta
time smalt map -n 2 TAIR_smalt.index Atha_DNA_PE_F.fastq.filt Atha_DNA_PE_R.fastq.filt > DNA_PE_smalt.sam
samtools view -bS DNA_PE_smalt.sam | samtools sort > DNA_PE_smalt.bam && rm DNA_PE_smalt.sam
```
>The index command will create a _k_-mer index of 15-mers (`-k 15`), that are sampled every 8 bp (`-s 8`).
>We use the `time` command to make a timing of how long the command that follows it takes. This allows us to compare the running time of different tools.
>Lastly, we use samtools to convert the resulting human-readable mapping file (`.sam` format) to a less space-consuming binary format (`.bam`).
>We will also sort the mappings, which will make it easier to do downstream analyses.

Now we will try doing the same using BWA-Mem. 
Since BWA-mem uses a different indexing structure than Smalt (FM-index rather than hash table), we need to create another index of our reference genome.

```
bwa-mem2 index -p TAIR_bwa.index TAIR.fasta
time bwa-mem2 mem -t 2 TAIR_bwa.index Atha_DNA_PE_F.fastq.filt Atha_DNA_PE_R.fastq.filt > DNA_PE_bwa.sam
samtools view -bS DNA_PE_bwa.sam | samtools sort > DNA_PE_bwa.bam && rm DNA_PE_bwa.sam
```

Lastly, we will do the same mapping using HiSat2.

```
hisat2-build TAIR.fasta TAIR_hisat.index
time hisat2 --no-spliced-alignment -x TAIR_hisat.index -1 Atha_DNA_PE_F.fastq.filt -2 Atha_DNA_PE_R.fastq.filt -S DNA_PE_hisat.sam
samtools view -bS DNA_PE_hisat.sam | samtools sort > DNA_PE_hisat.bam && rm DNA_PE_hisat.sam
```
>HiSat2 will give a warning about an "unsupported file format". 
>This is because our files end in `.filt` and not in `.fastq`. Don't worry, the software will still run as it should.

<details>
<summary>Which of the mappers ran the fastest, according to the "real" time?</summary>

_In my case, the "real" times were 2m46 for HiSat, 3m16 for BWA-mem, and 6m16 for smalt. However, your actual running times can differ depending on server load._
_Ideally, you perform the same task multiple times and take the average run time. This is outside the scope of this course, however._
_The "real" time is what we call "wall clock time". It is basically the difference in real time between when the command launched and when it finished._
_The "user" time is the so-called CPU time (total time the processors were running) of the user process._
_The "sys" time is the CPU time spent "outside" of the user process (i.e., communicating and coordinating between the different threads)._
</details>

Running time is one parameter to consider, but we also have to consider the mapping results. We will gather some statistics from the mapping files (`.bam`).
We will use samtools, awk, and grep for this, and put them in a for loop to have the results for all three mappings:

```
for map in *bam; do \
echo $map;
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}';
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}';
echo "Number of reads mapped:" && samtools view -c -F 260 $map;
echo ""
done
```
> Samtools depth will show us for every base, how many reads cover that base. 
> With the `awk` command, we will go over every line (= base) and at every line increase `c` with 1 (`c++`), 
> and increase `s` with the value of the third column (`s+=$3`)(the third column containins the coverage for that base).
> At the end, we will calculate the average coverage as the total coverage (`s`), divided by the total number of bases (`c`): `print s/c`.
> In the second command, we will again count all the bases with `c++`, 
> and each time we have a base which has a coverage (stored in the 3rd column) higher than zero, we will add 1 to the total (`if ($3>0) total+=1`). 
> In the end, we will calculate how much of the genome is covered by dividing the number of bases with coverage > 0 (`total`) by the total number of bases (`c`): `print (total/c)*100`. 
> We multiply by 100 to get the result in percentage instead of proportion.
> In the third command we count all reads in the BAM (`-c`), but exclude (`-F 260`) unmapped & secondary alignments.
> If you want to know more about what these commands do exactly, you can have a look [here](https://sarahpenir.github.io/bioinformatics/awk/calculating-mapping-stats-from-a-bam-file-using-samtools-and-awk/) and [here](https://www.metagenomics.wiki/tools/samtools/number-of-reads-in-bam-file).

<details>
<summary>Are there any large differences between the mapping statistics of the different mappers?</summary>

_Not really. HiSat has slightly lower coverage and a smaller fraction of the genome covered, and it mapped fewer reads than the other two mappers._
_However, the differences are relatively small._
</details>

<details>
<summary>Given that we have sequences from the whole genome, why is only 55-60% of the genome covered?</summary>

_We only used a subset of a normal sequencing run (1M reads in the case of short reads)._
_Since we have two read files (forward & reverse), we have 2 million reads of 100bp each (= 200 M bases in total)._
_Given that the A. thaliana genome is +- 136 Mbp large, an average coverage of 1.45 makes sense._
_However, since reads generally map randomly to the genome (if doing whole-genome sequencing), it is likely that a large part of reads will map to overlapping regions of the genome while other parts of the genome will have no mapped reads._
</details>
>Note: Here we have used the mappers using the default settings. All three mappers can be fine-tuned to improve mapping performance depending on the situation.

## ChIP-seq mapping

One of the datasets is derived from a ChIP-seq experiment. This means only selected regions of the genome are sequenced. Let's see what the influence is on our mapping performance.
We will use smalt here to map the reads, but feel free to use another mapper of your choice.

```
smalt map -n 2 TAIR_smalt.index Atha_ChIP_PE_F.fastq.filt Atha_ChIP_PE_R.fastq.filt \
| samtools view -bS | samtools sort > CHIP_smalt.bam
```
> We don't need to index the genome, as we already have created the index as part of the first section of this practical.
> Instead of saving the `.sam`, we will directly convert it into `.bam` and sort it. 
> We do this using the pipe symbol (`|`). This lets you use the output of one program directly into another program, without having to save the data to an intermediate file.

Again, we will calculate the mapping statistics:

```
echo "Average coverage:" && samtools depth -a CHIP_smalt.bam | awk '{c++;s+=$3}END{print s/c}'
echo "% of genome covered:" && samtools depth -a CHIP_smalt.bam | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}'
echo "Number of mapped reads:" && samtools view -c -F 260 CHIP_smalt.bam
```

We see that an even smaller fraction of the genome is covered (+- 46%) and we have a lower average coverage (+- 0.79), despite having the same amount of reads.
This is because ChIP-seq has a selection step where we only select parts of the genome to be sequenced (e.g., where a certain transcription factor binds).
Thus, most reads will be derived from these regions, and fewer reads will map to the other regions, lowering the overall covered fraction of the genome.
 
## Long Read Mapping

In this part, we will map both NanoPore and PacBio reads to the genome. 
Long read mappers use different approaches for mapping, as in general we are dealing with fewer, but longer reads. 
Many long-read mappers use approaches similar to those used for genome-to-genome alignment. 
One such mapper is [minimap2](https://github.com/lh3/minimap2), which can be used for long-read mapping but can also be used to align transcriptomes or whole genomes to a reference. 

In contrast to the short-read mappers we used, minimap2 creates the index as part of the mapping pipeline. 
We can thus go directly to mapping the reads:

```
minimap2 -t 2 -ax map-hifi TAIR.fasta Atha_DNA_HiFi.fastq.filt | samtools view -bS | samtools sort > DNA_HiFi.bam
minimap2 -t 2 -ax map-ont TAIR.fasta Atha_DNA_Nano.fastq.filt | samtools view -bS | samtools sort > DNA_Nano.bam
```
> We are telling minimap2 to do both indexing (`-x`) and aligning (`-a`), and using settings adapted to PacBio HiFi (`map-hifi`) or Oxford Nanopore (`map-ont`) data.

We will calculate the mapping statistics again:

```
for map in DNA_HiFi.bam DNA_Nano.bam; do \
echo $map;
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}';
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}';
echo "Number of mapped reads:" && samtools view -c -F 260 $map;
echo ""
done
```

<details>
<summary>Despite having a lot fewer reads (10 thousand instead of 1 million), we have similar mapping statistics (coverage and genome fraction) for the HiFi reads as the short read mappings. Why?</summary>

_Because the reads are longer. As such, we need a lot fewer reads to have the same number of bases to cover a certain proportion of the genome._
_In other words, you need fewer reads to get the same amount of total bases with longer reads._
</details>

<details>
<summary>Despite having a similar amount of total bases, the Nanopore data has worse mapping statistics compared to the HiFi reads. What could be the reason?</summary>

_This is likely due to the high error rate in Nanopore data. This means that it is more difficult to accurately map the reads to the genome, leading to fewer mapped reads, and thus lower coverage._
_Alternatively, it could be that the specimen sequenced using Nanopore is more divergent from the reference strain, leading to lower mapping performance._
</details>

## RNA-seq Mapping

Now that we have mapped both short and long DNA reads, we’ll have a look at mapping RNA-seq reads. 
Mapping RNA data is more difficult than mapping DNA data, because of transcript splicing.
In RNA-seq we normally sequence mature transcripts, which are already spliced (i.e. introns removed).
This means that parts of the transcript will map to different parts of the genome (exons), with introns inbetween.
If we want to map RNA-seq reads to a reference, we need so-called "splice-aware" aligners.
Here we'll have a look at HiSat2 and Minimap2 again, but also at the very popular [STAR](https://github.com/alexdobin/STAR) aligner.

We'll start by mapping the short reads, using HiSat2 and STAR.
To map using HiSat, we could use the same command as we used for the DNA reads, using the same index:

```
hisat2 -x TAIR_hisat.index -1 Atha_RNA_PE_F.fastq.filt -2 Atha_RNA_PE_R.fastq.filt -S RNA_PE_hisat.sam
samtools view -bS RNA_PE_hisat.sam | samtools sort > RNA_PE_hisat.bam && rm RNA_PE_hisat.sam
```
> Again, you can ignore the warning about the unsupported file format

However, since now are doings "splice-aware" alignment, we could help our mapper by adding information about the splice sites to the genome index.
Since we have a reference annotation (the `.gtf` file we downloaded earlier), we can extract the information about where the introns, exons, and splice sites are.
Then, we can use that information to create a new genome index that takes this information into account.

```
hisat2_extract_exons.py TAIR.gtf > TAIR.exons
hisat2_extract_splice_sites.py TAIR.gtf > TAIR.ss
hisat2-build -p 2 --exon TAIR.exons --ss TAIR.ss TAIR.fasta TAIR_hisat_splice.index
```

Let's have a look at the extra files we have created. We can easily look at the first few lines of a file by using the `head` command:

```
head TAIR.exons
head TAIR.ss
```

<details>
<summary>What do you think the different columns mean in these files?</summary>

- _First column: contig/chromosome_
- _Second column: start of exon/intron_
- _Third column: end of exon/intro_
- _Fourth column: strand_

</details>

Now we can map the reads using our new index:

```
hisat2 -x TAIR_hisat_splice.index -1 Atha_RNA_PE_F.fastq.filt -2 Atha_RNA_PE_R.fastq.filt -S RNA_PE_hisat_splice.sam
samtools view -bS RNA_PE_hisat_splice.sam | samtools sort > RNA_PE_hisat_splice.bam && rm RNA_PE_hisat_splice.sam
```

Let's compare the mapping statistics of both mappings:

```
for map in RNA_PE_hisat.bam RNA_PE_hisat_splice.bam; do \
echo $map;
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}';
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}';
echo "Number of mapped reads:" && samtools view -c -F 260 $map;
echo ""
done
```

<details>
<summary>Are there big differences in mapping statistics?</summary>

_Not really. The mapping statistics are highly similar._

_Also note the low proportion of the genome covered. This is because we are only mapping reads to exons, which are only a minor part of the genome._
_Most eukaryotic genomes mainly consist of non-coding sequences which will not be covered by RNA-seq data (as only coding transcripts (= mRNA) are sequenced)._
</details>

STAR is a very popular RNA-seq mapper. Let's try using it as well.
Before we can map using STAR, we need to put the genome in a separate folder. 
Just create a folder called "STAR_in", and make a copy of the genome in that folder:

```
mkdir STAR_in && ln -s $(pwd)/TAIR.fasta STAR_in/
```
> We use the `$(pwd)` here to create the exact path to the reference genome. This is preferred when making symbolic links. 
> The command `pwd` will print the full path to the current working directory.

Then, we will create a genome index. We need to add the annotation here as well, to make sure to annotate the splice junctions properly in the index.

```
STAR --runThreadN 2 --runMode genomeGenerate --genomeDir ./STAR_in \
--genomeFastaFiles ./STAR_in/TAIR.fasta --sjdbGTFfile TAIR.gtf \
--sjdbOverhang 100 --genomeSAindexNbases 10
```
> We use the `\` here to separate the command over multiple lines. Using this allows us to type one command over multiple lines, making it easier to read.
> So instead of running every line separately, it will interpret the three lines as one command.
> The `sjdb` related options in the command refer to splice junction database information. 

Then, we can map the reads using the following command:

```
STAR --runThreadN 2 --genomeDir ./STAR_in --sjdbGTFfile  TAIR.gtf --sjdbOverhang 100 --readFilesIn Atha_RNA_PE_F.fastq.filt Atha_RNA_PE_R.fastq.filt
samtools view -bS Aligned.out.sam | samtools sort > RNA_PE_STAR.bam && rm Aligned.out.sam
rm -r Log.* _STARgenome SJ.out.tab
``` 
> The last step is cleaning up some intermediate files that STAR creates

Then, let’s assess the mapping statistics:

```
map=RNA_PE_STAR.bam
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}'
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}'
echo "Number of mapped reads:" && samtools view -c -F 260 $map
```

<details>
<summary>How does STAR compare to HiSat2 in terms of mapping statistics?</summary>

_It has slightly better statistics: higher coverage, more of the genome covered, and a bit more reads could be mapped._
</details>

Lastly, we will use Minimap2 to map the long RNA reads we have. Again we need to calculate splice junctions to be able to do the spliced-aware mapping.

```
paftools.js gff2bed TAIR.gtf > TAIR.bed
minimap2 -t 2 -ax splice -uf -k14 --junc-bed TAIR.bed TAIR.fasta Atha_RNA_Nano.fastq.filt | samtools view -bS | samtools sort > RNA_NanoPore.bam
```
> We use `paftools` to convert the `.gtf` annotation file to a `.bed` file containing the coordinates of exons and introns.
> The `-uf` option forces minimap2 to only use the forward strand, as we're dealing with RNA transcripts. 
> The `-k` parameter sets the word length for the genome index. This is lowered compared to the default because we are dealing with noisy Nanopore data
> The `splice` option, as you might have guessed, tells minimap2 to do spliced alignment.

Now let's calculate the statistics:

```
map=RNA_NanoPore.bam
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}'
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}'
echo "Number of mapped reads:" && samtools view -c -F 260 $map"
```

<details>
<summary>How are the mapping results compared to the short reads?</summary>

_Not so good. All statistics are significantly lower. This is because the Nanopore reads have a lot of errors, leading to difficulties aligning all reads._ 
_Additionally, we have fewer total bases in the Nanopore set than in the short read set (83.5 Mbp vs 200 Mbp; before filtering), leading to lower coverage._
_Lastly, the experimental setup could also have played a role. Depending on the condition in which RNA was extracted, a different number of genes could be expressed, which can lead to reads mapping to fewer regions of the genome._
</details>

## Using the wrong tool

Picking the right tool for the job is important in bioinformatics. 
Earlier, we used different mappers depending on the input data.
In this part, we will try to have a look at what happens when we use the wrong tool for the job, and what would happen if we map reads from a different species.

We'll start by trying to see what would happen if we use a splice-aware aligner on DNA reads. 
We will map the PE DNA reads to the genome using STAR.
Since we already created the genome index with splice junctions before, we don't need to redo that.

```
STAR --runThreadN 2 --genomeDir ./STAR_in --sjdbGTFfile TAIR.gtf --sjdbOverhang 100 --readFilesIn Atha_DNA_PE_F.fastq.filt Atha_DNA_PE_R.fastq.filt
samtools view -bS Aligned.out.sam | samtools sort > DNA_PE_STAR.bam
rm -r Aligned.out.sam Log.* SJ.out.tab _STARgenome
map=DNA_PE_STAR.bam
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}'
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}'
echo "Number of mapped reads:" && samtools view -c -F 260 $map"
```
> Note: this mapping can take a bit of time (15-20 minutes, likely more on a busy server).

<details>
<summary>How are the mapping statistics? Do they look normal/similar to the STAR RNA mapping or smalt/bwa-mem/hisat DNA mapping?</summary>

_They look quite good and are very similar to the normal DNA mapping._
</details>

Now we will try the opposite. We will try to map the paired RNA-seq reads using a genomic mapper (in this case, smalt). 

<details>
<summary>Do you think this will work very well? What kind of problems do you expect?</summary>

_It will likely not work very well, since part of the reads should map to different part of the genome._ 
_We will either have few mapped reads, or reads that are only partially mapped (e.g. to only one of the exons)._
</details>

Again, we already have the index ready, so we can just do the mapping:

```
smalt map -n 2 TAIR_smalt.index Atha_RNA_PE_F.fastq.filt Atha_RNA_PE_R.fastq.filt \
| samtools view -bS | samtools sort > RNA_smalt.bam
map=RNA_smalt.bam
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}'
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}'
echo "% reads mapped:" && samtools flagstat $map | grep "mapped ("
```

<details>
<summary>Are the mapping statistics as you expected?</summary>

_The mapping statistics are similar to when using an RNA-mapper. This could mean that the reads get mapped, but maybe not split-up properly. We will investigate this later in this practical._
</details>

We have also investigated long reads in this practical. Let’s find out what happens when we try mapping long reads to the genome using the short-read mapper bwa-mem2:

```
bwa-mem2 mem -t 2 TAIR_bwa.index Atha_DNA_HiFi.fastq.filt > HiFi_bwa.sam
```

Sadly, it looks like the software quickly crashes. You could try using other short-read mappers (e.g. smalt), but you would quickly realize that the mapping would either take forever and/or crash. 
When running smalt to map the HiFi reads to the genome on a cluster using 32 cores the software crashed after ~20 minutes. 
As such, it seems that short-read mappers cannot handle long reads properly.

Let's try the opposite: mapping short reads with a long-read mapper (Minimap2). Since long reads are never paired, we will have to use the DNA_SE read data for this.
> Minimap2 can actually map short reads, but here we will use the long-read mapping mode instead.

```
minimap2 -t 2 -ax map-pb TAIR.fasta Atha_DNA_SE.fastq.filt | samtools view -bS | samtools sort > DNA_SE_minimapLR.bam
```

Since we haven't mapped the SE reads yet, we'll have to do that to be able to compare the statistics.
Let's use the minimap2 short read mode to do this:

```
minimap2 -t 2 -ax sr TAIR.fasta Atha_DNA_SE.fastq.filt | samtools view -bS | samtools sort > DNA_SE_minimapSR.bam
```

Then we can compare the statistics of both:

```
for map in DNA_SE_minimap*; do \
echo $map;
echo "Average coverage:" && samtools depth -a $map | awk '{c++;s+=$3}END{print s/c}';
echo "% of genome covered:" && samtools depth -a $map | awk '{c++; if ($3>0) total+=1}END{print (total/c)*100}';
echo "Number of mapped reads:" && samtools view -c -F 206 $map;
echo "";
done
```

<details>
<summary>How do the mappings compare in terms of statistics?</summary>

_Mapping using the short read option has better statistics: higher coverage, more of the genome covered, and more reads mapped._
</details>

Lastly, we'll have a quick look at what happens if we map reads from a different species. 
We'll map some _Brassica napes_ reads to the _A. thaliana_ genome and see what happens.
Pick one of the short-read mappers we have used, and try mapping the _B. napes_ reads (starting with `Bnap`) to `TAIR.fasta`, 
and compare the statistics to the one using _A. thaliana_ reads. 

<details>
<summary>What effect does mapping to the wrong genome have on the mapping statistics?</summary>

_The mapping statistics become significantly worse: lower coverage, a lot less of the genome is covered, and only about half of the reads could be mapped._
</details>

## Mapping visualisation

For now we have only looked at mapping statistics to assess the mapping quality.
However, it is not because all the reads could be mapped, that the mapping is good.
Using a genome browser (e.g. `IGV`), we can actually visualise the mappings.
To do this, we need to have sorted, indexed `.bam` files. We already sorted the `bam` files when creating them, but we still need to create their indexes.
We can quickly index all `bam` files using a for loop:

```
for bam in *bam;
do samtools index $bam;
done
```

This will create a `bam.bai` file for every bam file, containing the index (making it easier to search the file).
Now we have to download both the `bam` files and their indexes to our local pc (if you're working on NREC or another remote server).

> If you have problems downloading the files, you can find the relevant files on [Zenodo](https://zenodo.org/records/17119477).

Once you have downloaded the files, you can launch IGV.
Since _A. thaliana_ is a widely used model species, IGV has the TAIR genome included.
You can open that genome by going to "Genomes" -> "Select hosted genome", and select _A. thaliana_ from the list.
This will load both the genome, and the annotations. You should now see the 5 chromosomes at the top of your view.

We will start by loading the 3 DNA paired-end read mappings we started with (smalt, bwa-mem, and hisat2). 
You can load a `.bam` file by going to "File" -> "Load from file" and selecting the `.bam` files you want to add.
>As mentioned before, to properly display a `bam` file, it needs to be sorted, and there needs to be an associated index file (`.bai`) in the folder.
>If for some reason you lost the index file, or need to re-sort the `bam` file, you can go to "Tools" -> "Run igvtools" to sort and/or index a `.bam` file.

You should now see the three datasets added to your view. However, you likely only see gray and the message "zoom in to see coverage/alignments".
Because we are currently viewing the whole genome, we can't actually see the mapping data. We need to zoom in to have a better view of the mappings.

Select chromosome 3 by selecting it from the gray dropdown menu next to the name of the genome. Then click on the "+" button on the top right to zoom in until the blue marker is in the blue zone.
Now you should be able to see the mapping data. Zoom in until you see mapping data for all three `bam` files.

Each `bam` file displays two main "tracks": the coverage and the mappings. The hisat mapping also shows junctions, but this is not relevant for DNA-mapping.
The coverage is basically a histogram displaying how many reads cover a certain position. As you can see, the regions where more reads map have a higher coverage.
You can scroll through the chromosme by clicking and holding down the mouse botton while moving left/right. 

<details>
<summary>When we calculated the statistics, we had an average coverage of +- 1.5, and +- 60% of the genome covered. Is this visible on the mapping?</summary>

_Yes: We see that not every position has reads that map there, meaning that the reads don't cover the whole genome._ 
_There are regions with a lot of reads mapping, and regions with no reads mapping, so an average of 1.5 reads per position seems plausible._
</details>

Let's try looking at the mitochondria. Select it in the chromosome selection panel, and zoom in.

<details>
<summary>Why don't we observe any mapping to the mitochondrial genome?</summary>

_One explanation could be that we didn't have the mitochondrial sequence in our reference. However, if you check the contig names of your reference genome (`grep ">" TAIR.fasta`),_
_you will see that the mitochondrial sequence is there._
_It could be that there were no mitochondrial reads in our samples, or that there are slight differences in the reference genome we used (TAIR10.1) and the genome in IGV (TAIR10)._
</details>

Now navigate to the chloroplast sequence and zoom in.

<details>
<summary>Do we have more or less coverage than in chromosome 3?</summary>

_A lot more: there are reads mapped everywhere._
_This is because the DNA is likely derived from leaf tissue, which contains a lot of chloroplasts._
_When extracting DNA, a lot of it is thus derived from the chloroplast, leading to high coverage there._
</details>

We see some colored bars on some of the reads. Try zooming in to the maximum level to figure out what they are.

<details>
<summary>What do the colors on the reads mean? (the bars on the reads, not reads that are fully coloured)</summary>

_They indicate differences in the reads compared to the reference sequence (visible on the bottom). Each color represents a different base._
</details>
>Some reads are also fully colored in a certain color. These colorings are based on pairing of reads and insert sizes, but are not relevant here.

Since we have mapped paired-end reads, we could have a look at the pairings. Right-click on the read tracks title, and activate the "view as pairs" option.
You should now see that reads coming from the same pair are linked. You can also notice that some pairs overlap because the insert was smaller than the total size of the reads.

Let's have a look at some of the other mappings. You can delete the tracks by right-clicking and selecting "delete track". 
Then load the two long read mappings (DNA_Nano & DNA_HiFi).
Select the chloroplast again. You should be able to immediately see the mappings.

<details>
<summary>What obvious difference can you immediately see between both read sets?</summary>

_There are a lot more differences between the Nanopore reads and the reference than between the HiFi reads and the reference._
</details>

<details>
<summary>What kind of errors are the most common in the Nanopore reads?</summary>

_Indels (insertions and deletions), visualized by the black stripes (deletions) and the purple boxes (insertions)._
</details>

<details>
<summary>Zoom in all the way and scroll through the chloroplast genome. Look specifically for regions where many of the Nanopore reads have a deletion. Do you see a pattern in where these deletion hotspots occur?</summary>

_They  seem to mainly occur in so-called "homopolymer" stretches: regions where the same base is repeated multiple times (e.g._ `TTTTTTTTTT`_)_
</details>

Remove the tracks again, and now load the short-read RNA files (RNA_PE_hisat, RNA_PE_hisat_splice, and RNA_PE_STAR). 
Once loaded, use the search bar to go to gene "AT4G30600".

<details>
<summary>What are the blue and red arches that we can see on the "junctions" tracks?</summary>

_They are the splice junctions. They indicate where a read is split: one part of the read is mapped to one part of the gene, another part of the read is mapped to another part of the gene._
_If you look closely, you should see that these arches correspond to the introns, the parts of the gene that are spliced out when converting a pre-mRNA to a mature mRNA._
</details>

<details>
<summary>Do you see differences in the hisat mapping when using the normal index compared to the index where we added splice junctions?</summary>

_No. The mappings seem very similar. This does not mean, however, that adding splice junctions to the index is useless._
_In genes with complicated gene structures, adding splice junctions can help with mapping more reads correctly._
</details>

<details>
<summary>Zoom out a couple of times. Do you see a pattern in where in the genome the reads map?</summary>

_Yes, the reads map almost exclusively to genes (blue boxes at the bottom of the screen)._
_This is logical, since the reads are derived from RNA-transcripts, which can only be generated from genes._
</details>

<details>
<summary>Some genes do not have any reads mapping to them. What could this mean?</summary>

_This likely means that the gene is not expressed in the conditions of the experiment._
</details>

<details>
<summary>Go to gene "AT4G36850.1" and look at the STAR alignments. What difference can you see from the HISAT2 alignments?</summary>

_You should see some long lines on the junction track and on the first lines of the reads track._
_These indicate very long splice junctions, where parts of the reads are mapped to different genes._
_This problem could be resolved by specifying a maximal intron length when running STAR._
</details>

Remove the STAR tracks, and one of the two HISAT mappings. Then load the RNA_NanoPore mapping, and go back to gene "AT4G09150.1".

<details>
<summary>Look at the first read in the Nanopore mapping, and compare to the reads in the HISAT mapping. What benefit does Nanopore have here?</summary>

_Higher read length. The Nanopore read covers the whole gene, making it easier to identify all the splice junctions._
_With short reads, we have a lot more reads, but do not manage to cover the whole gene, and we do not manage to identify all the splice junctions._
_Long reads also makes it a lot easier to identify alternative splicing events._
</details>

Lastly, let us have a look at our faulty mappings (when using the wrong tool). Delete all tracks again, and load the Bnap_TAIR and DNA_PE_smalt files.
Pick a chromosome (not the chloroplast), zoom in and browse a bit. Try activating read-pair mode on both of them (right-click on a track to activate it).

<details>
<summary>What are the main differences between both read sets?</summary>

_There are way fewer mapped reads in the Bnap dataset, and most reads do not pair with another read (hence the different colours they have)._
_This is to be expected as we are mapping reads from another genome, and most of the reads will not map, or have a bad mapping._
</details>

Next, load the DNA_PE_smalt and DNA_PE_STAR mappings, and again browse the mappings.

<details>
<summary>Are there big difference between the mappings?</summary>

_Not so much. We have similar mappings in both sets._
_However, in the STAR mappings, there is a junctions track showing detected junctions. Since we are dealing with DNA reads here, of course these junctions are faulty._
</details>

Lastly, load the RNA_smalt and the RNA_PE_hisat_splice files. Go back to gene "AT4G30600" and compare the mappings.

<details>
<summary>What happens when using a DNA mapper on RNA reads?</summary>

_We still see that many of the reads map, but they are never split. From the coverage graph, we can still see where the introns are, but we have no reads covering the junctions._
</details>
 
## Cleanup

Once you have performed all the analyses, it is time to do some cleanup. We will remove some files that we don't need anymore, and will compress files to save space.

Remove the `.zip` archives that FastQC creates (once you have the multiqc report, they are not needed anymore):
```
rm *zip
```

Remove the genome indexes:
```
rm *index*
```

Compress the filtered reads and reference data so they take up less space on the disk:
```
gzip *filt TAIR*
```
> Since `bam` files are already in compressed format, there is no point in running gzip on them

Remove input data for STAR:

```
rm -r STAR_in
```

## Analysis on all reads

In this tutorial we have only mapped subsets of the reads.
This meant we had some low-coverage samples, leading to difficult interpretation from time to time.
If you want to have the mapping would look like (statistics + IGV) if we used all reads, you can have a look [here](../Other/Mapping_FullAnalysis.md).
