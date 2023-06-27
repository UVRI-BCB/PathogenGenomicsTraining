## Basic analysis commands

### Analysis tools

+ Quality control: `fastqc` and `multiqc`
+ Read-based tanonomic identification: `kraken2` 
+ Processing kraken results: `KrakenTools`
+ Visualising Kraken results: `krona`
+ Mapping reads onto reference genomes: `tanoti`
+ Visualising alignments: `tablet`

### Creating the analysis directory

We create a directory `analysis`, which will be our working space for this session. We use the famous `mkdir` command to achive this. Later, we shall be creating subdirectories inside the `analysis` directory for the different steps, includin; qc, mapping, e.t.c.

```
mkdir analysis
cd analysis
```

### The practice dataset

In interest of time, the data was downloaded from ENA ahead of time and stored on the server. Use the command below to create a copy of the data in your working directory.

```
cp -r /opt/metagenome/data .
```

### Quality control

First, let us look at the quality of the sequence data that we have using `fastqc` and `multiqc` tools.

```
mkdir qc
fastqc data/SRR19400451_*.fastq -o qc
multiqc qc/* -o qc 
```

At this point, we use `scp` to download the MultiQC report and have a look at it, according to our assessement of the report, we can choose whether or not to do some adaptor/quality trimming. To do this, open a new tab on your terminal/mobaxterm shell, navigate to where you would want to keep the MultQC report and run the command below.

```
scp acountname@xxx.xx.xxx.xx:/home/accountname/analysis/qc/multiqc_report.html .
```

For participants using Mobaxterm, we could simply download, by navigating to the `qc` directory in the left panel of mobaxterm window and downloading the MultiQC report to a desired folder on own computer. 

We use `trim_galore` for adaptor and quality trimming. Below is an example, the cut-offs on quality scores, length, e.t.c indicated below are arbitrary. In practice, the choice of these parameters guided by the assessment made on the QC plots generated above.

```
mkdir trimmed
trim_galore -q 30 --paired data/sample1_R1.fq data/sample1_R2.fq -o trimmed
```

In this case, we may not have to quality trim our data, but just in case we did, we would used the trimmed data from this point onwards.

### Read-based taxonomic identification using Kraken2

```
cp -r /opt/metagenome/KrakenTools .
cp -r /opt/metagenome/kraken krakenDB

mkdir kraken

kraken2 --db krakenDB --paired data/SRR19400485_1.fastq data/SRR19400485_2.fastq --report kraken-output/SRR19400485.report > kraken-output/SRR19400485.txt
python KrakenTools/kreport2krona.py -r kraken-output/SRR19400485.report -o kraken-output/SRR19400485.krona 
ktImportText kraken-output/SRR19400485.krona  -o kraken-output/SRR19400485.html
```

### Mapping mNGS onto reference genomes

Map the short reads onto the reference genome, and obtain mapping statistics

```
tanoti -r data/sars-cov-2.fasta -i data/SRR19400485_1.fastq data/SRR19400485_2.fastq -p 1
SAM_STATS FinalAssembly.sam
```

To keep order in our working directory, let us create a mapping directory add the alignment map to it and change it name to reflect the sample ID. 

```
mkdir mapping
mv FinalAssembly.sam mapping/SRR19400485.sam
```

Generate a consensus sequence from the alignment map

```
SAM2CONSENSUS -i mapping/SRR19400485.sam -o mapping/SRR19400485-SARS-CoV-2.fasta
```

### De novo assembly

This section comprises of three main stages; removing host reads from the read data, assembling short reads into longer contiguous sequences using a metagenome assembler, spades in this case and blasting the contigs against a protein database using `diamond`.

#### Removal of host reads 

Prior to performing de novo assembly, we remove the host reads from the data. Here we use `bowtie2` to map reads to the host genome and retain the reads that do not map to the host genome for de novo assembly. The Human reference genome can be downloaded from Ensembl, we have added a bowtie2-indexes of the Human genome to the server.

```
mkdir hosttfree
bowtie2 -x /opt/metagenome/GRCh38_noalt_as/GRCh38_noalt_as -1 data/SRR19400485_1.fastq -2 data/SRR19400485_2.fastq --un-conc-gz hosttfree/SRR19400485_clean > hosttfree/SRR19400485_host.sam &

mv hosttfree/SRR19400485_clean.1 hosttfree/SRR19400485_clean_1.gz
mv hosttfree/SRR19400485_clean.2 hosttfree/SRR19400485_clean_2.gz
gunzip hosttfree/*.gz
```

Assemble the short reads into longer contigs using `spades`.

```
mkdir spades_output
metaspades.py -1 hosttfree/SRR19400485_clean_2.fastq  -2 hosttfree/SRR19400485_clean_2.fastq  -o spades_output
```


