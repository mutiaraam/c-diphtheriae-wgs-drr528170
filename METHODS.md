# Methods
 
All analyses were performed on publicly available sequencing data obtained from the NCBI Sequence Read Archive. The complete pipeline comprised six sequential stages: data acquisition, quality control, genome alignment, BAM processing, variant calling, and functional annotation.
 
---
 
## 1. Data Acquisition
 
Paired-end Illumina MiSeq reads for *Corynebacterium diphtheriae* isolate DRR528170 were retrieved from NCBI SRA under BioProject [PRJDB12216](https://www.ncbi.nlm.nih.gov/bioproject/PRJDB12216) and extracted into paired FASTQ files using fastq-dump.
 
```bash
fastq-dump --split-files DRR528170/DRR528170.sra
```
 
The reference genome *C. diphtheriae* NCTC13129 (NC_002935.2, GCF_000195815.1) was downloaded from NCBI RefSeq, decompressed, and renamed for convenience.
 
```bash
# Download from NCBI RefSeq FTP
# https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/195/815/GCF_000195815.1_ASM19581v1/GCF_000195815.1_ASM19581v1_genomic.fna.gz
 
gunzip GCF_000195815.1_ASM19581v1_genomic.fna.gz
mv GCF_000195815.1_ASM19581v1_genomic.fna cdif_ref.fna
```
 
---
 
## 2. Quality Control
 
Raw reads were assessed using FastQC v0.11.9 to evaluate per-base quality scores, GC content distribution, sequence length distribution, and adapter contamination. Quality reports were generated for both forward and reverse reads independently.
 
```bash
fastqc DRR528170_1.fastq -o .
fastqc DRR528170_2.fastq -o .
```
 
Adapter sequences were automatically detected and removed using fastp v0.23 with the `--detect_adapter_for_pe` flag, which is optimized for paired-end library preparation. Default quality thresholds were applied. Post-trimming quality was verified with a second FastQC run.
 
```bash
fastp -i DRR528170_1.fastq -I DRR528170_2.fastq \
  -o data_clean_1.fastq -O data_clean_2.fastq \
  --detect_adapter_for_pe \
  --html fastp_report.html \
  --json fastp_report.json
 
fastqc data_clean_1.fastq -o .
fastqc data_clean_2.fastq -o .
```
 
---
 
## 3. Genome Alignment
 
Trimmed reads were aligned to the NCTC13129 reference genome using BWA MEM v0.7.17 with default parameters. BWA MEM is optimized for reads ≥70 bp and handles paired-end data natively.
 
```bash
bwa mem cdif_ref.fna data_clean_1.fastq data_clean_2.fastq > align_cdif.sam
```
 
Alignment statistics were assessed using `samtools flagstat` and `samtools coverage`.
 
```bash
samtools flagstat aligncdif_sorted.bam
samtools coverage aligncdif_sorted.bam
samtools depth aligncdif_sorted.bam | head -n 10
```
 
---
 
## 4. SAM/BAM Processing
 
The SAM output was converted to binary BAM format, coordinate-sorted, and integrity-checked using SAMtools v1.15. Intermediate files were removed after each step to conserve storage. The final sorted BAM was indexed to enable random access during variant calling.
 
```bash
# Convert SAM to BAM
samtools view -Sb align_cdif.sam > align_cdif.bam
 
# Sort by coordinate
samtools sort align_cdif.bam -o aligncdif_sorted.bam
 
# Verify integrity
samtools quickcheck aligncdif_sorted.bam && echo "OK"
 
# Index the sorted BAM
samtools index aligncdif_sorted.bam
 
# Remove intermediate files
rm align_cdif.sam
rm align_cdif.bam
```
 
---
 
## 5. Variant Calling
 
Variants were called using FreeBayes v1.3.6, a haplotype-based Bayesian variant detector. Parameters were set to ensure high-confidence calls: minimum mapping quality 20, minimum base quality 20, and minimum coverage depth 10×.
 
```bash
freebayes \
  -f cdif_ref.fna \
  --min-mapping-quality 20 \
  --min-base-quality 20 \
  --min-coverage 10 \
  aligncdif_sorted.bam > variantscdif_raw.vcf
```
 
Raw variants were filtered by quality (QUAL ≥30), depth (DP ≥10), and allele frequency (AF ≥0.1) using bcftools v1.15. Filtered variants were then normalized — left-aligning indels and decomposing multi-allelic sites — to ensure consistency before annotation.
 
```bash
# Filter by quality, depth, and allele frequency
bcftools filter \
  -e 'QUAL < 30 || INFO/DP < 10 || AF < 0.1' \
  variantscdif_raw.vcf -o varicdif_filtered.vcf
 
# Normalize indels
bcftools norm -f cdif_ref.fna -m- varicdif_filtered.vcf -o varcdif_norm.vcf
```
 
The chromosome identifier in the VCF was renamed to match the SnpEff database expectation.
 
```bash
sed 's/Chromosome/NC_002935.2/g' varcdif_norm.vcf > varcdif_norm_renamed.vcf
```
 
---
 
## 6. Functional Annotation
 
Variants were functionally annotated using SnpEff v5.4 with the *Corynebacterium_diphtheriae_nctc_13129* built-in database. SnpEff classifies each variant by predicted effect on protein function.
 
```bash
snpEff ann -v \
  -stats snpEffcdif_summary.html \
  -csvStats snpEffcdif_summary.csv \
  Corynebacterium_diphtheriae_nctc_13129 \
  varcdif_norm_renamed.vcf > varcdif_annotated.vcf
```
 
SnpEff impact categories used in this analysis:
 
| Impact Level | Definition |
|--------------|-----------|
| HIGH | Frameshift, stop codon gained/lost, start codon lost — likely disruptive to protein function |
| MODERATE | Missense substitution, inframe indel — possibly deleterious |
| LOW | Synonymous variant, splice region — unlikely to affect protein function |
| MODIFIER | Intergenic, upstream/downstream — no direct coding impact predicted |
 
HIGH-impact genes were extracted using:
 
```bash
grep "HIGH" varcdif_annotated.vcf | \
  grep -o "ANN=[^	]*" | \
  tr ',' '\n' | \
  grep "HIGH" | \
  cut -d'|' -f4,5 | \
  sort | uniq -c | sort -rn
```
 
---
 
## Software Summary
 
| Tool | Version | Purpose | Reference |
|------|---------|---------|-----------|
| fastq-dump | SRA Toolkit | Read extraction from SRA format | NCBI, 2022 |
| FastQC | v0.11.9 | Read quality assessment | Andrews, 2015 |
| fastp | v0.23 | Adapter trimming and filtering | Chen et al., 2018 |
| BWA MEM | v0.7.17 | Reference genome alignment | Li & Durbin, 2009 |
| SAMtools | v1.15 | BAM processing and coverage analysis | Li et al., 2009 |
| FreeBayes | v1.3.6 | Haplotype-based variant calling | Garrison & Marth, 2012 |
| bcftools | v1.15 | Variant filtering and normalization | Li, 2011 |
| SnpEff | v5.4 | Functional variant annotation | Cingolani et al., 2012 |
 
---
