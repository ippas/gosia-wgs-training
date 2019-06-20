# gosia-wgs-training
comparison of two wgs methods: illumina vs 10x genomics

## data 
one person's genome (stored in projects/ifpan-marpiech-genome)

# illumina:
1. convert 10x bcl to fastq wih bcl2fastq2 with [this code](https://gist.github.com/gosborcz/b31df08f6bb8b83c51f7a310f8f2bcc1)

2. fastqc

3. alignment bwa-mem do hg38 z broadinstitute to bam (https://storage.cloud.google.com/genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta?_ga=2.192558178.-935441401.1560518376) + fai

4. gatk best practices variant calling


# 10x genomics
1. use longranger to convert bcl to fastq:
`longranger mkfastq --run . --csv SampleSheet.csv --output-dir longrangerfq`
2. continue with the longranger pipelines (here)[https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger]
