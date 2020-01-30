# Battenberg
-----

## Advised installation and running instructions

Please visit the [cgpBattenberg page](https://github.com/cancerit/cgpBattenberg), download the code from there and run the setup.sh script. The battenberg.pl script can then be used to run the pipeline.


## Advanced installation instructions

The instructions below will install the latest stable Battenberg version. Please take this approach only when you'd like to do something not covered by cgpBattenberg.

### Standalone

#### Prerequisites

Installing from Github requires devtools and Battenberg requires readr, RColorBrewer and ASCAT. The pipeline requires doParallel. From the command line run:

```
R -q -e 'source("http://bioconductor.org/biocLite.R"); biocLite(c("devtools", "readr", "doParallel", "ggplot2", "RColorBrewer", "gridExtra", "gtools"));'
R -q -e 'devtools::install_github("Crick-CancerGenomics/ascat/ASCAT")'
```

#### Installation from Github

To install Battenberg, run the following from the command line:

```
R -q -e 'devtools::install_github("Wedge-Oxford/battenberg")'
```

#### Required reference files

Battenberg requires a number of reference files that should be downloaded.

  * ftp://ftp.sanger.ac.uk/pub/teams/113/Battenberg/battenberg_1000genomesloci2012_v3.tar.gz
  * ftp://ftp.sanger.ac.uk/pub/teams/113/Battenberg/battenberg_impute_1000G_v3.tar.gz
  * ftp://ftp.sanger.ac.uk/pub/teams/113/Battenberg/probloci_270415.txt.gz
  * ftp://ftp.sanger.ac.uk/pub/teams/113/Battenberg/battenberg_wgs_gc_correction_1000g_v3.tar.gz
  * ftp://ftp.sanger.ac.uk/pub/teams/113/Battenberg/battenberg_snp6_exe.tgz (SNP6 only)
  * ftp://ftp.sanger.ac.uk/pub/teams/113/Battenberg/battenberg_snp6_ref.tgz (SNP6 only)
  
#### Pipeline

Go into inst/example for example WGS and SNP6 R-only pipelines.
  
### Docker - experimental

Battenberg can be run inside a Docker container. Please follow the instructions below.

#### Installation

```
wget https://raw.githubusercontent.com/Wedge-Oxford/battenberg/dev/Dockerfile
docker build -t battenberg:2.2.8 .
```

#### Download reference data

To do


##### hg38 for Beagle5

Modified original code to derive the input vcf for Beagle5 and hg38:

```
#!/bin/bash
#
# READ_ME file (08 Dec 2015)
# 
# 1000 Genomes Project Phase 3 data release (version 5a) in VCF format for use with Beagle version 4.x
#    Data Source: ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/release/20130502/*
#
# NOTES:
# 
# 1) Markers with <5 copies of the reference allele or <5 copies of the non-reference alleles have been excluded.
#
# 2) Structural variants have been excluded.
#
# 3) All non-unique identifiers in the ID column are removed from the ID column
#
# 4) Additional marker filtering may be performed using the gtstats.jar and filterlines.jar utilities
#
# 5) Sample information is in files: 
#      integrated_call_samples.20130502.ALL.ped
#      integrated_call_samples_v3.20130502.ALL.panel
#
# 6) Male haploid chromosome X genotypes are encoded as diploid homozygous genotypes.
#
############################################################################
#  The following shell script was used to create the files in this folder  #
############################################################################
#

## required if loading modules
module load Java
module load HTSlib

## wget for GRCh37
## wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/release/20130502/*

# wget for GRCh38 (liftover from hg38)
wget ftp://ftp.1000genomes.ebi.ac.uk/vol1/ftp/release/20130502/supporting/GRCh38_positions/*
wget http://bochet.gcc.biostat.washington.edu/beagle/1000_Genomes_phase3_v5a/utilities/filterlines.jar
wget http://bochet.gcc.biostat.washington.edu/beagle/1000_Genomes_phase3_v5a/utilities/gtstats.jar
wget http://bochet.gcc.biostat.washington.edu/beagle/1000_Genomes_phase3_v5a/utilities/remove.ids.jar
wget http://bochet.gcc.biostat.washington.edu/beagle/1000_Genomes_phase3_v5a/utilities/simplify-vcf.jar

## Downloadable from Beagle5 web page
gts="java -ea -jar gtstats.jar"
fl="java -ea -jar filterlines.jar"
rmids="java -ea -jar remove.ids.jar"
simplify="java -ea -jar simplify-vcf.jar"
min_minor="5"

## Running directory 
src="./"
#mkdir ${src}
#cd ${src}
#cd -

## Go through autosomes and prepare vcf
for chr in $(seq 1 22); do
echo "chr${chr}"
input="${src}ALL.chr${chr}_GRCh38.genotypes.20170504.vcf.gz"
#vcf_removefield="${input}_removedfield.vcg.gz"
vcf="chr${chr}.1kg.phase3.v5a_GRCh38.vcf.gz"
excl="chr${chr}.1kg.phase3.v5a_GRCh38.excl"

#zcat ${input} | awk '/^[^#]/ {gsub(";GRC.*","",$9);print}' > ${vcf_removefield}
zcat ${input} | grep -v 'SVTYPE' | ${gts} | ${fl} \# -13 ${min_minor} 1000000 | cut -f2 > ${excl}
zcat ${input} | grep -v 'SVTYPE' | grep -v '^#' | cut -f3 | tr ';' '\n' | sort | uniq -d > chr${chr}.dup.id

# BEGIN: add 4 duplicate markers to exclusion list
if [ ${chr} == "8" ]; then echo "88316919"; fi >> ${excl}
#if [ ${chr} == "12" ]; then echo ""; fi >> ${excl}
if [ ${chr} == "14" ]; then echo "21181798"; fi  >> ${excl}
if [ ${chr} == "17" ]; then echo "1241338"; fi  >> ${excl}
# END:  add 4 duplicate markers to exclusion list

zcat ${input} | grep -v 'SVTYPE' | ${fl} \# \-2 ${excl} | ${simplify} | ${rmids} chr${chr}.dup.id | bgzip -c > ${vcf}
tabix ${vcf}
done

## Same for chromosome X                                                                                                  
chr="X"
in="${src}ALL.chr${chr}_GRCh38.genotypes.20170504.vcf.gz"
vcf="chr${chr}.1kg.phase3.v5a_GRCh38.vcf.gz"
excl="chr${chr}.1kg.phase3.v5a_GRCh38.excl"

### sed command recodes male chrom X genotypes as homozygote diploid genotypes
### sed command makes temporary change to floating point numbers and diploid genotypes to permit use of word-boundary '\>'
cat_in=" zcat ${in} | grep -v 'SVTYPE' | sed \
-e 's/\t0\t\.\t/\t111\tPASS\t/' \
-e 's/0\tPASS/222\tPASS/' \
-e 's/\([0-9]\)\./\1WXYZ/g' \
-e 's/\([0-9]\)|\([0-9]\)/\1X\2/g' \
-e 's/\t\([0-9]\)\>/\t\1|\1/g' \
-e 's/\([0-9]\)WXYZ/\1./g' \
-e 's/\([0-9]\)X\([0-9]\)/\1|\2/g';"

echo ${chr}
  eval ${cat_in} | grep -v '^#' | cut -f3 | tr ';' '\n' | sort | uniq -d > chr${chr}.dup.id
  eval ${cat_in} | ${gts} | ${fl} '\#' -13 ${min_minor} 1000000 | cut -f2 > ${excl}

  # BEGIN: add duplicate markers to exclusion list
  if [ ${chr} == "X" ]; then echo "5457254"; fi >> ${excl}
  if [ ${chr} == "X" ]; then echo "32344545"; fi >> ${excl}
  if [ ${chr} == "X" ]; then echo "68984437"; fi >> ${excl}
  # END:  add duplicate markers to exclusion list

eval ${cat_in} | ${fl} \# \-2 ${excl} | ${simplify} | ${rmids} chr${chr}.dup.id | bgzip -c > ${vcf}
tabix ${vcf}
```

Run R code to generate loci, allele and gc_content files:

```
##########################################################################
## set working directory to where the vcf files are located
setwd("./")
##########################################################################
ffs <- dir(full=T,pattern="1kg.phase3.v5a_GRCh38.vcf.gz$")
MC.CORES <- 10 ## set number of cores to use
##########################################################################


##########################################################################
library(parallel)
##########################################################################
## Further remove unreferenced alleles and duplicates
##########################################################################
mclapply(ffs[length(ffs):1],function(x)
{
    cat(paste(x,"read file"))
    out  <- gsub(".vcf.gz","nounref.vcf.gz",x)
    cmd <- paste0("zcat ",
                  x,
                  " | grep -v '",
                  "\\",
                  ".",
                  "|","' ",
                  " | grep -v '",
                  "|",
                  "\\",
                  ".",
                  "' "
                 ,"|"," awk '!seen[$2]++' |  gzip > ",
                  out)
    system(cmd)
},mc.cores=MC.CORES)
##########################################################################

ffs <- dir(full=T,pattern="1kg.phase3.v5a_GRCh38.nounref.vcf.gz$")
## Generate Loci files
##########################################################################
mclapply(ffs[length(ffs):1],function(x)
{
    cat(paste(x,"read file"))
    out  <- gsub(".vcf.gz","_loci.txt",x)
    cmd <- paste0("zcat ",
                  x," | grep -v '^#' | awk -v OFS='\t' '{print $1, $2}' > ",
                  out)
    system(cmd)
},mc.cores=10)
##########################################################################
## with chr string (for BAMs that contain "chr")
mclapply(ffs[length(ffs):1],function(x)
{
    ##cat(paste(x,"read file"))
    out  <- gsub(".vcf.gz","_loci_chrstring.txt",x)
    cmd <- paste0("zcat ",
                  x," | grep -v '^#' | awk -v OFS='\t' '{print ",
                  '"chr"'
                 ,"$1, $2}' > ",
                  out)
    system(noquote(cmd))
},mc.cores=10)
##########################################################################
## Generate Allele Files
##########################################################################
mclapply(ffs[length(ffs):1],function(x)
{
    cat(paste(x,"read file"))
    out  <- gsub(".vcf.gz","_allele_letter.txt",x)
    cmd <- paste0("zcat ",
                  x," | grep -v '^#' | awk -v OFS='\t' '{print $2, $4, $5}' > ",
                  out)
    system(cmd)
},mc.cores=10)
##########################################################################
## Convert Alleles into Indices for BB input
indices <- c("A"=1,"C"=2,"G"=3,"T"=4)
##########################################################################
mclapply(ffs[length(ffs):1],function(x)
{
    cat(".")
    inp <- gsub(".vcf.gz","_allele_letter.txt",x)
    out <- gsub(".vcf.gz","_allele_index.txt",x)
    df <- as.data.frame(data.table::fread(inp))
    ref <- indices[df[,2]]
    alt <- indices[df[,3]]
    ndf <- data.frame(position=df[,1],
                      a0=ref,
                      a1=alt)
    write.table(ndf,file=out,
                row.names=F,col.names=T,sep="\t",quote=F)
},mc.cores=5)
##########################################################################


##########################################################################
## Symlink loci to change the names allowing for a "prefix" before chr in BB
##########################################################################
allfs <- dir(full=T)
allfs_loci <- allfs[grepl("loci.txt$",allfs)]
tnull <- lapply(allfs_loci,function(x)
{
    cmd <- paste0("ln -s ",x," ",gsub("chr(.*?)\\.(.*).txt","\\2_chr\\1.txt",x))
    system(cmd)
    if(grepl("chrX",x))
    {
        cmd <- paste0("ln -s ",x," ",gsub("chr(.*?)\\.(.*).txt","\\2_chr23.txt",x))
        system(cmd)
    }
})
##########################################################################
allfs <- dir(full=T)
allfs_loci <- allfs[grepl("loci_chrstring.txt$",allfs)]
tnull <- lapply(allfs_loci,function(x)
{
    cmd <- paste0("ln -s ",x," ",gsub("chr(.*?)\\.(.*).txt","\\2_chr\\1.txt",x))
    system(cmd)
    if(grepl("chrX",x))
    {
        cmd <- paste0("ln -s ",x," ",gsub("chr(.*?)\\.(.*).txt","\\2_chr23.txt",x))
        system(cmd)
    }
})
##########################################################################
## Symlink alleles: same as for loci, symlink to change name for the
use of prefixes
##########################################################################
allfs <- dir(full=T)
allfs_index <- allfs[grepl("allele_index",allfs)]
tnull <- lapply(allfs_index,function(x)
{
    cmd <- paste0("ln -s ",x," ",gsub("chr(.*?)\\.(.*).txt","\\2_chr\\1.txt",x))
    system(cmd)
    if(grepl("chrX",x))
    {
        cmd <- paste0("ln -s ",x," ",gsub("chr(.*?)\\.(.*).txt","\\2_chr23.txt",x))
        system(cmd)
    }
})
       
##########################################################################
##  Derive GC content files
##########################################################################
library(Rsamtools)
library(data.table)
library(Biostrings)
##########################################################################
gcTrack <- function(chr,
                    pos,
                    dna,
                    window=5000)
{
    gc <- rowSums(letterFrequencyInSlidingView(dna[[chr]],
                                               window,
                                               c("G","C")))/window
    gc[pos]
}
##########################################################################
FASTA <- "genome.fa" ## Link to genome reference fasta file 
CHRS <- paste0("", c(1:22, "X", "Y"))
BEAGLELOCI.template <- "chrCHROMNAME.1kg.phase3.v5a_GRCh38_loci.txt"
##########################################################################
REFGENOME <- getRefGenome(fasta = FASTA, CHRS = CHRS) ## Loads genome
in memory
##########################################################################
OUTDIR <- "1000genomes_2012_v3_gcContent_hg38"
system(paste0("mkdir ",OUTDIR))
##########################################################################

##########################################################################
windows <- c(25,
             50,
             100,
             200,
             500,
             1000,
             2000,
             5000,
             10000,
             20000,
             50000,
             100000,
             200000,
             500000,
             1000000,
             2000000,
             5000000,
             10000000)
names(windows) <- formatC(windows,format="f",digits=0)
names(windows) <- gsub("000000$","Mb",names(windows))
names(windows) <- gsub("000$","kb",names(windows))
names(windows) <- sapply(names(windows),function(x) if(grepl("[0-9]$",x)) paste0(x,"bp") else x)
##########################################################################

writeGC <- function(gccontent,chr,outdir)
{
    write.table(gccontent,
                file=gzfile(paste0(outdir,"/1000_genomes_GC_corr_chr_",chr,".txt.gz")),
                col.names=T,
                row.names=T,quote=F,sep="\t")
}

for(chr in CHRS)
{
    snppos <- as.data.frame(data.table::fread(gsub("CHROMNAME",chr,BEAGLELOCI.template)))
    gccontent <- sapply(windows,function(window) gcTrack(chr=chr,
                                                         pos=snppos[,2],
                                                         dna=REFGENOME,
                                                         window=window))*100
    gccontent <- cbind(rep(chr,nrow(gccontent)),snppos[,2],gccontent)
    colnames(gccontent)[1:2] <- c("chr","Position")
    rownames(gccontent) <- snppos[,2]
    writeGC(gccontent,chr,OUTDIR)
}
   
```


#### Run interactively

These commands run the Battenberg pipeline within a Docker container in interactive mode. This command assumes the data is available locally in `$PWD/data/pcawg/HCC1143_ds` and the reference files have been placed in `$PWD/battenberg_reference`

```
docker run -it -v `pwd`/data/pcawg/HCC1143_ds:/mnt/battenberg/ -v `pwd`/battenberg_reference:/opt/battenberg_reference battenberg:2.2.8
```

Within the Docker terminal run the pipeline, in this case on the ICGC PCAWG testing data available [here](https://s3-eu-west-1.amazonaws.com/wtsi-pancancer/testdata/HCC1143_ds.tar).

```
R CMD BATCH '--no-restore-data --no-save --args HCC1143 HCC1143_BL /mnt/battenberg/HCC1143_BL.bam /mnt/battenberg/HCC1143.bam FALSE /mnt/battenberg/' /usr/local/bin/battenberg_wgs.R /mnt/battenberg/battenberg.Rout
```
  
### Building a release

In RStudio: In the Build tab, click Check Package

Then open the NAMESPACE file and edit:

```
S3method(plot,haplotype.data)
```  

to:

```
export(plot.haplotype.data)
```
