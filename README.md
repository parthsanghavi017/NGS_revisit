# Next-Generation Sequencing (NGS) Analysis Revisit: Short-Read & Structural Variant Pipelines

Welcome to the comprehensive, hands-on teaching materials for Next-Generation Sequencing (NGS) analysis. This repository provides an end-to-end curriculum covering **Short-Read Somatic Secondary Analysis**, **Copy Number Variation (CNV) Calling**, and **Oncogenic Gene Fusion Detection**, specifically reduced to **Chromosome 17 (`chr17`)** for optimal learning and computational efficiency.

---

## 🧬 Dataset Optimization & Architecture

To allow seamless execution on laptop and workstation environments without sacrificing human genomic structure:
- **Reference Genome**: Reduced to Chromosome 17 ([`References/chr17.fa`](References/chr17.fa), ~81 MB).
- **Target Capture BED**: Filtered to Chromosome 17 target exons ([`References/Target_regions_chr17.bed`](References/Target_regions_chr17.bed), 16,233 capture windows).
- **Input Read Pairs**: Compact paired FASTQs ([`Samples/SRR7890850_1.fastq.gz`](Samples/SRR7890850_1.fastq.gz) & [`Samples/SRR7890850_2.fastq.gz`](Samples/SRR7890850_2.fastq.gz), ~876 MB total), extracted directly from real human exome sequencing data mapped to `chr17`.

---

## 📋 Prerequisites & Conda Environments

We provide three dedicated Conda environments tailored to each specific analysis session:

| Session | Topic / Focus | Conda Env Name | Core Bioinformatics Tools Included | YAML Spec File |
| :--- | :--- | :--- | :--- | :--- |
| **Section 1** | **ShortVar Secondary Analysis** | `shortvar_sec` | `fastp`, `bwa`, `samtools`, `picard`, `mosdepth`, `gatk4`, `jupyter`, `pandas` | [`shortvar_sec.yml`](shortvar_sec.yml) |
| **Section 2a** | **Copy Number Variation (CNV)** | `strvar_cnv` | `cnvkit`, `pyfaidx`, `pysam`, `scipy`, `matplotlib`, `pandas`, `jupyter` | [`strvar_cnv.yml`](strvar_cnv.yml) |
| **Section 2b** | **Oncogenic Gene Fusions** | `strvar_fus` | `genefuse` (v0.8.0), `jupyter`, `notebook`, `ipykernel` | [`strvar_fus.yml`](strvar_fus.yml) |

---

## 🚀 Step-by-Step Environment Setup & Launch

Run the following commands in your bash terminal to build the environments and register their kernels in Jupyter:

### 1. Build & Register Section 1 (`shortvar_sec`)
```bash
conda env create -f shortvar_sec.yml
conda activate shortvar_sec
python -m ipykernel install --user --name shortvar_sec --display-name "Python 3 (shortvar_sec)"
```

### 2. Build & Register Section 2a (`strvar_cnv`)
```bash
conda env create -f strvar_cnv.yml
conda activate strvar_cnv
python -m ipykernel install --user --name strvar_cnv --display-name "Python 3 (strvar_cnv)"
```

### 3. Build & Register Section 2b (`strvar_fus`)
```bash
conda env create -f strvar_fus.yml
conda activate strvar_fus
python -m ipykernel install --user --name strvar_fus --display-name "Python 3 (strvar_fus)"
```

### 4. Launch Jupyter Lab
Start the interactive Jupyter Lab server from the repository root:

```bash
jupyter lab
```

In Jupyter Lab, select the corresponding kernel (**`Python 3 (shortvar_sec)`**, **`Python 3 (strvar_cnv)`**, or **`Python 3 (strvar_fus)`**) for each notebook.

---

## 📚 Curriculum & Notebook Breakdown

### 1. [`01_ShortVar_Secondary_Analysis.ipynb`](01_ShortVar_Secondary_Analysis.ipynb)
*Recommended Kernel: `Python 3 (shortvar_sec)`*

Covers the short-read secondary analysis pipeline from raw reads to filtered somatic variants:
- **Step 1: FastP Quality Control & Trimming**: Phred Q-scores ($Q = -10 \log_{10} P$), Q30 standard ($99.9\%$ accuracy), adapter read-through removal, poly-G tail trimming.
- **Step 2: Reference Genome Indexing**: Burrows-Wheeler Transform (BWT), FM-Index, Suffix Arrays, `.fai` lookup table, and GATK sequence dictionary (`gatk CreateSequenceDictionary`).
- **Step 3: Alignment & Read Group Tagging**: `bwa-mem` aligned and piped to `samtools sort`. Theory of Soft-clipping (`S`) vs Hard-clipping (`H`) and Read Group (`@RG`) metadata (`ID`, `SM`, `PL`, `LB`, `PU`).
- **Step 4: PCR Duplicate Marking**: `picard MarkDuplicates` tagging duplicate reads with SAM flag `0x400` (1024). Theory of PCR amplification bias vs optical duplicates.
- **Step 5: BAM Quality Audit**: `samtools flagstat` & `samtools stats`. Formatted as Pandas DataFrames displaying Summary Numbers (`SN`).
- **Step 6: Mosdepth Target Region Coverage**: High-speed coverage array calculation across 16,233 capture regions. Output inspected via Pandas summary and distribution tables (`describe()`).
- **Step 7: GATK Mutect2 Somatic Calling**: De Bruijn graph local re-assembly, Bayesian likelihoods, Tumor-Only vs Tumor-Normal vs Panel of Normals (PoN).
- **Step 8: GATK FilterMutectCalls**: Machine-learning orientation bias (OxoG/FFPE), strand bias, and candidate variant filtering. Filtered VCF parsed into Pandas DataFrames.

