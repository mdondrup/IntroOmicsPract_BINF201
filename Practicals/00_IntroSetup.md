# Practical 0: Introduction and Setup

This practical contains general information necessary to complete the practicals, 
and information on how to set up certain software necessary for the practicals.

## IGV
In practical 1 and in some other practicals, you will be using a genome browser to visualise genome data. 
We will be using [IGV](https://igv.org/doc/desktop) for this. 
IGV is an easy-to-use, java-based genome browser. 
It can automatically download a range of reference genomes and their annotations, but can also be used for visualisation of your own data.
IGV can be downloaded from [this link](https://igv.org/doc/desktop/#DownloadPage/).

## NREC & Linux
In praticals 2-6 we will use a range of -omics tools. 
Most -omics/bioinformatic tools are developped for linux, and require some computational resources. 
Students of the BINF201 course will have access to an [NREC](https://www.nrec.no/) instance with the necessary computational power and pre-installed software. 
Please check the BINF201 MittUiB course page for more information on how to get access to the BIN201 NREC server.

The NREC servers run a Linux operation system (in our case Ubuntu), which is different to navigate than a Windows system. 
You can only interface with the server using the command line. 
If you are not familiar with working on the command line, please have a look at [this excellent shell tutorial](https://swcarpentry.github.io/shell-novice/).
A short overview of useful commands, hints, and tips can also be found [here](../Other/ShellTips.md)

If you are working on NREC, you can skip the next parts, and go straight to the `R & Rstudio` section.

It is also possible to do the practicals on your own computer:
If you have a Linux system, you can just install the necessary software and download the necessary data yourself (instructions are provided in the tutorials themselves).
If you have a Windows system, it is advised to use [WSL (Windows subsystem for Linux)](https://learn.microsoft.com/en-us/windows/wsl/install).
Then, you can install and run the necessary software on the WSL terminal as if you are working on Linux.
For Mac users, it should be possible to install and run most of the software on the command line similar to Linux users.
However, some packages are not fully up to date for Mac compared to Linux, which can cause problems.

## Shell
Most software will be controlled from the command line (the shell).
If you are not familiar with working on the command line, please have a look at [this excellent shell tutorial](https://swcarpentry.github.io/shell-novice/).
A basic mastery of the command line will make going through the tutorials and identifying possible problems a lot easier.
A short overview of useful commands, hints, and tips can be found [here](../Other/ShellTips.md)

## Conda
All necessary software is already installed on the NREC server. 
Most software have been installed using the [Mamba](https://github.com/mamba-org/mamba) package manager.
Mamba is an upgrade of the [Conda](https://docs.conda.io/en/latest/) package manager.
Conda allows users to easily install a range of (bio)informatics software and packages (for python, perl, R, ...). 
The main benefit of using Conda is that it automatically tries to solve dependencies.
This means that if you ask Conda to install a certain software package (or python package), it will recursively check which other packages are required, and install them. 

On NREC, software has been installed using [Micromamba] (https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html)
Micromamba is a lightweight re-implementation of Conda with the main advantage being its speed.  

Note: if you see install commands starting with `conda`, in most cases, you can simply replace them with `mamba` or `micromamba` depending on the package manager you are using.  


Many of the bioinformatic packages that we will use are available through [Bioconda](https://bioconda.github.io/index.html). 
If you are installing the packages yourself, it is advised to set up the bioconda channel as part of the default by running the following code:

```
micromamba config set channel_priority strict
micromamba config prepend channels bioconda
micromamba config prepend channels conda-forge
```

Or for conda:
```
conda config --remove channels defaults
conda config --set channel_priority strict
conda config --prepend channels bioconda
conda config --prepend channels conda-forge

```

## Data download

All data used in these practicals is publically available. 
For students of the UiB BINF201 course, all necessary data is already present on the NREC server.

For people who do not have access to NREC and/or want to do the practicals on their own computer, all data used in this practicals is available on [Zenodo](https://zenodo.org/uploads/13120340).
Download links to the specific files needed for each tutorial are available in the tutorials themselves.

Alternatively, you can download and pre-process the data yourself. 
To do this, it is advised to create an environment dedicated to downloading and processing the necessary data.
You can create this enviroment by running the following command (after having installed `mamba`, see above):

```
mamba create -n downloader sra-tools ncbi-datasets-cli seqtk pigz
```

- [SRA tools](https://github.com/ncbi/sra-tools) is used for downloading data from [SRA](https://www.ncbi.nlm.nih.gov/sra) (mainly raw sequencing data in `.fastq` format).
- [NCBI datasets](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/download-and-install/) is used for downloading assembled sequences (genomes, genes, proteins) from NCBI.
- [SeqTK](https://github.com/lh3/seqtk) is a sequence toolkit, which we will mainly use to subset larger datasets into smaller ones.
- [Pigz](https://github.com/madler/pigz) is a parallel implementation of GZip, allowing fast (de)compression of files.

The commands needed to download and process the data are included in each practical as well.

## R and Rstudio

For practicals 7-9 we will be using R and R studio. 
These have to be downloaded and installed yourself. 
Please follow the instructions [here](https://posit.co/download/rstudio-desktop/) on how to install both R and Rstudio for your operating system.
For these practicals, it is not necessary to be on Linux or the NREC server.
