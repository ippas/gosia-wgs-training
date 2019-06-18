FROM centos:5
MAINTAINER Feiyu Du <fdu@wustl.edu>

LABEL \
  version="v2.18" \
  description="bcl2fastq2 docker image"

RUN yum -y install wget unzip

WORKDIR /opt
RUN wget --no-check-certificate https://support.illumina.com/content/dam/illumina-support/documents/downloads/software/bcl2fastq/bcl2fastq2-v2-18-0-12-linux-x86-64.zip && \
  unzip bcl2fastq2-v2-18-0-12-linux-x86-64.zip

RUN yum -y --nogpgcheck localinstall bcl2fastq2-v2.18.0.12-Linux-x86_64.rpm

ENTRYPOINT ["/usr/local/bin/bcl2fastq"]
