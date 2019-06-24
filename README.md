# ifpan-marpiech-wgs
Comparison of two wgs methods: illumina vs 10x genomics

## data 
one person's genome (stored in projects/ifpan-marpiech-genome)

# illumina:
1. convert 10x bcl to fastq wih bcl2fastq2 with [this code](https://gist.github.com/gosborcz/b31df08f6bb8b83c51f7a310f8f2bcc1)

2. fastqc with [this docker image](https://hub.docker.com/r/pegi3s/fastqc):
`docker run -d --rm -v $PWD:/data pegi3s/fastqc /data/<fastq_name>`
* fastqc reports [for R1](http://149.156.177.112/projects/ifpan-marpiech-wgs/illumina-fq/mp_S0_L002_R1_001_fastqc.html) and [for R2](http://149.156.177.112/projects/ifpan-marpiech-wgs/illumina-fq/mp_S0_L002_R2_001_fastqc.html)
* Warnings (in both cases): Per tile sequence quality and per sequence GC content

3. alignment with bwa-mem to [hg38 from broadinstitute](https://storage.cloud.google.com/genomics-public-data/resources/broad/hg38/v0/Homo_sapiens_assembly38.fasta?_ga=2.192558178.-935441401.1560518376) to bam:
* in an intelliseq docker container (add dockerfile from Marcin)
* indexing with bwa:
`docker run -v $PWD:/data intelliseq/bwa:latest bwa index /data/hg38/Homo_sapiens_assembly38.fasta`
* alignment with bwa: 
`docker run -t 8 -v $PWD:/data intelliseq/bwa:latest bwa mem -v 3 -Y -K 10000000 -t 8 data/hg38/Homo_sapiens_assembly38.fasta data/illumina-fq/mp_S0_L002_R1_001.fastq.gz data/illumina-fq/mp_S0_L002_R2_001.fastq.gz > mp-aln-illumina.sam`

4. gatk best practices variant calling


# 10x genomics
1. longranger to convert bcl to fastq:
`longranger mkfastq --run . --csv SampleSheet.csv --output-dir longrangerfq`


*note: The output generated four folders with fastqs according to the supplied Samplesheet:*

|Lane|Sample_ID|Sample_Name|index|Sample_Project|
|----|----------|-------------|-------|-------------|
|1|SI-GA-F3_1|1-AK1255|TTCAGGTG|Chromium_20190402|
|1|SI-GA-F3_2|1-AK1256|ACGGACAT|Chromium_20190402|
|1|SI-GA-F3_3|1-AK1257|GATCTTGA|Chromium_20190402|
|1|SI-GA-F3_4|1-AK1258|CGATCACC|Chromium_20190402|


[From 10x website](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/using/fastq-input): "It is likely that an input samplesheet was used that explictly separated the four oligos in a 10x sample index set into four separate sample names. You probably want to be able to merge All samples from the SI-GA-A1 index into a single analysis. If you only run one index at a time, you will see a smaller number of reads than expected, which may translate to lower coverage or cell count than you expect for your experiment."

This step can be alternatively done with Illuminas bcl2fastq with [this code](https://gist.github.com/gosborcz/bc6896406b776c41e83c37d7568cbe1a)

2. fastqc only on R2 files as I1 contain indexes and R1 barcodes, with [this docker image](https://hub.docker.com/r/pegi3s/fastqc):
`docker run -d --rm -v $PWD:/data pegi3s/fastqc /data/<fastq_name>`

* reports: [SI-GA-F3_1](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_1/1-AK1255_S1_L001_R2_001_fastqc.html), [SI-GA-F3_2](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_2/1-AK1256_S2_L001_R2_001_fastqc.html), [SI-GA-F3_3](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_3/1-AK1257_S3_L001_R2_001_fastqc.html), [SI-GA-F3_4](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_4/1-AK1258_S4_L001_R2_001_fastqc.html)
* warnings: per sequence GC content, **failed: per tile sequence quality** (they all seem to have problems in the same tiles towards the ends)

3. continue with the [longranger pipelines](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger)
**longranger wgs with gatk** 
*longranger does not like the hg38 from Broad institute: so the download is from their reference*
* code to run long ranger:
`longranger wgs --id mp10x --fastqs <directory-with-fastqs> --vcmode gatk:/opt/tools/gatk-4.0.3.0/gatk-package-4.0.3.0-local.jar --reference <path-to-10x-provided-reference-directory> --sample=1-AK1255,1-AK1256,1-AK1257,1-AK12581`


# software versions:
1. bcl2fastq v2.20.0.422 (on server)
2. logranger 2.2.2 (on server)
3. FastQC v0.11.7 (docker container)
4. gatk 4.03 *this is for longranger as it does not suppor newer versions for now* (on server)
5. bwa (intelliseq container): Version: 0.7.17-r1188

#reference genome:
1. For Illumina: https://console.cloud.google.com/storage/browser/genomics-public-data/resources/broad/hg38/v0 + .fai
2. For 10x: http://cf.10xgenomics.com/supp/genome/refdata-GRCh38-2.1.0.tar.gz
