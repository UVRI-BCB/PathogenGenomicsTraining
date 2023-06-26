## Basic analysis commands

### Obtain the dataset from ENA

```
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR194/085/SRR19400485/SRR19400485_1.fastq.gz
wget ftp://ftp.sra.ebi.ac.uk/vol1/fastq/SRR194/085/SRR19400485/SRR19400485_1.fastq.gz
```

### Quality control

```
fastqc SRR19400451_*.fastq -o .
```

### Mapping mNGS onto reference genomes

```
bowtie2-build sars-cov-2.fasta sars

samtools view -b SRR19400485.sam > SRR19400485.bam

samtools sort  SRR19400485.bam -o SRR19400485_sorted.bam

bcftools mpileup -f sars-cov-2.fasta SRR19400485_sorted.bam | bcftools call -c | vcfutils.pl vcf2fq > SRR19400485.fastq

seqtk seq -a SRR19400485.fastq > SRR19400485.fasta

seqkit seq SRR19400485.fasta --upper-case > SRR19400485_cns.fasta

samtools index SRR19400485_sorted.bam
```

### Extract mapped reads and do a de novo assembly on them

```
samtools view -F 4 SRR19400485_sorted.bam > SRR19400485_mapped.bam

awk '{OFS="\t";  print ">"$1"\n"$10}' SRR19400485_mapped.bam > SRR19400485_mapped.fasta
```

### Taxonomic identification using Kraken2

```
kraken2 --db kraken --paired SRR19400485_1.fastq SRR19400485_2.fastq --report SRR19400485.report > SRR19400485.txt

python KrakenTools/kreport2krona.py -r SRR19400485.report -o SRR19400485.krona 

ktImportText SRR19400485.krona  -o SRR19400485.html
```
