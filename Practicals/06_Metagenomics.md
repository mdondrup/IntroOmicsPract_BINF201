# Practical 6 – Metagenomics

In this practical we will have a look at metagenomic data: data containing reads from multiple species.
We will perform metagenome assemblies and perform binning to try to recover full genomes.
In addition we will have a look at marker-gene based analysis (e.g. 16S) and how to analyse those.

## Software installation and data retrieval

In this tutorial, we will have a look at the following metagenomics and related software:

- [MetaSPAdes](https://github.com/ablab/spades) - An adaptation of the popular SPAdes assembler for metagenomic data
- [MetaFlye](https://github.com/mikolmogorov/Flye) - An adaptation of the Flye long-read assembler for metagenomes
- [Concoct](https://github.com/BinPro/CONCOCT) - An unsupervised binner for metagenomic contigs
- [Kraken2](https://github.com/DerrickWood/kraken2) - A taxonomic _k_-mer based taxonomic classifier
- [Metaphlan](https://huttenhower.sph.harvard.edu/metaphlan/) - A metagenomic community profiling tool
- [CheckM](https://github.com/Ecogenomics/CheckM) - A quality assesment tool for mircrobial genomes
- [KrakenTools](https://github.com/jenniferlu717/KrakenTools) - Toolkit for anlysing Kraken/Bracken output
- [Krona](https://github.com/marbl/Krona/releases) - Taxonomy visualisation tool
- [Bracken](https://github.com/jenniferlu717/Bracken) - Statistical methods for species abundance calculation

### For students using NREC

The data and software has been setup on the NREC server. 
Before starting the practical, make sure to activate the correct environment before each part of the tutorial 
(`QC` for the QC part, `Metagenomics` for the metagenomics part)

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
mkdir -p Practical6
ln -s /storage/data/06_Metagenomics/* Practical6/
```
> `mkdir -p` creates a folder called Practical6. The "-p" option tells mkdir to create subdirectories if necessary, and to not give an error if the folder(s) already exist
> `ln -s` creates what we call a "symbolic link". This creates a small file that just says "Instead of this file, use the file that I'm liking to". 
> This allows you to "copy" files without actually having to make a physical copy.

Now, go to the newly created directory (by running `cd Practical6`), and you are ready to start!

### For students running on their own pc

You will first have to set up the correct environment with the necessary tools. 
See [the intro practical](00_IntroSetup.md) on how to install micromamba and how to create an environment for downloading the necessary data.

To create a new environment including the necessary tools for this practical, run the following command:

```
micromamba create -n Metagenomics spades kraken2 krona metaphlan python=3.11 checkm-genome krakentools concoct=1.1.0 bwa-mem2 quast bracken flye minimap2 flash2 cutadapt
micromamba run -n pip install "setuptools<=80.10.2" # this is required for checkm to work

```
> This will create a new environment called "Transcriptome" with the necessary tools.
> We'll use bwa-mem2 and minimap2 for mapping reads, and flash2 for merging paired reads together.

This will print some information to the screen, telling us to do some additional setup:

- Get taxonomy information for Krona:

```
ktUpdateTaxonomy.sh
```

- Set CheckM root:

```
checkm data setRoot {path to CheckM data}
```
> Check the output on your screen for the exact comment.
> You can also run `checkm -h` to check the root folder that should be set.

- Download necessary kraken2 databases:

```
kraken2-build --special silva --threads 2 --db KrakenSILVADB
bracken-build -d KrakenSILVADB -t 2 -k 35 -l 300
```
> This will install the KrakenDB in your current directory.
> Change the `--db` argument if you want to store the database elsewhere.

<details>
<summary> Fix a Concoct bug (likely no longer necessary with the explicit versions given) </summary>

A bug in Concoct can be easily fixed with a small script correction.
The problem is in a script called "validation.py" from the sklearn package.
To fix the bug, you need to know where your Mamba/Conda envs are located.
Normally, they're in your home folder, but if they're not, modify the commands below as needed.
First, we need to find the line of the script that contains the bug:

```
grep -n "feature_names = np\.asarray(X" ~/miniforge3/envs/Metagenomics/lib/python3.10/site-packages/sklearn/utils/validation.py
```

This will output the line and line number of the bug.
We can open `nano` on a specific line to fix the bug:
>Replace the XXXX with the line number you got from the `grep` command.

```
nano +XXXX ~/miniforge3/envs/metagenomics/lib/python3.10/site-packages/sklearn/utils/validation.py
```

Then, replace the line `feature_names = np.asarray(X.columns, dtype=object)`
to `feature_names = np.asarray(X.columns.astype(str), dtype=object)`.
Save the file by tapping `Ctrl+O`, and quit by tapping `Ctrl+X`.
This should fix the bug and make Concoct run without problems.

</details>

Lastly, create a new folder (e.g. `Practical6`), go into it (`cd Practical6`), and then download the necessary data.
You can either:

- Download directly from the [Zenodo repository](https://zenodo.org/uploads/13120340):

```
wget https://zenodo.org/records/13120340/files/06_Metagenomics.zip
unzip 06_Metagenomics.zip
pigz -d 06_Metagenomics/*gz
```

- Download and filter the data manually using the commands below:

> Remember to activate the download environment (see [here](00_IntroSetup.md)).
> This will download the data, and create subsamples of the read sets.
> Note: Largest downloaded file: 7G, total size after subsampling: 7.5G

<details>
<summary>Click here to expand the command necessary for setting up the data yourself</summary>

```
prefetch SRR27874410
fasterq-dump -p --outdir ./ --split-files SRR27874410/SRR27874410.sra
seqtk sample -s666 SRR27874410_1.fastq 2500000 > Plant_F.fastq
seqtk sample -s666 SRR27874410_2.fastq 2500000 > Plant_R.fastq
rm -r SRR27874410*

prefetch SRR23255609
fasterq-dump -p --outdir ./ --split-files SRR23255609/SRR23255609.sra
seqtk sample -s666 SRR23255609_1.fastq 2500000 > Stool_F.fastq
seqtk sample -s666 SRR23255609_2.fastq 2500000 > Stool_R.fastq
rm -r SRR23255609*

prefetch SRR25440994
fasterq-dump -p --outdir ./ --split-files SRR25440994/SRR25440994.sra
mv SRR25440994_1.fastq Soil_1F.fastq
mv SRR25440994_2.fastq Soil_1R.fastq
rm -r SRR25440994*

prefetch SRR25440995
fasterq-dump -p --outdir ./ --split-files SRR25440995/SRR25440995.sra
mv SRR25440995_1.fastq Soil_2F.fastq
mv SRR25440995_2.fastq Soil_2R.fastq
rm -r SRR25440995*

prefetch SRR25440996
fasterq-dump -p --outdir ./ --split-files SRR25440996/SRR25440996.sra
mv SRR25440996_1.fastq Soil_3F.fastq
mv SRR25440996_2.fastq Soil_3R.fastq
rm -r SRR25440996*

prefetch ERR12117715	
fasterq-dump -p --outdir ./ --split-files ERR12117715/ERR12117715.sra
seqtk sample -s666 ERR12117715.fastq 75000 > Sea_HiFi.fastq
rm -r ERR12117715*
```
</details>

## The Data

The data we will use in this tutorial are different kinds of metagenomes:

- A plant metagenome (short paired-end reads)
- A stool (faeces) metagenome (short paired-end reads)
- An ocean metagenome (long PacBio HiFi reads)
- Three soil metagenomes (16S amplicon sequencing, paired Illumina reads)

Most samples have been downsampled to speed up their analysis.
For an overview of the data used in this practical, please see the [information sheet on Zenodo](https://zenodo.org/uploads/13120340).

## Shotgun metagenomics

In this first part, we will have a look at the shotgun metagenomics data. 
This data is generated by sequencing all DNA from a sample. 
We have two short-read samples (Plant & Stool), and one long-read one (Sea).

### Quality control

As usual, we'll start by performing quality control of our sequences.
Since our read sets are a bit larger in size, FastQC would take a while to run on NREC.
We have thus provided the FastQC reports with the data on NREC.
Download and have a look at the QC reports.

> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>Do we need any trimming?</summary>

_Some trimming could be good. Adapter levels are generally low, but in some read sets the quality drops off near the end._
</details>

<details>
<summary>Look at the %GC content plots. Why do we observe some of these weird patterns?</summary>

_Because we are dealing with metagenome samples: the samples contain DNA from multiple organisms, which often have different % GC, leading to the erratic %GC plots._
</details>

We will do some basic trimming using cutadapt to get rid of low-quality bases and adapters in the short reads:
> We will override our read files with the trimmed versions. 
> If for some reason you need the original reads again, rerun the link command at the beginning of the tutorial.

```
cutadapt -q 30,30 -m 30 -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -o tempF.fq -p tempR.fq Plant_F.fastq Plant_R.fastq
mv tempF.fq Plant_F.fastq
mv tempR.fq Plant_R.fastq
cutadapt -q 30,30 -m 30 -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -o tempF.fq -p tempR.fq Stool_F.fastq Stool_R.fastq
mv tempF.fq Stool_F.fastq
mv tempR.fq Stool_R.fastq
```

### Taxonomic profiling of reads

Now that we have QC’d and filtered our reads, we can try a first classification on read level.
We will first use [kraken2](https://ccb.jhu.edu/software/kraken2/) for this. 
Kraken extracts _k_-mers from our reads and compares them to a database to classify them. 
We will first concatenate the paired-end reads from each sample together:
> Here we will classify the trimmed reads. 
> It is, however, debated whether it is better to classify trimmed or raw reads. 
> Raw reads are more representative, but can lead to some errors due to low-quality bases.

```
cat Plant_F.fastq Plant_F.fastq > Plant_all.fastq
cat Stool_R.fastq Stool_R.fastq > Stool_all.fastq
```

Then we will run Kraken2, using a 16S database from [SILVA](https://www.arb-silva.de/):

```
kraken2 --db /storage/data/KrakenSILVADB/ --threads 2 --output Plant.kraken.out --report Plant.kraken.report Plant_all.fastq
kraken2 --db /storage/data/KrakenSILVADB/ --threads 2 --output Stool.kraken.out --report Stool.kraken.report Stool_all.fastq
kraken2 --db /storage/data/KrakenSILVADB/ --threads 2 --output Sea.kraken.out --report Sea.kraken.report Sea_HiFi.fastq
```

Take a look at the generated reports:

```
less *report
```
> This will open all 3 files ending in "report" using the `less` command. 
To switch between files, you can press `:n` for the next file, or `:p` for the previous file.

In the first column, we can see the proportion of reads that is classified as a certain taxonomy (the last column). 
However, this overview is quite long and complex and difficult for a human to interpret.
We will use krakenTools and kronaTools to make a visual overview of the taxonomic distribution in our samples:

```
kreport2krona.py -r Plant.kraken.report -o Plant.krona
ktImportText Plant.krona -o Plant.krona.html
kreport2krona.py -r Stool.kraken.report -o Stool.krona
ktImportText Stool.krona -o Stool.krona.html
kreport2krona.py -r Sea.kraken.report -o Sea.krona
ktImportText Sea.krona -o Sea.krona.html
```
> `kreport2krona.py` will convert the kraken output to an output that Kronatools can read.
> `ktImportText` will transform the modified output to a Krona plot.

Now download and look at the generated `.html` files. 
> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

You will probably see that most of the plot is unclassified reads.
This is because we are only classifying the reads using a 16S database. 
16S is only one of the many genes in bacteria, so if we sequence all the DNA from a sample, only a few reads will be part of the 16S.

Try double-clicking on the "Bacteria" section in the bottom right corner.
This will only show the reads classified as bacterial and give the taxonomic distribution inside that part.

<details>
<summary>What is the most abundant group in each sample?</summary>

- _Plant: Actinobacteriota-Streptomyces_
- _Stool: Actinobacteriota-Bifidobacterium_
- _Sea: Proteobacteria-Klebsiella_

</details>

Look a bit at the different reports. 
You can "zoom in" on certain slices by double clicking, and return to levels above using the circles in the top right.

In a real example, you would not perform this analysis on a database of 16S sequences but on a database containing full genomes.
Kraken2 has such a database, but it takes up >100Gb of space which is too much for this tutorial.
However, in your data folder we have provided you with the results of the Kraken2 output this larger database containing full genome sequences. 
Download and take a look at the `Plant.full.krona.html`, `Stool.full.krona.html`, and `Sea.full.krona.html` files.
> If you don't like downloading the files from NREC to your personal computer, you can download the relevant files directly from [Zenodo](https://zenodo.org/uploads/13120340).

<details>
<summary>What is the most common species in each sample?</summary>

- _Plant: Streptomyces rutgersensis_
- _Stool: Bifidobacterium longum_
- _Sea: Difficult to say, but probably a Candidatus Pelagibacter species_ 
</details>

<details>
<summary>Are there a lot of eukaryotic reads in the sample? If so, from which species?</summary>

- _Plant: almost none, the few that are there are from a Poaceae species_
- _Stool: almost none, the few that are there are from different plant species_
- _Sea: a significant part: 11%. Mostly different plant species as well_
</details>

<details>
<summary>How diverse are the samples? Which sample looks to be the most diverse?</summary>

_The plant sample is not diverse at all, the stool sample has some more species, but the sea sample is very diverse._
</details>

Kraken can give a good overview of the diversity, and since it’s read based also of the abundance of each species in the sample.
> This however assumes that every copy of a genome in the metagenome sample has the same probability of being sequenced, which is not always the case!

Another neat tool for taxonomic profiling and quantification, is [Metaphlan](https://huttenhower.sph.harvard.edu/metaphlan).
Metaphlan will map the reads to a database of marker genes (genes typical for certain clades/species).
Since the Metaphlan database is quite large (>20Gb), and the software can be quite slow, we have provided the output in the data for this tutorial.

> If you want to run these yourself, you can use following commands:
>```
>metaphlan Plant_all.fastq --nproc 2 --input_type fastq --unclassified_estimation --index /storage/dbs/Metaphlan -o Plant_metaphlan
>metaphlan Stool_all.fastq --nproc 2 --input_type fastq --unclassified_estimation --index /storage/dbs/Metaphlan -o Stool_metaphlan
>metaphlan Sea_HiFi.fastq --nproc 2 --input_type fastq --unclassified_estimation --index /storage/dbs/Metaphlan -o Sea_metaphlan
>```

Take a look at the provided files: `Plant.metaphlan.txt`, `Stool.metaphlan.txt`, and `Sea.metaphlan.txt` (e.g. using `cat` or `less`).
As you can see, these files are not straightforward to interpret. 
We can get a rough estimation of the number of identified species using `grep`:

```
grep "s__" Plant.metaphlan.txt | grep -v "t__" | wc -l
grep "s__" Stool.metaphlan.txt | grep -v "t__" | wc -l
grep "s__" Sea.metaphlan.txt | grep -v "t__" | wc -l
```
> Here we select all lines wich have "s__" in them (= identified at species level), and then filter everything out that is classified at the strain level ("t__").

<details>
<summary>How many different species are identified in each sample?</summary>

_None in the sea sample, 1 in the plant sample, 36 in the stool sample._
> The 0 in the sea sample is likely because Metaphlan can't really handle long reads very well.
</details>

<details>
<summary>Are these results in line with the Kraken results?</summary>

_More or less. We found only one main species in the plant sample and a bit more specis in the stool sample._
_We can't comment on the sea sample, since the metaphlan analysis seems to have failed._
</details>

Metaphlan can also do some extra analysis on metagenomic samples.
See the [Metaphlan](https://huttenhower.sph.harvard.edu/metaphlan) page for a full overview of all functionalities.

###  Metagenome assembly

We made a taxonomic profile of the reads, and have a good idea of what is in there. 
However, for a more functional analysis it would be good to also have the genomes of some of these species. 
To get to these, we will first do a metagenome assembly using metaSPAdes for the short reads, and MetaFlye for the long reads.
In metagenome assembly we're just creating contigs from the reads. 
This works similar to a normal genome assemblies, but takes into account that multiple species are present (and thus multilpe DBGgraphs will be formed).
Metagenome assembly is a bit more resource intensive on short reads. 
Using the quick and efficient [MegaHit](https://github.com/voutcn/megahit) assembler, the assemblies would still take 15m each on 2 cores, and they would take 30m with spades (1h if error-correction is performed).
To save you some time, the metaSPAdes assemblies have been run beforehand, and the resulting assembly and assembly graph have been provided (`Plant_meta.fasta`, `Stool_meta.fasta`, `Plant_meta.gfa`, `Stool_meta.gfa`).
For your information, the commands used for running the assemblies can be found below:

>```
>metaspades.py -t 2 -1 Plant_F.fastq -2 Plant_R.fastq -o Plant_metaspades
>metaspades.py -t 2 -1 Stool_F.fastq -2 Stool_R.fastq -o Stool_metaspades
>```

The reason we provided the assembly graphs (`.gfa`) is because we want to take a look at them.
A good tool to visualise the structure of our assembly graphs is [BandageJS](https://jbrowse.org/demos/bandagejs/). 

There's other software like the original [Bandage](https://github.com/rrwick/Bandage) and [BandageNG](https://github.com/asl/BandageNG) software, but a lot of these is poorly maintained now and cannot be easily downloaded or installed due to missing certificates. If you need to inspect larger assembly graphs, you may want to build BandageNG from source.

Another recommended tool that runs as a local web application is [MetaGenome](https://github.com/marbl/MetagenomeScope). It is easy to install with pip. But it also struggles with larger graphs. Optionally, try it out with the assembly graphs. 

> If you have problems downloading the graph files, you can find the relevant files on [Zenodo](https://zenodo.org/records/17120797).

Go the [BandageJS](https://jbrowse.org/demos/bandagejs/) demo page and upload the `.gfa` files of both assemblies. The stools sample will trigger a warning because a large number of nodes. Click "render anyway". 
You can draw a frame over a collection of lines (contigs), and see their total length on the right side of the screen.

<details>
<summary>How many large clusters do you see? What do they correspond to according to you?</summary>

- _Stool sample: There is one big connected cluster (total size +- 14 Mbp). This is likely one bacterial genome (or two closely related). Otherwise, we have smaller components (100kb and lower). These are likely contigs from different species that are present in lower amounts in the sample._
- _Plant sample: There is also one big connected cluster (total size +- 9Mbp). Again, this is likely one bacterial genome. The other components are similar, but there are less short pieces in this sample._
</details>

Now let's assemble the long reads, using the metagenomic mode of Flye:

```
flye -t 2 --meta --pacbio-hifi Sea_HiFi.fastq -o Sea_metaFlye
```

Download the assembly graph (`assembly_graph.gfa` in the `Sea_metaFlye` folder) and have a look at it.
> If you have problems downloading the files, you can find the relevant files on [Zenodo](https://zenodo.org/records/17120797).

<details>
<summary>How is the assembly graph similar/different from the short-read assemblies?</summary>

_We have a lot fewer components, and all components are one contig and non-connected. There are no large clusters, and the contigs are quite small._
</details>

### Metagenome binning

We now have our metagenome: a collection of contigs. 
However, we have no idea which contigs belongs to which species/genome.
So, we need to try to separate contigs from different genomes into so-called genome bins.
Here we will use Concoct for binning, but other good binning tools are [Autometa](https://autometa.readthedocs.io/en/latest/index.html) or [MetaBat](https://bitbucket.org/berkeleylab/metabat).
We will first create a new directory for binning and put our metagenome assemblies there.

```
mkdir binning
ln -s ${PWD}/Plant_meta.fasta binning/Plant_meta.fasta
ln -s ${PWD}/Stool_meta.fasta binning/Stool_meta.fasta
ln -s ${PWD}/Sea_metaFlye/assembly.fasta binning/Sea_meta.fasta 
cd binning
```

One type of information that can help separate genomes into bins is genomic coverage coverage.
We expect different genomes to have different coverages, in other words we expected contigs coming from the same genome to have simlar coverage.
To get coverage information, we’ll have to map the reads to the metagenome. 
We’ll be using Bwa-mem2 for mapping the short reads, and Minimap2 for mapping the long reads:

```
bwa-mem2 index -p PMindex Plant_meta.fasta
bwa-mem2 index -p SMindex Stool_meta.fasta
mkdir mapping
bwa-mem2 mem -t 2 PMindex ../Plant_F.fastq ../Plant_R.fastq | samtools view -bS | samtools sort > mapping/Plant.bam
bwa-mem2 mem -t 2 SMindex ../Stool_F.fastq ../Stool_R.fastq | samtools view -bS | samtools sort > mapping/Stool.bam
minimap2 -t 2 -ax map-hifi Sea_meta.fasta ../Sea_HiFi.fastq | samtools view -bS | samtools sort > mapping/Sea.bam
samtools index mapping/Plant.bam
samtools index mapping/Stool.bam
samtools index mapping/Sea.bam
```

To increase binning performance, Concoct wants us to cut up our contigs in equally-sized pieces. We’ll do this by running:

```
cut_up_fasta.py Plant_meta.fasta -c 10000 -o 0 --merge_last -b Plant_10k.bed > Plant_10k.fa
cut_up_fasta.py Stool_meta.fasta -c 10000 -o 0 --merge_last -b Stool_10k.bed > Stool_10k.fa
cut_up_fasta.py Sea_meta.fasta -c 10000 -o 0 --merge_last -b Sea_10k.bed > Sea_10k.fa
```
> We will tell Concoct to cut up the contigs in pieces of 10000 bp (`-c 10000`), without allowing overlap between pieces (`-o 0`), and we will concatenate the remaining part (< 10000bp) to the final contig (`--merge_last`)

Then Concoct can create a coverage table from the mapping and the created `.bed` files from the cutting process:

```
concoct_coverage_table.py Plant_10k.bed mapping/Plant.bam > Plant_cov.tsv
concoct_coverage_table.py Stool_10k.bed mapping/Stool.bam > Stool_cov.tsv
concoct_coverage_table.py Sea_10k.bed mapping/Sea.bam > Sea_cov.tsv
```

Then we can run concoct itself to identify the different bins:

```
concoct --threads 2 --composition_file Plant_10k.fa --coverage_file Plant_cov.tsv -b Plant_binning
concoct --threads 2 --composition_file Stool_10k.fa --coverage_file Stool_cov.tsv -b Stool_binning
concoct --threads 2 --composition_file Sea_10k.fa --coverage_file Sea_cov.tsv -b Sea_binning
```
>This might print a lot of warnings for the first command, but you can ignore them.

However, we only clustered the cut-up pieces of our assembly. We need to get back to the original contigs:

```
merge_cutup_clustering.py Plant_binning_clustering_gt1000.csv > Plant_binning_clustering_merged.csv
merge_cutup_clustering.py Stool_binning_clustering_gt1000.csv > Stool_binning_clustering_merged.csv
merge_cutup_clustering.py Sea_binning_clustering_gt1000.csv > Sea_binning_clustering_merged.csv
```

Then finally, we can extract the clustered bins as fasta files:

```
mkdir Plant_bins Stool_bins Sea_bins
extract_fasta_bins.py Plant_meta.fasta Plant_binning_clustering_merged.csv --output_path Plant_bins
extract_fasta_bins.py Stool_meta.fasta Stool_binning_clustering_merged.csv --output_path Stool_bins
extract_fasta_bins.py Sea_meta.fasta Sea_binning_clustering_merged.csv --output_path Sea_bins
```

### Binning QC

We now have genomic bins for each sample. 
A genomic bin is a set of contigs that is assumed to come from the same genome/species. 
We can thus treat it as a genome and use genome QC tools to assess their quality.
We can quickly have an overview of the genome statistics using Quast:

```
cd Plant_bins && quast.py *fa
cd ../Stool_bins && quast.py *fa
cd ../Sea_bins && quast.py *fa
```

Find and download the quast reports, and have a look at them.
> You can use the `pwd` command to check in what directory you currently are
> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>Given that most bacterial genomes are between 2 and 10 Mbp, how many full genomes do you think we recovered from every sample?</summary>

- _Plant: 1 large genome (+- 7Mbp)_
- _Stool: 1 large genome (+- 12.2 Mbp), possible some smaller or uncomplete genomes (3-5 genomes between 2Mbp and 4Mbp)
- _Sea: Only 1 genome >2Mbp, so possibly no complete genomes_
</details>

<details>
<summary>Is this more or less then expected based on the taxonomic analysis (kraken/metaphlan)?</summary>

_For plant and stool yes: Plant had 1 dominant species, Stool a couple._
_There was a lot of diversity in the sea sample, but it looks like we did not get any good genomes (based on just their size)._
</details>

<details>
<summary>Why do you think we could not recover the genomes of all?</summary>

_Mainly because we are lacking coverage (especially since we only worked on a subset of reads), so we have not enough information to recover all genomes._
_Species which are present in low abundance are difficult to recover from metagenomic samples._
</details>

Lastly, we will be using [CheckM](https://github.com/Ecogenomics/CheckM/wiki) to check how good the bins are that we reconstructed. 
This software will try to identify marker genes in our bins, and use that to estimate the completeness, purity (~ 1-contamination), and tries to classify our bin.
Running CheckM takes a wile, and requires some memory (40G, or 14G when running in a less accurate low-memory mode (`--reduced_tree`)).
We have provided the relevant output below, so you don't have to run the relevant commands.

```
checkm lineage_wf Plant_bins/ Plant_checkm -x .fa -t 2
---------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Bin Id           Marker lineage           # genomes   # markers   # marker sets   0     1    2   3   4   5+   Completeness   Contamination   Strain heterogeneity
---------------------------------------------------------------------------------------------------------------------------------------------------------------------
  1        f__Streptomycetaceae (UID2048)       60         460           233        2    452   6   0   0   0       99.46            0.93               0.00
  9                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  8                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  7                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  6                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  5                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  4                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  3                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  2                 root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  12                root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  11                root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
  10                root (UID1)                5656         56            24        56    0    0   0   0   0        0.00            0.00               0.00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------

checkm lineage_wf Stool_bins/ Stool_checkm -x .fa -t 2
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Bin Id            Marker lineage           # genomes   # markers   # marker sets    0     1     2     3    4   5+   Completeness   Contamination   Strain heterogeneity
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  1        f__Bifidobacteriaceae (UID1462)       65         476           217         1    474    1     0    0   0       99.54            0.12               0.00
  11          o__Bacteroidales (UID2657)        160         492           269         32    34   309   109   8   0       98.83           122.37             38.30
  3              k__Bacteria (UID203)           5449        104            58         19    46    28    11   0   0       79.69           24.87              67.21
  10          o__Clostridiales (UID1226)        155         278           158        106   147    23    2    0   0       61.89            7.14              20.69
  27         o__Selenomonadales (UID1024)        64         334           167        146   183    5     0    0   0       55.96            1.80              20.00
  22             k__Bacteria (UID203)           5449        104            58         67    37    0     0    0   0       55.52            0.00               0.00
  2           o__Clostridiales (UID1212)        172         261           147        103   135    23    0    0   0       53.36            5.55               4.35
  23          o__Clostridiales (UID1226)        155         278           158        169    83    24    2    0   0       37.13            9.24              10.00
  6        f__Enterobacteriaceae (UID5124)      134         1173          336        732   429    12    0    0   0       35.15            0.28              16.67
  4              k__Bacteria (UID203)           5449        103            57         90    13    0     0    0   0        5.58            0.00               0.00
  0              k__Bacteria (UID203)           5449        103            58         99    4     0     0    0   0        2.04            0.00               0.00
  9                  root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  8                  root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  7                  root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  5                  root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  28                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  26                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  25                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  24                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  21                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  20                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  19                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  18                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  17                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  16                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  15                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  14                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  13                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
  12                 root (UID1)                5656         56            24         56    0     0     0    0   0        0.00            0.00               0.00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------

checkm lineage_wf Sea_bins/ Sea_checkm -x .fa -t 2
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Bin Id            Marker lineage            # genomes   # markers   # marker sets    0     1    2    3   4    5+   Completeness   Contamination   Strain heterogeneity
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  7              k__Bacteria (UID203)            5449        104            58         23    53   13   5   10   0       63.79           18.65              22.73
  5              k__Bacteria (UID203)            5449        104            58         32    55   16   0   1    0       57.29           15.37              63.64
  15        o__Rhodospirillales (UID3754)         63         336           201        166   143   25   2   0    0       50.60            7.91               3.77
  11             k__Bacteria (UID203)            5449        104            58         67    28   9    0   0    0       34.48            2.82               0.00
  13             k__Bacteria (UID203)            5449        104            58         85    19   0    0   0    0       27.59            0.00               0.00
  6              k__Bacteria (UID203)            5449        104            58         87    17   0    0   0    0       22.02            0.00               0.00
  1        c__Gammaproteobacteria (UID4443)      356         451           270        364    85   2    0   0    0       10.15            0.08               0.00
  8                  root (UID1)                 5656         56            24         55    1    0    0   0    0        4.17            0.00               0.00
  2                  root (UID1)                 5656         56            24         55    1    0    0   0    0        4.17            0.00               0.00
  14             k__Bacteria (UID203)            5449        102            56         97    5    0    0   0    0        3.90            0.00               0.00
  4              k__Bacteria (UID203)            5449        102            56        100    2    0    0   0    0        1.79            0.00               0.00
  9                  root (UID1)                 5656         56            24         56    0    0    0   0    0        0.00            0.00               0.00
  3                  root (UID1)                 5656         56            24         56    0    0    0   0    0        0.00            0.00               0.00
  12                 root (UID1)                 5656         56            24         56    0    0    0   0    0        0.00            0.00               0.00
  10                 root (UID1)                 5656         56            24         56    0    0    0   0    0        0.00            0.00               0.00
  0                  root (UID1)                 5656         56            24         56    0    0    0   0    0        0.00            0.00               0.00
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```
> We tell CheckM to analyze files ending in `.fa` (`-x .fa`), using two threads (`-t 2`). 

<details>
<summary>How many good quality genomes are recovered from the data based on the CheckM output?</summary>

- _Plant: 1 high-quality genome part of the Streptomycetaceae family. The other bins show no markers._
- _Stool: Many genomes are discovered. 1 high-quality (Bifidobacteriaceae), 1 near-complete but possibly contaminated (or 2 close strains; Bacteroidales), some half-complete genomes (50-80% completeness), and the rest is likely nothing._
- _Sea: There are some genomes detected, but none are fully complete. Most of them are also difficult to classify (classified only as Bacteria)._
</details>

<details>
<summary>Do you think we could have recovered more genomes if we would have used the full dataset instead ofa subsample of reads?</summary>

_Using the full data, we might be able to discover more high-quality bins in some samples:_

- _In the plant sample, it looks like there is only one species really present, and we managed to recover its genome._
- _In the stool sample, there are more genomes present, and having the full dataset could have helped us recover more complete genomes._
- _In the sea sample, we didn't manage to recover a high-quality genome, but could detect some uncomplete genomes. Using all data could help complete some of them._
</details>

As a last note: binning analysis on long reads is not always recommended. 
Most binning tools use coverage as an important factor to cluster contigs together.
Since long reads tend to have fewer reads, and thus lower coverages, it is not always easy/possible to cluster contigs together using coverage.
If enough coverage is available, a meteagenome assembly alone could also be sufficient to recover the full genomes of separate species instead of going through a binning step.
> If you are interested in long-read metagenome assembly, you can find a nice review [here](https://translational-medicine.biomedcentral.com/articles/10.1186/s12967-024-04917-1).

##  Amplicon sequencing data

In the previous part we have used whole metagenome sequencing data to recover genomes of some of the members of the bacterial communities.
We have not looked at the exact community structure and diversity. 
In this part we will look at amplicion sequencing data (only the 16S gene is sequenced, instead of all the DNA in the sample), and how we can use it to compare microbial communities.
The analysis presented here for comparing the communities can also be used on shotgun sequencing data. 
However, this would require using the larger Kraken2 database to be able to classify most of the reads.

We will analyze three sets of 16S amplicon sequencing data from soil metagenomes.
You can find them in the same data folder as the other data we used, and are named Soil_1, Soil_2, and Soil_3.

First, run have a look at the FastQC reports that we provided for the read sets.
> If you are not on NREC you have to run the fastqc commands yourself.
> Remember to go back to the main Pract6 folder.

<details>
<summary>Why do we observe such a high level of duplication in the data?</summary>

_Because we are dealing with amplicon data: we sequenced an amplification of markers genes. We thus only have copies of the same gene in different bacteria._
_Since we amplified (= copied) these genes multiple times, they will have a lot of reads with exactly the same sequence._
</details>

There are a tiny bit of adapters present, and the quality drops quite low near the end of the reads.
We'll do a quick cutadapt filtering to remove most of these problems:

```
cutadapt -q 30,30 -m 30 -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -o tempF.fq -p tempR.fq Soil_1F.fastq Soil_1R.fastq
mv tempF.fq Soil_1F.fastq
mv tempR.fq Soil_1R.fastq

cutadapt -q 30,30 -m 30 -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -o tempF.fq -p tempR.fq Soil_2F.fastq Soil_2R.fastq
mv tempF.fq Soil_2F.fastq
mv tempR.fq Soil_2R.fastq

cutadapt -q 30,30 -m 30 -a AGATCGGAAGAGCACACGTCTGAACTCCAGTCA -A AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -o tempF.fq -p tempR.fq Soil_3F.fastq Soil_3R.fastq
mv tempF.fq Soil_3F.fastq
mv tempR.fq Soil_3R.fastq
```

Since we sequenced PCR-fragments (amplicons), in many cases the insert size (size of the amplicon) will be shorter than 2x read length.
This means there will often be overlap between the read pairs. We will use [FLASH](https://github.com/dstreett/FLASH2) to merge the reads of the same pair together.
Remember to reactivate the Metagenmics environment first!
> It is debated wether it is best to merge reads first or do read trimming first. 
> Here we pick trimming first since the quality dropped quite low near the end of the reads.
> Doing merging first might be better if you have low amount of reads and want to keep as many as possible.

```
flash2 Soil_1F.fastq Soil_1R.fastq -M 250
mv out.extendedFrags.fastq Soil_1M.fastq
mv out.notCombined_1.fastq Soil_1F.fastq
mv out.notCombined_2.fastq Soil_1R.fastq

flash2 Soil_2F.fastq Soil_2R.fastq -M 250
mv out.extendedFrags.fastq Soil_2M.fastq
mv out.notCombined_1.fastq Soil_2F.fastq
mv out.notCombined_2.fastq Soil_2R.fastq

flash2 Soil_3F.fastq Soil_3R.fastq -M 250
mv out.extendedFrags.fastq Soil_3M.fastq
mv out.notCombined_1.fastq Soil_3F.fastq
mv out.notCombined_2.fastq Soil_3R.fastq
```
> The `-M` parameter increases the maximal allowed overlap, since we're dealing with relatively long reads (250bp)

Merging reads can also be used to improve genome assembly, as longer reads will make assembly easier.
As an additional bonus, it will lower the number of total reads, leading to shorter running times.

Let's analyse our soil samples with Kraken2 first.
We have to again put all reads in one file:

```
cat Soil_1*.fastq > Soil_1A.fastq
cat Soil_2*.fastq > Soil_2A.fastq
cat Soil_3*.fastq > Soil_3A.fastq
```

Since we are dealing with 16S data, using the SILVA database should give better results then when we were working with shotgun data.

```
kraken2 --db /storage/data/KrakenSILVADB/ --threads 2 --output Soil_1.kraken.out --report Soil_1.kraken.report Soil_1A.fastq
kraken2 --db /storage/data/KrakenSILVADB/ --threads 2 --output Soil_2.kraken.out --report Soil_2.kraken.report Soil_2A.fastq
kraken2 --db /storage/data/KrakenSILVADB/ --threads 2 --output Soil_3.kraken.out --report Soil_3.kraken.report Soil_3A.fastq
```

We will again visualise the results in a krona plot:
> An alternative way to visualise the kraken reports, is on the webserver [Pavian](https://fbreitwieser.shinyapps.io/pavian/).

```
kreport2krona.py -r Soil_1.kraken.report -o Soil_1.krona && ktImportText Soil_1.krona -o Soil_1.krona.html
kreport2krona.py -r Soil_2.kraken.report -o Soil_2.krona && ktImportText Soil_2.krona -o Soil_2.krona.html
kreport2krona.py -r Soil_3.kraken.report -o Soil_3.krona && ktImportText Soil_3.krona -o Soil_3.krona.html
```

Now download and open the krona plots and have a look at the results.
> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>Are the samples diverse, or are there only few dominating species?</summary>

_All 3 samples look very divers, with not really a single species dominating the community._
</details>

<details>
<summary>Do the 3 samples have similar species distributions on first glance?</summary>

_Soil1 and Soil2 have very similar distributions, Soil3 has the same large groups, but the proportions are a bit different._
</details>

Now we’ll use a tool called [Bracken](https://ccb.jhu.edu/software/bracken/) to calculate species abundance for all three samples based on the Kraken results.
Bracken re-estimates the read counts from kraken, to make the counts more robust and comparable between samples. 
Since the SILVA database only classifies on genus level, we will do the analysis on genes level. 
Furthermore, we will limit ourselves to species with at least 10 reads assigned to them:

```
bracken -d /storage/data/KrakenSILVADB/ -i Soil_1.kraken.report -o Soil_1.bracken -w Soil_1.bracken.kreport -r 300 -l "G" -t 10
bracken -d /storage/data/KrakenSILVADB/ -i Soil_2.kraken.report -o Soil_2.bracken -w Soil_2.bracken.kreport -r 300 -l "G" -t 10
bracken -d /storage/data/KrakenSILVADB/ -i Soil_3.kraken.report -o Soil_3.bracken -w Soil_3.bracken.kreport -r 300 -l "G" -t 10
```
> We specify the average read length (`-l 300`), and specify we want to classify on Genus level (`-l "G"`). 
> We use a read length of 300 since most of our merged reads are that length (= size of the amplicon).
> `-t 10` filters out results with fewer than 10 mapped reads.
> The `-o` and `-w` options determine the output files (bracken, and kraken-like output).

Have a look at the Bracken files (using `less`, `head`, or `cat`).
You will see for every discovered genus the assigned reads, added reads (from higher/lower-level taxonomies), and the new amounts of reads.

Let's try finding the most abundant genera:

```
sort -nrk6 Soil_1.bracken | head -n10
sort -nrk6 Soil_2.bracken | head -n10
sort -nrk6 Soil_3.bracken | head -n10
```
> We sort numerically (`-n`), in reverse order (`-r`), and use field 6 (corrected reads) (`k6`) to sort.
> Then, we take the first 10 lines using the `head`.

<details>
<summary>What is the 3 most abundant genera in each sample (excluding uncultured species)?</summary>

- _Soil1: Acidothermus, Conexibacter, Burkholderia-Caballeronia-Paraburkholderia_
- _Soil2: Acidothermus, Candidatus Solibacter, Conexibacter_
- _Soil3: Nocardioides, Gaiella, Streptomyces_
</details>

Using the Bracken estimations, we can try to estimate the alpha diversity of our samples using the [KrakenTools](https://github.com/jenniferlu717/KrakenTools) package. 
Many diversity measurements are implemented in the software, but here we will only be using the Shannon alpha diversity:

```
alpha_diversity.py -f Soil_1.bracken -a Sh
alpha_diversity.py -f Soil_2.bracken -a Sh
alpha_diversity.py -f Soil_3.bracken -a Sh
```
> `-a Sh` specifies that we want to use the Shanon Index

<details>
<summary>Which of the samples has the highest alpha diversity?</summary>

_Soil3 (4.769)_
</details>

We can also calculate the beta-diversity between the three samples. 
The `beta_diversity.py` script from KronaTools calculates the Bray-Curtis dissimilarity matrix, 
where a score of 0 means samples have exactly the same species distribution, and a score of 1 means we have totally different samples.
This means that the score is more of "dissimilarity" measure, as a higher score means less similar samples.

```
beta_diversity.py -i *bracken --type bracken
```

<details>
<summary>Do the observed differences in diversity between samples are what you expected based on the Krona plots?</summary>

_Yes, Soil1 and Soil2 have a low dissimilarity (0.382), while Soil3 is more dissimilar to the other 2 (0.426-0.562)._
</details>

<details>
<summary>Why do we have X's in some places of the table?</summary>

_Because the table is symmetrical: the dissimilarity between 1 & 3 is the same as the dissimilarity between 3 & 1._
</details>

We can also use Metaphlan to analyse the samples.
Sadly, 16S sequences are not included in the Metaphlan marker database, so running metaphlan itself is of no use here. 
However, using KronaTools, we can convert our Kraken2 output to a metaphlan output. 
As not all libraries contain the same number of reads, we will use percentages instead of read counts (which is also used by metaphlan itself):

```
kreport2mpa.py -r Soil_1.bracken.kreport -o Soil_1.metaphlan.txt --percentages
kreport2mpa.py -r Soil_2.bracken.kreport -o Soil_2.metaphlan.txt --percentages
kreport2mpa.py -r Soil_3.bracken.kreport -o Soil_3.metaphlan.txt --percentages
```

Our kraken analysis only goes to genus level, so we’ll try counting the number of genera in the samples:

```
grep "g__" Soil_1.metaphlan.txt | wc -l
grep "g__" Soil_2.metaphlan.txt | wc -l
grep "g__" Soil_3.metaphlan.txt | wc -l
```

It looks like there are many genera identified, but many of them have only few reads assigned to them.
Let’s do some filtering. We will remove “uncultured” genera, and only keep genera with a proportion above 0.5%.
We'll do this using some build-in Linux functions:

```
grep "g__" Soil_1.metaphlan.txt | awk '{ if ($NF > 0.5) print}' | grep -v "uncultured" > Soil_1.metaphlan.reduced.txt
grep "g__" Soil_2.metaphlan.txt | awk '{ if ($NF > 0.5) print}' | grep -v "uncultured" > Soil_2.metaphlan.reduced.txt
grep "g__" Soil_3.metaphlan.txt | awk '{ if ($NF > 0.5) print}' | grep -v "uncultured" > Soil_3.metaphlan.reduced.txt
```
> In this command, we filter the lines that have identifications on genus level `grep "g__"`, 
> and then only retain lines where the last column has a value strictly higher than 0.5 (`awk '{ if ($NF > 0.5) print}'`. 
> Lastly, we use grep again to filter out lines which contain "uncultured" somewhere (`grep -v "uncultured"`)

Since we filtered out the "genus" lines only, we can simply count the number of discovered genera by counting the lines in the files:

```
wc -l Soil*reduced.txt
```

This reduced the number of genera significantly. 
We can merge the Metaphlan profiles, to create one big table:

```
combine_mpa.py -i *reduced.txt -o Soil_merged.tbl
```

<details>
<summary>The combine_mpa script tells us that it printed 56 classifications, but if we count the genera in all 3 samples, we have 94. Why is there a difference?</summary>

_Because there is overlap in genera between samples. There are thus 56 unique genera found, but some of them are present in multiple samples._
</details>

If you look at the merged table, the names of the genera are quite long, containing the whole taxonomy.
We’ll clean up the table a bit using `sed`, by removing everything before the last `|` character:

```
sed -i "s/^.*|//g" Soil_merged.tbl
```
> Here, we are using the substitute command of  `sed`: `sed s/pattern/replacement/g`
> The leading `s` tells `sed` to do a substitution, and the trailing `g` tells `sed` to do it "globally" (so don't stop after the first replacement).
> We tell `sed` to substitute the pattern `^.*|` and replace it with nothing (there is nothing between the last forward slashes: `//`) 
> The `^.*|` pattern roughly translates to "any number or any character (`.*`) after the start of the string (`^`) until you meet a `|`.  
> This will effectively remove everything before (and including) the last `|` character.

Lastly, we need to slightly change the header before we can visualize the result.
Open the merged table using nano (`nano Soil_merged.tbl`), and and change the header by removing the leading `#` in the first line, 
and change the sample headers to not contain whitespace (e.g. Sample #1 in Soil1).
Press `Ctrl-O` to save the file, and `Ctrl-X` to quit.
Now let's have a look at the table:

```
column -t Soil_merged.tbl
```
> The `column` command will nicely align your columns

Since the table is quite large it is not so straightforward to look for the most abundant species in each sample.
We can use the sort and head commands again to have the most abundant species per sample:

```
sort -nrk2 Soil_merged.tbl | column -t | head -n10
sort -nrk3 Soil_merged.tbl | column -t | head -n10
sort -nrk4 Soil_merged.tbl | column -t | head -n10
```

<details>
<summary>Compare the 3 most abundant genera from the metaphlan analysis here to the those of the Bracken output. Do they match?</summary>

- _Soil1: Acidothermus, Conexibacter, Candidatus solibacter_
- _Soil2: Candidatus solibacter, Acidothermus, Conexibacter_
- _Soil3: Nocardioides, Gaiella, Streptomyces_

_This does not match with the Bracken output._ 
_If we have a closer look at the Bracken output, we can see that there is a space in some species names (e.g. Candidatus Solibacter)._
_Since sort treats any whitespace (spaces and tabs) as column delimiters, it will wrongly count the columns when there is a space in the any column value._
_We can tell sort to only count tabs as column delimiters: `sort -nrk6 -t$'\t' Soil_1.bracken | head -n 10` should give the correct sorting.
</details>

Here we only created a metaphlan merged table. There is much more you can do with this table, but we won't go into it here.
See the [Metaphlan](https://huttenhower.sph.harvard.edu/metaphlan) page for a full overview of all functionalities.
Additionally, if you want to have an overview of metagenomic tools in general, you can find a good review [here](https://link.springer.com/article/10.1007/s13238-020-00724-8).

## Cleanup

Once you have performed all the analyses, it is time to do some cleanup. We will remove some files that we don't need anmyore, and will compress files to save space.

Compress the filtered reads:
```
gzip *fastq
```
> Compressing the `fastq` files can take some time. You can launch it in the background by running 'nohup gzip *fastq &' instead.
> This will make it so that the command will keep running untill it is done, even when you log out of the server.

## Analysis using all reads

In this tutorial we only used a subset of reads for the metagenome assembly.
If you're interested in seeing how the results would change if we used the full datasets, 
you can have a look [here](../Other/Metagenomics_FullAnalysis.md).
