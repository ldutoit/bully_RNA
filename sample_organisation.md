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

The data is good with 20-26mio paired-end 150bp reads per samples.
