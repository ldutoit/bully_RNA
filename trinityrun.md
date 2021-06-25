# Trinity run

Trinity is run on the mahuika server on the nobackup partition nobackup following nesi instructions https://support.nesi.org.nz/hc/en-gb/articles/360000980375-Trinity


```
cd /nesi/nobackup/uoo00116/mitra_RNASeq/
```


After checking the quality of the data within [sample_organisation.md](sample_organisation.md)

I learn that the data is very good with 20-26Mio paired end reads per sample, 2x150bp. But we need to remove rare adapters.

```
#/nesi/nobackup/uoo00116/mitra_RNASeq/source_files/CB
mkdir trimmed
for forward in *1.fq.gz
  do
  echo  $forward
  basename=$(basename $forward 1.fq.gz)
  reverse=${basename}2.fq.gz
  echo $reverse
  cutadapt  -a AGATCGGAAGAG  -q 20 -m 50    -A AGATCGGAAGAG -o trimmed/${basename}1.fq  -p trimmed/${basename}2.fq  -j 8 $forward  $reverse
done
```



I create a [samples_file.txt](samples_file.txt) modifying the [metadata.txt](metadata.txt) modifying the file.

I then run  Trinity_phase1

```
#!/bin/bash -e
#SBATCH --job-name=trinity-phase1_RF
#SBATCH --account=uoo00116   # your NeSI project code
#SBATCH --time=2-00:00:00       # maximum run time
#SBATCH --ntasks=1            # always 1
#SBATCH --cpus-per-task=16    # number of threads to use for Trinity
#SBATCH --mem=220G            # maximum memory available to Trinity
#SBATCH --partition=bigmem    # based on memory requirements
#SBATCH --hint=nomultithread  # disable hyper-threading

# load a Trinity module
module load Trinity/2.8.5-gimkl-2018b

# run trinity, stop before phase 2
srun Trinity --no_distributed_trinity_exec \
  --CPU ${SLURM_CPUS_PER_TASK} --max_memory 200G --seqType fq \
  --samples_file samples_file.txt --SS_lib_type RF   --output RF_trinity_output
```


And phase2:
```
#!/bin/bash -e
#SBATCH --job-name=trinity-phase2grid
#SBATCH --account=uoo00116  # your NeSI project code
#SBATCH --time=30:00:00      # enough time for all sub-jobs to complete
#SBATCH --ntasks=1           # always 1 - this is the master process
#SBATCH --cpus-per-task=1    # always 1
#SBATCH --mem=20G            # memory requirements for master process
#SBATCH --partition=hugemem   # submit to an appropriate partition
#SBATCH --hint=nomultithread

# load Trinity and HPC GridRunner
module load Trinity/2.8.5-gimkl-2018b
module load HpcGridRunner/20181005

# run Trinity - this will be the master HPC GridRunner process that handles
#   submitting sub-jobs (batches of commands) to the Slurm queue
srun Trinity --CPU ${SLURM_CPUS_PER_TASK} --max_memory 20G \
  --grid_exec "hpc_cmds_GridRunner.pl --grid_conf ${SLURM_SUBMIT_DIR}/SLURM.conf -c" \
   --seqType fq --samples_file samples_file.txt --SS_lib_type RF   --output RF_trinity_output
```


get some stats

```
module load Trinity
 /opt/nesi/CS400_centos7_bdw/Trinity/2.11.0-gimkl-2020a/trinityrnaseq-v2.11.0/util/TrinityStats.pl Trinity.fasta > Trinity.stats
```
  
### Obtaining iscount matrix


The next step is to obtain a count matrix
I used the tutorial at:
https://southgreenplatform.github.io/trainings/trinityTrinotate/TP-trinity/

The combination of modules and environment below is extremely weird, but it works.



```
module purge
module load Trinity/2.8.4-gimkl-2017a
module load Miniconda3
source activate transdecoder #modu that contains rm #rsem and other of my trinity utils
module load  Trinity/2.8.4-gimkl-2017a SAMtools/1.8-gimkl-2017a Bowtie/1.2.0-gimkl-2017a RSEM/1.3.1-gimkl-2017a
```

Then the alignment could be run at once but it is super slow so we split the samples_files.txt into one file per sample and run the job:
```sh
mkdir jobs/
while read p; do
  echo "$p"
  sample_name=$(echo $p | cut -f 2 -d " ")
  echo $p > jobs/${sample_name}".txt"
  #create sample file
  echo '#!/bin/sh' > jobs/job_${sample_name}.sh
  echo "align_and_estimate_abundance.pl --transcripts RF_trinity_output/Trinity.fasta --seqType fq --samples_file jobs/"${sample_name}".txt --est_method RSEM --aln_method bowtie2 --trinity_mode --prep_reference --thread_count 16 --coordsort_bam > " ${sample_name}".log" >> jobs/job_${sample_name}.sh
  sbatch -A uoo00116 -t 2-00:00 -c 16  jobs/job_${sample_name}.sh
done < samples_file.txt #samples_files_minusran.txt # 
```

We obtain gene counts organised in one folder per sample. We need to group that to make one matrix

```bash

find -name RSEM.genes.results > quant_gene_files.txt # path to all the gene files

mkdir RSEM_results
cd RSEM_results
```     

```python
###python
import os
with open("../quant_gene_files.txt") as f:
  for line in f:
    os.system("ln -s ."+ line.strip() +" "+ line.split("/")[1]+".txt") # that extra . after ln -s is because the file starts with ./
```

In order to create a singleclean matrix we go into R and use the tximport library. For some reason I am struggling a bit to do it on NeSI so I do that locally.

```R
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install("tximport")

```
from within ```gene_counts/RSEM_results```

```R
library("tximport")
files <- dir()
txi.rsem <- tximport(files, type = "rsem", txIn = FALSE, txOut = FALSE)
head(txi.rsem$counts)
colnames(txi.rsem$counts) <- gsub(".txt","",files)
head(txi.rsem$counts)
#I checked manually that the colnames match the counts in the single files

write.table(txi.rsem$counts,"RSEM_gene_counts.txt",row.names=T,col.names=T,sep="\t")
```


We can use the expected counts into DeSEQ2 as outlined by Love (DESeq2 author) [here](https://support.bioconductor.org/p/90672/)

I saved the count matrix in this repository [results_files/RSEM_gene_counts.txt](results_files/RSEM_gene_counts.txt)


