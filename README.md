<div align="center">

# 🧬 VGP Genome Assembly Pipeline
**End-to-end diploid genome assembly of *Saccharomyces cerevisiae* using PacBio HiFi + Hi-C + Bionano optical maps**

[![Galaxy](https://img.shields.io/badge/Workflow-Galaxy-1f6feb?style=flat-square&logo=galaxy&logoColor=white)](https://usegalaxy.org/)
[![HiFi](https://img.shields.io/badge/Reads-PacBio%20HiFi-2563eb?style=flat-square)](https://www.pacb.com/technology/hifi-sequencing/)
[![Hi-C](https://img.shields.io/badge/Scaffolding-Hi--C-16a34a?style=flat-square)](https://dovetailgenomics.com/)
[![Bionano](https://img.shields.io/badge/Scaffolding-Bionano%20Optical%20Maps-7c3aed?style=flat-square)](https://bionano.com/)
[![VGP](https://img.shields.io/badge/Pipeline-VGP%20Assembly-e11d48?style=flat-square)](https://vertebrategenomesproject.org/)
[![Organism](https://img.shields.io/badge/Organism-S.%20cerevisiae%20S288C-f59e0b?style=flat-square)](https://www.ncbi.nlm.nih.gov/datasets/taxonomy/4932/)

[Overview](#overview) · [Dataset](#dataset) · [Pipeline](#pipeline) · [Tools & Outputs](#tools--outputs) · [Assembly Stats](#assembly-statistics) · [References](#references)

</div>

---

## Overview

This repository documents a diploid genome assembly pipeline following the **Vertebrate Genomes Project (VGP)** methodology, implemented in Galaxy. The pipeline takes raw PacBio HiFi reads and produces chromosome-scale haplotype-resolved scaffolds using a combination of HiFi contig assembly, Bionano optical map scaffolding, and Hi-C chromatin contact scaffolding.

The test organism is *Saccharomyces cerevisiae* S288C (haploid genome ~11.7 Mb), assembled as a synthetic diploid. This allows for high-confidence evaluation of assembly completeness and accuracy.

**Analysis trajectory used: B** — HiFi + Hi-C (Hi-C phased contigging → Bionano scaffolding → Hi-C scaffolding)

---

## Dataset

| Data type | Format | Source | Purpose |
|-----------|--------|--------|---------|
| PacBio HiFi reads (3 files) | `.fasta` | [Zenodo 6098306](https://zenodo.org/record/6098306) | Contig assembly |
| Illumina Hi-C reads (F + R) | `.fastqsanger.gz` | [Zenodo 5550653](https://zenodo.org/record/5550653) | Contig phasing & Hi-C scaffolding |
| Bionano optical map | `.cmap` | [Zenodo 5887339](https://zenodo.org/records/5887339) | Optical map scaffolding |

> **Coverage targets:** ≥30× HiFi, ~60× Hi-C (recommended for diploid genomes)

---

## Pipeline

The pipeline consists of five sequential stages:

```
Raw HiFi Reads
      │
      ▼
[1] Adapter Trimming (Cutadapt)
      │
      ▼
[2] Genome Profiling (Meryl → GenomeScope2)
      │
      ▼
[3] Hi-C Phased Contig Assembly (hifiasm)
      │
      ├── Hap1 contigs (GFA → FASTA)
      └── Hap2 contigs (GFA → FASTA)
            │
            ▼
      [4] Assembly QC (gfastats, BUSCO, Merqury)
            │
            ▼
      [5a] Bionano Hybrid Scaffolding
            │
            ▼
      [5b] Hi-C Scaffolding (BWA-MEM2 → YaHS)
            │
            ▼
      Final Chromosome-Scale Scaffolds
            │
            ▼
      [6] Contact Map Evaluation (PretextMap + Pretext Snapshot)
```

---

## Tools & Outputs

### Stage 1 — Adapter Trimming

**Tool:** `Cutadapt v4.4`

Removes HiFi reads that contain internal adapter sequences. Unlike short-read trimming, adapters in PacBio SMRT reads can appear anywhere in the read — so entire reads are discarded rather than trimmed.

| Parameter | Value |
|-----------|-------|
| Adapter 1 | `ATCTCTCTCAACAACAACAACGGAGGAGGAGGAAAAGAGAGAGAT` |
| Adapter 2 | `ATCTCTCTCTTTTCCTCCTCCTCCGTTGTTGTTGTTGAGAGAGAT` |
| Max error rate | 0.1 |
| Min overlap | 35 bp |
| Reverse complement search | Yes |
| Action on match | Discard entire read |

**Input:** `HiFi_collection` (3× `.fasta` files as a Galaxy collection)  
**Output:** `HiFi_collection (trimmed)` — adapter-free HiFi reads

---

### Stage 2 — Genome Profiling

#### 2a. K-mer Counting

**Tool:** `Meryl v1.3` (run 3×)

Decomposes reads into k-mers of length 31, counts occurrences, and merges databases across all three HiFi input files into a single unified k-mer database.

| Run | Operation | Input | Output |
|-----|-----------|-------|--------|
| 1 | Count k-mers (k=31) | HiFi collection (trimmed) | `meryldb` (collection) |
| 2 | Union-sum merge | `meryldb` collection | `Merged meryldb` |
| 3 | Generate histogram | `Merged meryldb` | `meryldb histogram` |

**Output used downstream:** `Merged meryldb` (for Merqury QC), `meryldb histogram` (for GenomeScope2)

#### 2b. Genome Size & Heterozygosity Estimation

**Tool:** `GenomeScope2 v2.0`

Fits a mixture of negative binomial distributions to the k-mer frequency histogram to estimate genome properties.

| Parameter | Value |
|-----------|-------|
| Ploidy | 2 |
| k-mer length | 31 |

**Input:** `meryldb histogram`  
**Output:** Linear/log k-mer plots, model fit, summary statistics

**Key results:**

| Property | Estimated Value |
|----------|----------------|
| Haploid genome size | ~11.7 Mb |
| Heterozygosity | ~0.576% |
| Diploid coverage peak | ~50× |
| Haploid coverage peak | ~25× |
| Model fit | >93% |

> These values were used to parameterize downstream tools (e.g., expected genome size in gfastats).

---

### Stage 3 — Hi-C Phased Contig Assembly

**Tool:** `hifiasm v0.19.8`

Assembles HiFi reads into contigs and uses Hi-C reads from the same individual to phase them into two haplotypes (hap1 / hap2). Outputs assembly graphs in GFA format.

| Input | Description |
|-------|-------------|
| `HiFi_collection (trimmed)` | Adapter-filtered HiFi reads |
| `Hi-C_dataset_F` | Hi-C forward reads |
| `Hi-C_dataset_R` | Hi-C reverse reads |

| Output (GFA) | Renamed to |
|--------------|------------|
| Hi-C hap1 balanced contig graph | `Hap1 contigs graph` |
| Hi-C hap2 balanced contig graph | `Hap2 contigs graph` |

**GFA → FASTA conversion**

**Tool:** `gfastats v1.3.6`

Converts hifiasm GFA outputs to FASTA format for downstream tools.

| Input | Output |
|-------|--------|
| `Hap1 contigs graph` | `Hap1 contigs FASTA` |
| `Hap2 contigs graph` | `Hap2 contigs FASTA` |

---

### Stage 4 — Assembly Quality Control

#### 4a. Summary Statistics

**Tool:** `gfastats v1.3.6`

Generates contig-level assembly statistics. Run on both hap1 and hap2 GFA files with expected genome size of `11747160` bp. Outputs are column-joined for side-by-side comparison.

**Key stats reported:** contig count, total length, largest contig, N50, NG50, GC content

**Hap1 / Hap2 results (approximate):**

| Metric | Hap1 | Hap2 |
|--------|------|------|
| # Contigs | 16 | 17 |
| Total length | ~11.3 Mb | ~12.2 Mb |

#### 4b. Gene Completeness (BUSCO)

**Tool:** `BUSCO v5.5.0`

Evaluates assembly completeness by searching for lineage-specific conserved single-copy orthologs.

| Parameter | Value |
|-----------|-------|
| Mode | Genome assembly (DNA) |
| Lineage | Saccharomycetes |
| Gene predictor | Metaeuk |

**Input:** `Hap1 contigs FASTA`, `Hap2 contigs FASTA`  
**Output:** Short summary text + summary image per haplotype (`BUSCO hap1`, `BUSCO hap2`)

#### 4c. K-mer Based QC (Merqury)

**Tool:** `Merqury v1.3`

Reference-free assessment of assembly completeness, accuracy (QV), and phasing quality by comparing assembly k-mers to the read k-mer database.

| Input | Description |
|-------|-------------|
| `Merged meryldb` | K-mer database from raw reads |
| `Hap1 contigs FASTA` | First haplotype assembly |
| `Hap2 contigs FASTA` | Second haplotype assembly |

**Outputs:** Completeness stats, QV stats, spectra-cn plot (copy-number), spectra-asm plot (haplotype assignment)

**Interpretation:**
- `spectra-cn` plot: diploid coverage peak at ~50×; confirms expected sequencing depth
- `spectra-asm` plot: haploid k-mers (~25×) split between hap1 and hap2, confirming phasing; homozygous k-mers shared between both assemblies as expected

---

### Stage 5a — Bionano Hybrid Scaffolding

**Tool:** `Bionano Hybrid Scaffold v3.7.0`

Integrates Bionano optical maps with the contig assembly to orient, order, and join contigs into larger scaffolds. Resolves chimeric joins and estimates gap sizes using optical map information.

| Input | Description |
|-------|-------------|
| `Hap1 contigs FASTA` | Contig-level assembly |
| `Bionano_dataset` (`.cmap`) | Optical map data |

| Parameter | Value |
|-----------|-------|
| Configuration | VGP mode |
| Genome maps conflict | Cut contig at conflict |
| Sequences conflict | Cut contig at conflict |

**Outputs used:**
- `NGScontigs scaffold NCBI trimmed` — contigs that were scaffolded using optical maps
- `NGScontigs not scaffolded trimmed` — contigs that could not be incorporated

These two outputs are concatenated (via `Concatenate datasets`) into:  
→ `Hap1 assembly bionano`

**Post-Bionano gfastats evaluation** (expected genome size: `11747160`):  
Output: `Bionano stats` — shows improvement in scaffold count and largest scaffold vs. raw contigs

---

### Stage 5b — Hi-C Scaffolding

#### Read Mapping

**Tool:** `BWA-MEM2 v2.2.1` (run twice — once per read direction)

Hi-C reads are mapped **individually** (not as pairs) against the Bionano-scaffolded assembly, because insert sizes in Hi-C data vary from 1 bp to hundreds of Mb and do not fit standard paired-end assumptions.

| Run | Input reads | Reference | Sort mode | Output |
|-----|-------------|-----------|-----------|--------|
| 1 | `Hi-C_dataset_F` | `Hap1 assembly bionano` | By read name (QNAME) | `BAM forward` |
| 2 | `Hi-C_dataset_R` | `Hap1 assembly bionano` | By read name (QNAME) | `BAM reverse` |

**Tool:** `Filter and merge v1.0` (Arima Genomics chimeric read filter)

Merges and filters the two BAM files, handling chimeric Hi-C read pairs.

**Input:** `BAM forward`, `BAM reverse`  
**Output:** `BAM Hi-C reads`

#### Initial Contact Map (Pre-scaffolding)

**Tools:** `PretextMap v0.1.9` → `Pretext Snapshot v0.0.3`

Generates a Hi-C contact heatmap to visualize scaffold order before YaHS scaffolding, used as a baseline for comparison.

**Input:** `BAM Hi-C reads`  
**Outputs:** `PretextMap output` → contact map PNG image (grid view)

#### YaHS Scaffolding

**Tool:** `YaHS v1.2a.2`

Uses Hi-C contact frequencies to orient and order Bionano scaffolds into chromosome-scale sequences. Does not require a pre-specified chromosome count.

| Input | Description |
|-------|-------------|
| `Hap1 assembly bionano` | Bionano-scaffolded contigs |
| `BAM Hi-C reads` | Merged Hi-C alignments |

| Parameter | Value |
|-----------|-------|
| Restriction enzyme sequence | `CTTAAG` |

**Output:** `YaHS Scaffolds FASTA` — final chromosome-scale scaffolded assembly

---

### Stage 6 — Final Assembly Evaluation

Hi-C reads are re-mapped to the YaHS scaffolds and a new contact map is generated to confirm correct scaffold ordering.

**Tools used (same as pre-scaffolding contact map):**

| Tool | Input | Output |
|------|-------|--------|
| BWA-MEM2 (×2) | `Hi-C_dataset_F/R` vs. `YaHS Scaffolds FASTA` | `BAM forward/reverse YaHS` |
| Filter and merge | `BAM forward/reverse YaHS` | `BAM Hi-C reads YaHS` |
| PretextMap | `BAM Hi-C reads YaHS` | `PretextMap output YaHS` |
| Pretext Snapshot | `PretextMap output YaHS` | Final contact map PNG |

---

## Assembly Statistics
 
| Metric | Final Assembly | Reference Genome |
|--------|---------------|-----------------|
| Total sequence length | 12,161 kb | 12,157 kb |
| Scaffold count | 17 | 17 |
| N50 | 922 kb | 924 kb |
| N75 | 667 kb | 667 kb |
| L50 | 6 | 6 |
| GC% | 38.18 | 38.00 |
 
The final assembly closely matches the reference genome across all key metrics, with scaffold count, N50, and L50 all in excellent agreement.
 
---
 
## Glossary
 
| Term | Definition |
|------|------------|
| **Contig** | A gapless sequence inferred from overlapping reads |
| **Scaffold** | One or more contigs joined by gap sequences (N's) |
| **Haplotype** | One of the two inherited copies of each chromosome |
| **Phasing** | Assigning contigs to their correct parental haplotype |
| **N50** | Length at which contigs ≥ this size cover 50% of the assembly |
| **QV** | Phred-scaled quality value; QV50 = 1 error per 100,000 bp |
| **BUSCO** | Completeness score based on conserved single-copy orthologs |
 
---
 
## References
 
- Cheng et al. (2021). Haplotype-resolved de novo assembly using phased assembly graphs. *Nature Methods*
- Rhie et al. (2020). Merqury: reference-free quality, completeness, and phasing assessment for genome assemblies. *Genome Biology*
- Formenti et al. (2022). Gfastats: conversion, evaluation and manipulation of genome sequences. *Bioinformatics*
- Simão et al. (2015). BUSCO: assessing genome assembly and annotation completeness. *Bioinformatics*
- Ranallo-Benavidez et al. (2020). GenomeScope 2.0 and Smudgeplots. *Nature Communications*
- Zhou et al. (2022). YaHS: yet another Hi-C scaffolding tool. *Bioinformatics*
- Wenger et al. (2019). Accurate circular consensus long-read sequencing. *Nature Biotechnology*

- Zhou et al. (2022). YaHS: yet another Hi-C scaffolding tool. *Bioinformatics*
- Wenger et al. (2019). Accurate circular consensus long-read sequencing. *Nature Biotechnology*