---

### 2. [`02a_Structural_Variant_CNV.ipynb`](02a_Structural_Variant_CNV.ipynb)
*Recommended Kernel: `Python 3 (strvar_cnv)`*

Covers tumor-only copy number alteration analysis using `CNVkit`:
- **Step 1: Target BED Splitting** (`cnvkit.py target`): Subdividing target capture regions into uniform coverage bins (~200-500 bp).
- **Step 2: Access & Antitarget Derivation** (`cnvkit.py access` & `antitarget`): Deriving background off-target coverage bins from non-specifically captured intronic/intergenic reads.
- **Step 3: Bin Coverage Counting** (`cnvkit.py coverage`): Calculating mean log2 read depth across target and antitarget bins.
- **Step 4: Flat Baseline Reference** (`cnvkit.py reference`): Constructing a reference-free baseline from FASTA GC-content and repeat-masking models.
- **Step 5: Coverage Fix & Normalization** (`cnvkit.py fix`): GC bias regression and log2 copy ratio computation ($\log_2(\text{Tumor Depth} / \text{Reference Depth})$).
- **Step 6: Genome Segmentation** (`cnvkit.py segment -m haar`): Grouping noisy bin log2 ratios into contiguous copy number segments.
- **Step 7 & 8: Purity, Ploidy & BAF Integration** (`cnvkit.py call --vcf`): Translating continuous ratios to discrete integer copy numbers ($CN = 0, 1, 2, 3, 4+$) and integrating GATK Mutect2 VCF **B-Allele Frequency (BAF)** to detect Loss of Heterozygosity (LOH).
- **Step 9: Pandas Table Inspection**: Python cell inspecting `.cnr`, `.cns`, and `.call.cns` DataFrames, highlighting copy number alteration segments ($CN \neq 2$).

---

### 3. [`02b_Structural_Variant_Fusion.ipynb`](02b_Structural_Variant_Fusion.ipynb)
*Recommended Kernel: `Python 3 (strvar_fus)`*

Covers oncogenic gene fusion detection using `GeneFuse`:
- **Theoretical Foundations**: Mechanisms of chromosomal translocations, inversions, and interstitial deletions generating oncogenic chimeric transcripts (*EML4-ALK*, *BCR-ABL1*, *RET*, *ROS1*, *NTRK1/2/3*, *TMPRSS2-ERG*).
- **Sequence Read Evidence**: **Split Reads** spanning fusion junctions vs **Discordant Paired-End Reads**.
- **K-Mer Junction Indexing**: Fast k-mer matching against hg38 cancer/druggable gene panels ([`References/GeneFuse/cancer.hg38.csv`](References/GeneFuse/cancer.hg38.csv) & [`References/GeneFuse/druggable.hg38.csv`](References/GeneFuse/druggable.hg38.csv)).
- **Execution & Report Parsing**: Running `genefuse` on raw read pairs, generating `genefuse_report.html` and `genefuse_report.json`, and parsing candidate fusion events into Pandas DataFrames.

---

## 📁 Repository Directory Layout

```
NGS_Revisit/
├── 01_ShortVar_Secondary_Analysis.ipynb   # Section 1: ShortVar Secondary Analysis Notebook
├── 02a_Structural_Variant_CNV.ipynb       # Section 2a: CNV Analysis Notebook (CNVkit)
├── 02b_Structural_Variant_Fusion.ipynb    # Section 2b: Gene Fusion Notebook (GeneFuse)
├── README.md                              # Main Setup & Teaching Guide
├── shortvar_sec.yml                       # Conda Environment Specification for Section 1
├── strvar_cnv.yml                         # Conda Environment Specification for Section 2a
├── strvar_fus.yml                         # Conda Environment Specification for Section 2b
├── create_notebook.py                     # Script generating Section 1 notebook
├── create_session2_notebooks.py           # Script generating Section 2 notebooks
├── References/                            # Reference Resources
│   ├── chr17.fa                           # Reference FASTA (chr17)
│   ├── chr17.fa.fai                       # FASTA Index
│   ├── chr17.dict                         # GATK Sequence Dictionary
│   ├── Target_regions_chr17.bed           # Target Capture Regions BED (chr17)
│   └── GeneFuse/                          # GeneFuse Panel Files
│       ├── cancer.hg38.csv                # Cancer Gene Fusion Library
│       └── druggable.hg38.csv             # Druggable Gene Fusion Library
├── Samples/                               # Compact Sample Input Reads
│   ├── SRR7890850_1.fastq.gz              # Paired Read 1 (432 MB)
│   └── SRR7890850_2.fastq.gz              # Paired Read 2 (444 MB)
└── Results/                               # Clean Output Subdirectories
    ├── QC/                                # FastP Reports & Trimmed Reads
    ├── Align/                             # Sorted BAM Files
    ├── Deduplication/                     # Deduplicated BAM Files & Metrics
    ├── Stats/                             # Flagstat & Samtools Stats Summaries
    ├── Coverage/                          # Mosdepth Coverage Reports
    ├── Variants/                          # Raw & Filtered GATK Somatic VCFs
    ├── CNV/                               # CNVkit .cnn, .cnr, .cns, and .call.cns Files
    └── Fusion/                            # GeneFuse HTML & JSON Reports
```
