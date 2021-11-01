### go enchrichment analysis


denovoassembly_annotation_report_bully.xls is the original trinotate output from  [annotate.md](annotate.md)

https://github.com/trinityrnaseq/trinityrnaseq/wiki/Running-GOSeq contains the methodology for the go enrichment analysis



### Extract go terms

Process the annotation report to get a GO table

```
  cd annotate
  module load Miniconda3
  source activate transdecoder
  extract_GO_assignments_from_Trinotate_xls.pl --gene \
                           --Trinotate_xls  denovoassembly_annotation_report_bully.xls \
                           -G --include_ancestral_terms \
                           > go_annotations.txt
````

## Extract GENE lengths files

extract gene lengths to be able to account for gene length in GOseq annotation.

```
/opt/nesi/CS400_centos7_bdw/Trinity/2.8.5-gimkl-2018b/trinityrnaseq-Trinity-v2.8.5/util/misc/fasta_seq_length.pl /Trinity.fasta > Trinity.fasta.seq_lens
```


isoforms.TMM.EXPR.matrix  is then created using counts generated using salmon and the script below. 


```
module load Salmon/0.14.0-gimkl-2018b # version matters!
module load R # EdgeR need to be installed!
/opt/nesi/CS400_centos7_bdw/Trinity/2.8.5-gimkl-2018b/trinityrnaseq-Trinity-v2.8.5/util/align_and_estimate_abundance.pl \
--transcripts FR_trinity_output/Trinity.fasta \
--seqType fq \
--samples_file samples_file.txt \
--est_method salmon \
--trinity_mode \
--prep_reference \
 > salmon_align_and_estimate_abundance.log 2>&1 &


/opt/nesi/CS400_centos7_bdw/Trinity/2.8.5-gimkl-2018b/trinityrnaseq-Trinity-v2.8.5/util/abundance_estimates_to_matrix.pl \
--est_method salmon \
--out_prefix Trinity_trans \
--name_sample_by_basedir \
--gene_trans_map none \
CB_FW_01_Hayes/quant.sf \
CB_FW_01_Otokia/quant.sf \
CB_FW_01_Waiko/quant.sf \
CB_FW_01_Wanaka/quant.sf \
CB_FW_02_Hayes/quant.sf \
CB_FW_02_Otokia/quant.sf \
CB_FW_03_Hayes/quant.sf \
CB_FW_03_Otokia/quant.sf \
CB_FW_03_Waiko/quant.sf \
CB_FW_03_Wanaka/quant.sf \
CB_FW_04_Otokia/quant.sf \
CB_FW_04_Waiko/quant.sf \
CB_FW_04_Wanaka/quant.sf \
CB_FW_05_Hayes/quant.sf \
CB_FW_05_Waiko/quant.sf \
CB_FW_05_Wanaka/quant.sf \
CB_SW_01_Hayes/quant.sf \
CB_SW_01_Otokia/quant.sf \
CB_SW_01_Waiko/quant.sf \
CB_SW_01_Wanaka/quant.sf \
CB_SW_02_Hayes/quant.sf \
CB_SW_02_Otokia/quant.sf \
CB_SW_02_Waiko/quant.sf \
CB_SW_03_Hayes/quant.sf \
CB_SW_03_Otokia/quant.sf \
CB_SW_03_Waiko/quant.sf \
CB_SW_03_Wanaka/quant.sf \
CB_SW_04_Wanaka/quant.sf \
CB_SW_05_Hayes/quant.sf \
CB_SW_05_Otokia/quant.sf \
CB_SW_05_Waiko/quant.sf \
CB_SW_05_Wanaka/quant.sf 
```

Finish the gene lengths:

```
/opt/nesi/CS400_centos7_bdw/Trinity/2.8.5-gimkl-2018b/trinityrnaseq-Trinity-v2.8.5/util/misc/TPM_weighted_gene_length.py  \
         --gene_trans_map Trinity.fasta.gene_trans_map \
         --trans_lengths Trinity.fasta.seq_lens \
         --TPM_matrix Trinity_trans.isoform.TMM.EXPR.matrix >Trinity.gene_lengths.txt
```



## Install GOSeq dependencies ( not covered in tutorial)


 I installed goseq using conda inside the transdecoder conda environment:

```bash
module load Miniconda3
conda create -n env_goseq #create the environment for goseq
conda activate goseq
conda install -c bioconda bioconductor-goseq 

 module load R/3.4.2-gimkl-2017a
```

Then I install qvalue inside R using:

```r
source("http://bioconductor.org/biocLite.R")
biocLite("qvalue") # that on
biocLite("goseq")  
```

## Run the GO analysis:
(done by Mitra)
https://github.com/trinityrnaseq/trinityrnaseq/wiki/Running-GOSeq

```bash
/opt/nesi/CS400_centos7_bdw/Trinity/2.8.5-gimkl-2018b/trinityrnaseq-Trinity-v2.8.5/Analysis/DifferentialExpression/run_GOseq.pl \
                       --factor_labeling  factor_labeling.txt \
                       --GO_assignments go_annotations.txt \
                       --lengths Trinity.gene_lengths.txt \
                       --background  backgroundGO.txt
mv diff.GOseq.depleted  allDE.GOseq.depleted 
mv diff.GOseq.enriched allDE.GOseq.enriched                 
```




```

