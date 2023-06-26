## Basic analysis commands

### Analysis tools

```
fastqc
tanoti
samtools
bcftools
tablet
seqtk
seqkit
kraken2
KrakenTools
krona
```

### Creating the analysis directroy

We create a directory `analysis`, which will be our working space for this session. We use the famous `mkdir` command to achive this. Later, we shall be creating subdirectories inside the `analysis` directory for the different steps, includin; qc, mapping, e.t.c.

```
mkdir analysis
```

### Obtain the dataset from ENA

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

### Read-based taxonomic identification using Kraken2

```
cp -r /opt/metagenome/KrakenTools .
cp -r /opt/metagenome/kraken krakenDB

mkdir kraken

kraken2 --db krakenDB --paired data/SRR19400485_1.fastq data/SRR19400485_2.fastq --report kraken/SRR19400485.report > kraken/SRR19400485.txt
python KrakenTools/kreport2krona.py -r kraken/SRR19400485.report -o kraken/SRR19400485.krona 
ktImportText kraken/SRR19400485.krona  -o kraken/SRR19400485.html
```

### Mapping mNGS onto reference genomes

Map the short reads onto the reference generate, and obtain mapping statistics

```
tanoti -r data/sars-cov-2.fasta -i data/SRR19400485_1.fastq data/SRR19400485_2.fastq -p 1
SAM_STATS FinalAssembly.sam
```

Convert `.sam` file to `.bam` format, sort it, and generate a consensus sequence from the alignment map.

```
samtools view -b SRR19400485.sam > SRR19400485.bam
samtools sort  SRR19400485.bam -o SRR19400485_sorted.bam
bcftools mpileup -f sars-cov-2.fasta SRR19400485_sorted.bam | bcftools call -c | vcfutils.pl vcf2fq > SRR19400485.fastq
seqtk seq -a SRR19400485.fastq > SRR19400485.fasta
seqkit seq SRR19400485.fasta --upper-case > SRR19400485_cns.fasta
```


### Extract mapped reads and do a de novo assembly on them

```
mkdir denovo
samtools view -F 4 SRR19400485_sorted.bam > SRR19400485_mapped.bam
awk '{OFS="\t";  print ">"$1"\n"$10}' SRR19400485_mapped.bam > SRR19400485_mapped.fasta
```
