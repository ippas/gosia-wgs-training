# ifpan-marpiech-wgs

[interactive view of aligmnents](http://149.156.177.112/projects/ifpan-marpiech-wgs/alignments.html)

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
* getting the rg_id:
`zcat -f mp_S0_L002_R1_001.fastq.gz | head -1 | cut -d ':' -f 3,4 | sed 's/:/\./g'`, out: `HYFFWCCXY.2`
* alignment with bwa
`docker run -d -v $PWD:/data intelliseq/bwa:latest bwa mem -t 6 -R "@RG\tID:HYFFWCCXY.2\tPU:HYFFWCCXY.2.mp-illumina\tPL:Illumina\tLB:mp-illumina.library\tSM:mp-illumina" -K 10000000 -v 3 -Y /data/hg38/Homo_sapiens_assembly38.fasta /data/illumina-fq/mp_S0_L002_R1_001.fastq.gz /data/illumina-fq/mp_S0_L002_R2_001.fastq.gz 2> mp.illumina.bwa.stderr.log > mp.illumina.sam`
* samblaster
`docker run -v $PWD:/data intelliseq/bwa:latest samblaster -i /data/mp.illumina.sam -o /data/mp.illumina.markdup.sam 2> mp.illumina.bwa.samblaster.stderr.log`
* samtools sort
`docker run -v $PWD:/data intelliseq/bwa:latest samtools sort -o /data/mp.illumina.markdup.bam -@ 6 /data/mp.illumina.sam`
* after .bam creation I have rm the .sam files (they are huuuuuge)

4. gatk:
* generate .bai with picard: `docker run -v $PWD:/data broadinstitute/picard:latest BuildBamIndex I=/data/mp.illumina.markdup.bam`
* run gatk: `docker run -v $PWD:/gatk/data broadinstitute/gatk:latest gatk HaplotypeCaller -R data/hg38/Homo_sapiens_assembly38.fasta -I data/mp.illumina.markdup.bam -output data/output.raw.snps.indels.vcf`



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

2. fastqc only on R2 files as I1 contains indexes and R1 barcodes, with [this docker image](https://hub.docker.com/r/pegi3s/fastqc):
`docker run -d --rm -v $PWD:/data pegi3s/fastqc /data/<fastq_name>`

* reports: [SI-GA-F3_1](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_1/1-AK1255_S1_L001_R2_001_fastqc.html), [SI-GA-F3_2](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_2/1-AK1256_S2_L001_R2_001_fastqc.html), [SI-GA-F3_3](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_3/1-AK1257_S3_L001_R2_001_fastqc.html), [SI-GA-F3_4](http://149.156.177.112/projects/ifpan-marpiech-wgs/10x-fq/Chromium_20190402/SI-GA-F3_4/1-AK1258_S4_L001_R2_001_fastqc.html)
* warnings: per sequence GC content, **failed: per tile sequence quality** (they all seem to have problems in the same tiles towards the ends)

3. continue with the [longranger pipelines](https://support.10xgenomics.com/genome-exome/software/pipelines/latest/what-is-long-ranger)
**longranger wgs with gatk**
* longranger does not like the hg38 from Broad institute: so the download is from their reference and it needs java 1.8
* preparing the docker container (adding gatk 4.0.3) based on [biocontainer/longranger](https://hub.docker.com/r/biocontainers/longranger):
```
docker run -it biocontainers/longranger:v2.2.2_cv2 /bin/bash
      # cd /home/biodocker
      # wget https://github.com/broadinstitute/gatk/releases/download/4.0.3.0/gatk-4.0.3.0.zip
      # unzip gatk-4.0.3.0.zip
      # exit
docker commit jovial_dewdney longrangergatk:l2.2.2g4.03
```

* code to run longranger wgs:
`docker run -d -v $PWD:/data longrangergatk:l2.2.2g4.03 longranger wgs --id mp10x --fastqs /data/10x-fq --vcmode gatk:/home/biodocker/gatk-4.0.3.0/gatk-package-4.0.3.0-local.jar --reference /data/hg38/refdata-GRCh38-2.1.0 --sample=1-AK1255,1-AK1256,1-AK1257,1-AK12581`

* using 10x loupe software to visualise 10x data (download instructions here: https://support.10xgenomics.com/genome-exome/software/downloads/latest). `ssh -L 3001:localhost:3001 ifpan "LOUPE_PORT=3001 LOUPE_SERVER=../loupe-dir ../loupe/start_loupe.sh"`. Then navigate in you local browser: localhost:3001

* In the fastq files tht 151th bp was not trimmed. [seqtk](https://github.com/lh3/seqtk) was used [in this docker container](https://hub.docker.com/r/biocontainers/seqtk): 
```docker run -d --rm -v /:/data biocontainers/seqtk:v1.2-1-deb_cv1 bash
      $ seqtk trimfq -e 1 /data/[path-to-input] > /data/[path-to-input]```


## Bam visualisation in IGV
* page source code can be found [here](alignments.html)

# software versions:
1. bcl2fastq v2.20.0.422 (on server)
2. logranger 2.2.2 (docker container)
3. FastQC v0.11.7 (docker container)
4. gatk 4.03 *this is for longranger as it does not suppor newer versions for now* (on server)
5. bwa (intelliseq container): Version: 0.7.17-r1188
6. samblaster (intelliseq container): Version 0.1.24
7. samtools (intelliseq container): Version 1.8

# reference genome:
1. For Illumina: https://console.cloud.google.com/storage/browser/genomics-public-data/resources/broad/hg38/v0 + .fai
2. For 10x: http://cf.10xgenomics.com/supp/genome/refdata-GRCh38-2.1.0.tar.gz
