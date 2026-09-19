# 🧬 Bioinformatics NGS Pipeline Interview Prep — Complete Guide

**Comprehensive Interview Resource** | Covers: Conda · Docker · Git/GitLab · CI/CD · Nextflow · Variant Calling · RNA-seq · Single-Cell · Bash Scripting · CNV · Fusion Calling

---

## Table of Contents

1. [Conda & Environment Management](#conda--environment-management)
2. [Docker Containerization](#docker-containerization)
3. [Git & GitLab](#git--gitlab)
4. [CI/CD & Workflow Integration](#cicd--workflow-integration)
5. [Nextflow Fundamentals](#nextflow-fundamentals)
6. [Nextflow Workflows & Integration](#nextflow-workflows--integration)
7. [SNV Calling — Secondary Analysis](#snv-calling--secondary-analysis)
8. [Short Variant Calling — Tertiary Analysis](#short-variant-calling--tertiary-analysis)
9. [Variant Interpretation & Guidelines](#variant-interpretation--guidelines)
10. [RNA-seq Analysis](#rna-seq-analysis)
11. [Single-Cell & Spatial Transcriptomics](#single-cell--spatial-transcriptomics)
12. [Bash Scripting Essentials](#bash-scripting-essentials)
13. [CNV Calling](#cnv-calling)
14. [Fusion Calling](#fusion-calling)
15. [Pipeline-Specific Questions](#pipeline-specific-questions)
16. [Quick Reference Commands](#quick-reference-commands)

---

## Conda & Environment Management

### Q1: What is the technical difference between Conda, Miniconda, and Anaconda? How do you choose which to use for an NGS pipeline deployment?

**Answer:**

- **Conda** is a language-agnostic package and environment management system. It resolves dependencies using a built-in satisfiability solver (like libsolv in newer versions) to ensure all binary packages are mutually compatible.

- **Miniconda** is a minimal bootstrap distribution that includes only Conda, Python, their dependencies, and a small number of core packages (like pip and zlib).

- **Anaconda** is a full distribution that includes Conda along with over 1,500 pre-installed scientific, data science, and machine learning packages.

**Selection Strategy:** For production clinical pipelines, Miniconda (or Minimalist variations like Mambaforge) is almost always preferred. Anaconda creates massive container images and environments filled with unused packages, increasing the attack surface and introducing potential version conflicts. Miniconda allows you to build lightweight, reproducible environments containing only the specific tools needed (e.g., specific versions of samtools, bwa, or bcftools), keeping Docker images lean and pipelines predictable.

---

### Q2: Why is the bioconda channel order critical, and what happens if channels are configured incorrectly in a .condarc file?

**Answer:**

**Channel Priority:** Channels are the locations where Conda looks for packages. For bioinformatics, the official configuration requires a strict priority order: conda-forge (highest), followed by bioconda, and defaults (lowest).

**The Risk:** Many packages in bioconda depend on core libraries (like libgcc, openssl, or specific Python builds) compiled and maintained by the conda-forge community.

**Consequences:** If defaults is placed above conda-forge or channel priority is set to flexible instead of strict, Conda might pull a core dependency from defaults and a bioinformatics tool from bioconda. Because these packages were compiled against different binary interfaces (ABIs), this mismatch frequently causes silent runtime execution errors, segmentation faults (segfault), or unresolved dependency loops during environment creation.

---

### Q3: How do you achieve absolute environment reproducibility across different compute environments using Conda?

**Answer:**

**Standard Export (environment.yml):** Running `conda env export > environment.yml` captures the package names and versions. However, this file is often platform-specific and can fail if transferred between different operating systems (e.g., macOS to Linux) because it includes builds tied to specific OS architectures.

**Explicit Specification Files (Best Practice):** For strict cryptographic reproducibility on identical operating systems (like deploying from a Linux development machine to a Linux production server), you should generate an explicit specification file:

```bash
conda list --explicit > spec-file.txt
```

This file bypasses the dependency solver entirely. It contains exact URLs and MD5/SHA256 hashes of the pre-compiled binaries. Recreating it via `conda create --name my_env --file spec-file.txt` ensures that every single bit of the environment is identical, preventing silent updates from breaking a validated clinical pipeline.

---

### Q4: When a pipeline step requires two tools with mutually exclusive dependencies (e.g., Tool A requires Python 2.7 and Tool B requires Python 3.10), how do you resolve this using Conda within a workflow manager?

**Answer:**

**Isolation Strategy:** You must never force conflicting dependencies into a single environment. Instead, split them into modular, task-specific Conda environments.

**Workflow Orchestration Integration:** Rather than manual activation, you delegate the environment switching to your workflow runner (like Snakemake or Nextflow).

- In **Snakemake**, you define a separate `environment.yml` for each rule using the `conda:` directive. Snakemake automatically deploys and isolates these environments at runtime.

- In **Nextflow**, you can use the `conda` directive at the process level to specify either a path to an environment, a `.yml` file, or a package string, ensuring that Process A runs in its Python 2.7 silo while Process B executes smoothly in its Python 3.10 silo.

---

### Q5: Walk me through your standard debugging process when Conda takes an excessive amount of time or throws a "CondaHTTPError / Solver Failure" during environment creation.

**Answer:**

**Solver Hangs:** If Conda takes hours to resolve dependencies, it is usually because the environment has too many unpinned packages, forcing the solver to evaluate millions of potential combinations. I resolve this by explicitly pinning major and minor versions (e.g., `python=3.9`, `samtools=1.15`) or switching to Mamba/Micromamba, a C++ based drop-in replacement for Conda that utilizes parallel downloading and a significantly faster dependency solver (libsolv).

**CondaHTTPError:** This usually points to proxy issues, strict firewall configurations in institutional servers, or SSL certificate issues. I debug this by testing connection via `curl`, configuring the `.condarc` file to bypass SSL verification temporarily if inside a secure intranet (`ssl_verify: false`), or adjusting proxy settings (`proxy_servers`).

**Solver Failure:** If the solver fails explicitly, it means a conflict exists. I run `conda search package_name --info` to inspect the package's exact dependencies and look for hidden conflicts, or I rebuild the environment layer-by-layer to isolate the culprit package.

---

### Q6: In a production team environment, what are the pros and cons of sharing Conda environments via a shared network drive versus using code-managed configuration files?

**Answer:**

| Approach | Pros | Cons |
|----------|------|------|
| **Shared Network Drive** | Saves disk space on cluster; teammates don't need to spend time installing heavy tools | **High risk:** If a team member accidentally modifies, updates, or installs a package within that shared environment, it instantly alters the environment for everyone, breaking existing production runs and ruining audit trails |
| **Code-Managed Configs** | Complete version control. Every environment state is documented in Git, easily auditable, and can be rebuilt on-demand by any developer or CI/CD runner | Requires internet access or internal channel mirror to download packages; takes initial execution time to build locally |

**Verdict:** For clinical and validation-heavy settings, code-managed configurations in Git are the standard to maintain data integrity.

---

### Q7: How do you handle disk space management on high-throughput analysis servers where multiple users are building Conda environments?

**Answer:**

**The Problem:** Conda caches downloaded package tarballs and unindexed package binaries in the user's home directory (`~/.conda/pkgs`). Over time, this easily hoards hundreds of gigabytes of storage, leading to "disk space full" errors on shared nodes.

**Resolution Steps:**

1. **Regular Maintenance:** Run `conda clean --all` (or `mamba clean --all`) regularly to purge cached tarballs and unused packages without breaking active environments.

2. **Hard Links:** By default, Conda uses hard links to connect the central package cache to individual environments on the same filesystem, which saves space. I ensure that environments are built on the same partition as the cache directory so hard links can work effectively.

3. **Global Cache Configuration:** Set up a centralized, read-only shared package cache (`pkgs_dirs` in `.condarc`) across the server so multiple users can share the same binary source pool without duplicating files in their individual directories.

---

### Q8: If an essential open-source bioinformatics tool isn't available on Bioconda or Conda-Forge, how do you cleanly integrate it into your Conda-managed workflow?

**Answer:**

**Method 1: Local Pip Installation (For Python packages):** If it's a Python tool available on PyPI, I include a pip section inside the `environment.yml` file under the dependencies block. This ensures that Pip installs the tool after Conda resolves the base system libraries.

**Method 2: Custom Conda Environment Active Path:** If it is a compiled C/C++ binary or a custom shell script, I download or compile the source code, then move the executable into the `bin/` directory of the active Conda environment (e.g., `$CONDA_PREFIX/bin`). Because the environment's bin path is automatically appended to the system `$PATH` upon activation, the custom tool becomes globally accessible within that environment without manual environment path hijacking.

---

## Docker Containerization

### Q1: Bioinformatics containers often bloat to several gigabytes when packaging tools like GATK, Conda environments, and various dependencies. How do you optimize a Dockerfile to keep the image size minimal and efficient?

**Answer:** There are three primary strategies to minimize image size:

1. **Chain RUN commands using &&** to combine updates, installations, and cache-clearing (e.g., `apt-get clean` or `conda clean --all`) into a single layer, preventing temporary files from being permanently baked into the image.

2. **Use lightweight base images** like `python:3.9-slim` or `miniconda3` instead of full OS images.

3. **For compiled tools, use multi-stage builds:** one stage to compile the C/C++ binaries, and a second, minimal stage that only copies the compiled executable, leaving the heavy build tools behind.

---

### Q2: If you are running a secondary analysis pipeline that needs to process a 50GB BAM file and output a VCF, how do you handle this data within the Docker environment?

**Answer:** You should **never** copy large genomic datasets directly into the container using the COPY command in the Dockerfile. Containers are meant to be ephemeral and stateless. Instead, I use **Bind Mounts** at runtime:

```bash
docker run -v /host/path/to/data:/container/path/to/data my_image
```

This maps a directory on the local host directly into the container, allowing the pipeline to read the BAM file and write the VCF directly to the host storage without bloating the container's writable layer or losing data when the container stops.

---

### Q3: Explain the difference between ENTRYPOINT and CMD in a Dockerfile. How would you configure them for a standalone CLI tool?

**Answer:**

- **ENTRYPOINT** defines the main executable that the container will always run. It makes the container behave like a native command.

- **CMD** provides default arguments to that executable, but these can be easily overridden by the user at runtime.

**Usage:** If I am containerizing a variant prioritization script:

```dockerfile
ENTRYPOINT ["python", "variant_caller.py"]
CMD ["--help"]
```

- If a user runs `docker run my_image`, it outputs the help menu.
- If they run `docker run my_image --input sample.bam`, the `--input` flag overrides `--help` and is passed directly to the Python script.

---

### Q4: A container keeps exiting immediately after you start it with docker run. How do you troubleshoot what went wrong inside the container?

**Answer:** An immediate exit usually means the primary foreground process finished or failed.

1. **First, check logs:**
   ```bash
   docker logs <container_id>
   ```
   This shows any explicit standard error (stderr) outputs or tracebacks.

2. **If logs are empty, override the entrypoint:**
   ```bash
   docker run -it --entrypoint /bin/bash <image_name>
   ```
   This allows you to manually execute the pipeline step-by-step from inside the container to identify missing dependencies or path errors.

---

### Q5: In clinical translation and precision medicine, reproducibility is legally and scientifically critical. How do you ensure your Dockerized pipelines produce the exact same results three years from now?

**Answer:** Reproducibility starts with strict versioning:

1. **Never use the `:latest` tag** for base images or software pulls; pin exact versions (e.g., `ubuntu:20.04` or `bwa:0.7.17`).

2. **Use explicit specification files** to install dependencies rather than resolving them dynamically during the build.

3. **For clinical auditing**, reference the specific **SHA256 image digest** rather than just the tag name when deploying to production, ensuring that even if a tag is overwritten in the registry, the exact binary environment remains immutable.

---

### Q6: Why is it considered a security risk to run Docker containers as the root user, particularly when processing sensitive datasets? How do you prevent it?

**Answer:** By default, processes inside a Docker container run as root. If a vulnerability exists in an open-source tool within the pipeline, an attacker or rogue process could potentially exploit a container breakout and gain root access to the underlying host machine.

To mitigate this:

```dockerfile
RUN useradd -m appuser
USER appuser
ENTRYPOINT ["my_pipeline"]
```

Create a dedicated, non-privileged user within the Dockerfile and switch to it using the `USER` appuser directive before the ENTRYPOINT.

---

### Q7: How does Docker integrate into an Agile CI/CD pipeline (like GitLab CI) when developing analytical tools?

**Answer:** Docker acts as the standard unit of deployment. When a developer commits new code to Git:

1. The CI/CD pipeline is triggered
2. The runner uses the Dockerfile to build a new image
3. Spins it up and runs automated unit and integration tests (e.g., processing a synthetic "golden" patient file to check for expected variant outputs)
4. If clinical concordance tests pass, the pipeline automatically pushes the validated image to a container registry (like Docker Hub or AWS ECR) with a new version tag
5. The image is instantly available for production

---

### Q8: When orchestrating a workflow, how do you decide whether to define a Docker container natively within a Nextflow script versus managing it manually?

**Answer:** Manual container management is prone to human error and doesn't scale across clusters. I integrate Docker natively within Nextflow by defining `container 'repository/image:tag'` directly inside the process block, and enabling Docker in the `nextflow.config` file. This allows Nextflow to handle:

- Automated pulling of images
- Volume mounting of input channels
- Lifecycle management of containers for every single parallel task
- Isolated execution without manual intervention

---

## Git & GitLab

### Q1: In a .gitlab-ci.yml file, how do you manage dependencies and outputs between pipeline stages, particularly when dealing with the intermediate files of a variant calling pipeline?

**Answer:** You manage these using `cache` and `artifacts`, and knowing the difference is critical.

- **Cache** is used to speed up job execution by storing downloaded dependencies (like Conda environments or Python packages) between runs. It is **not** guaranteed to exist.

- **Artifacts** are the explicit outputs of a job (like a generated VCF file or a QC report) that **must** be passed to the next stage. Because genomic files can be large, I ensure artifacts have a strict `expire_in` directive (e.g., `expire_in: 1 week`) to prevent GitLab storage bloat.

---

### Q2: You and a collaborator are both updating different modules of a variant prioritization suite. They push to the remote branch before you. How do you choose between `git pull --rebase` and a standard `git merge`, and what are the implications?

**Answer:**

- **`git pull --rebase`:** I use this when working on a local, unpushed feature branch and want to incorporate the main branch's latest updates. It temporarily removes my local commits, pulls the new remote commits, and replays mine on top. This creates a clean, linear history.

- **`git merge`:** I use this when integrating two shared or public branches. Merging creates a specific "merge commit" that documents exactly when the two lines of development diverged and joined. In clinical environments, preserving this exact historical context is often preferred for auditability, so I avoid rebasing public branches.

---

### Q3: You are running a GitLab CI job to validate an AMP/ASCO/CAP interpretation script. The job requires a specific, heavy Docker image and high memory. How do you configure the GitLab architecture to handle this without timing out?

**Answer:** Standard shared runners will likely time out or fail due to memory constraints. I would:

1. Register a custom GitLab Runner on a dedicated high-performance instance (or cloud VM)
2. Assign it specific tags (e.g., `high-mem`, `clinical-validation`)
3. In the `.gitlab-ci.yml` file, use the `tags:` keyword so the job is routed exclusively to that capable runner
4. Use the `image:` keyword to specify the exact pinned Docker container required for the execution environment

---

### Q4: A newly deployed Snakemake pipeline is suddenly failing in production. You need to find which specific commit introduced the bug. Walk me through your local Git debugging process.

**Answer:** If reviewing the recent Git log doesn't immediately reveal the issue, I use **git bisect**. It performs a binary search through the commit history:

1. `git bisect start` — start the bisect process
2. `git bisect bad` — tag the current broken commit
3. `git bisect good <commit_hash>` — tag the last known working commit
4. Git automatically checks out a commit halfway between the two
5. I run my pipeline tests; if it passes, I mark it `good`, and if it fails, I mark it `bad`
6. Git repeats this process, halving the search space each time until it pinpoints the exact commit that broke the pipeline

---

## CI/CD & Workflow Integration

### Q1: Git is notoriously bad at handling large binary files like BAM or VCF files. How do you version control the "golden" reference datasets used in your CI/CD concordance testing?

**Answer:** You should **never** commit large genomic files directly to the Git repository, as it will permanently bloat the `.git` folder and stall clone times. Instead, use:

- **Git LFS (Large File Storage)** or **DVC (Data Version Control)**

These tools replace the large files in the repository with tiny text pointers, while the actual heavy datasets are stored on an external server or cloud bucket (like AWS S3). When the GitLab CI pipeline runs, it checks out the code and uses those pointers to pull the exact versions of the reference datasets needed for that specific test run.

---

### Q2: Describe how you would design a CI/CD pipeline to automatically validate a newly updated machine learning model (e.g., for tumor primary site identification) before it is allowed to merge into the main production branch.

**Answer:** The pipeline must enforce clinical concordance before any code is merged. I would structure it in four distinct stages:

1. **Linting:** Checks Python code formatting and syntax to ensure standard compliance.

2. **Build:** Assembles the Docker image containing the updated ML model and its specific Conda environment.

3. **Test (The Gatekeeper):** The runner executes the model against a standardized cohort of synthetic patient data. It uses isotonic regression and influence diagnostics to compare the output against a known baseline. If the clinical concordance drops below the required threshold (e.g., 95%), the pipeline automatically fails, blocking the merge request.

4. **Deploy:** If tests pass, the validated image is pushed to the secure container registry.

---

### Q3: Your CI/CD pipeline needs to pull proprietary genomic annotations from a secured, external database during its run. How do you securely manage the API keys in GitLab?

**Answer:** API keys and credentials must **never** be hardcoded into the pipeline scripts or the `.gitlab-ci.yml` file. I store them in **GitLab CI/CD Variables** within the repository settings. I specifically mark these variables as:

- **"Masked"** (so they are scrubbed from the job logs if accidentally printed)
- **"Protected"** (so they are only accessible to pipelines running on protected branches or tags)

The script then accesses them securely as environment variables at runtime.

---

### Q4: If a buggy commit accidentally makes it into the main branch and corrupts the production variant reporting logic, how do you handle the rollback to maintain a clean clinical audit trail?

**Answer:** I would **absolutely avoid** using `git reset --hard` and force-pushing to main, as this erases history and violates the audit trail of what actually happened.

Instead, I use:

```bash
git revert <commit_hash>
```

This creates a brand new commit that perfectly reverses the changes introduced by the bug. The history remains intact, showing that the bug was introduced and then explicitly removed. Pushing this revert commit automatically triggers the CI/CD pipeline to redeploy the previously stable state.

---

## Nextflow Fundamentals

### Q1: What are the fundamental differences between Nextflow DSL1 and DSL2, and why is DSL2 considered essential for modern clinical pipelines?

**Answer:**

**DSL1 Architecture:** In DSL1, processes are strictly coupled to the workflow execution. A process inherently consumes a channel and outputs to another channel, making the script a single, monolithic flow. You cannot reuse a process (like a specific BWA-MEM alignment step) twice in the same script without copying and renaming the entire block of code.

**DSL2 Architecture:** DSL2 separates the definition of a process from its invocation. It introduces modularity. Processes, functions, and channels can be defined in separate `.nf` files (modules) and imported into a main workflow script.

**The Clinical Advantage:** DSL2 allows you to build a standardized, version-controlled library of modules. If you need to update a GATK HaplotypeCaller module to align with new AMP guidelines, you update it once in the module file, and every pipeline (e.g., both an HRD pipeline and a generic somatic mutation pipeline) that imports that module inherits the validated update. It dramatically reduces code duplication and validation overhead.

---

### Q2: Nextflow's `-resume` flag is critical for debugging long NGS pipelines. Explain exactly how Nextflow determines whether a process needs to be re-run or if it can use cached results.

**Answer:** Nextflow uses a cryptographic hashing mechanism. When a process runs, Nextflow generates a unique 128-bit MD5 hash based on four specific elements:

1. **Input variable values** (e.g., sample names, metadata)
2. **Complete contents and paths of input files** (e.g., BAM or FASTQ files)
3. **Complete text of the command script** executed in the process block
4. **Specific container or Conda environment** specified for that process

If all these elements remain identical in a subsequent run:
- Nextflow recognizes the hash
- Skips execution
- Simply pulls the outputs from the `work/` directory cache

If even a single character in the script or a single byte in an input file changes, the hash changes, and the process (along with downstream processes) is re-executed.

---

### Q3: You are running a somatic variant calling pipeline (Tumor/Normal pairing). How do you correctly structure the Nextflow channels to ensure the right tumor sample is matched with the correct normal sample?

**Answer:** You must **avoid relying on the order of files** in a directory, as channels process items asynchronously.

**The standard approach:** Create a tuple channel holding metadata alongside the file paths.

1. Start with a sample sheet (CSV) containing: `[Patient_ID, Tumor_FastQ, Normal_FastQ]`

2. Using the `splitCsv()` operator, read this file and structure a channel that emits tuples:
   ```
   tuple val(patient_id), path(tumor_fastq), path(normal_fastq)
   ```

3. When this channel is fed into the variant calling process, the `patient_id` acts as the key, guaranteeing that the tumor and normal samples are biologically paired, preventing any catastrophic sample swaps during execution.

---

### Q4: An alignment process frequently fails because it runs out of memory (OOM) on large oncology samples, but you don't want to blindly over-allocate RAM for every single sample. How do you handle this dynamically in Nextflow?

**Answer:** Nextflow handles this elegantly using dynamic resource allocation combined with the retry error strategy.

In the process directive, I configure:

```groovy
errorStrategy { task.exitStatus == 137 ? 'retry' : 'terminate' }
maxRetries 3
memory { 8.GB * task.attempt }
```

- **Exit status 137** is the standard Linux out-of-memory kill code
- If the process is killed for OOM, Nextflow catches it, increments the built-in `task.attempt` variable from 1 to 2, and resubmits the job requesting 16 GB of RAM instead of 8 GB
- This ensures small samples use minimal cluster resources, while heavy samples automatically scale up without manual intervention or pipeline failure

---

### Q5: What is the critical distinction between Tier IA and Tier IIC in the AMP guidelines?

**Answer:**

- **Tier IA** requires the biomarker to be FDA-approved or in professional guidelines for the patient's specific tumor type (e.g., BRAF V600E in Melanoma).

- **Tier IIC** is an FDA-approved therapy for a different tumor type (e.g., finding a BRAF V600E in Colorectal Cancer, which serves as an inclusion criterion for basket trials).

---

### Q6: What is ${projectDir} in Nextflow?

**Answer:** `${projectDir}` is a built-in variable pointing to the directory where the main Nextflow script (`.nf` file) resides. It is used to reference files bundled with the pipeline, such as scripts in the `bin/` directory.

**Example from pipelines:**
```groovy
script:
"""
bash ${projectDir}/bin/CNV.sh "${params.output_dir}" "${sample_file}"
"""
```

**Other important implicit variables:**

| Variable | Description |
|----------|-------------|
| `projectDir` | Directory containing the main .nf script |
| `launchDir` | Directory where nextflow run was invoked |
| `workDir` | Nextflow's work directory (default: ./work) |
| `baseDir` | Same as projectDir (deprecated in DSL2) |

**Note:** Scripts placed in the `bin/` directory of the project root are automatically added to the `$PATH` and can be called directly by name (e.g., `Rename_combined.py`) without the full path. This is why most Python scripts in production pipelines are called by name only.

---

### Q7: How does Nextflow handle the bin/ directory?

**Answer:** Any executable script placed in the `bin/` directory at the root of the Nextflow project is automatically added to the system PATH during execution. This means:

- You can call scripts by name without providing the full path
- Scripts must be executable (`chmod +x bin/*.py`)
- Scripts need a proper shebang line (`#!/usr/bin/env python3`)
- Line endings must be Unix-style (LF, not CRLF)

The pipeline's `setup.sh` typically handles all three requirements:

```bash
chmod +x bin/*.py                          # Make executable
sed -i 's/\r$//' bin/*.py 2>/dev/null      # Fix line endings
```

---

## Nextflow Workflows & Integration

### Q1: Compare Nextflow to Snakemake. In what scenarios would you choose Nextflow for a clinical-genomic database project over Snakemake?

**Answer:**

**Snakemake:**
- Operates on a "pull" paradigm (similar to GNU Make)
- You define the desired final output files, and Snakemake works backward to determine which rules to run
- Heavily Python-centric and excellent when workflows are highly file-dependent and you want native Python logic inside the rules

**Nextflow:**
- Operates on a Dataflow "push" paradigm
- Data passes through asynchronous channels, triggering processes as inputs become available

**When to choose Nextflow:**
Nextflow is vastly superior for production-level cloud deployments and enterprise orchestration. Its abstraction of infrastructure allows you to write the pipeline logic once, and run it locally, on a Slurm cluster, or on AWS Batch/Google Cloud Life Sciences simply by swapping a configuration profile. If the goal is a sovereign, institutionalized end-to-end pipeline that needs to scale to hundreds of patients securely, **Nextflow's architecture is the industry standard**.

---

### Q2: How do you separate the pipeline's core logic from the execution infrastructure (e.g., moving from a local test environment to a production AWS cluster)?

**Answer:** You achieve this strict separation using the `nextflow.config` file and the concept of **Profiles**.

- The `.nf` scripts contain only the clinical logic (inputs, outputs, tool execution)
- The `nextflow.config` file defines profiles like:
  - `standard` (for local development using Docker)
  - `slurm` (for an on-premise HPC, specifying queues and Conda paths)
  - `awsbatch` (specifying IAM roles, S3 buckets, and compute environments)

By executing:
```bash
nextflow run main.nf -profile awsbatch
```

The engine injects the cloud infrastructure directives at runtime, keeping the core pipeline code perfectly immutable and clinically validated across any compute environment.

---

### Q3: AMP/ASCO/CAP guidelines and CLIA audits require strict traceability of how a sample was processed. How does Nextflow help you generate this audit trail?

**Answer:** Nextflow has robust, built-in introspection tools that do not require extra scripting. By executing the run with specific flags:

```bash
nextflow run main.nf -with-report -with-timeline -with-trace
```

Nextflow automatically generates comprehensive HTML and text-based metrics.

**The Trace file is particularly critical for audits.** It logs every single process execution, capturing:
- Task hash
- Start and end timestamps
- CPU/memory utilization
- Exit statuses
- Exact computational node used

This provides a mathematically verifiable audit trail proving exactly how a variant call was generated.

---

### Q4: You have developed a custom container for complex variant detection (like your bees_cpu image). How do you integrate this directly into a Nextflow pipeline to ensure all users execute the exact same environment?

**Answer:** Rather than requiring users to manually pull and run the Docker image before executing the pipeline, I define the container directly inside the Nextflow script or config.

In the `nextflow.config` file, I enable the Docker engine and assign the image path to the specific process:

```groovy
docker {
    enabled = true
}

process {
    withName: 'CALL_COMPLEX_VARIANTS' {
        container = 'parthsanghavi017/bees_cpu:v1.2'
    }
}
```

---

## SNV Calling — Secondary Analysis

### Q1: What is an SNV (Single Nucleotide Variant)? How does it differ from an SNP?

**Answer:**

An **SNV** is a single base-pair change at a specific position in the genome. It is the most general term — it can be **somatic** (arising in tumor tissue) or **germline** (inherited).

An **SNP** (Single Nucleotide Polymorphism) is a **subset of SNVs**: it refers specifically to germline variants present at ≥1% frequency in the population. **All SNPs are SNVs, but not all SNVs are SNPs.**

---

### Q2: What variant callers are used in this pipeline for SNV calling?

**Answer:**

- **DRAGEN Enrichment (v3.9.5)** — the primary caller launched on Illumina BaseSpace. It performs alignment + variant calling in a single pipeline using the hg19-altaware-cnv-anchor reference. It produces hard-filtered VCFs.

- **FreeBayes** — used as an alternate/secondary variant caller for hotspot analysis. It is an open-source, haplotype-based Bayesian caller. In the pipeline, it is specifically run with:
  - `-F 0.008` (minimum allele frequency threshold of 0.8%)
  - `-t` (restricted to hotspot BED regions)
  - Targeted against `all_hotspot_variant_gs.bed`

---

### Q3: What is the difference between somatic and germline variant calling?

**Answer:**

| Feature | Somatic | Germline |
|---------|---------|----------|
| Origin | Acquired (tumor cells) | Inherited (all cells) |
| Allele Frequency | Low (often <10%) | ~50% (het) or ~100% (hom) |
| vc-type in DRAGEN | 1 (somatic mode) | 0 (germline mode) |
| Paired analysis | Often tumor-normal pair | Single sample |
| AF thresholds (this pipeline) | call: 5%, filter: 10% | No AF filtering |
| Clinical significance | Actionable cancer targets | Hereditary disease risk |

In this pipeline, the distinction is made by the `-F` / `-B` suffix in the sample name:
- `-F` → somatic (Forward/Fresh tissue)
- `-B` → germline (Blood/Normal)
- `-cf-` → cfDNA (liquid biopsy, uses lower AF thresholds: call 1%, filter 5%)

---

### Q4: What is a VCF file? Describe its key columns.

**Answer:** VCF (Variant Call Format) is the standard file format for storing sequence variants. It has:

- **Header lines** (start with ##): metadata about the file, reference genome, INFO/FORMAT field definitions, filters applied
- **Column header line** (starts with #):

```
#CHROM  POS  ID  REF  ALT  QUAL  FILTER  INFO  FORMAT  SAMPLE
```

---

### Q5: Explain allele frequency (AF) and how it is used in somatic variant filtering.

**Answer:**

**AF (Allele Frequency)** = Number of variant reads / Total reads at that position.

In somatic calling, AF is critical because tumor samples are often impure (mixed with normal tissue). The pipeline uses two thresholds:

- **vc-af-call-threshold:** Minimum AF to call a variant (5% for solid, 1% for cfDNA)
- **vc-af-filter-threshold:** Minimum AF to keep a variant after hard filtering (10% for solid, 5% for cfDNA)

cfDNA samples use lower thresholds because circulating tumor DNA is typically at very low allelic fractions (0.1%–5%).

---

### Q6: What is strand bias and why does it matter in variant calling?

**Answer:** Strand bias occurs when a variant is supported predominantly by reads from only one strand (forward or reverse). A real variant should be detected approximately equally on both strands.

The pipeline checks strand bias using FreeBayes' INFO fields:
- **SAF** (Supporting Allele Forward) — reads supporting ALT on forward strand
- **SAR** (Supporting Allele Reverse) — reads supporting ALT on reverse strand
- **SRF/SRR** — reference strand support

If `SAF == SAR`, the variant is labeled `NO_STRAND_BIAS`; otherwise `STRAND_BIAS`. Variants with strong strand bias are more likely to be artifacts (e.g., from library preparation or sequencing errors).

---

### Q7: What is a BED file and how is it used in targeted sequencing?

**Answer:** A BED file defines genomic intervals (regions of interest). Format:

```
chr1    11873    14409    DDX11L1
chr1    69090    70008    OR4F5
```

Columns: chromosome, start (0-based), end, name (optional).

In targeted sequencing:
- Used to restrict variant calling to capture kit regions only (e.g., `TarGT_First_v2_CDS_and_FEV2F2_GRCh37_30_Mar_23.bed`)
- Passed to DRAGEN as `target_bed_id` parameter
- Passed to FreeBayes with `-t` flag for hotspot calling
- Used by mosdepth for gene coverage calculation
- Used by bedtools for CNV annotation

---

### Q8: What is the purpose of hotspot analysis in this pipeline?

**Answer:** Hotspot analysis identifies variants at known clinically significant positions (e.g., EGFR L858R, BRAF V600E, KRAS G12D). The pipeline:

1. Runs FreeBayes at a very low AF threshold (0.8%) specifically on hotspot BED regions
2. Parses VCF output to extract read depth (DP), reference observation count (RO), alternate observation count (AO)
3. Calculates `AO_percentage = (AO / DP) * 100` per variant
4. Tags each variant with a gene name by intersecting POS with the appropriate capture kit's BED file
5. Checks for strand bias (SAF/SAR)
6. Exports results to Excel

This catches clinically actionable variants that might be below the primary caller's AF threshold.

---

### Q9: How is variant calling from GATK different from Samtools (pileup)?

**Answer:** This is a critical distinction between legacy bioinformatics and modern clinical pipelines.

**Samtools (mpileup/bcftools):** This uses a naive, position-based algorithm. It looks vertically down a column of aligned reads at a specific genomic coordinate and counts the bases. It trusts the alignment provided by BWA completely. If BWA misaligned a read around an indel, Samtools will likely call a false positive SNP.

**GATK (HaplotypeCaller / Mutect2):** This performs local de novo assembly. When it finds an active region (an area with high variation), it throws away the BWA alignment. It builds a De Bruijn graph to re-assemble the reads locally and mathematically determines the most likely haplotypes.

**The Verdict:** GATK is vastly superior for calling Indels and complex variants because it fixes alignment artifacts on the fly. Samtools is computationally lighter but prone to false positives in complex genomic regions.

---

### Q10: How can PCR bias or a false positive short variant call be detected in the VCF?

**Answer:** A VCF file contains incredibly rich metadata in the INFO and FORMAT fields. To spot artifacts without looking at the BAM file in IGV, you interrogate these metrics:

- **Strand Bias (FS or SOR):** If the variant is only seen on the forward strand reads and never on the reverse strand (or vice versa), it is almost certainly a sequencing artifact or PCR bias.

- **Read Position Bias (ReadPosRankSum):** Sequencing chemistry degrades at the ends of reads. If the alternate allele is always found at the extreme 3' or 5' end of the reads, it's likely a false positive.

- **Mapping Quality (MQ):** If the reads supporting the variant have terrible mapping quality compared to the reads supporting the reference allele, the variant might be mapped to a homologous pseudogene.

- **Depth (DP) and Allele Fraction (AF):** A variant with an allele fraction of 1% supported by only 2 reads in a high-background region is highly suspect unless validated by UMIs.

**Solution:** We apply **GATK VariantFiltration** (Hard Filtering based on these exact metrics) or **Machine Learning approaches** (like VQSR) to systematically filter out these false positives before they ever reach the clinical reporting stage.

---

### Q11: Why is Base Quality Score Recalibration (BQSR) a mandatory step in clinical pipelines, and how does it work?

**Answer:**

**The Problem:** The quality scores (Phred scores) assigned by the sequencing machine (like an Illumina NovaSeq) are mathematically biased. The machine might systematically overestimate base quality at the ends of reads or after specific di-nucleotide contexts (e.g., following a 'CG' motif). If left uncorrected, these systematic errors will be called as low-allele-frequency somatic mutations.

**The Solution:** BQSR uses machine learning to correct this. It takes the BAM file and a database of known, highly validated germline SNPs (like dbSNP or the 1000 Genomes Project). It assumes that any mismatch in the BAM file that is not in dbSNP is a sequencing error. It calculates the empirical error rate across various covariates (machine cycle, read position, nucleotide context) and builds a recalibration table.

**The Outcome:** It rewrites the BAM file with corrected, accurate Phred scores, drastically reducing the false-positive rate for variant calling.

---

### Q12: When building a somatic pipeline for clinical reporting, how does Mutect2 differ fundamentally from HaplotypeCaller, and why is a Panel of Normals (PoN) critical?

**Answer:**

**HaplotypeCaller vs. Mutect2:**
- **HaplotypeCaller** is designed for germline variants. It assumes standard ploidy (diploid) and expects allele frequencies around 50% (heterozygous) or 100% (homozygous).
- **Mutect2** is designed for somatic (cancer) variants. Tumors are highly heterogeneous and often contaminated with normal healthy cells. Mutect2 can detect subclonal mutations at extremely low allele fractions (e.g., 2% or 5%) without assuming diploidy.

**The Role of the PoN:** A Panel of Normals is an absolute necessity in oncology. It is a VCF generated by running Mutect2 on dozens of healthy, normal samples sequenced on the exact same pipeline and machines.

**Why it matters:** If a "variant" shows up in your tumor sample and also appears in the PoN, the pipeline instantly flags it as a machine-specific sequencing artifact or a common, uncharacterized germline SNP. **It is the strongest filter against pipeline-induced false positives.**

---

### Q13: Standard variant callers often miss Internal Tandem Duplications (ITDs) and Compound EGFR mutations. How do you adjust your secondary analysis to detect these complex variants?

**Answer:**

**The ITD Challenge:** Large ITDs (like FLT3-ITD, which is critical in leukemia) are often longer than the standard read length. Standard callers drop these reads because BWA-MEM cannot align a sequence that is massively duplicated within itself, resulting in a pile of unmapped or heavily soft-clipped reads.

**The Solution:** You cannot rely solely on standard VCF outputs. You must use specialized algorithms (like Pindel or specialized ITD-hunters) that specifically look for discordant read pairs and re-assemble the soft-clipped sequences (the parts of the read BWA gave up on) to span the duplication.

**Compound Mutations (e.g., EGFR):** If a patient has two different mutations in the EGFR gene, standard callers output two separate VCF lines. However, clinical response (like TKI resistance) depends on whether those mutations are in **cis** (on the same DNA strand) or in **trans** (on opposite strands). Secondary analysis must include read phasing (e.g., using GATK ReadBackedPhasing) to determine if a single read spans both variants, proving they are in cis.

---

## Short Variant Calling — Tertiary Analysis

### Q1: What is the fundamental difference in purpose between the ACMG/AMP and AMP/ASCO/CAP guidelines?

**Answer:**

- **ACMG/AMP (2015)** is designed for germline variants to determine **pathogenicity** (does this variant cause a heritable disease?). It uses a **5-tier system** (Pathogenic to Benign).

- **AMP/ASCO/CAP (2017)** is designed for somatic variants in oncology to determine **clinical actionability** (does this variant impact patient management, prognosis, or targeted therapy?). It uses a **4-tier system** (Tier I to Tier IV).

---

### Q2: Walk me through the 4 Tiers of the AMP/ASCO/CAP somatic guidelines.

**Answer:**

- **Tier I (Variants of Strong Clinical Significance):** FDA-approved therapies or included in professional guidelines (like NCCN) for the patient's specific tumor type.

- **Tier II (Variants of Potential Clinical Significance):** FDA-approved therapies for a different tumor type, or supported by well-powered studies/consensus expert panels.

- **Tier III (Variants of Unknown Clinical Significance - VUS):** Not observed at significant allele frequencies in general populations, no convincing evidence of altering protein function, no targeted therapies.

- **Tier IV (Benign/Likely Benign):** High allele frequency in populations (e.g., >1% in gnomAD) and no literature supporting cancer association.

---

### Q3: What are the 5 classification categories for ACMG/AMP germline interpretation?

**Answer:** Pathogenic (P), Likely Pathogenic (LP), Variant of Uncertain Significance (VUS), Likely Benign (LB), and Benign (B). These are calculated by combining evidence criteria (e.g., PVS1, BA1, PP3) that have assigned weights (Very Strong, Strong, Moderate, Supporting).

---

### Q4: What is the ClinGen/VICC/CGC harmonized somatic interpretation protocol, and why was it created?

**Answer:** The original AMP/ASCO/CAP guidelines lacked a formalized mechanism to evaluate the actual **oncogenicity** (cancer-causing nature) of a mutation separately from its clinical actionability. The **ClinGen/CGC/VICC framework (2022)** bridges this gap by creating a standardized scoring system for somatic oncogenicity (Oncogenic, Likely Oncogenic, VUS, Likely Benign, Benign), highly mirroring the ACMG germline logic but adapted for tumor biology.

---

### Q5: How do you handle reporting a Variant of Uncertain Significance (VUS) in a clinical oncology report versus a rare disease report?

**Answer:**

In rare genetic diseases, a VUS might be the only clue and is often reported to encourage family segregation studies or functional assays.

In somatic oncology (AMP/ASCO/CAP Tier III), a VUS is generally **kept out of the primary clinical actionability summary** because it cannot guide therapy, though it is usually listed in an appendix for completeness.

---

### Q6: When interpreting a tumor-only NGS panel, how do you differentiate a somatic mutation from a pathogenic germline variant without a matched normal?

**Answer:** Without a matched normal, absolute certainty is impossible. However, we infer germline status by looking at the **Variant Allele Frequency (VAF)**:

- If a variant is near 50% or 100%, and is known to be a common cancer susceptibility gene (e.g., BRCA1/2), it is flagged as a potential germline variant
- We cross-reference population databases (gnomAD); if it has a high population frequency, it's likely a benign germline SNP
- True somatic driver mutations often have variable VAFs depending on tumor purity and subclonal architecture

---

### Q7: Explain the ClinVar "Star Rating" system and how you programmatically weigh it in a variant prioritization suite.

**Answer:** The star rating indicates the level of review supporting the classification:

- **0 stars:** No assertion criteria provided
- **1 star:** Single submitter
- **2 stars:** Multiple submitters with no conflicts
- **3 stars:** Reviewed by an expert panel (VCEP)
- **4 stars:** Practice guideline

**Programmatic weighting:** In automated pipelines, 3 or 4-star variants can often bypass manual review. Conflicting interpretations (1 star) require manual curation or automated RAG/LLM extraction from primary literature.

---

### Q8: What is the difference in structural focus between ClinVar and CIViC?

**Answer:**

- **ClinVar** is primarily a repository of germline assertions (Pathogenic/Benign) with some somatic data, focused on genotype-phenotype relationships.

- **CIViC** (Clinical Interpretation of Variants in Cancer) is purpose-built for oncology. It structures data around clinical actionability, linking specific variants to specific drugs, evidence levels (A-E), and evidence types (Predictive, Prognostic, Diagnostic).

---

### Q9: In CIViC, what is the difference between Evidence Level A and Evidence Level C?

**Answer:**

- **Level A** indicates the variant-drug association is proven and established in medical practice (e.g., FDA approved or NCCN guidelines).

- **Level C** indicates the evidence comes from individual case studies or small observational cohorts.

---

### Q10: How do you resolve a "Conflicting Interpretations of Pathogenicity" status in ClinVar during routine clinical reporting?

**Answer:** You cannot rely on ClinVar alone. You must dig into the primary submissions. I check:

- The dates of the submissions (newer is generally better)
- The methodology of the submitters (did they use ACMG 2015 criteria?)
- Whether the conflicting submitter provided functional evidence

If unresolved, I resort to calculating the ACMG criteria de novo using population databases and in-silico predictors.

---

### Q11: What is a ClinGen VCEP, and why is their data considered the gold standard?

**Answer:** **VCEP** stands for **Variant Curation Expert Panel**. These are disease-specific working groups that modify the generic ACMG guidelines for a specific gene (e.g., the TP53 VCEP). They define exactly what constitutes a "loss of function" or what allele frequency threshold defines "Benign" for that specific gene. Their classifications are granted **3 stars in ClinVar**.

---

### Q12: When building a neoplasm-associated germline variant database, why can't you just write a script to download ClinVar once and use it forever?

**Answer:** Variant classifications are dynamic. A VUS today might be reclassified as Likely Pathogenic next month due to a new functional study. Clinical labs require rigorous database versioning. The pipeline must:

1. Pull the monthly ClinVar XML/VCF release
2. Diff it against the internal database
3. Automatically flag previously reported patient cases where a variant's classification has changed for re-reporting

---

### Q13: How would you extract structured information from CIViC to automate Tier I/II AMP/ASCO/CAP classifications?

**Answer:** I would query the **CIViC GraphQL API**. For a specific variant, I would filter for evidence items where:

- The disease matches the patient's primary tumor site
- The evidence type is "Predictive" (therapeutic response)
- The evidence level is A or B

This directly maps to Tier I and Tier II actionability.

---

### Q14: What is the critical distinction between Tier IA and Tier IIC in the AMP guidelines?

**Answer:**

- **Tier IA** requires the biomarker to be FDA-approved or in professional guidelines for the patient's specific tumor type (e.g., BRAF V600E in Melanoma).

- **Tier IIC** is an FDA-approved therapy for a different tumor type (e.g., finding a BRAF V600E in Colorectal Cancer, which serves as an inclusion criterion for basket trials).

---

### Q15: If you detect an EML4-ALK fusion in a lung adenocarcinoma sample, what evidence types must be included in the report?

**Answer:** It must be reported as a **Tier IA variant**. The report must state:

- **Predictive evidence:** Sensitivity to ALK inhibitors like Alectinib or Crizotinib
- **Diagnostic evidence:** Confirming NSCLC
- **Genomic breakpoints** (if relevant to therapy resistance)

---

### Q16: How do you handle a known oncogenic mutation that confers resistance to a therapy?

**Answer:** Resistance mutations (e.g., EGFR T790M following Erlotinib therapy) are **highly actionable**. They are classified as **Tier I** because they directly alter patient management (switching the patient to Osimertinib). The report must explicitly highlight this as "**Predictive - Resistance.**"

---

### Q17: What is the role of Biomarker/Therapeutic Response Indexes (TRI) in relation to standard guidelines?

**Answer:** Standard guidelines treat variants somewhat in isolation. **TRI approaches** look at the **composite genomic signature** (e.g., combining HRD scores, TP53 status, and MYC amplifications) to predict drug efficacy. While a specific component might only be Tier II, a validated TRI acts as a complex biomarker that requires rigorous clinical concordance testing against gold-standard cohorts before it can be reported.

---

### Q18: You find a pathogenic BRCA1 mutation in a breast cancer tumor-only sample. It is Tier I for PARP inhibitors. Do you also report it using ACMG germline guidelines?

**Answer:** **Yes, this is a dual-reporting scenario.** It is reported as:

- **Tier I somatic** for targeted therapy (PARP inhibitors)
- However, because BRCA1 is a highly penetrant cancer susceptibility gene, the report must include a **strong recommendation for orthogonal germline testing and genetic counseling**, noting that the mutation may be constitutional

---

### Q19: When curating clinical trials for Tier II/III variants, what filtering criteria must an automated pipeline apply to avoid overwhelming the oncologist?

**Answer:** The pipeline must not just string-match the gene name. It must filter by:

- The **specific variant** (or variant class, like "activating mutations")
- The patient's **specific tumor primary site**
- **Trial status** (must be "Recruiting" or "Active")
- Patient's **geographic location** or institution

---

## Variant Interpretation & Guidelines

### Q20: The PVS1 (Very Strong) criterion is for null variants (nonsense, frameshift, canonical splice sites). When is it inappropriate to apply PVS1 to a frameshift mutation?

**Answer:** PVS1 cannot be applied if:

1. The gene is not known to cause disease through a **loss-of-function mechanism** (e.g., if the disease is caused by gain-of-function)

2. The frameshift occurs in the **last exon** or the **3' end of the penultimate exon**, as it may escape **nonsense-mediated decay (NMD)** and result in a partially functional protein

---

### Q21: Explain the difference between BA1 (Stand-alone Benign) and BS1 (Strong Benign) regarding allele frequency.

**Answer:**

- **BA1** is applied when a variant has an exceptionally high allele frequency (>5%) in a large outbred population (like gnomAD), proving it is a benign polymorphism.

- **BS1** is applied when the allele frequency is greater than expected for the disorder (> baseline disease prevalence), but doesn't quite reach the massive >5% threshold

---

### Q22: How do you properly use in-silico predictors (PP3 / BP4) without overestimating their weight?

**Answer:** Tools like REVEL, CADD, or SIFT only predict the **impact on protein structure**, not clinical pathogenicity. Under ACMG, they can only ever provide "**Supporting**" evidence (PP3 for pathogenic, BP4 for benign). Furthermore, if multiple tools are used, they only count as **one piece of evidence**, because their underlying algorithms and training datasets heavily overlap.

---

### Q23: When building an LLM or RAG-based chatbot to assist with variant prioritization, what is the biggest risk regarding ACMG criteria, and how do you mitigate it?

**Answer:** The biggest risk is **hallucination of functional evidence (PS3/BS3) or patient segregation data (PP1)**. LLMs might conflate a paper mentioning a variant with a paper proving its function.

**Mitigation requires strict RAG:**
- Force the LLM to only extract data from retrieved PMC full-text XML
- Instruct it to cite the specific sentence
- Keep a **human-in-the-loop** (the clinical scientist) to verify the extracted text before applying the PS3 tag

---

### Q24: Tools like InterVar automate ACMG classifications. What are their primary limitations in a production clinical environment?

**Answer:** Automated tools are excellent at:
- Querying population databases for BA1/PM2
- Running in-silico tools for PP3

However, they consistently fail at:
- Evaluating functional assays (PS3)
- Determining if a variant is de novo (PS2)
- Reading complex literature for segregation data
- They often default to "VUS" for variants that a human curator would easily classify as Pathogenic based on recent literature

---

### Q25: If you are developing a sovereign, end-to-end pipeline to interpret variants, what must you do to validate the tertiary analysis component for a CLIA environment?

**Answer:** The pipeline's automated classification logic must be tested against a **validated truth set** (e.g., hundreds of previously reported cases from a CAP-accredited lab). You must prove **clinical concordance**—calculating sensitivity, specificity, and reproducibility.

If the pipeline assigns Tier I/II classifications, it must do so **accurately >95% of the time** compared to the human expert baseline, and all discrepancies must be mathematically or biologically justified through influence diagnostics.

---

## RNA-seq Analysis

### Q1: What is the key difference between raw counts, RPKM, FPKM, and TPM, and explain them mathematically?

**Answer:**

**Raw counts** simply represent the absolute number of reads mapping to a gene. They cannot be used directly to compare expression between different genes or different samples because they do not account for sequencing depth (total reads) or gene length.

**RPKM (Reads Per Kilobase of transcript, per Million mapped reads):** Used for single-end RNA-seq. It normalizes for sequencing depth and gene length.

$$RPKM = \frac{N \cdot L}{C \cdot 10^9}$$

Where C is the number of mapped reads to the gene, N is the total mapped reads in the experiment, and L is the length of the transcript in base pairs.

**FPKM (Fragments Per Kilobase of transcript, per Million mapped reads):** Conceptually identical to RPKM, but used for paired-end sequencing where two reads represent a single DNA/RNA fragment.

**TPM (Transcripts Per Million):** The modern standard. It normalizes for gene length before normalizing for sequencing depth. This ensures that the sum of all TPMs in every sample is exactly identical (10⁶), making it the only metric suitable for comparing the relative expression of a gene across different samples.

$$TPM_i = \left(\frac{C_i / L_i}{\sum_j (C_j / L_j)}\right) \cdot 10^6$$

---

### Q2: What is the key algorithmic difference between DESeq2, edgeR, and Limma?

**Answer:**

| Feature | DESeq2 | edgeR | Limma (voom) |
|---------|--------|-------|-------------|
| **Statistical Model** | Negative Binomial distribution | Negative Binomial distribution | Linear models (Gaussian distribution) |
| **Normalization** | Median of Ratios | Trimmed Mean of M-values (TMM) | TMM or Quantile normalization |
| **Variance Handling** | Estimates dispersion by borrowing information across genes; uses empirical Bayes shrinkage for fold changes | Uses empirical Bayes methods to moderate gene-specific dispersions towards a common dispersion | voom function transforms RNA-seq counts to log2-cpm and estimates the mean-variance relationship |
| **Best Use Case** | Small sample sizes (n < 10) where variance estimation is difficult | Similar to DESeq2, slightly more sensitive to outliers | Large sample sizes, complex experimental designs, or integrating RNA-seq with microarray data |

---

### Q3: What normalization technique is used in DESeq2?

**Answer:** DESeq2 uses the **Median of Ratios method**. It assumes that the majority of genes are not differentially expressed between samples. It accounts for library size and RNA composition bias.

**Mathematical workflow:**

1. **Create a pseudo-reference sample:** Calculate the geometric mean for each gene across all samples
   $$\text{pseudo-ref}_i = \left(\prod_{v=1}^m K_{iv}\right)^{1/m}$$

2. **Calculate ratios:** For every gene in a sample, calculate the ratio of its read count to the pseudo-reference

3. **Determine the size factor:** Calculate the median of these ratios for a given sample. This median is the size factor $s_j$
   $$s_j = \text{median}_i\left(\frac{\text{pseudo-ref}_i}{K_{ij}}\right)$$

4. **Normalize:** Divide the raw counts of the sample by this size factor

---

### Q4: What is bulk deconvolution and what information does it provide?

**Answer:** Bulk RNA-seq measures the average gene expression across an entire tissue (like a solid tumor biopsy), which is a complex mixture of cancer cells, immune cells, and fibroblasts. **Bulk deconvolution** is a computational technique (using tools like CIBERSORT or EPIC) that applies linear regression algorithms to estimate the relative proportions of distinct cell types within that bulk mixture.

It relies on a **"signature matrix"**—a reference panel of gene expression profiles for purified cell types (often derived from single-cell RNA-seq).

In oncology, it provides critical information about the **tumor microenvironment**, such as the fraction of infiltrating CD8+ T-cells, which is predictive of a patient's response to immunotherapy.

---

### Q5: What is the difference between STAR and HISAT2 aligners for RNA-seq?

**Answer:** Both are **splice-aware aligners**, which is critical for RNA-seq because reads can span exon-exon junctions (introns are spliced out).

**STAR (Spliced Transcripts Alignment to a Reference):**
- Extremely fast and highly accurate
- Performs uncompressed suffix array searches and can discover novel splice junctions de novo
- Incredibly memory-intensive (often requiring >30GB of RAM for the human genome)

**HISAT2:**
- Uses a graph-based alignment approach and hierarchical FM-indexes
- Significantly more memory-efficient (running on as little as 4GB-8GB of RAM)
- Slightly less sensitive to novel splicing events than STAR

---

### Q6: Why do we use pseudo-aligners like Salmon or Kallisto instead of traditional aligners in some pipelines?

**Answer:** Traditional aligners (STAR/HISAT2) perform base-by-base mapping to determine exact genomic coordinates, which generates massive BAM files.

**Pseudo-aligners (Salmon/Kallisto)** do not perform base-level alignment. Instead, they use **k-mer hashing** to determine which transcript a read likely originated from. This approach is:

- Exponentially faster
- Requires a fraction of the computational resources
- Directly outputs transcript-level quantifications

They are the **standard for standard differential expression**, whereas traditional aligners are still required if you need to perform variant calling or novel splice site discovery.

---

### Q7: How do you detect and handle ribosomal RNA (rRNA) contamination in secondary analysis?

**Answer:** rRNA makes up about 80% of total cellular RNA. If library depletion protocols fail, rRNA reads will consume the sequencing depth.

During secondary QC, I map a subset of reads to a database of human rRNA sequences using a fast aligner like Bowtie2. If the rRNA mapping rate is unusually high (e.g., >10%), it indicates a **library preparation failure**.

In the pipeline, these reads are **strictly filtered out** before quantification because they skew the normalization algorithms used in differential expression.

---

### Q8: Explain the significance of the RIN (RNA Integrity Number) and how it affects downstream analysis.

**Answer:** **RIN** is a metric (from 1 to 10) generated by instruments like the Bioanalyzer that assesses RNA degradation:

- RIN of 10 = fully intact RNA
- RIN of 2 = highly degraded RNA

RNA degrades unevenly (often from the 5' end). If you sequence low-RIN samples, you will see a **massive 3' bias** in your transcript coverage. This skews **transcript length normalization** and causes **false-positive differential expression results** if high-RIN and low-RIN samples are compared directly.

---

### Q9: What is the difference between a stranded and unstranded RNA-seq library, and why does it matter?

**Answer:**

**Unstranded libraries:** You know a read came from a specific genomic locus, but you do **not** know which of the two DNA strands it was transcribed from.

**Stranded libraries:** The biochemistry preserves the strand information.

This is critical in the human genome because **many genes overlap on opposite strands**. With unstranded data, a read in an overlapping region cannot be confidently assigned to either gene. **Stranded data** allows the quantification software (like featureCounts) to accurately assign the read to the correct gene based on its directionality.

---

### Q10: In differential expression, what is the problem with multiple testing, and how is it corrected?

**Answer:** A standard human RNA-seq experiment tests roughly **20,000 genes** for differential expression simultaneously. If you use a standard p-value threshold of 0.05, you will expect **1,000 false positives purely by chance** (20,000 × 0.05).

This is the **multiple testing problem**.

To correct this, we use the **Benjamini-Hochberg (BH) procedure** to calculate the **False Discovery Rate (FDR)** or adjusted p-value ($p_{adj}$). A $p_{adj}$ of 0.05 ensures that **across all genes deemed significant, only 5% are expected to be false discoveries**.

---

### Q11: How do you handle batch effects in RNA-seq data, and what tools do you use?

**Answer:** **Batch effects** occur when non-biological factors (e.g., sequencing samples on different days, using different lots of reagents) cause systemic shifts in the data.

I first **detect** them using **Principal Component Analysis (PCA)**; if samples cluster by processing date rather than by biological condition, a batch effect exists.

I handle them by **including the batch as a covariate** directly in the DESeq2 design formula:
```R
design = ~ batch + condition
```

If I need to extract batch-corrected normalized counts for downstream machine learning, I use tools like **ComBat-seq**.

---

### Q12: Explain Gene Set Enrichment Analysis (GSEA) versus Over-Representation Analysis (ORA).

**Answer:**

**ORA:** Requires a hard threshold. You take a list of strictly differentially expressed genes (e.g., $p_{adj} < 0.05$ and $|\log_2FC| > 1$) and test via a hypergeometric distribution if a specific pathway (like apoptosis) is statistically over-represented in that short list compared to the background genome.

**GSEA:** Does not use a hard threshold. You rank all genes in the experiment by their fold change or p-value. GSEA calculates a running sum statistic to see if the genes defining a specific pathway aggregate at the very top or bottom of your ranked list.

**GSEA is superior** for detecting pathways where many genes change by a small, sub-threshold amount.

---

### Q13: How is RNA-seq used for fusion gene detection in oncology, and what algorithms are utilized?

**Answer:** Unlike DNA-seq where introns can make fusion breakpoints difficult to find, RNA-seq directly captures the **transcribed fused exons**.

Algorithms like **STAR-Fusion** or **Arriba** map reads to the genome and specifically search for:
- Discordant read pairs
- Chimeric junction-spanning reads

These tools then filter the candidates against databases of known oncogenic fusions and artifacts (like read-through transcripts) to identify clinically actionable targets, such as **NTRK or ALK fusions**.

---

### Q14: What is the MA plot, and what does it tell you about your differential expression results?

**Answer:** An MA plot visualizes differential expression by plotting the **Log2 Fold Change (M)** on the y-axis against the **Mean Normalized Expression (A)** on the x-axis for every gene.

It immediately shows:
- If normalization was successful (the bulk of the genes should center tightly around y=0)
- The mean-variance relationship: genes with low mean expression (far left) typically show massive, noisy fold-changes (wide spread on the y-axis)

---

### Q15: Why is log2 fold change (LFC) shrinkage important in differential expression analysis?

**Answer:** As seen on an MA plot, genes with very low read counts suffer from high statistical noise, often producing exaggerated and biologically meaningless log2 fold changes (e.g., a change from 1 read to 4 reads is an LFC of 2).

**LFC shrinkage algorithms** (like apeglm used in DESeq2) use empirical Bayes techniques to shrink these noisy, low-count fold changes toward zero, while leaving the fold changes of highly expressed genes untouched. This ensures that **downstream ranking of genes by LFC is driven by robust biological signals** rather than statistical artifacts.

---

### Q16: How do you approach calling single nucleotide variants (SNVs) from RNA-seq data, and what are the primary caveats?

**Answer:** Calling SNVs from RNA is complex because RNA undergoes editing, splicing, and allele-specific expression. The standard pipeline requires:

1. Mapping with a **splice-aware aligner** (STAR 2-pass method)
2. **Splitting reads** that span introns (using GATK SplitNCigarReads)
3. **Recalibrating base qualities**
4. Using **GATK HaplotypeCaller**

**Main caveats:**

- **RNA Editing:** Enzymes (like ADAR) naturally alter RNA sequences (e.g., A-to-I editing), creating false-positive DNA SNV calls

- **Coverage bias:** You can only call variants in genes that are actively expressed. A true driver mutation in a silenced gene will be invisible in RNA-seq

---

## Single-Cell & Spatial Transcriptomics

### Q1: What is the fundamental difference between scRNA-seq and spatial transcriptomics?

**Answer:**

**scRNA-seq** requires the mechanical and enzymatic dissociation of tissue into a suspension of individual cells. This provides:
- True single-cell resolution
- Massive cellular heterogeneity
- **Completely destroys histological context** (where the cell was located in the tumor)

**Spatial transcriptomics** preserves the 2D architecture of the tissue slice. However, standard spatial tech (like 10x Visium) operates on **"spots"** rather than single cells, meaning a single spatial barcode might capture mRNA from 3 to 10 overlapping cells, requiring computational deconvolution later.

---

### Q2: What are the primary normalization techniques available in Seurat?

**Answer:** Seurat primarily relies on two approaches:

**LogNormalize (Standard):** Divides feature counts for each cell by the total counts for that cell, multiplies by a scale factor (default $10^4$), and applies a natural log transformation:
$$\ln(1 + \text{normalized\_counts})$$

**SCTransform (Advanced):** Replaces the standard NormalizeData, ScaleData, and FindVariableFeatures steps. It uses regularized negative binomial regression to model technical noise (like sequencing depth). It effectively removes the influence of technical variation while preserving biological variance, making it vastly superior for complex datasets.

---

### Q3: In terms of memory usage, how are Scanpy and Seurat fundamentally different?

**Answer:**

**Seurat (R-based):** Loads the entire dataset into active RAM. For massive datasets (>100k cells), Seurat quickly exhausts standard workstation memory, leading to crashes unless run on a high-memory HPC node.

**Scanpy (Python-based):** Built entirely around the **AnnData (Annotated Data)** object structure, which can be backed by HDF5 files (`.h5ad` format) on disk. Scanpy can stream data directly from the hard drive, processing data in chunks. This makes it vastly more memory-efficient, allowing you to process datasets of a million cells on standard hardware.

---

### Q4: Explain how droplet-based scRNA-seq (like 10x Genomics) isolates and barcodes individual cells.

**Answer:** It uses microfluidics to encapsulate three things into an oil droplet:

1. A single cell
2. A gel bead covered in barcoded primers
3. Lysis buffer

Inside this microscopic droplet:
- The cell lyses
- The mRNA binds to the bead's primers
- Reverse transcription occurs

Each primer contains:
- A **Cell Barcode** (identifying which cell the RNA came from)
- A **UMI** (Unique Molecular Identifier, identifying the specific RNA transcript)

This allows bulk sequencing later while tracing every read back to its exact cell of origin.

---

### Q5: What are the three standard Quality Control (QC) metrics used to filter cells in scRNA-seq?

**Answer:**

- **Total UMI counts per cell:** Extremely low counts indicate an empty droplet (just ambient RNA); extremely high counts indicate a multiplet (multiple cells)

- **Number of unique features (genes) per cell:** Measures library complexity

- **Percentage of mitochondrial reads:** High mitochondrial percentage indicates a dying or ruptured cell

---

### Q6: Why is mitochondrial read percentage a critical QC metric in single-cell analysis?

**Answer:** When a cell is stressed, apoptotic, or physically ruptured during the dissociation process, its cell membrane breaks and cytoplasmic mRNA leaks out. However, **mitochondrial mRNA is protected** inside the double membrane of the mitochondria.

Therefore, a droplet containing a dead cell will sequence a **disproportionately massive percentage of mitochondrial genes** compared to nuclear genes.

---

### Q7: How do you identify and handle doublets (or multiplets) computationally?

**Answer:** A **doublet** occurs when two cells are encapsulated in one droplet. You can sometimes spot them manually if a cell expresses mutually exclusive markers (e.g., both CD3 for T-cells and CD19 for B-cells).

Computationally, tools like **DoubletFinder** or **Scrublet** are used:

1. Generate artificial doublets by randomly combining real single-cell profiles
2. Project them into the PCA/UMAP space
3. Calculate the proportion of artificial doublets in the neighborhood of each real cell to flag and filter true doublets

---

### Q8: Explain the difference between PCA, t-SNE, and UMAP in the context of single-cell visualization.

**Answer:**

**PCA (Principal Component Analysis):** 
- Linear reduction
- Captures global variance
- Cannot resolve complex, non-linear biological manifolds (like developmental trajectories)

**t-SNE:** 
- Non-linear
- Excellent at grouping similar cells into distinct clusters
- Distance between different clusters is **meaningless** (loses global topology)

**UMAP (Uniform Manifold Approximation and Projection):** 
- Non-linear
- Computationally faster than t-SNE
- **Preserves both local clustering and global topology**, meaning the relative distances between clusters actually represent biological similarity

---

### Q9: How does the Louvain or Leiden algorithm work for clustering single-cell data?

**Answer:** They are **graph-based clustering algorithms**:

1. Seurat builds a **K-Nearest Neighbor (KNN)** graph in PCA space, connecting cells with similar expression profiles

2. Louvain/Leiden iteratively group cells into **"communities"** to maximize a metric called **modularity**—ensuring that:
   - Connections within a cluster are very dense
   - Connections between different clusters are very sparse

**Leiden is generally preferred** as it guarantees well-connected communities, whereas Louvain can sometimes leave disconnected sub-communities.

---

### Q10: What is the "resolution" parameter in clustering algorithms, and how do you determine the optimal value?

**Answer:** The **resolution parameter** dictates the granularity of the downstream clustering:

- **Higher resolution** (e.g., 1.2) forces the algorithm to split the data into many small, highly specific sub-clusters
- **Lower resolution** (e.g., 0.2) yields fewer, broader clusters (like just "T-cells" vs. "Macrophages")

You optimize it using tools like **clustree** to visualize cluster stability across different resolutions or by manually checking if sub-clusters possess biologically distinct marker genes.

---

### Q11: What is a "batch effect" in scRNA-seq, and how does Seurat's integration workflow correct for it?

**Answer:** A **batch effect** is technical variation (e.g., samples processed on different days or different sequencing runs) that causes identical cell types to cluster separately.

**Seurat corrects this using Canonical Correlation Analysis (CCA)**:

1. Identifies **"anchors"**—pairs of cells across batches that are mutually nearest neighbors in this shared space
2. Calculates a **correction vector** to physically pull the datasets together
3. Aligns identical cell types across batches

---

### Q12: How does Harmony differ from Seurat's anchor-based integration?

**Answer:**

**Seurat's integration** calculates a massive pairwise distance matrix to find anchors, which is extremely computationally heavy and scales poorly with huge datasets.

**Harmony** uses a faster, iterative approach:

1. Projects all batches into a shared PCA space
2. Calculates a clustering objective function that **penalizes batch-specific clusters**
3. Applies a soft correction to the cell coordinates
4. Repeats until batches are thoroughly mixed without losing cell-type distinctions

It runs much faster and uses less memory.

---

### Q13: How do you approach annotating cell clusters after dimensionality reduction?

**Answer:**

**Manual Annotation:**
- Running `FindAllMarkers` to identify the most upregulated genes in a cluster
- Comparing them to known biological literature (e.g., MS4A1 for B-cells)

**Automated Reference Mapping:**
- Using tools like **SingleR** or **Seurat's MapQuery**
- These take a gold-standard annotated reference atlas (like Azimuth)
- Automatically transfer cell-type labels onto your query dataset based on transcriptomic similarity

---

### Q14: Why is traditional bulk RNA-seq differential expression (like DESeq2) not always suitable for single-cell data?

**Answer:** Bulk RNA-seq algorithms assume **continuous negative binomial distributions**. scRNA-seq data is highly **sparse and heavily zero-inflated** (due to technical "dropouts" where transcripts are missed).

While specialized models like **MAST** account for this zero-inflation, the **default standard** in single-cell analysis is **non-parametric testing**, specifically the **Wilcoxon Rank Sum test**. It compares the rank of expression values between two clusters rather than assuming a specific distribution, making it highly robust to the noise of single-cell data.

---

### Q15: How does the 10x Visium spatial transcriptomics technology map transcripts back to tissue histology?

**Answer:** A thin slice of fresh frozen or FFPE tissue is placed onto a specialized slide printed with thousands of microscopic spots. Each spot contains oligos with a specific **Spatial Barcode**.

The tissue is:
1. Imaged (H&E stain)
2. Permeabilized
3. mRNA falls directly onto the spots below

During sequencing, every transcript is tagged with that spot's spatial barcode. The computational pipeline then uses those barcodes to **map the transcript counts perfectly back to the X-Y coordinates of the H&E image**.

---

### Q16: What is the limitation of "spot resolution" in spatial transcriptomics, and how is it addressed computationally?

**Answer:**

**The limitation:** Standard Visium spots are 55 μm in diameter, meaning they encompass 1 to 10 overlapping cells depending on tissue density. It is **not** true single-cell resolution.

**To address this**, we use **Spatial Deconvolution tools** (like Cell2location, RCTD, or Seurat's transfer anchors). These algorithms use an **annotated scRNA-seq reference dataset** to estimate the proportional mixture of different cell types within a single spatial spot.

---

### Q17: What are the emerging "true" single-cell spatial technologies, and how do they differ from Visium?

**Answer:** Technologies like **Xenium (10x Genomics), CosMx (NanoString), and MERSCOPE (Vizgen)** bypass the "spot" limitation entirely. They do not rely on standard NGS sequencing. Instead, they use:

**Highly multiplexed in situ hybridization (smFISH):** Using fluorescent probes to image individual transcripts directly inside intact cells on the slide.

This provides:
- True sub-cellular resolution
- Exact cell boundaries
- Currently limited to targeted panels of hundreds or thousands of genes (unlike the whole-transcriptome capture of Visium)

---

## Bash Scripting Essentials

### Q1: What does `#!/usr/bin/env bash` mean at the top of a script? How is it different from `#!/bin/bash`?

**Answer:**

- `#!/bin/bash` — hardcoded path; assumes bash is at `/bin/bash`

- `#!/usr/bin/env bash` — uses the `env` command to find bash in the user's `$PATH`; more portable across systems where bash may be installed in different locations (e.g., `/usr/local/bin/bash` on macOS with Homebrew)

**The env form is considered best practice** for scripts that need to run across different environments.

---

### Q2: Explain the difference between $() and backticks \` \` for command substitution.

**Answer:** Both perform command substitution (capture the output of a command):

```bash
result=$(ls -l)    # Modern form
result=`ls -l`     # Legacy form
```

| Feature | $() | Backticks |
|---------|-----|-----------|
| Nesting | Easy: `$(cmd1 $(cmd2))` | Painful: \`cmd1 \\`cmd2\\` \` |
| Readability | Clear | Easy to confuse with single quotes |
| Escaping | Standard rules | Extra backslash needed |
| POSIX compliant | Yes | Yes (but discouraged) |

**Always prefer $()**

---

### Q3: What is the difference between > and >> in shell redirection?

**Answer:**

- `>` — overwrites the file (truncates to zero then writes)
- `>>` — appends to the file (adds to the end)

```bash
echo "first"  > file.txt    # file contains: "first"
echo "second" > file.txt    # file contains: "second" (overwrites)
echo "third"  >> file.txt   # file contains: "second\nthird" (appends)
```

**Also important:**
- `2>` — redirects stderr
- `&>` or `2>&1` — redirects both stdout and stderr
- `2>/dev/null` — silences errors

---

### Q4: What does `set -e` do and why is it important in bioinformatics pipelines?

**Answer:** `set -e` makes the script exit immediately if any command returns a non-zero exit status (i.e., fails).

Without it, a failed step (e.g., a failed alignment) would be silently ignored and the pipeline would continue with missing/corrupt data.

**Common safety flags:**
```bash
set -euo pipefail
```

- `-e` — exit on error
- `-u` — treat unset variables as errors
- `-o pipefail` — a pipeline fails if any command in the pipe fails (not just the last one)

**This is critical** in bioinformatics because silent failures can lead to incorrect clinical results.

---

### Q5: How do you iterate over lines in a file using a while loop?

**Answer:**

```bash
while IFS=',' read -r sample_id project_name; do
    echo "Processing Sample: ${sample_id}, Project: ${project_name}"
    # ... do work ...
done < "$input_file"
```

**Key points:**
- `IFS=','` — sets the Internal Field Separator to comma (for CSV parsing)
- `read -r` — `-r` prevents backslash interpretation
- `< "$input_file"` — redirects file into the loop's stdin
- **Always quote variables** (`"$sample_id"`) to handle spaces/special chars

---

### Q6: Explain pipes and how they work in the CNV annotation command from this pipeline.

**Answer:**

A typical CNV annotation command chains multiple tools:

```bash
bedtools intersect -a ${sample_id}/${sample_id}.bam_CNVs \
    -b /path/to/FE_V2_CDS.bed -loj | \
    sort -V | \
    awk -F"\t" '{print $1"\t"$2"\t"$3"\t"$4"\t"$5"\t"$9}' | \
    awk -vOFS="\t" '$1=$1; BEGIN { str="Chromosome Start End Predicted_copy_number Type_of_alteration Gene"; \
    split(str,arr," "); for(i in arr) printf("%s\t", arr[i]); print}' | \
    awk '$6 != "."' > ./${sample_id}_cnv_combined.txt
```

**Step by step:**
1. `bedtools intersect -loj` — left outer join: for each CNV, find overlapping genes from the BED file
2. `sort -V` — version sort (natural chromosome ordering: chr1, chr2, ..., chr10)
3. `awk #1` — extract columns: chr, start, end, copy_number, alteration_type, gene_name
4. `awk #2` — add a header row
5. `awk #3` — filter out rows where gene is . (no annotation)
6. `>` — write to output file

**Each | (pipe)** sends the stdout of one command to the stdin of the next.

---

### Q7: Write a bash script that takes a CSV file and a directory as arguments, creates an output folder, and loops over samples.

**Answer:**

```bash
#!/usr/bin/env bash

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <location> <csv_file>"
    exit 1
fi

location=$1
csv_file=$2
output_dir="$location/results"

# Create directory if it doesn't exist
mkdir -p "$output_dir"

# Extract sample IDs (skip header, get 3rd-from-last column)
awk -F',' 'BEGIN{OFS=","} {if(NR>1) print $(NF-2),$4}' \
    "$csv_file" > "$output_dir/list.txt"

input="$output_dir/list.txt"
cd "$output_dir" || exit 1

while IFS=',' read -r sample_id project_name; do
    echo "Processing Sample: ${sample_id}"
    mkdir -p "${sample_id}"
    # ... processing steps ...
done < "$input"

echo "All files done."
```

---

### Q8: Write a command to find all .vcf.gz files in a directory, move them to a target folder, and clean up empty directories.

**Answer:**

```bash
# Find and move VCF files
find "$output_dir" -type f -name "*.vcf.gz" -exec mv {} "$output_dir" \;

# Remove empty directories matching a pattern
find . -type d -name "*_ds*" -exec rm -rf {} \;
```

---

### Q9: Write an awk one-liner to print columns where a field matches a pattern.

**Answer:**

```bash
# Print rows where column 3 is "somatic" and column 5 matches "CE"
awk -F',' '$3 == "somatic" && $5 == "CE" {print $0}' samples.csv

# Count samples per capturing kit
awk -F',' 'NR>1 {count[$3]++} END {for (kit in count) print kit, count[kit]}' samples.csv
```

---

### Q10: Write a bash snippet that checks if a file exists and is non-empty before processing.

**Answer:**

```bash
#!/usr/bin/env bash

bam_file="/path/to/sample.bam"

if [ ! -f "$bam_file" ]; then
    echo "ERROR: BAM file not found: $bam_file"
    exit 1
fi

if [ ! -s "$bam_file" ]; then
    echo "ERROR: BAM file is empty: $bam_file"
    exit 1
fi

echo "Processing $bam_file ..."
samtools index "$bam_file"
```

**Key test operators:**
- `-f` — file exists and is a regular file
- `-d` — directory exists
- `-s` — file exists and is non-empty
- `-r` / `-w` / `-x` — readable / writable / executable

---

### Q11: Write a bash function that decompresses .gz files for a list of samples.

**Answer:**

```bash
#!/usr/bin/env bash

decompress_thresholds() {
    local sample_list="$1"
    
    while IFS= read -r sample; do
        threshold_file="${sample}.thresholds.bed.gz"
        if [ -f "$threshold_file" ]; then
            echo "Decompressing $threshold_file"
            gunzip "$threshold_file"
        else
            echo "WARNING: $threshold_file not found, skipping"
        fi
    done < "$sample_list"
}

# Usage
decompress_thresholds "sample_ids.txt"
```

---

### Q12: Write a sed command to fix Windows line endings (CRLF → LF).

**Answer:**

```bash
# Remove carriage returns from all Python scripts
sed -i 's/\r$//' bin/*.py

# Check if a file has Windows line endings
file bin/CNV.sh    # will say "with CRLF line terminators" if present

# Alternative using dos2unix
dos2unix bin/*.py
```

---

## CNV Calling

### Q1: What is a CNV (Copy Number Variation)?

**Answer:** A **CNV** is a structural variant where a segment of DNA is duplicated (gain) or deleted (loss) compared to the reference genome. CNVs are typically ≥1 kb in size.

---

### Q2: What tool does this pipeline use for CNV calling and how does it work?

**Answer:** The pipeline uses **Control-FREEC v11.6** (Control-FREE Copy number caller).

**How Control-FREEC works:**

1. **Read count normalization** — counts reads in windows across the genome
2. **GC content correction** — normalizes for GC bias in library preparation
3. **Segmentation** — uses a LASSO-based algorithm to identify regions of consistent copy number
4. **Ploidy estimation** — estimates overall sample ploidy
5. **Copy number prediction** — assigns integer copy numbers to each segment

**Key config parameters** (from `cnv_config_somatic.pl`):
- `ploidy = 2` — Expected ploidy
- `intercept = 1` — Use intercept in regression
- `minMappabilityPerWindow = 0.7` — Skip low-mappability regions
- `breakPointType = 2` — Use both read count and BAF
- `degree = 3` — Polynomial degree for GC normalization
- `coefficientOfVariation = 0.05` — Noise threshold
- `breakPointThreshold = 0.6` — Sensitivity for breakpoints
- `maxThreads = 10` — Parallelism
- `noisyData = TRUE` — Handle noisy WES data

---

### Q3: What is a CNV baseline and why is it needed?

**Answer:** A **CNV baseline** (Panel of Normals / PoN) is a set of normal samples processed identically to the tumor samples. It captures:

- Systematic biases (GC content, mappability, capture kit artifacts)
- Batch effects (reagent lot variations)
- Region-specific noise (poor-performing capture probes)

The pipeline uses DRAGEN's CNV baseline for certain panels:
```
cnv_baseline = "cnv-baseline-id:26595964844,26596768352,..."  # ~49 normal samples for CE
cnv_baseline_se8 = "cnv-baseline-id:25791243964,..."          # ~19 normal samples for SE8
```

By subtracting the baseline signal, you get a cleaner copy-number profile that reflects true biological variation rather than technical noise.

---

### Q4: How are CNVs annotated with gene names in this pipeline?

**Answer:** After Control-FREEC produces raw CNV calls (`.bam_CNVs`), the pipeline uses BEDTools intersect to annotate them:

```bash
bedtools intersect \
    -a ${sample_id}/${sample_id}.bam_CNVs \
    -b /path/to/FE_V2_CDS.bed \
    -loj | \
    sort -V | \
    awk ... | \
    awk '$6 != "."' \
    > ${sample_id}_cnv_combined.txt
```

The `-loj` (left outer join) flag ensures all CNV segments are kept, even if they don't overlap a gene. Unannotated entries (.) are then filtered out.

**Output columns:** Chromosome | Start | End | Predicted_copy_number | Type_of_alteration | Gene

---

### Q5: What is BAF (B-Allele Frequency) and how does it help CNV calling?

**Answer:** **BAF** measures the relative frequency of the B (alternate) allele at heterozygous SNP positions:

$$BAF = \frac{B\_allele\_count}{A\_allele\_count + B\_allele\_count}$$

**Normal diploid state:**
- Heterozygous → BAF ≈ 0.5 (50/50 split)
- Homozygous ref → BAF ≈ 0.0
- Homozygous alt → BAF ≈ 1.0

**In CNV regions:**
- **Deletion** → BAF shifts to 0 or 1 (loss of one allele = LOH)
- **Gain** → BAF shifts to 0.33 or 0.67 (e.g., AAB = 0.33, ABB = 0.67)
- **CN-LOH** → BAF at 0 or 1 without read depth change

Control-FREEC config includes:
```ini
[BAF]
minimalCoveragePerPosition = 5    # Minimum depth to calculate BAF
```

---

### Q6: Write the Perl script that generates a Control-FREEC config file.

**Answer:**

```perl
#!/usr/bin/env perl

$path     = $ARGV[0];   # BAM file path
$bed_file = $ARGV[1];   # Capture regions BED
$output   = $ARGV[2];   # Output directory name

open(O, ">$output/config_CNV.txt");
print O "[general]

chrLenFile = /path/to/genome.fa.fai
chrFiles = /path/to/chromFa
window = 0
ploidy = 2
intercept = 1
minMappabilityPerWindow = 0.7
outputDir = $output
sex = XY
breakPointType = 2
degree = 3
coefficientOfVariation = 0.05
breakPointThreshold = 0.6
maxThreads = 10
sambamba = /usr/bin/sambamba
SambambaThreads = 10
noisyData = TRUE
printNA = FALSE

[sample]

mateFile = $path
inputFormat = BAM
mateOrientation = FR

[BAF]

minimalCoveragePerPosition = 5

[target]

captureRegions = $bed_file";
```

---

### Q7: Write a complete bash script for CNV calling: extract samples from CSV, run Control-FREEC, and annotate results.

**Answer:**

```bash
#!/usr/bin/env bash

if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <location> <csv_file>"
    exit 1
fi

location=$1
csv_file=$2
cnv_dir="$location/cnv"
mkdir -p "$cnv_dir"

# Extract Sample_ID and Project_name from CSV
awk -F',' 'BEGIN{OFS=","} {if(NR>1) print $(NF-2),$4}' \
    "$csv_file" > "$cnv_dir/list.txt"

input="$cnv_dir/list.txt"
cd "$cnv_dir" || exit 1

while IFS=',' read -r sample_id project_name; do
    echo "Processing: ${sample_id}"

    bam_path="$location/basespace/Projects/${project_name}/AppResults/${sample_id}/Files/${sample_id}.bam"

    mkdir -p "${sample_id}"

    # Step 1: Generate Control-FREEC config
    perl /path/to/cnv_config_somatic.pl \
        "$bam_path" \
        /path/to/capture.bed \
        "${sample_id}"

    # Step 2: Run Control-FREEC
    freec -conf "${sample_id}/config_CNV.txt"

    # Step 3: Annotate with gene names
    bedtools intersect \
        -a "${sample_id}/${sample_id}.bam_CNVs" \
        -b /path/to/genes.bed \
        -loj | \
        sort -V | \
        awk -F"\t" '{print $1"\t"$2"\t"$3"\t"$4"\t"$5"\t"$9}' | \
        awk -vOFS="\t" '$1=$1; BEGIN {
            str="Chromosome Start End Predicted_copy_number Type_of_alteration Gene";
            split(str,arr," ");
            for(i in arr) printf("%s\t", arr[i]); print
        }' | \
        awk '$6 != "."' > "./${sample_id}_cnv_combined.txt"

done < "$input"

echo "All CNV analysis complete."
```

---

### Q8: Write a Nextflow process for CNV calling that passes output_dir and sample_file.

**Answer:**

```groovy
process CNV {
    publishDir params.output_dir, mode: 'copy'

    input:
        path output_loc
        path sample_file

    output:
        path "*", optional: true

    script:
    """
    bash ${projectDir}/bin/CNV.sh "${params.output_dir}" "${sample_file}"
    """
}

workflow {
    CNV(params.output_dir, params.sample_file)
}
```

---

## Fusion Calling

### Q1: What is a gene fusion and why is it clinically important?

**Answer:** A **gene fusion** is a hybrid gene formed when parts of two different genes join together, typically due to:

- Chromosomal translocation (e.g., BCR-ABL in CML — t(9;22))
- Inversion (e.g., EML4-ALK in NSCLC)
- Deletion bringing two genes together
- Tandem duplication

**Clinical importance:**

| Fusion | Cancer | Therapy |
|--------|--------|---------|
| BCR-ABL | CML | Imatinib (Gleevec) |
| EML4-ALK | NSCLC | Crizotinib, Alectinib |
| TMPRSS2-ERG | Prostate | Diagnostic marker |
| NTRK fusions | Pan-cancer | Larotrectinib, Entrectinib |
| RET fusions | Thyroid/NSCLC | Selpercatinib |

**Fusions are key driver events** and often have targeted therapies available.

---

### Q2: How does this pipeline detect fusions from WES (exome) data?

**Answer:** The pipeline uses **FuSeq_WES v1.0.0** — a method designed specifically for fusion detection from whole-exome sequencing data (not RNA-seq). The workflow in `FuSeq_BAM_FUS_auto.sh`:

**Step 1: BAM Intersection**
```bash
intersectBed -abam sample.bam \
    -b knowngeneFusions.bed -f 1 > sample_intersected.bam
```
Extracts only reads overlapping known fusion-prone regions, dramatically reducing the search space.

**Step 2: Read Extraction (Python)**
```bash
python3 fuseq_wes.py --bam sample_intersected.bam \
    --gtf reference.json --mapq-filter --outdir output/
```
Identifies split reads (SR) and mate-pair reads (MR) supporting fusion breakpoints.

**Step 3: Fusion Calling (R)**
```bash
Rscript process_fuseq_wes.R in=output/ sqlite=ref.sqlite \
    fusiondb=Mitelman_fusiondb.RData paralogdb=paralogs.RData out=output/
```
Cross-references against the Mitelman database (known fusions) and filters paralogs to reduce false positives.

**Step 4: Summary Generation (Python)**
```bash
python3 fusion_summary.py
```
Generates the final `.fus` summary report.

---

### Q3: What is the difference between fusion detection from WES vs RNA-seq?

**Answer:**

| Feature | WES (DNA-based) | RNA-seq |
|---------|---|---|
| Detects at | Genomic breakpoint level | Transcript (mRNA) level |
| Evidence | Split reads + discordant pairs | Chimeric reads + spanning pairs |
| Expression | Cannot confirm expression | Confirms fusion IS expressed |
| Sensitivity | Lower (exon-only coverage) | Higher (mRNA amplification) |
| False positives | Read-through transcripts not an issue | Read-through contamination |
| Tools | FuSeq_WES, Delly, Manta | STAR-Fusion, Arriba, FusionCatcher |
| Best for | Structural variants, all breakpoints | Clinically active fusions |

This pipeline uses WES-based fusion calling because it operates on enrichment panel data (not RNA-seq), using the `knowngeneFusions.bed` as a prior.

---

### Q4: What is intersectBed and how is it used in the fusion pipeline?

**Answer:** `intersectBed` (from BEDTools) finds overlaps between two sets of genomic intervals.

```bash
intersectBed -abam sample.bam \
    -b knowngeneFusions.bed \
    -f 1 \
    > sample_intersected.bam
```

- `-abam` — input is a BAM file (not BED)
- `-b` — the reference intervals (known fusion regions)
- `-f 1` — require 100% overlap (the entire read must fall within the fusion region)

**Output:** a filtered BAM containing only reads in fusion-prone regions

This dramatically reduces compute time by pre-filtering the BAM to only relevant regions before running the computationally expensive fusion detection algorithm.

---

### Q5: Write the Nextflow process for DNA fusion calling.

**Answer:**

```groovy
process DNA_fusion {
    publishDir "${params.output_dir}/Fusion", mode: 'copy'

    input:
        path output_dir
        path sample_file

    output:
        path "*_fusions", optional: true
        path "*intersected.bam", optional: true
        path "*intersected.bam.bai", optional: true

    script:
    """
    awk -F',' 'BEGIN {OFS=","} {if (NR>1) print $(NF-2)}' "${sample_file}" \
        > ${projectDir}/bin/FuSeq_WES_v1.0.0/list_test.txt

    bash ${projectDir}/bin/FuSeq_WES_v1.0.0/FuSeq_BAM_FUS_auto.sh \
        "${params.output_dir}" "${sample_file}"
    """
}
```

---

### Q6: Write a bash script that intersects a BAM with a BED file and indexes the result.

**Answer:**

```bash
#!/usr/bin/env bash

sample_id=$1
bam_path=$2
fusion_bed="/path/to/knowngeneFusions.bed"

# Intersect BAM with fusion regions
intersectBed -abam "$bam_path" \
    -b "$fusion_bed" \
    -f 1 \
    > "${sample_id}_intersected.bam"

# Index the resulting BAM
samtools index "${sample_id}_intersected.bam"

echo "Created ${sample_id}_intersected.bam and index"
```

---

### Q7: Write a bash loop that processes multiple samples for fusion calling, renames output files, and generates a summary.

**Answer:**

```bash
#!/usr/bin/env bash

input="sample_list.txt"
fusion_dir="./fusion_results"
mkdir -p "$fusion_dir"

while IFS= read -r sample; do
    echo "> Processing ${sample} for fusions"

    output_dir="${fusion_dir}/${sample}_fusions"
    mkdir -p "$output_dir"

    # Step 1: Extract reads from fusion regions
    intersectBed -abam "${sample}.bam" \
        -b knowngeneFusions.bed -f 1 > "${sample}_intersected.bam"
    samtools index "${sample}_intersected.bam"

    # Step 2: Run fusion detection
    python3 fuseq_wes.py --bam "${sample}_intersected.bam" \
        --gtf ref.json --mapq-filter --outdir "$output_dir"

    # Step 3: Process with R
    Rscript process_fuseq_wes.R in="$output_dir" \
        sqlite=ref.sqlite fusiondb=Mitelman.RData \
        paralogdb=paralogs.RData out="$output_dir"

    # Step 4: Rename outputs to include sample ID
    for f in FuSeq_WES_FusionFinal.txt FuSeq_WES_SR_fge_fdb.txt; do
        mv "$output_dir/$f" "$output_dir/${sample}-${f}" 2>/dev/null
    done

    echo "> Done: ${sample}"
done < "$input"

echo "All samples processed."
```

---

## Pipeline-Specific Questions

### Q1: What capturing kits does this pipeline support and what are their corresponding bed_ids?

**Answer:**

```python
bed_id_mapping = {
    "GE":         32335159063,    # Germline Enrichment panel
    "CE":         25985869859,    # Comprehensive Enrichment (INDIEGENE)
    "SE8":        23683257154,    # SureSelect V8 (ABSOLUTE)
    "FEV2F2both": 34857530934,    # Target First (FEV2 + F2)
    "CDS":        31946210817,    # CDS panel
    "CEfu":       32246847276     # CE Fusion panel
}
```

---

### Q2: How does the pipeline determine if a sample is somatic vs germline?

**Answer:** By regex pattern matching on the sample ID:

```python
pattern_B = re.compile(r"-B[0-9]+|BB[0-9]+|B[0-9]+|-B", re.IGNORECASE)
pattern_F = re.compile(r"-F[0-9]+|FF[0-9]+|F[0-9]+|-F", re.IGNORECASE)
```

- `-F` suffix → somatic (Forward/Fresh tissue)
- `-B` suffix → germline (Blood/Normal)
- `-cf-` → cfDNA (liquid biopsy)

The `vc_type` is also set based on this:
- `vc_type = 0` → germline mode (when -B detected)
- `vc_type = 1` → somatic mode (default)

---

### Q3: How does the pipeline batch samples for DRAGEN launch?

**Answer:** Samples are grouped by Capturing_Kit and Somatic_Germline classification:

```python
grouped_samp = file1.groupby(
    ["Capturing_Kit", "Somatic_Germline"]
)[["Biosample_ID"].apply(
    lambda x: ",".join(map(str, map(int, x)))
)
```

Then, for each group, a single `bs launch` command is issued with all biosample IDs in that group as a comma-separated list. This allows DRAGEN to process multiple samples in a single app session, which is more efficient on BaseSpace.

---

### Q4: What QC metrics does the pipeline extract from DRAGEN results?

**Answer:** The `VCF_download_QC_extract.py` script:

1. Polls BaseSpace until the app session status is "Complete" (checking every 60 seconds)
2. Downloads `.summary.csv` files from DRAGEN (contains alignment and variant calling metrics)
3. Merges metrics across all samples into a single `metrics.csv`
4. Downloads hard-filtered VCF files (`.hard-filtered.vcf.gz`)
5. Mounts BaseSpace filesystem using `basemount`

**Typical DRAGEN metrics include:**
- Total reads, mapped reads, duplicate rate
- Mean target coverage
- PCT target bases > 30X, > 100X
- Insert size statistics
- Variant calling statistics (total SNVs, indels, Ti/Tv ratio)

---

### Q5: How does the pipeline calculate gene-level coverage?

**Answer:** Using mosdepth with per-region thresholds:

```bash
mosdepth --by capture.bed --thresholds 1,10,20,50,100 \
    sample_id sample.bam
```

This produces files showing what fraction of each capture region is covered at 1X, 10X, 20X, 50X, and 100X depth.

The `Gene_Coverage_via_Mosdepth_V2.py` script then:

1. Runs mosdepth for each sample
2. Decompresses `.thresholds.bed.gz` output
3. Groups regions by gene name
4. Calculates mean coverage percentage per gene per sample
5. Exports per-panel Excel sheets with gene-level coverage

**Output example:** Gene EGFR covered at 99.2% at 1X depth across all samples

---

### Q6: What is BaseSpace and the bs CLI tool? How does this pipeline interact with it?

**Answer:** **BaseSpace** is Illumina's cloud platform for sequencing data management and analysis. The `bs` CLI (BaseSpace Command Line Interface) lets you interact with it from the terminal.

**Pipeline interaction:**

```bash
# List projects
bs list project

# Upload FASTQ data to a project
bs upload dataset --project=<project_id> R1.fastq.gz R2.fastq.gz

# Get biosample info
bs get biosample -n <sample_name> --terse

# Launch DRAGEN analysis
bs launch application -n "DRAGEN Enrichment" --app-version 3.9.5 -o ...

# Check analysis status
bs appsession list | grep <session_name>

# Download results
bs appsession download -i <session_id> --extension=hard-filtered.vcf.gz -o output/
```

**The pipeline automates this entire cycle:** upload → launch → poll → download.

---

### Q7: Explain the sample CSV structure used by this pipeline.

**Answer:** The sample CSV has these columns:

```
Test_Name, Sample_Type, Capturing_Kit, Project_name, file_name,
Project_ID, Biosample_ID, appsession_name, bed_id, liquid_tumor,
vc-af-call-threshold, vc-af-filter-threshold, cnv_baseline_Id, baseline-noise-bed
```

---

## Quick Reference Commands

### Bioinformatics Tools

```bash
# Align reads
bwa mem -t 8 ref.fasta R1.fq.gz R2.fq.gz | samtools sort -o aligned.bam

# Index BAM
samtools index sample.bam

# View BAM header
samtools view -H sample.bam

# BAM statistics
samtools flagstat sample.bam

# Variant calling (FreeBayes)
freebayes -f ref.fasta -F 0.008 -t hotspot.bed sample.bam > variants.vcf

# BEDTools intersection
bedtools intersect -a variants.bed -b genes.bed -loj > annotated.bed

# Coverage analysis
mosdepth --by targets.bed --thresholds 1,10,20,50,100 sample sample.bam

# CNV calling
freec -conf config_CNV.txt
```

---

### Nextflow Commands

```bash
# Run pipeline
nextflow run main.nf

# Run with params override
nextflow run main.nf --input_dir /data/fastq --output_dir results

# Resume from checkpoint
nextflow run main.nf -resume

# Run with a profile
nextflow run main.nf -profile cluster

# View execution report
nextflow log

# Clean work directory
nextflow clean -f
```

---

### Bash Essentials

```bash
# CSV column extraction
awk -F',' 'NR>1 {print $3}' file.csv
cut -d',' -f3 file.csv

# Find files
find /data -name "*.vcf.gz" -type f

# File checks
[ -f file.txt ]    # exists and is regular file
[ -s file.txt ]    # exists and is non-empty
[ -d dir/ ]        # exists and is directory

# String matching
[[ "$sample" =~ -cf- ]]     # regex match
[[ "$sample" == *"-F"* ]]   # glob match

# Process substitution
while read line; do ...; done < <(tail -n +2 file.csv)
```

---

**Good luck with your interview! 🍀**

Remember:
- Explain your thought process aloud
- Ask clarifying questions when needed
- Don't rush through code — correctness > speed
- Be prepared to discuss trade-offs and clinical implications of pipeline decisions
- Have concrete examples from real projects ready
