# Pig-Gut-Genomes
Genome collection of isolates and MAGs from the pig gut

# Description
This collection include genomes (isolates and MAGs) from following studies:

PIBAC: Isolate and MAG collection:
Wylensek, 2020, A collection of bacterial isolates from the pig intestine reveals functional and taxonomic diversity.
https://doi.org/10.1038/s41467-020-19929-w
https://github.com/tillrobin/PIBAC
https://www.dsmz.de/pibac

Chen, 2021, Expanded catalog of microbial genes and metagenome-assembled genomes from the pig gut microbiome." Nature communications 12.1 (2021): 1-13.
https://doi.org/10.1038/s41467-021-21295-0

Holman, 2022, Novel insights into the pig gut microbiome using metagenome-assembled genomes
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

# Pipelines for using the collection


## Genome based taxnomic abundance profilling to representative genomes

### 1. We recommend the use of [Bioconda](http://bioconda.github.io/) eg create bioconda environment with bwa2 and bbmap with:

    conda create -n PIGrun bwa2 bbmap
	conda activate PIGrun

## 2. Download bwa2-index (Warning 25,7GB but you can use option 2b as an alternative)

    download [Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.fasta.gz](https://1drv.ms/f/s!Am-fED1L6602hcVH65QUhQZse5vOzA) 
	gzip -d Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.fasta.gz


## 3. Map the samples with bwa2 to the Pig_genomes_dRep9095-mMAGs-dereplicated_genomes_v01.fasta

	Cores=24                      # please check your server
	RefFasta=/path/to/index/      # bwa2 index
	FastqPathR1=/path/to/file/R1  # set path to read 1
	FastqPathR2=/path/to/file/R2  # set path to read 2
	SampleName=MySampleName       # create name for the samples

### 3. Mapping to sam-file and sumup via pipeup from bbmap-tools (create sam-file I/O-weighted)

    bwa-mem2 mem -t ${Cores} ${RefFasta} ${FastqPathR1} ${FastqPathR2} | pigz --fast > /tmp/${SampleName}.sam.gz
	pileup.sh in=/tmp/${SampleName}.sam.gz covstats=${SampleName}.covstats 32bit=t 2> ${SampleName}.log
	rm /tmp/${SampleName}.sam.gz


## 4. Convert covstats to TPM (normalize count data to genome size and relative to 1 million reads)

    bash TPM-Script ${SampleName}.covstats # create TPM-${SampleName}.txt
	bash create-abundance-table.sh         # summarizing all Samples into one matrix file

