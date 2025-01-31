
# RNAseq Data QC, Mapping, and Quantification Pipeline

### Overview

This pipeline takes advantage of a genome mapper STAR, which performs transcript alignment by mapping to a reference genome. Importantly, STAR is suited for the de novo discovery of splice junctions, which can be leveraged for identifying novel exons and isoforms. This pipeline couples the STAR mapper with RSEM for transcript quantification. This approach attempts to probabilistically estimate transcript abundance rather than simply count the reads.This may be beneficial for improving transcript count estimates, by probabilistically resolving reads which map to multiple genes (multimappers).  

## Table of Contents

1. [Data](#data)
2. [Brief Description and Literature on Required Tools and Scripts](#description)
3. [Step 1 - Trimming, adapter removal, and QC](#one)
4. [Step 2 - Creating STAR index](#two)
5. [Step 3 - Mapping with STAR](#three)
6. [Step 4 - Running RSEM](#four)
7. [Step 5 - Filtering, Creating DGEList Object, and Normalization (with limma-voom)](#five)
8. [Step 6 - Clustering gene expression data with WGNCA, and correlating phenotypic and environmental variables with gene clusters](#six)  

---

## Data <a name="data"></a>

* [**Link to data**](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/tree/master/data/RNAseq)
* Reference genome: from NCBI ([GCA_002022765.4 C_virginica-3.0](https://www.ncbi.nlm.nih.gov/genome/?term=crassostrea+virginica))

## Brief Description and Literature on Required Tools and primary R packages <a name="description"></a>

**Trimming and Quality Control**

*dDocent (wrapper for trimming and QC steps)* -  dDocent pipeline uses Trimmomatic trimming tool to remove adapter and low quality sequences from the ends of reads. Within the dDocent code, it is specified to be paired-end (which is automatically recognized based on our file naming scheme), removes adapters based on thresholds for how well the adapter sequences align to reads (2:30:10; see Trimmomatic manual for more details), removes leading bases with phred quality score less than 20, removes trailing bases with phred quality score less than 20, scans the reads at a 5bp window and cuts when the average quality of the five bases is less than 10, and makes sure all reads are a minimum length after this cutting (greater than the shortest read/2).

* [Website](https://www.ddocent.com/)
* [Publication](https://peerj.com/articles/431/)

*Trimmomatic* - A flexible read trimming tool for Illumina NGS data. Used by dDocent to trim raw RNAseq fragments and remove adapters.

* [Github](https://github.com/timflutre/trimmomatic)
* [Publication](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4103590/)

**File Conversion**

*gffread* - Program to add in the conversion between different gene annotation file structures. Used here to convert from the `.gff` file format provided by NCBI to `.gtf` (preferred by STAR mapper).

* [Github](https://github.com/gpertea/gffread)

**Mapping**

*STAR* - Fast RNA-seq aligner than can make to a reference genome and identify identify canonical as well as novel splice junctions. It will output mapped reads as `.sam` or `.bam` files, and with the `--quantMode` it can also create a tab delimited read count output (similar to HT-Seq). In addition, mapped reads can be ouputed as a `.bam` file with transcript coordinates. This can be used downstream by the transcript quantification program RSEM. 

* [Github](https://github.com/alexdobin/STAR)  
* [Publication](https://academic.oup.com/bioinformatics/article/29/1/15/272537)

**Transcript Quantification**

*RSEM* - Transcript quantifier, that can estimate counts at either the transcript (isoform) or gene level. It has a direct workflow with `STAR`, which enables a single line command for both mapping and quantification. Alternatively, it can take`STAR` outputs (specifically `.bam` files with transcript coordinates), and then perform the estimation.

* [Github](https://deweylab.github.io/RSEM/)
* [Publication](https://bmcbioinformatics.biomedcentral.com/articles/10.1186/1471-2105-12-323)

**Gene filtering, standardization, and normalization**

*edgeR* - R package used for analyzing transcriptomic sequence data. Primarily using it to create a `DGEList` object type in R which will be used by `limma` package functions downstream. Also using it for the `cpm` function which converts within sample counts into a `count per million (cpm)`. 

* [Manual](https://www.bioconductor.org/packages/release/bioc/vignettes/edgeR/inst/doc/edgeRUsersGuide.pdf)

*limma* - R package used for analyzing transcriptomic sequence data. Here we are using limma for `TMMwsp` standardization approach to account for variable library sizes among samples, to transform our counts into `log2-cpm` using the `voom` function, account for random tank effects. Also used to perform differential expression analysis with planned contrasts in RNAseq_Analysis workflow.

* [Manual](https://www.bioconductor.org/packages/release/bioc/vignettes/limma/inst/doc/usersguide.pdf)

---

## Step 1 - Trimming and Adapter Removal <a name = "one"></a>

### Overview
Step performs trimming and quality control step implemented in the program dDocent, which uses trimmomatic to perform the trimming step.

Details [here](https://www.ddocent.com/UserGuide/)

**Input**
* Files from [BioProject](https://www.ncbi.nlm.nih.gov/bioproject/594029)

Command Line:
```
./dDocent RNA.config
```
---

## Step 2 - Creating STAR index <a name="two"></a>

### Overview 
Create a index for STAR mapping. This is done using a reference genome and gene annotations, which in this case were taken from NCBI. STAR prefers using a `gtf` rather than the `gff` gene annotation file format provided by NCBI, so we also provide a step for converting between file formats using the program `gffread`. 

**Additional thoughts and performance**
* This step only needs to be done once, unless the genome or gene annotations have been updated.
* Indexing should be relatively quick on a cluster (<10min)

**Inputs**
* [Reference genome and gene annotations : GCA_002022765.4 C_virginica-3.0](https://www.ncbi.nlm.nih.gov/genome/?term=crassostrea+virginica))
* [reference genome files used in this pipeline](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/references/compressed/GCF_002022765.2_C_virginica-3.0_rna_from_genomic.tar.xz)
* [gff Annotation file used in this pipeline](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/references/compressed/gff_002022765.2_CV_3.0.tar.xz)

**Output**
* [gtf file created by conversion step](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/references/compressed/gff_002022765.2_CV_3.0.tar.xz)

### Step 2.1 - File conversion

Command line code for converting from `.gff` to `.gtf`:
```
gffread my.gff -T -o my.gtf
```
* `-T` flag needed for conversion from `.gff` to `.gtf` format.

### Step 2.2 - Create STAR index using oyster gene annotation file

* Script : [`STAR_genomeCreate.sh`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/02_STAR_genomeCreate.sh)

Command line code:
```
downey-wall.a@comp5[references]# STAR_genomeCreate.sh 
/shared_lab/20180226_RNAseq_2017OAExp/RNA/scripts/STAR_scripts/STAR_genomeCreate.sh: line 1: !#/bin/bash: No such file or directory
Please put in the base directory:
/shared_lab/20180226_RNAseq_2017OAExp/RNA/references/
Please put in the output folder name:
star_ref2
Outputs saving to :  /shared_lab/20180226_RNAseq_2017OAExp/RNA/references/star_ref2
Directory Created
Select genome file (.fna format, should include entire path)
/shared_lab/20180226_RNAseq_2017OAExp/RNA/references/genome/GCF_002022765.2_C_virginica-3.0_genomic.fna
Select gene annotation file (.gtf, should includ entire path)
/shared_lab/20180226_RNAseq_2017OAExp/RNA/references/gene_annotation/KM_CV_genome.gtf 
```

Output code if run successfully:
```
Jul 15 12:23:09 ..... started STAR run
Jul 15 12:23:09 ... starting to generate Genome files
Jul 15 12:23:22 ... starting to sort Suffix Array. This may take a long time...
Jul 15 12:23:25 ... sorting Suffix Array chunks and saving them to disk...
Jul 15 12:24:53 ... loading chunks from disk, packing SA...
Jul 15 12:25:29 ... finished generating suffix array
Jul 15 12:25:29 ... generating Suffix Array index
Jul 15 12:27:55 ... completed Suffix Array index
Jul 15 12:27:55 ..... processing annotations GTF
Jul 15 12:28:06 ..... inserting junctions into the genome indices
Jul 15 12:29:46 ... writing Genome to disk ...
Jul 15 12:29:48 ... writing Suffix Array to disk ...
Jul 15 12:30:07 ... writing SAindex to disk
Jul 15 12:30:12 ..... finished successfully
```

--- 

## Step 3 - Mapping with STAR <a name="three"></a>

### Overview
Map samples to reference genome with two-step STAR mapping protocol. 

**Additional Thoughts and Performance**
* This will likely take a long time and require extensive RAM (>30GB), so will likely need to be done on a computing cluster.
* It would be a good idea to create a dettachable session via tmux as each sample takes ~1-2 hours to process.

**Input**: 

* Post trimming and QC reads reads (trimmed and QCed)
    * Files stored as `.fq.gz` format
    * Forward Read Example : `P1_17005.R1.fq.gz`
    * Reverse Read Example : `P1_17005.R2.fq.gz`
* Reference index Folder (from previous step)

### **Step 3.1** : Start STAR mapping 1st Pass

* Full Script : [`STAR_1Pass_all.sh`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/03A_STAR_1Pass.sh)

Command Line:
```
downey-wall.a@comp5[references]# ./STAR_1Pass_all.sh 
Please put in raw file directory:
/pathway/to/trimmedRNASeqFiles
Please put in name of new folder for output
NAME_outputFile
```

### **Step 3.3** :  Move output files from 1st pass and Create `m3` folder for 2nd Pass

Command Line:
```
cd /pathway/to/output/folder
mkdir m2
mkdir m3
mv *m2_* m2 
```

### **Step 3.4** : Start STAR mapping 2nd Pass

* Full Script : [`STAR_2Pass_all.sh`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/03B_STAR_2Pass.sh)

Command Line:
```
downey-wall.a@comp5[references]# STAR_2Pass_all.sh
Please put in raw file directory:
/shared_lab/20180226_RNAseq_2017OAExp/RNA/rawfiles
```

**STAR command**

Core function STAR 1st pass:
```
/shared_lab/scripts/STAR --runThreadN 10 \
--genomeDir /path/toStarReferenceIndex \
--outFilterMatchNminOverLread 0.17 --outFilterScoreMinOverLread 0.17 \
--readFilesIn/path/toForwardStrand /path/toReverseStrand \
--outSAMmapqUnique 40 \
--outSAMtype BAM Unsorted SortedByCoordinate \
--outFileNamePrefix /path/toOutput \
--readFilesCommand zcat
```

Core function STAR 2nd pass:
```
/shared_lab/scripts/STAR --runThreadN 19 \
--genomeDir /path/toStarReferenceIndex \
--readFilesIn /path/toForwardStrand /path/toReverseStrand \
--outSAMmapqUnique 40 \
--outSAMtype BAM Unsorted SortedByCoordinate \
--quantMode TranscriptomeSAM GeneCounts --limitSjdbInsertNsj 1500000 \
--outFileNamePrefix /path/toOutput \
--readFilesCommand zcat \
--sjdbFileChrStartEnd /path/toSpliceJunctionFolder
```

## Step 4 - Running RSEM  <a name="four"></a>

### Overview
Performed transcript quantification with RSEM.

**Additional Thoughts and Performance**
* This will likely also take a long time and require extensive RAM (>30GB), so will likely need to be done on a computing cluster.
* It would be a good idea to create a dettachable session via tmux as each sample takes ~3 hours to process.

**Creating Index folder for RSEM**

* Full Script : [`RSEM_createRefFromStar.sh`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/04A_RSEM_createRefFromStar.sh)

Core function `rsem-prepare-reference`: 
```
rsem-prepare-reference \
--gtf /path/toGeneAnnotation.gtf \
--star \
-p 8 \
/path/toRefGenome_GCF_002022765.2_C_virginica-3.0_genomic.fna \
/path/toOuput
```

**Performing RSEM Transcript Quantification**

* Full Script : [`RSEM_calcExp.sh`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/04B_RSEM_calcExp.sh)

Core function `rsem-calculate-expression`:

```
rsem-calculate-expression --star --paired-end \
--star-gzipped-read-file \
-p 20 \
/path/toForwardStrand \
/path/toReverseStrand \
/path/toRSEM_reference \
/path/toOutputFolder
```
**Combine Sample Transcript Files into Single Matrix**

** Full Script : [`/01B_RSEM_countMatrix.R`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/01B_RSEM_countMatrix.R)

**Outputs**

* [`/RSEM_outputs`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/tree/master/data/RNAseq/RSEM_output) : Folder of sample RSEM transcript quantification outputs.
* [`/RSEM_gene_Summary.Rdata`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/RNAseq/RSEM_gene_Summary.Rdata) : RSEM transcript quantification `RData` file (all counts).
* [`/RSEM_gene_EstCount.csv`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/RNAseq/RSEM_gene_EstCount.csv) : Estimated counts from RSEM
* [`/RSEM_gene_FPKM.csv`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/RNAseq/RSEM_gene_FPKM.csv) : FPKM counts from RSEM
* [`/RSEM_gene_TPM.csv`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/RNAseq/RSEM_gene_TPM.csv) : TPM counts from RSEM

## Step 5 - Filtering, Creating DGEList Object, and Normalization (with limma-voom) <a name="five"></a>

### Overview  

Takes a raw RSEM count estimation matrix and filters out genes that have low coverage (<1 cpm in at least 5 individuals in at least one trt/time combination), and performs normalization and transformation steps using `EdgeR` and `limma` packages.

* Full R Script: [`05_filtering_CreatingDGEListObj.R`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/05_filtering_CreatingDGEListObj.R)

**Inputs**
* [`/RSEM_gene_Summary.Rdata`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/RNAseq/RSEM_gene_Summary.Rdata) : RSEM transcript quantification `RData` file (all counts).
* [`STAR_gnomon_tximportGeneFile.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/references/STAR_gnomon_tximportGeneFile.RData) : Transcript annotation file.
* [`/AE17_RNAmetaData.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/meta/AE17_RNAmetaData.RData) : Sequenced sample meta data file.

**Outputs**
* [`/RNA_gene_preNormalization_DGEListObj.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/results/RNA/RNA_gene_preNormalization_DGEListObj.RData) : DGEListObj of gene expression data prior to normalization with `limma`.
* [`/RNA_gene_postVoomAndNormalization_DGEListObj.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/results/RNA/RNA_gene_postVoomAndNormalization_DGEListObj.RData) : DGEListObj of gene expression data post-normalization with `limma`.

## Step 6 - Clustering gene expression data for WGNCA <a name="six"></a>

### Overview
  
Step clusters co-expressed genes and generate WGCNA objects which are used for downstream weighted co-gene expression network analysis. Used to create figure 7.

* Full R Script: [`06_CreatingWGCNAObj.R`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/src/RNA_seq/06_CreatingWGCNAObj.R)

**Input**
* [`/RNA_gene_postVoomAndNormalization_DGEListObj.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/results/RNA/RNA_gene_postVoomAndNormalization_DGEListObj.RData) : DGEListObj of gene expression data post-normalization with `limma`.
* [`/AE17_RNAmetaData.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/data/meta/AE17_RNAmetaData.RData) : Sequenced sample meta data file.

**Output**
* [`/RNA_Limma_Expression_Data_forWGCNA.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/results/RNA/RNA_Limma_Expression_Data_forWGCNA.RData) 
* [`/RNA_Limma_networkConstruction_WGCNA.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/results/RNA/RNA_Limma_networkConstruction_WGCNA.RData)
* [`/RNA_Limma_WGCNA_ModuleMembership.RData`](https://github.com/epigeneticstoocean/AE17_Cvirginica_MolecularResponse/blob/master/results/RNA/RNA_Limma_WGCNA_ModuleMembership.RData)
