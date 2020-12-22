# Geoduck TagSeq Bioinformatics

## <span style="color:blue">**Table of Contents**</span>
  - [Upon upload to HPC...](#Initial-diagnostics-upon-sequence-upload-to-HPC)
      - [Count raw reads](#Count-the-number-of-read-files)
      - [Digital fingerprint md5sums](#run-checksum) (HPC script)
  - [Initial quality check](#Quality-check-of-raw-reads) (HPC script)
  - [Trim and post-trim quality check](#Trimming-and-post-trim-quality-check-of-'clean'-reads) (HPC script)
  - [HISAT2: Alignment of cleaned reads to reference](#Alignment-of-cleaned-reads-to-reference)
	- [Upload reference genome](#Reference-genome-upload-to-HPC)
	- [About HISAT2](#HISAT2-alignment)
	- [About samtools](#samtools)
	- [Run index and alignment](#HPC-Job-HISAT2-Index-Reference-Alignment) (HPC script)
  - [StringTie: Assembly](#Assembly-quantification)  
	- [About StringTie](#StringTie)
## Initial diagnostics upon sequence upload to HPC
--------------------------------------------
### Count the number of read files
ls -1 | wc -l

should equal 141 seq samples *2lanes per sample + 1 md5 = 283

### run checksum

```
nano transfer_checks.sh
```
```
#!/bin/bash
#SBATCH -t 120:00:00
#SBATCH --nodes=1 --ntasks-per-node=10
#SBATCH --mem=500GB
#SBATCH --account=putnamlab
#SBATCH --export=NONE
#SBATCH -D /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/
#SBATCH -p putnamlab
#SBATCH --cpus-per-task=3

# load modules needed

# generate md5
md5sum /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/*.gz > /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/URI.md5

# Count the number of sequences per file

zcat /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/*fastq.gz | echo $((`wc -l`/4)) > /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/rawread.counts.txt
```

```
sbatch transfer_checks.sh
```

- check the digital fingerprint of the files with md5sum 
- compare md5sum of our output URI.md5 file to the UT Library Information pdf; okay the upload - move forward


## Quality check of raw reads
-------------------------------------------
### RUN FASTQC and MUTLIQC
```
nano fastqc_raw.sh
```

```
#!/bin/bash
#SBATCH -t 120:00:00
#SBATCH --nodes=1 --ntasks-per-node=10
#SBATCH --mem=500GB
#SBATCH --account=putnamlab
#SBATCH --export=NONE
#SBATCH -D /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/
#SBATCH -p putnamlab
#SBATCH --cpus-per-task=3

# load modules needed
module load all/FastQC/0.11.9-Java-11
module load MultiQC/1.9-intel-2020a-Python-3.8.2

for file in /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/*gz
do
fastqc $file
done

multiqc /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/

```

```
sbatch fastqc_raw.sh
```
- view the multiqc.html report 

## Trimming and post-trim quality check of 'clean' reads
-------------------------------------------
- notes from the previous raw data multiqc.html report:
	- *Per Sequence GC Content*: double peak of ~10% GC and 30-40% GC
	- *Mean Quality Scores*: > 30
	- *Per Base Sequence Content*: 15-25% C and G, 25-28% T, 35-40+% A
	- *Adapter Content*: high adapters present, 1-6% of sequences 
- trim adapter sequence
- output mutliqc report of the 'clean' reads
### RUN FASTP, FASTQC, and MUTLIQC 
```
#!/bin/bash
#SBATCH -t 120:00:00
#SBATCH --nodes=1 --ntasks-per-node=10
#SBATCH --mem=500GB
#SBATCH --account=putnamlab
#SBATCH --export=NONE
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=samuel_gurr@uri.edu
#SBATCH --output=../../../sgurr/Geoduck_TagSeq/output/clean/"%x_out.%j"
#SBATCH --error=../../../sgurr/Geoduck_TagSeq/output/clean/"%x_err.%j"
#SBATCH -D /data/putnamlab/KITT/hputnam/20201217_Geoduck_TagSeq/
#SBATCH -p putnamlab
#SBATCH --cpus-per-task=3

# load modules needed
module load fastp/0.19.7-foss-2018b
module load FastQC/0.11.8-Java-1.8
module load MultiQC/1.7-foss-2018b-Python-2.7.15

# Make an array of sequences to trim
array1=($(ls *.fastq.gz))

# fastp loop; trim the Read 1 TruSeq adapter sequence
for i in ${array1[@]}; do
	fastp --in1 ${i} --out1 ../../../sgurr/Geoduck_TagSeq/output/clean/clean.${i} --adapter_sequence=AGATCGGAAGAGCACACGTCTGAACTCCAGTCA
    fastqc ../../../sgurr/Geoduck_TagSeq/output/clean/clean.${i}
done

echo "Read trimming of adapters complete." $(date)

# Quality Assessment of Trimmed Reads

cd ../../../sgurr/Geoduck_TagSeq/output/clean #The following command will be run in the /clean directory

multiqc ./ #Compile MultiQC report from FastQC files

echo "Cleaned MultiQC report generated." $(date)
```

### EXPORT MUTLIQC REPORT 
*exit bluewaves and run from terminal*
- save to gitrepo as multiqc_clean.html
```
scp samuel_gurr@bluewaves.uri.edu:/data/putnamlab/sgurr/Geoduck_TagSeq/output/clean/multiqc_report.html  C:/Users/samjg/Documents/My_Projects/Geoduck_TagSeq
```

## Alignment of cleaned reads to reference
-------------------------------------------

### Reference genome upload to HPC 

*exit bluewaves and run from terminal*

```
scp C:/Users/samjg/Documents/Bioinformatics/genomes/Panopea-generosa-genes.fna samuel_gurr@bluewaves.uri.edu:/data/putnamlab/sgurr/refs/
```

-  file name: Panopea-generosa-genes.fna
	- Roberts, S., White, S., Trigg, S. A., and Putnam, H.M. (2020). Panopea_generosa_genome. OSF. June 24. doi:10.17605/OSF.IO/YEM8N.
	 [Roberts et al. 2020; link to genome](doi:10.17605/OSF.IO/YEM8N)
-  reference genome file size: 368MB
-  uploaded to sgurr/refs/

### HISAT2 alignment

**About:** HISAT2 is a sensitive alignment program for mapping sequencing reads.
In our case here, we will use HISAT2 to **(1)** index our *P. generosa* reference genome **(2)** align our clean TagSeq reads to the indexed reference genome. 

More information on HISAT2 can be read [here](http://daehwankimlab.github.io/hisat2/manual/)!

**Main arguments used below**: 

**(1)** *Index the reference genome*

``` hisat2-build ``` =
builds a HISAT2 index from a set of DNA sequences. Outputs 6 files that together consitute the index. 
ALL output files are needed to slign reads to the reference genome and the original sequence FASTA file(s) 
are no longer used for th HISAT2 alignment 

``` -f <reads.fasta> ``` =
the reads (i.e <m1>, <m2>, <m100>)
FASTA files usually have extension .fa, .fasta, .mfa, .fna or similar. 
FASTA files do not have a way of specifying quality values, so when -f is set, 
the result is as if --ignore-quals is also set

Note: other options for your reads are ```-q ``` for FASTQ files, 
 ```--qseq ``` for QSEQ  files, etc. - check [here](http://daehwankimlab.github.io/hisat2/manual/) for more file types
 
**(2)** *align reads to reference*

``` -x <hisat2-indx> ``` =
base name for the index reference genome, first looks in the current directory then in the directory specified in HISAT_INDEXES environment variable

``` --dta ``` =
 **important!** reports the alignments tailored for *StringTie* assembler.
 With this option, HISAT2 requires longer anchor lengths for de novo discovery of splice sites. 
 This leads to fewer alignments with short-anchors, which helps transcript assemblers improve 
 significantly in computation and memory usage. 
 
 This is important relative to Cufflinks ```--dta-cufflinks``` 
 in which HISAT2 looks for novel splice sites with three signals (GT/AG, GC/AG, AT/AC)

``` -U <r> ``` =
Comma-separated list of files contained unparied reads to be aligned. 
*In other words...*, our array of post-trimmed 'clean' TagSeq reads

``` -p NTHREADS``` =
Runs on separate processors, increasing -p increases HISAT2's memory footprint, increasing -p from 1 to 8 
increased the footprint by a few hundred megabytes 

Note: Erin Chile from Putnam Lab ran -p 8 for HISAT2 alignment of *Montipora* [here](https://github.com/echille/Montipora_OA_Development_Timeseries/blob/master/Amil/amil_RNAseq-analysis.sh)

``` -S <hit> ``` =
file to write SAM alignments to. By default, alignments are written to the “standard out” or “stdout” filehandle (i.e. the console).

**Note:** HISAT2 also has several criteria to trim such as the phred score and base count 5' (left) and 3' (right) 
Since we already assessed quality and trimmed, we will not use these commands


### samtools 
**About:** used to manipuate alignments in BAM format. quickly extracts alignments overlapping 
particular genomic regions - outputs allow viewers to quickly display alignments in each genomic region

``` sort ``` =
sorts the alignments by the leftmost coordinates

```-o <out.bam``` = 
outputs as a specified .bam file 
	 
```-@ threads``` = 
Set number of sorting and compression threads. By default, operation is single-threaded
	 
more information on samtools commands [here](http://www.htslib.org/doc/1.1/samtools.html)
	 
## HPC Job: HISAT2 Index Reference & Alignment 

- create directory output\hisat2  

``` mkdir hisat2 ```

- index reference and alignment 

**input**
- Panopea-generosa-genes.fna *= reference genome*
- clean/*.fastq.gz *= all clean TagSeq reads*

**ouput**
- Pgenerosa_ref *= indexed reference by hisat2-build; stored in the output/hisat2 folder*
- <clean.fasta>.sam *=hisat2 output, readable text file; removed at the end of the script*
- <clean.fasta>.bam *=converted binary file complementary to the hisat sam files*
```
#!/bin/bash
#SBATCH -t 120:00:00
#SBATCH --nodes=1 --ntasks-per-node=10
#SBATCH --mem=500GB
#SBATCH --account=putnamlab
#SBATCH --export=NONE
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=samuel_gurr@uri.edu
#SBATCH --output="%x_out.%j"
#SBATCH --error="%x_err.%j"
#SBATCH -D /data/putnamlab/sgurr/Geoduck_TagSeq/output/hisat2

#load packages
module load HISAT2/2.1.0-foss-2018b #Alignment to reference genome: HISAT2
module load SAMtools/1.9-foss-2018b #Preparation of alignment for assembly: SAMtools

# symbolically link 'clean' reads to hisat2 dir
ln -s ../clean/*.fastq.gz ./

# index the reference genome for Panopea generosa output index to working directory
hisat2-build -f ../../../refs/Panopea-generosa-genes.fna ./Pgenerosa_ref
echo "Referece genome indexed. Starting alingment" $(date)

# This script exports alignments as bam files
# sorts the bam file because Stringtie takes a sorted file for input (--dta)
# removes the sam file because it is no longer needed
array=($(ls *.fastq.gz)) # call the symbolically linked sequences - make an array to align
for i in ${array[@]}; do
        hisat2 --dta -x Pgenerosa_ref -U ${i} -S ${i}.sam
        samtools sort -@ 8 -o ${i}.bam ${i}.sam
                echo "${i} bam-ified!"
        rm ${i}.sam
done
```
- HISAT2 complete with format prepared for StringTie assembler!

## Assembly & quantification

### StringTie

**About:** StringTie is a fast and efficient assembler of RNA-Seq alignments to assemble and quantitate full-length transcripts.  putative transcripts. 
For our use, we will input short mapped reads from HISAT2. StringTie's ouput can be used to identify DEGs in programs such as DESeq2 and edgeR

More information on StringTie can be read [here](https://ccb.jhu.edu/software/stringtie/)!

[Cufflinks](http://cole-trapnell-lab.github.io/cufflinks/) is another assembler option we will not use. [Read this](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4643835/#:~:text=On%20a%20simulated%20data%20set,other%20assembly%20software%2C%20including%20Cufflinks.)

