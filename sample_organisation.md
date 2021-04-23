### Organise files and quality control

We start in the right folder

```bash
cd /nesi/nobackup/uoo00116/mitra_RNASeq/
```

Get all the CB files together

```bash
cd source_files
mkdir  CB/
cp X201SC21011063-Z01-F001/raw_data/CB*/*  CB/
#small issue, one sample is named Cs. we copy it and rename it
cp X201SC21011063-Z01-F001/raw_data/Cs*/*  CS/
mv Cs_FW_05_Taieri_1.fq.gz CS_FW_05_Taieri_1.fq.gz
mv Cs_FW_05_Taieri_2.fq.gz CS_FW_05_Taieri_1.fq.gz

```

quality control:

```bash
#module load FastQC MultiQC
cd CB
mkdir qc
fastqc --threads 16 *.gz -o qc/ 
cd qc 
multiqc .
#cp multiqc_report.html ~/repos/scripts/bully_RNA/
```

[multiqc_report.html](multiqc_report.html)

