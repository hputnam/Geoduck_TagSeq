# Geoduck TagSeq Bioinformatics

## Count the number of files
ls -1 | wc -l

should equal 141 seq samples *2lanes per sample + 1 md5 = 283

## Run Checksum

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


## fastqc
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