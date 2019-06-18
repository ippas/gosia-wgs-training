# gosia-wgs-training
comparison of two wgs methods: illumina vs 10x genomics

## data 
One person's genome (stored in projects/gosia-wgs-training)

# illumina:

1. convert 10x bcl to fastq
* [dockerfile][bcl2fastq-dockerfile] of Illumina bcl2fastq image


2. fastqc


3. alignment bwa-mem do hg38 z broadinstitute to bam (https://storage.cloud.google.com/genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta?_ga=2.192558178.-935441401.1560518376) + fai


4. gatk best practices variant calling


10x genomics - inny protokół
