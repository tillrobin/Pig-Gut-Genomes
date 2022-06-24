# Pig-Gut-Genomes
Genome collection of isolates and MAGs from the pig gut

# Description
This collection include genomes (isolates and MAGs) from following studies:

PIBAC: Isolate and MAG collection:
Wylensek, 2020, "A collection of bacterial isolates from the pig intestine reveals functional and taxonomic diversity." Nature communications
https://doi.org/10.1038/s41467-020-19929-w
https://github.com/tillrobin/PIBAC
https://www.dsmz.de/pibac

Chen, 2021, "Expanded catalog of microbial genes and metagenome-assembled genomes from the pig gut microbiome." Nature communications
https://doi.org/10.1038/s41467-021-21295-0

Holman, 2022, "Novel insights into the pig gut microbiome using metagenome-assembled genomes" bioRxiv
https://doi.org/10.1101/2022.05.19.492759

# Pipelines for creating the collection

## Collection

Download all genomes and create a checkM-data.txt including all CheckM information of all genomes you want to cluster:

	genome,completeness,contamination
	PIG0001.fa,97.07,1.27
	PIG0002.fa,58.9,0.67
	PIG0003.fa,79.84,0.23
	[...]


## Dereplication

Run dRep via bioconda using all mMAGs (comp >= 50, con < 10) and cluster to 95% ANI

	# create dRep environment
	conda create -n dRep dRep
	# run for mMAGs
	conda activate dRep
	dRep dereplicate drep-outout \
	--genomeInfo checkM-data.txt \
	-comp 50 -con 10 -nc 0.3 \
	-sa 0.95 -p 32 \
	-g FileListPath.txt

This will resulting in 3,532 clusters, see contributions of the studies in the venn plot:

<img src="https://github.com/tillrobin/Pig-Gut-Genomes/raw/main/drep5010venn.png"  height="400">


## Taxonomic classification for final genome set

Run gtdbtk for all dereplicated genomes

	# create gtdbtk environment
	conda create -n gtdbtk gtdbtk
	# run for dereplicated_genomes
	conda activate gtdbtk
	gtdbtk classify_wf \
	--genome_dir dereplicated_genomes \
	--out_dir gtdbtk-dereplicated_genomes \
	--cpus 64 -x fa 

The results you can find in the summary files:
[Classifications archaeal genomes](/gtdbtk.ar53.summary.tsv)
[Classifications bacterial genomes](/gtdbtk.bac120.summary.tsv)

Here a visualisation of the top families:

<img src="https://github.com/tillrobin/Pig-Gut-Genomes/raw/main/sunbrust.png"  height="500">

# Pipelines for using the collection


## Genome based taxnomic abundance profilling to representative genomes

### 1. We recommend the use of [Bioconda](http://bioconda.github.io/) eg create bioconda environment with bwa2 and bbmap with:

    conda create -n PIGrun bwa2 bbmap
	conda activate PIGrun

## 2. Download bwa2-index (Warning 22GB but you can use option 2b as an alternative)

download [Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.tar.gz](https://onedrive.live.com/download?cid=36ADEB4B3D109F6F&resid=36ADEB4B3D109F6F%21114785&authkey=AMy3k92ykHzmXwk)

	wget -O "Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.tar.gz" "https://onedrive.live.com/download?cid=36ADEB4B3D109F6F&resid=36ADEB4B3D109F6F%21114785&authkey=AMy3k92ykHzmXwk"
	tar xzf Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.tar.gz

### 2b. Download mMAG-fasta and run bwa2-index (2GB, will take 2-3 hours to process)

download [Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.fasta.gz](https://1drv.ms/f/s!Am-fED1L6602hcVH65QUhQZse5vOzA)

	wget -O "Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.fasta.gz" "https://onedrive.live.com/download?cid=36ADEB4B3D109F6F&resid=36ADEB4B3D109F6F%21114785&authkey=AMy3k92ykHzmXwk"
	gzip -d Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.fasta.gz
	bwa-mem2 index Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.fasta


## 3. Map the samples with bwa2 to the dereplicated genomes

	Cores=24                      # please check your server
	RefFasta=/path/to/index/      # bwa2 index
	FastqPathR1=/path/to/file/R1  # set path to read 1
	FastqPathR2=/path/to/file/R2  # set path to read 2
	SampleName=MySampleName       # create name for the samples

### 4. Mapping to sam-file and sumup via pipeup from bbmap-tools (create sam-file I/O-weighted)

    bwa-mem2 mem -t ${Cores} ${RefFasta} ${FastqPathR1} ${FastqPathR2} | pigz --fast > /tmp/${SampleName}.sam.gz
	pileup.sh in=/tmp/${SampleName}.sam.gz covstats=${SampleName}.covstats 32bit=t 2> ${SampleName}.log
	rm /tmp/${SampleName}.sam.gz


## 5. Convert covstats to TPM

- filter out genomes with less than 20% genomic coverage
- normalize count data to genome size and relative to 1 million reads
- download bash script [make-TPM-cov20-biom.sh] to process your covstats files

 this script will grep all *.covstats files in the working folder and create TPM and biom-files 
 
	bash make-TPM-cov20-biom.sh
	

## 6. Use otu-table/biom-file for analyse the data

- create mapping file containing matching "#SampleID" with a group assignment
- Local: Import biom-file and metadata mapping file into R phyloseq https://github.com/joey711/phyloseq
- Online: Analyse with Microbiomeanalyst 
	- Import biom-file and metadata mapping file via Marker Data Profiling https://www.microbiomeanalyst.ca/
	- Data Upload -> biom-format
	  -  use TPM or raw-count table in biom format
	- Data filtering
	  - please note that the TPM version already include normalication to genomesize and a filtering of low covarge genomes


