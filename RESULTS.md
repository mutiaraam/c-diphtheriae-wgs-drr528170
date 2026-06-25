# Results

Findings are presented sequentially by analytical pipeline stage. Full methodology is documented in [METHODS.md](./METHODS.md).

---

## 1. Data Preparation and Quality Control

Raw paired-end reads were assessed using FastQC and preprocessed with fastp. Post-trimming metrics confirmed adequate quality for downstream alignment.

| Parameter | Before Trimming | After Trimming | Threshold | Status |
|-----------|----------------|----------------|-----------|--------|
| Total reads | 4,405,628 | 4,286,188 | ≥1,000,000 | ✅ |
| Read length (bp) | 150 | 150 | 100–300 | ✅ |
| GC content (%) | 53 | 53 | ~53 | ✅ |
| Q20 bases (%) | 90.22 | 91.93 | ≥90 | ✅ |
| Q30 bases (%) | 80.46 | 82.51 | ≥80 | ✅ |
| Adapter contamination | Present | Removed | None | ✅ |

---

## 2. Genome Mapping

Trimmed reads were aligned to *C. diphtheriae* NCTC13129 (NC_002935.2) using BWA MEM. Alignment quality was assessed via `samtools flagstat` and `samtools coverage`.

| Parameter | Value | Threshold | Status |
|-----------|-------|-----------|--------|
| Total reads processed | 4,324,446 | — | — |
| Mapped reads | 3,992,824 | — | — |
| **Mapping rate (%)** | **92.33** | >85 | ✅ |
| Properly paired (%) | 91.44 | >85 | ✅ |
| Singletons (%) | 0.14 | <5 | ✅ |
| Inter-chromosomal mismatches | 0 | 0 | ✅ |
| **Mean sequencing depth (×)** | **360.53** | ≥30 | ✅ |
| **Genome coverage breadth (%)** | **95.32** | >90 | ✅ |
| Mean base quality (Phred) | 34 | ≥30 | ✅ |
| Mean mapping quality | 58.6 | ≥30 | ✅ |

---

## 3. SAM/BAM Processing

The SAM output from BWA MEM was processed using SAMtools to generate a sorted, indexed BAM file suitable for variant calling.

| Step | Command | Input | Output | Status |
|------|---------|-------|--------|--------|
| Format conversion | `samtools view -Sb` | align_cdif.sam | align_cdif.bam | ✅ |
| Coordinate sorting | `samtools sort` | align_cdif.bam | aligncdif_sorted.bam | ✅ |
| Index generation | `samtools index` | aligncdif_sorted.bam | aligncdif_sorted.bam.bai | ✅ |

---

## 4. Variant Calling and Functional Annotation

### 4.1 Variant Landscape

Variants were called with FreeBayes, filtered and normalized with bcftools, and functionally annotated using SnpEff v5.4 against the *Corynebacterium_diphtheriae_nctc_13129* database.

| Variant Type | Count | Percentage |
|--------------|-------|------------|
| SNP | 23,070 | 82.63% |
| MNP | 4,170 | 14.93% |
| Insertion | 219 | 0.78% |
| Deletion | 211 | 0.76% |
| Complex | 245 | 0.88% |
| **Total** | **27,915** | **100%** |

### 4.2 Functional Impact Classification

| Impact Level | Count | Percentage | Predicted Consequence |
|--------------|-------|------------|-----------------------|
| HIGH | 188 | 0.67% | Frameshift, stop codon gained/lost, start lost |
| MODERATE | 6,607 | 23.66% | Missense substitution, inframe indel |
| LOW | 17,291 | 61.94% | Synonymous variant, splice region |
| MODIFIER | 3,829 | 13.72% | Intergenic, upstream/downstream variant |
| **Total** | **27,915** | **100%** | — |

Of the 188 HIGH-impact variants, the predominant effect types were frameshift mutations and stop codon changes, distributed across **130 genes**. A complete ranked list of all 130 genes is available in the [METHODS.md](./METHODS.md) section on functional annotation. For clinical interpretation, detailed analysis focuses on six genes of known significance: five vaccine-relevant genes and one antimicrobial resistance marker.

### 4.3 Genes of Clinical Significance

Vaccine-relevant genes and antimicrobial resistance markers were evaluated together for clinical interpretation. Despite genome-wide divergence, no HIGH-impact variants were identified in primary vaccine antigen genes. One resistance-associated gene carried a disruptive variant.

| Gene ID | Gene Symbol | Encoded Protein | Variant Effect | Function | Status |
|---------|-------------|----------------|----------------|----------|--------|
| — | *tox* | Diphtheria toxin | None (HIGH-impact) | Primary vaccine antigen | ✅ Conserved |
| — | *dtxR* | Diphtheria toxin repressor | None (HIGH-impact) | Regulates toxin expression | ✅ Conserved |
| — | *spaA* | Major pilus subunit | None (HIGH-impact) | Surface adhesin | ✅ Conserved |
| — | *spaB* | Minor pilus subunit | None (HIGH-impact) | Surface adhesin | ✅ Conserved |
| — | *spaC* | Pilus tip adhesin | None (HIGH-impact) | Host cell attachment | ✅ Conserved |
| DIP0102 | *tipA* | TipA efflux pump | Stop lost + splice region | Tetracycline resistance | ⚠️ Disrupted |

---

### 4.4 Pipeline Summary

| Stage | Tool | Result |
|-------|------|--------|
| Quality control | FastQC + fastp | Q30 82.51%, adapters removed |
| Genome alignment | BWA MEM | 92.33% mapping rate |
| Coverage analysis | SAMtools | 95.32% breadth, 360× depth |
| Variant calling | FreeBayes | Raw variants identified |
| Filtering + normalization | bcftools | QUAL ≥30, DP ≥10, AF ≥0.1 |
| Functional annotation | SnpEff v5.4 | 27,915 variants annotated |
| **Total variants** | — | **27,915** |
| **HIGH-impact variants** | — | **188 across 130 genes** |
| **Vaccine target genes with HIGH-impact** | — | **0** |
| **Resistance marker genes with HIGH-impact** | — | **1 (*tipA*)** |

---

*See [DISCUSSION.md](./DISCUSSION.md) for biological interpretation of these findings.*
