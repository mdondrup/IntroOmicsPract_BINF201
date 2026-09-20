# Practical 3 – Genome Assembly

In this practical, we will assemble bacterial genomes using different assembly software and methods (both for short and long reads), and compare the results.

## Software installation and data retrieval

In this tutorial, we will have a look at the following assembly (or related) software:

- [SPAdes](https://github.com/ablab/spades) - Likely the most popular De Bruijn Graph assembler, used very commonly for bacterial or small eukaryotic genome assembly with short reads
- [ABySS](https://github.com/bcgsc/abyss) - Another short read assembler using De Bruijn Graphs
- [Megahit](https://github.com/voutcn/megahit) - An ultrafast, memory-efficient De Bruijn Graph assembler for short reads
- [Canu](https://github.com/marbl/canu): A long read assembler for both Nanopore and early (noisy) PacBio reads
- [Flye](https://github.com/mikolmogorov/Flye): Another long-read assembler for all kinds of long reads (high or low error, Nanopore or PacBio)
- [HifiAsm](https://github.com/chhylp123/hifiasm): An assembler specific for PacBio HiFi reads
- [Quast](https://github.com/ablab/quast) - An assembly QC tool that generates assembly statistics
- [BUSCO](https://busco.ezlab.org/) - A tool to assess assembly completeness
- [FastK](https://github.com/thegenemyers/FASTK) - A fast k-mer counter used to build k-mer databases from sequencing reads
- [kmc](https://github.com/refresh-bio/KMC) – Another fast k-mer counter
- [Smudgeplot](https://github.com/KamilSJaron/smudgeplot) - A tool that uses heterozygous k-mer pairs to infer genome ploidy and identify signatures of genome structure (e.g., duplications, heterozygosity) directly from raw reads
- [Seqkit](https://bioinf.shenwei.me/seqkit/) – Versatile and ultrafast toolkit for FASTA/Q file manipulation


### For students using NREC

The data and software have been set up on the NREC server. 
Before starting the practical, make sure to activate the correct environment before each part of the tutorial!
(e.g., `QC` for the QC part, `Assembly` for the assembly part)

Then navigate to your work folder.
We will not work on the home folder (`~` or `/home/{your_username}`) because there is only limited storage space (20Gb).
You will be working on a mounted drive (200Gb) which is located in `/storage`.
All students on NREC will have their own folder `/storage/{your_username}` (e.g., `/storage/brdan` if your username is "brdan").

First of all, go to your work folder:

```
cd /storage/{your_username}
```
> Replace `{your_username}` with your username on the server.

You can make a copy of the data you will be working on by running this command from your work directory:

```
mkdir -p Practical3
ln -s /storage/data/03_Assembly/* Practical3/
```
> `mkdir -p` creates a folder called Practical3. The "-p" option tells mkdir to create subdirectories if necessary, and not to give an error if the folder(s) already exist
> `ln -s` creates what we call a "symbolic link". This creates a small file that just says "Instead of this file, use the file that I'm linking to". This allows you to "copy" files without actually having to make a physical copy.

Now, go to the newly created directory (by running `cd Practical3`), and you are ready to start!

### For students running on their own pc

You will first have to set up the correct environment with the necessary tools. 
See [the intro practical](00_IntroSetup.md) on how to install micromamba and how to create an environment for downloading the necessary data.

We need to create two environments: one for assembly, and one for BUSCO. 
BUSCO has some very specific requirements, which are difficult to combine with other tools. 
We will thus install it in its own environment. 
To create the two environments, including the necessary tools for this practical, run the following commands:

```
micromamba create -n BUSCO busco
micromamba create -n Assembly spades abyss megahit quast canu flye hifiasm fastk smudgeplot kmc seqkit
```
> This will create two new environments called "BUSCO" and "Assembly" with the necessary tools.

Create a new folder (e.g., `Practical3`), go into it (`cd Practical3`), and then download the necessary data.
You can either:

- Download directly from the [Zenodo repository](https://zenodo.org/uploads/13120340):

```
wget https://zenodo.org/records/13120340/files/03_Assembly.zip
unzip 03_Assembly.zip
```

- Download manually using the commands below:

> Remember to activate the download environment (see [here](00_IntroSetup.md))
> Note: Total download size after decompression is +- 1,25 Gb

<details>
<summary>Click here to expand the command necessary for setting up the data yourself</summary>

```
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR486/ERR486840/ERR486840_1.fastq.gz
mv ERR486840_1.fastq.gz MycGen_1.fastq.gz

wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/ERR486/ERR486840/ERR486840_2.fastq.gz
mv ERR486840_2.fastq.gz MycGen_2.fastq.gz
gunzip *gz

prefetch SRR28208385
fasterq-dump -p --outdir ./ --split-files SRR28208385/SRR28208385.sra
mv SRR28208385.fastq MycOvi_Nano.fastq
rm -r SRR28208385/

prefetch SRR24462972
fasterq-dump -p --outdir ./ --split-files SRR24462972/SRR24462972.sra
seqtk sample -s666 SRR24462972.fastq 50000 > MycOvi_HiFi.fastq
rm -r SRR24462972*
```
</details>

## The Data

The data for short-read assembly is composed of paired-end Illumina sequencing reads from _Mycoplasmoides genitalium_, 
a pathogenic bacterium that causes infection of the urinary and genital tracts in humans. 
The data for long-read assembly is composed of Nanopore and PacBio HiFi reads from _Mycoplasma ovipneumoniae_, 
a pathogenic bacterium causing pneumonia in sheep and goats.
The reads used here are only subsets of the total data, to make sure the analyses don't take too long to run.
For an overview of the data used in this practical, please see the [information sheet on Zenodo](https://zenodo.org/uploads/13120340).

## Quality Control

Before assembling a genome, it is important to take a look at the data first. 
Activate the QC environment created in practical 2: `micromamba activate QC`.
Then, run FastQC on all four files, download the files, and look at the reports.

> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>How many reads are in this dataset, and how long are the reads?</summary>

- _Illumina: There are 387 568 reads in the dataset, all of them 150bp long._
- _Nanopore: 54857 reads, ranging from 117 to 34528 bp_
- _HiFi: 50000 reads, ranging from 641 to 40383 bp_
</details>

<details>
<summary>Based on the reports, is adapter and/or quality trimming necessary?</summary>

_Not for Illumina: The quality scores are all above Q30, and there are no adapters detected in both read files._
_The long reads show some adapters, and Nanopore has low-quality reads, but we will not trim the reads for this practical._
</details>

<details>
<summary>Given that the genome size of M. genitalium is +- 580 kbp, what coverage do we expect to have with the Illumina read set?</summary>

_200x coverage_

_To calculate this, we need to first calculate the total number of bases in our read set._ 
_We have two read sets with 387568 reads of 150bp. Thus, the total number of bases is 2\*387568\*150 = 116,270,400 bp._
_Since we know the genome is 580 kbp (580 000bp) long, that means we should have an average coverage of 116270400/580000 or 200x coverage._
</details>

## Genome assembly using Spades

We will assemble the genome from the short reads using three different assemblers. 
The first one, [SPAdes](https://github.com/ablab/spades), is one of the most popular genome assemblers for small genomes (viral, bacterial, yeast). 
It is also very popular for doing metagenome assembly (see practical 6).

### Estimating genome properties with GenomeScope2

Before starting the assembly, it can be useful to inspect the Illumina reads with [GenomeScope2](http://genomescope.org/genomescope2.0/). GenomeScope2 fits a model to a k-mer histogram and can give a first estimate of genome size, repeat content, sequencing error rate, and heterozygosity directly from the raw reads. To do this, we first count k-mers from the paired-end reads with KMC, and then use the resulting histogram as input for the GenomeScope2 web interface.

If you are still in the QC environment, switch back to the assembly environment first:

```
micromamba deactivate && micromamba activate Assembly
```

Then create a 21-mer histogram with KMC:

```
echo -e "MycGen_1.fastq.gz\nMycGen_2.fastq.gz" > reads.fof
mkdir -p kmc_tmp
kmc -k21 -t2 -m8 -ci1 @reads.fof mycgen_k21 kmc_tmp
kmc_tools transform mycgen_k21 histogram mycgen_k21.histo
```
> `reads.fof` is a simple text file listing the paired-end read files that KMC should process.
> `kmc` counts all 21-mers in the reads and stores them in a database called `mycgen_k21`.
> `kmc_tools transform ... histogram` converts that database into the `mycgen_k21.histo` file that GenomeScope2 expects.

Next, open the GenomeScope2 website at `http://genomescope.org/genomescope2.0/`, upload `mycgen_k21.histo`, and make sure you enter the same k-mer length (`21`) in the web form. **Do not use `https://` for this site: the GenomeScope2 website does not support HTTPS, so you must use the HTTP address.**

<details>
<summary>What do you have to set for ploidy and why?</summary>

  - _ploidy = 1_
  - _This is a bacterial genome. Bacteria normally have only one copy of their chromosome._

</details>

We will run SPAdes in two different modes: the "isolate" and the "careful" mode. Remember to switch back to the assembly environment first (`micromamba deactivate && micromamba activate Assembly`).
> The `&&` in the above command tells the command line, "Do the first command (deactivate), and if that succeeds, run the second command (activate).

```
spades.py -t 2 -o spades_isolate -1 MycGen_1.fastq.gz -2 MycGen_2.fastq.gz --isolate
spades.py -t 2 -o spades_careful -1 MycGen_1.fastq.gz -2 MycGen_2.fastq.gz --careful
```

The assembly can take some minutes to complete. In the meantime, you can try answering the following questions regarding genome sequencing and assembly:
> You can find the necessary information in the [SPAdes manual](https://ablab.github.io/spades/) and/or by searching on the internet.

<details>
<summary>What do the "isolate" and "careful" options do?</summary>

_According to the SPAdes manual:_

- _The `--isolate` option: This flag is highly recommended for high-coverage isolate and multi-cell Illumina data; it improves the assembly quality and running time. We also recommend trimming your reads prior to the assembly._
- _The `--careful` option: Tries to reduce the number of mismatches and short indels. Also runs MismatchCorrector - a post processing tool, which uses the BWA tool (comes with SPAdes). This option is recommended only for assembly of small genomes. We strongly recommend not using it for large and medium-sized eukaryotic genomes._
</details>

<details>
<summary>SPAdes will create both contigs and scaffolds. What is the difference between both?</summary>

- _Contigs are contiguous sequences that are built from overlapping reads_
- _Scaffolds are the combination of contigs in a certain order and orientation_
</details>

<details>
<summary>What is mate pair sequencing, and how is it similar and/or different from classic paired-end sequencing?</summary>

_Mate Pair sequencing is a type of paired-end sequencing where the size of the DNA fragments is significantly larger._ 
_Mate pair sequencing is often used in combination with normal paired-end sequencing to improve difficult genome assemblies and to generate longer scaffolds from the contigs._
</details>

Once SPAdes is done running, it will have created two new directories: `spades_isolate` and `spades_careful`.
These contain the intermediate and final output files. The files we are interested in are the `contigs.fasta` and `scaffolds.fasta` files. 
We will count the total number of sequences in each file using the following commands:

```
seqkit stats spades_*/contigs.fasta spades_*/scaffolds.fasta
```
Seqkit is a fast and versatile toolkit for manipulation of FastA/Q type files.
`seqkit stats` computes some basic statistics for each of the files, including the total number of sequences. 
If you want to learn more about useful Seqkit commands, there's a [Sandbox tutorial](https://sandbox.bio/tutorials/seqkit-intro).


<details>
<summary>Are there any differences in the number of contigs or scaffolds between both runs?</summary>

_Yes: the careful run has slightly more contigs and scaffolds compared to the isolate run (57 vs 54/53)._ 
_There is no difference between contigs and scaffolds in the `careful` run, but there is one fewer scaffold than contigs in the `isolate` run._
> _Note: the exact numbers you have may vary from the results given here._
</details>

<details>
<summary>How does the assembly size turn out compared to the GenomeScope prediction and the known genome size</summary>
  
- _The size predicted by GenomeScope is ~564 kbp, and therefore underestimates the true genome size of ~580 kbp._
- _The assembly sizes are slightly larger at ~586 kbp, which is pretty good_

</details>

We will have a more detailed look at these assemblies later in the practical.

## Genome assembly using Abyss

[Abyss](https://github.com/bcgsc/abyss) is a genome assembler that can be used for assemblies of all sizes using short reads. 
We will run it here using the default settings.
As you might have noticed, SPAdes tried assembling the reads using different values for _k_, and then picks the best one. 
ABySS does not have this capacity, so we will have to run ABySS by specifying our own _k_-values. 
We will run ABySS here using _k_-values of 31 and 75:

```
mkdir -p Abyss_k31 Abyss_k75
abyss-pe k=31 name=abyss_k31  B=1G in="MycGen_1.fastq.gz MycGen_2.fastq.gz"
abyss-pe k=75 name=abyss_k75  B=1G in="MycGen_1.fastq.gz MycGen_2.fastq.gz"
mv abyss_k31* Abyss_k31
mv abyss_k75* Abyss_k75
```
> The `k` option sets the _k_-mer length, and the `B` option sets the size of the Bloom filter (a specific data structure ABySS uses to store the De Bruijn Graph).

Once both assemblies are finished, the final contig files will be stored in the output directory as `abyss_kXX-contigs.fa`. 
Use `grep` again to find the number of contigs and scaffolds in the two ABySS assemblies.

<details>
<summary>Which of the two ABySS assemblies gives the most contigs? Do you think this is better or worse?</summary>

_The assembly using a k-value of 31 gave a lot more contigs: 570 vs 49._
_Assuming the total assembly length is the same, having more contigs is worse than having fewer contigs, as more contigs means the assembly is more fragmented (more, but smaller contigs)._
</details>

<details>
<summary>Are there differences between scaffolds and contigs in the assemblies?</summary>

_Yes: In general, there are fewer scaffolds than contigs (582 vs 570 in k31; 29 vs 49 in k75)._
</details>

<details>
<summary>Why do we observe such a large difference in the number of contigs between k31 and k75?</summary>

_Using shorter k-mers (lower k-value) means using shorter sections of our reads to find overlaps._ 
_This means it is more difficult to solve repeats using shorter k-mers, leading to more fragmentation of the genome._
</details>

## Genome assembly using Megahit

Lastly, we will use [Megahit](https://github.com/voutcn/megahit) to assemble the short reads. 
Megahit is one of the faster assemblers, and was originally designed for metagenomes. 
However, it can also be used for single genomes or single-cell assemblies. 
Similar to the other assemblers, we will run Megahit using the default settings.
Megahit will assemble using multiple _k_-values, and pick the best assembly from all of them.

```
megahit -1 MycGen_1.fastq -2 MycGen_2.fastq -o megahit_assembly
```

Megahit does not create scaffolds, and will output the contigs in a file called `final.contigs.fasta`

<details>
<summary>How many contigs are in the megahit assembly?</summary>

_22_
</details>

## Assembly QC

We have now created multiple short read assemblies using different assemblers and methods. 
Now, we will use [Quast](https://github.com/ablab/quast) to assess and compare assembly statistics between assemblies.
Since there exists a public genome sequence for _M. genitalium_, we will also compare our assemblies to that one.

First, we need to download the _M. genitalium_ reference genome:

```
wget -O ref_genome.zip "https://api.ncbi.nlm.nih.gov/datasets/v2alpha/genome/accession/GCF_000027325.1/download?include_annotation_type=GENOME_FASTA"
unzip ref_genome.zip
mv ncbi_dataset/data/GCF_000027325.1/GCF_000027325.1_ASM2732v1_genomic.fna ./reference.fasta
rm -r ncbi_dataset/ README.md ref_genome.zip
```

Then we can run QUAST on all our assemblies (using the scaffolds file if available) using the following command:

```
quast -o assembly_QC -r ./reference.fasta spades_*/scaffolds.fasta Abyss_k*/*scaffolds.fa megahit_assembly/final.contigs.fa
```

This will create a folder called "assembly_QC" containing the report on the statistics (`report.html`). 
Download and go through the report and answer the questions below:
> You can click "extend report" to get more statistics.
> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>Which of the assemblies gave the largest contig?</summary>

_The ABYSS (k=75) assembly: 521282 bp (you can find this in "Largest contig" in the "Statistics without reference" section._
</details>

<details>
<summary>Which assembler had the most contigs? Which one the fewest?</summary>

_The ABySS assembly using k31 has the most contigs (482), the megahit assembly (final.contigs) has the fewest contigs (22)._
_You can find this under "# contigs (>= 0bp)" in the "Statistics without reference" section._
</details>

<details>
<summary>Which assembly has the best N50? Is this the highest or the lowest value?</summary>

_The Abyss (k=75) assembly have the best N50: 521282._
_This is the highest value, as a higher N50 is better._ 
_This is because a higher N50 means that the largest contigs (making up at least 50% of the assembly) are larger compared to assemblies with lower N50._
_The N50 value can be found in the "Statistics without reference" section._
</details>

<details>
<summary>Compare the size of the largest contig with the N50 of the SPAdes assembly. Is there anything that stands out? How would you explain this?</summary>

_Yes: they are the same. This is because the largest contig (417kbp) takes up over half of the assembly (577kbp)._ 
_Thus, half of the assembly is contained in the largest contig, which is 417kbp._
_This also explains why the L50 is 1: you only need 1 contig to get at least 50% of the assembly._
</details>

<details>
<summary>In the "Genome statistics" section, there is a metric called NG50. How does this differ from N50? Which metric would you prefer to use to assess the quality of an assembly?</summary>

_NG50 is relative to the reference genome size, while N50 is relative to the assembly size._ 
_Using NG50 is preferred because it avoids bias due to incomplete assemblies (see example below)._

_Example: We have two assemblies of a genome:_

- _AssemblyA has 12 contigs of each 250kbp (total size 2.5 Mbp)_
- _AssemblyB has 2 contigs of 500 kbp each, and 25 contigs of 10 kbp each (total size 1.25 Mbp)_

_We know from previous studies that the genome of this species should be around 2.5 Mbp._
_Using this information, we can calculate the N50:_

_The N50 of AssemblyA is 250kbp (half of the assembly is in contigs of size 250kbp or larger), the N50 of AssemblyB is 500kbp (half of the assembly is in contigs of size 500kbp or larger)._

_Based on N50 alone, it looks like AssemblyB is better (higher value). However, let's calculate NG50 now._ 
_This means we have to check the size of the contigs that we need to have 50% of the reference genome (= 50% of 2.5Mbp = 1.25 Mbp)._

_The NG50 of AssemblyA is still 250kbp (we can build an assembly of 1.25 Mbp using contigs of 250kbp or higher)._ 
_However, the NG50 of AssemblyB is now 10 kbp: We need all contigs to get to an assembly of 1.25 Mbp; thus, the NG50 is equal to the smallest contig (10 kbp)._

_Thus, NG50 is a better metric to assess an assembly, but it requires you to have a reference genome, or to know how large the genome should be._
</details>

<details>
<summary>Which assembly covers the largest fraction of the reference genome?</summary>

_The SPAdes (isolate) assembly: (98.988%). You can find this in the "Genome fraction" row under the Genome statistics section._
</details>

<details>
<summary>Which assembly shows the most/fewest mismatches and indels compared to the reference?</summary>

_The SPAdes (isolate) assembly has the most mismatches and indels; the Abyss k31 has the fewest._
</details>

<details>
<summary>Given that the read quality was very good, what could be the reason for observing mismatches and indels in our assemblies, compared to the reference?</summary>

_Because we are sequencing a different strain than the reference genome. Bacteria, and especially pathogenic bacteria, evolve very quickly._ 
_Thus, if we sequence a bacterium of a certain species, it is very unlikely that we find the same genome as the reference._
_The observed mismatches and indels are thus likely differences that evolved between the reference strain and the strain that was sequenced in our dataset._
</details>

<details>
<summary>Which of the genome assemblies do you think is best, and why?</summary>

_Either one of the SPAdes assemblies or the ABySS k75. The ABySS k75 assembly has a higher N50, fewer total contigs, and the largest contig._ 
_However, SPAdes (isolate) captures a slightly higher fraction of the reference genome._
</details>

## BUSCO analysis

In this case, we compared our assemblies to the reference genome to assess if we had a good assembly or not. 
Of course, when sequencing a new species, there will be no reference genome available to compare with, so it gets more difficult to properly assess assembly quality. 
In that case, tools like [BUSCO](https://busco.ezlab.org/) come in handy. BUSCO is a tool that will try to detect the presence of conserved genes in your assembly.
The BUSCO tool uses a database that contains sets of conserved genes (universal single-copy orthologs) that are present in the majority of species of a certain phylogenetic lineage (e.g., plants, primates, ...).
By checking how many of the genes that should be in your assembly are actually in your assembly, you can have a rough idea of how complete your assembly is.

We will run BUSCO on the SPAdes-isolate assembly, the ABySS-k75 assembly, and the ABySS-k31 assembly. The most important step in running BUSCO is figuring out what lineage to use.
BUSCO doesn't always run nicely with other programs, so we had to install it in a separate environment (called "BUSCO"). So first, activate the BUSCO environment (`micromamba activate BUSCO`).
To get an overview of which lineages are available, you can run the following commands:

```
busco --list-datasets
```

This will give you a list of lineages that you can use. But how do we find the correct lineage? There are 3 options:

- You look up the taxonomy (e.g., on [NCBI taxonomy](https://www.ncbi.nlm.nih.gov/taxonomy), and check if any of the taxonomic levels are available in BUSCO.
- Let BUSCO figure out the best lineage (using the `--auto-lineage` option).
- Only use a general lineage (e.g., Bacteria or Eukaryota) 

While the automatic lineage selection looks tempting, it is quite computationally heavy. In addition, it doesn't always select the best lineage, especially in taxa that are underrepresented in the underlying database.
Here, we will run BUSCO both on a general and a specific dataset:

```
busco -m geno -c 2 -i spades_isolate/scaffolds.fasta --lineage bacteria -o busco_spades_bacteria
busco -m geno -c 2 -i spades_isolate/scaffolds.fasta --lineage mycoplasmatales -o busco_spades_mycoplasmatales
```
> The `-m` option selects the running mode (here `geno` for genome; other options are `trans` (transcriptome) or `prot`(protein)).

Review the results, which are printed to the screen. The summary data is also saved in a text file in the 
output directories (-o option):

```
less busco_spades_bacteria/short_summary.specific.bacteria_odb12.2.busco_spades_bacteria.txt
less busco_spades_mycoplasmatales/short_summary.specific.mycoplasmatales_odb12.2.busco_spades_mycoplasmatales.txt
```

For this tutorial, we will look only at the proportion of complete BUSCOs (C).

<details>
<summary>Compare the results for the bacteria dataset with the results for the Mycoplasmatales. Why could there be such a big difference?</summary>

_BUSCO only finds 57.8% of conserved bacterial genes, but 95.6% of Mycoplasmatales genes._ 
_This is because Mycoplasmoides species are very different from normal bacteria._
_They have small genomes (+- 500-750 kbp) compared to most bacteria (which are mostly around 3-10 Mbp) and are known pathogens._
_This means that these bacteria are very specialized and don't have many of the "general" genes that other bacteria have._ 
_However, if we only look at conserved genes within the Mycoplasmatales, it does have most genes that we expect._
_This is because most species in the Mycoplasmatales are specialized in the same way, and thus share a lot more genes._
_As a general rule, a BUSCO score >90% is considered good, and >95% is considered very good._
</details>

Now try running BUSCO yourself on the two ABySS assemblies, but only using the `mycoplasmatales` lineage. 

<details>
<summary>Are there large differences in BUSCO score between the assemblies?</summary>

_There are no large differences, but the BUSCO score is slightly lower in the ABySS-k31 assembly (96.6% vs 97.7%)._
</details>

## Smudgeplot and GenomeScope analysis

So far, we have evaluated our short-read assemblies after they were assembled. Another useful approach is to inspect the raw reads directly before assembly. [Smudgeplot](https://github.com/KamilSJaron/smudgeplot) uses k-mer pair coverage information from a [FastK](https://github.com/thegenemyers/FASTK) database to infer the ploidy and heterozygosity structure of a genome without needing an assembly first.
This can help detect genome properties such as diploidy, polyploidy, high heterozygosity, or genome duplications directly from the sequencing reads.

On the NREC server, both `smudgeplot` and `FastK` are already installed in the `Assembly` micromamba environment. If you are not already in that environment, activate it now:

```
micromamba activate Assembly
```

For this example, we will not use the _M. genitalium_ reads from the rest of the practical. Instead, we will use the _S. cerevisiae_ demo dataset from the Smudgeplot documentation, which is already available on the server. Create a new folder in your own workspace, link the reads into it, and move into that folder:

```
cd /storage/{your_username}/Practical3
mkdir -p Smudgeplot
ln -s /storage/data/03_Assembly/smudgeplot/SRR5678680_1.fastq.gz Smudgeplot/
ln -s /storage/data/03_Assembly/smudgeplot/SRR5678680_2.fastq.gz Smudgeplot/
cd Smudgeplot
```
> Replace `{your_username}` with your own username on the server.
> `mkdir -p` creates the `Smudgeplot` folder in your work directory.
> `ln -s` makes symbolic links to the existing FASTQ files, so you can work with the data without copying it.

Now create a k-mer database with FastK:

```
FastK -v -t4 -k31 -M16 -T4 -NFastK_Table SRR5678680_[12].fastq.gz
```
> This command builds a 31-mer FastK database called `FastK_Table` from the two paired-end read files.
> The example uses 4 threads and up to 16 GB of memory, following the Smudgeplot documentation.

Next, extract heterozygous k-mer pairs from the FastK database:

```
smudgeplot hetmers -L 12 -t 4 -o kmerpairs --verbose FastK_Table
```
> `hetmers` finds pairs of k-mers that differ by one base and are informative about genome structure.
> `-L 12` filters out very low-frequency k-mers.
> `-o kmerpairs` sets the output prefix for the extracted k-mer pairs, including the `kmerpairs_text.smu` file used in the next step.

This command creates the file `kmerpairs_text.smu`, which is the exact input used in the final `smudgeplot all` step below.

Finally, infer ploidy and generate the smudgeplot:

```
smudgeplot all -o trial_run kmerpairs_text.smu
```

This will generate several output files, including PDF plots, summary tables, and logs, all beginning with the `trial_run` prefix. To see which `trial_run` outputs were created, run:

```
ls -d trial_run*
```

Look for the generated smudgeplot PDF files in that output, then download and inspect them.

For example, from your own computer you can download the resulting `trial_run` output files or folder with:

```
scp -r {your_username}@{nrec_server}:/storage/{your_username}/Practical3/Smudgeplot/trial_run* ./
```
> Replace `{your_username}` with your own username on the server.
> Replace `{nrec_server}` with the address of your NREC server.

<details>
<summary>What does the main smudge in the plot represent, and what does its position tell you about the ploidy of the <i>S. cerevisiae</i> strain?</summary>

_The main smudge is the most prominent cluster of heterozygous k-mer pairs, usually corresponding to the dominant allele relationship in the genome._
_Here, the dominant AB smudge indicates a diploid genome structure, consistent with a standard diploid_ _S. cerevisiae_ _strain showing a 1:1 allele ratio._
</details>

<details>
<summary>If the genome had a more complex structure, what would you expect to see in the smudge plot?</summary>

_If multiple genome structures were present, you would expect additional smudges at different positions or ratios in the plot._
_Those extra smudges could indicate polyploidy, segmental duplications, or more complex heterozygosity patterns._
</details>

## Long Read Assembly

Long read sequencing is becoming more and more common, even for small genomes. 
The range of tools used for assembling long reads is different than the ones we have seen above.
Many of these tools take a bit longer to run, so some patience is often required. 
We will assemble Nanopore and PacBio reads from _Mycoplasma ovipneumoniae_ (now classified as _Mesomycoplasma ovipneumoniae_).
We will use 3 different tools: [Canu](https://github.com/marbl/canu), [Flye](https://github.com/mikolmogorov/Flye), and [HifiAsm](https://github.com/chhylp123/hifiasm).

First, we'll assemble the Nanopore reads using Canu. 
Canu needs an estimate of the genome size to be able to calculate expected coverage. 
Luckily, there are already [a lot of _M. ovipneumoniae_ genomes](https://www.ncbi.nlm.nih.gov/datasets/genome/?taxon=29562) sequenced.
Based on these genome assemblies, we can say that the expected genome size is roughly 1.1 Mbp.

Long-read assembly usually takes more time than short-read assembly and has higher RAM requirements (depending on input size, of course).
To save you time, we have provided you with the Canu assembly (`MycOvi_Canu.fasta`), 
the Flye assemblies (`MycOvi_Nano_Flye.fasta` and `MycOvi_HiFi_Flye.fasta`),
and the HiFiAsm assembly (`MycOvi_HiFiasm.fasta`).
The Canu, Flye, and HiFiasm commands are included below in case you want to run them yourself.

> The Canu assembly:
>```
>canu -p canu -d MycOvi_Canu genomeSize=1.1m maxThreads=4 -nanopore MycOvi_Nano.fastq
>```
> `-p` determines the prefix (so in our case all output files will start with "canu"); `-d` sets the name of the output directory.
>
>The Flye assembly for Nanopore:
>```
>flye --nano-raw MycOvi_Nano.fastq --out-dir MycOvi_Flye --threads 2
>``` 
>
>The Flye assembly for PacBio HiFi:
>```
>flye --pacbio-hifi MycOvi_HiFi.fastq --out-dir MycOvi_HiFi_Flye --threads 2
>```
>
>The hifiasm assembly for PacBio HiFi:
>```
>hifiasm -o hifiasm -t2 MycOvi_HiFi.fastq
>awk '/^S/{print ">"$2;print $3}' hifiasm.bp.hap1.p_ctg.gfa > hifiasm.bp.hap1.p_ctg.fasta
>mkdir -p MycOvi_HiFiAsm
>mv hifiasm* MycOvi_HiFiAsm/
>```
> Since hifiasm doesn't output `fasta` directly, only `.gfa`, we use the `awk` command to convert from one to the other format.
> GFA files contain multiple lines, one of them the sequence line (starting with S).
> These sequence lines contain three columns/fields: the category (S for sequence in this case), the identifier (sequence name), and the sequence itself.
> The `awk` command here will process all lines starting with `S` (`/^S/`) and will print ">" followed by the second field in the line (`print ">"$2`).
> Then it will print the third field of the line on a new line (`print $3`).

Run Quast on the 4 assemblies, and include the reference genome for this species.
Since we are working on another species than the long reads, we'll have to find another reference species.
Try modifying the download command from the first part of the tutorial (where we downloaded the first reference genome) to download the _Mesomycoplasma ovipneumoniae_ reference genome.
You can search for the accession number of the reference genome on [NCBI genomes](https://www.ncbi.nlm.nih.gov/genome/).
Then, download and look at the QUAST report.
> If you have problems downloading the files, you can find the relevant files in [this folder](../Outputs).

<details>
<summary>Based on assembly statistics alone, are these assemblies better or worse than the assemblies we have made using short reads? (the expected genomes size is +- 1 Mbp)</summary>

_A lot better. Most assemblers manage to form one big contig close to the expected genome size._
</details>

<details>
<summary>How are the genomes compared to the reference?</summary>

_While the genomes are similar in size to the reference, they only have a low percentage of recovered genome fraction._
_They also have many mismatches, unaligned regions, indels, ..._
_This could be because:_ 

- _Our assemblies are not good (e.g. not enough trimming, bad assembly parameters, ...)_
- _Our assemblies are good, but divergent from the reference. Either our samples have been identified as the wrong species, and we are aligning to the wrong reference, or this species is very diverse, and there is a lot of genomic variation within the same species._
</details>

To check the completeness of the genome, run BUSCO on one assembly from the Nanopore reads and one assembly from the PacBio HiFi reads.

<details>
<summary>Do the BUSCO scores indicate a good assembly?</summary>

- _For the HiFi reads: yes (99.4% completeness for Mycoplasmatales)_
- _For the Nanopore reads: no (18-26% completeness for Mycoplasmatales)_

_Thus, while both Nanopore and HiFi assemblies had very good assembly statistics, the Nanopore ones have very low BUSCO completeness scores, while the HiFi showed very good BUSCO scores._
_This is because the Nanopore reads used old Nanopore technology, which had error rates up to 15-20%. Assembling a genome using these reads alone leads to assemblies full of mistakes._
_Because of that, Nanopore was often combined with short reads to fix their mistakes._ 
_The most recent Nanopore technologies have a significantly lower error rate (<5%) and can now create good assemblies on their own as well._
</details>

## Cleanup

Once you have performed all the analyses, it is time to do some cleanup. We will remove some files that we don't need anymore and will compress files to save space.

Remove the `.zip` archives that FastQC creates (Once you have the MultiQC report, they are not needed anymore):
```
rm *zip
```

Compress the assemblies, so they take up less space on the disk:
```
gzip *fasta */*fasta
```

Remove the downloaded BUSCO data:
```
rm -r busco_downloads
```

## Analysis using full readsets

In the long read assembly, we only used partial HiFi data and did not properly pre-process the long reads (due to time and computational constraints).
If you are curious about what the results would have looked like if we used all data and properly processed it, you can have a look [here](../Other/Assembly_FullAnalysis.md).
