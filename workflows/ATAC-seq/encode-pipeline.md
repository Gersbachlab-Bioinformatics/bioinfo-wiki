# ENCODE ATAC-seq pipeline instructions

Here are some notes on how to run the ATAC-seq processing workflow developed by the ENCODE consortium. For further details about the pipeline, please refer to the original documentation [here](https://docs.google.com/document/d/1f0Cm4vRyDQDu0bMehHD7P7KOMxTOP-HiNoIvL1VcBt8/edit?tab=t.0) and in [github](https://github.com/ENCODE-DCC/atac-seq-pipeline). 

- [Introduction](#introduction)
- [Installation](#installation)
- [Configuration](#configuration)
- [Metadata](#metadata)
- [Run](#run)
- [Organize outputs](#outputs)
- [Gotchas](#gotchas)

## Introduction

The ENCODE ATAC-seq pipeline has been mostly developed across the Kundaje and Cherry labs at Stanford. The workflows are written in WDL which use Cromwell as the executing engine. To make these pipelines more portable, the team developed a companion open source tool called [caper](https://github.com/ENCODE-DCC/caper), which comes handy to manage the cromwell job orchestration and can be configured to work with many backends including `SLURM`. In addition, there is a third companion tool to organize and summarize Cromwell outputs named [croo](https://github.com/ENCODE-DCC/croo).

Beyond the above technical details, this bulk ATAC-seq processing pipeline will generate BAMs, peak sets and signal files that need to be further processed if trying to identify differentially accessible regions. That said, many of the QC metrics and stats computed and introduced here are now part of the community standard, like TSS enrichment scores. It is not uncommon to see ENCODE pipelines cited in the literature and they are a safe bet for most users processing producing ATAC-seq samples  

```sh
$ source /hpc/group/gersbachlab/software/miniforge3/bin/activate /hpc/group/gersbachlab/aeb84/mamba_envs/encode_atac
(encode_atac)$ caper --help

usage: caper [-h] [-v] {init,run,server,submit,abort,unhold,list,metadata,troubleshoot,debug,hpc,gcp_monitor,gcp_res_analysis,cleanup} ...

Caper (Cromwell-assisted Pipeline ExecutioneR)

positional arguments:
  {init,run,server,submit,abort,unhold,list,metadata,troubleshoot,debug,hpc,gcp_monitor,gcp_res_analysis,cleanup}
    init                Initialize Caper's configuration file. THIS WILL OVERWRITE ON A SPECIFIED(-c)/DEFAULT CONF FILE. e.g. ~/.caper/default.conf.
    run                 Run a single workflow without server
    server              Run a Cromwell server
    submit              Submit a workflow to a Cromwell server
    abort               Abort running/pending workflows on a Cromwell server
    unhold              Release hold of workflows on a Cromwell server
    list                List running/pending workflows on a Cromwell server
    metadata            Retrieve metadata JSON for workflows from a Cromwell server
    troubleshoot        Troubleshoot workflow problems from metadata JSON file or workflow IDs
    debug               Identical to "troubleshoot"
    hpc                 Subcommand for HPCs
    gcp_monitor         Tabulate task's resource data collected on instances run on Google Cloud Compute. Use this for any workflows run with Caper>=1.2.0 on gcp backend. This is for gcp backend only.
    gcp_res_analysis    Linear resource analysis on monitoring data collected on instances run on Google Cloud Compute. This is for gcp backend only. Use this for any workflows run with Caper>=1.2.0 on gcp backend. Calculates coefficients/intercept for task's required resources based on input file size of a task.
                        For each task it solves a linear regression problem of y=Xc + i1 + e where X is a matrix (m by n) of input file sizes and c is a coefficient vector (n by 1) and i is intercept and 1 is ones vector. y is a vector (m by 1) of resource taken and e is residual to be minimized. m is number of
                        dataset and n is number of input file variables. Each resource metric will be solved separately. Refer to --target-resources for details about available resource metrics. Output will be a tuple of coefficient vector and intercept.
    cleanup             Cleanup outputs of workflows.

optional arguments:
  -h, --help            show this help message and exit
  -v, --version         Show version
```

## Configuration
A few things are needed to run `caper` in an HPC with SLURM. In paricular, you need to have a `default.conf` file under a `.caper` folder in your HOME directory. If you don't have that config file, please create a `~/.caper/default.conf` file with the following content: 

```sh
backend=slurm

# define one of the followings (or both) according to your
# cluster's SLURM configuration.

# SLURM partition. Define only if required by a cluster. You must define it for Stanford Sherlock.
slurm-partition=common
# SLURM account. Define only if required by a cluster. You must define it for Stanford SCG.
slurm-account=gersbachlab


# This parameter is NOT for 'caper submit' BUT for 'caper run' and 'caper server' only.
# This resource parameter string will be passed to sbatch, qsub, bsub, ...
# You can customize it according to your cluster's configuration.

# Note that Cromwell's implicit type conversion (String to Integer)
# seems to be buggy for WomLong type memory variables (memory_mb and memory_gb).
# So be careful about using the + operator between WomLong and other types (String, even Int).
# For example, ${"--mem=" + memory_mb} will not work since memory_mb is WomLong.
# Use ${"if defined(memory_mb) then "--mem=" else ""}{memory_mb}${"if defined(memory_mb) then "mb " else " "}
# See https://github.com/broadinstitute/cromwell/issues/4659 for details

# Cromwell's built-in variables (attributes defined in WDL task's runtime)
# Use them within ${} notation.
# - cpu: number of cores for a job (default = 1)
# - memory_mb, memory_gb: total memory for a job in MB, GB
#   * these are converted from 'memory' string attribute (including size unit)
#     defined in WDL task's runtime
# - time: time limit for a job in hour
# - gpu: specified gpu name or number of gpus (it's declared as String)

slurm-resource-param=-n 1 --ntasks-per-node=1 --cpus-per-task=${cpu} ${if defined(memory_mb) then "--mem=" else ""}${memory_mb}${if defined(memory_mb) then "M" else ""} ${if defined(time) then "--time=" else ""}${time*60} ${if defined(gpu) then "--gres=gpu:" else ""}${gpu} 

# If needed uncomment and define any extra SLURM sbatch parameters here
# YOU CANNOT USE WDL SYNTAX AND CROMWELL BUILT-IN VARIABLES HERE
#slurm-extra-param=

# Hashing strategy for call-caching (3 choices)
# This parameter is for local (local/slurm/sge/pbs/lsf) backend only.
# This is important for call-caching,
# which means re-using outputs from previous/failed workflows.
# Cache will miss if different strategy is used.
# "file" method has been default for all old versions of Caper<1.0.
# "path+modtime" is a new default for Caper>=1.0,
#   file: use md5sum hash (slow).
#   path: use path.
#   path+modtime: use path and modification time.
local-hash-strat=path+modtime

# Metadata DB for call-caching (reusing previous outputs):
# Cromwell supports restarting workflows based on a metadata DB
# DB is in-memory by default
db=in-memory

# If you use 'caper run' and want to use call-caching:
# Make sure to define different 'caper run ... --db file --file-db DB_PATH'
# for each pipeline run.
# But if you want to restart then define the same '--db file --file-db DB_PATH'
# then Caper will collect/re-use previous outputs without running the same task again
# Previous outputs will be simply hard/soft-linked.

# If you use 'caper server' then you can use one unified '--file-db'
# for all submitted workflows. In such case, uncomment the following two lines
# and defined file-db as an absolute path to store metadata of all workflows
#db=file
#file-db=

# Local directory for localized files and Cromwell's intermediate files
# If not defined, Caper will make .caper_tmp/ on local-out-dir or CWD.
# /tmp is not recommended here since Caper store all localized data files
# on this directory (e.g. input FASTQs defined as URLs in input JSON).
local-loc-dir=

cromwell=${HOME}/.caper/cromwell_jar/cromwell-65.jar
womtool=${HOME}/.caper/womtool_jar/womtool-65.jar
```

## Metadata

JSON metadata files are used to configure and run the pipeline. 

### Genome tsv
All the references are listed in a tab-delimited file. In DCC, there is one available for human (hg38):
```sh
# %load /hpc/group/gersbachlab/reference_data/encode_atac_seq/hg38.tsv
genome_name	hg38
ref_fa	/hpc/group/gersbachlab/reference_data/encode_atac_seq/GRCh38_no_alt_analysis_set_GCA_000001405.15.fasta.gz
ref_mito_fa	/hpc/group/gersbachlab/reference_data/encode_atac_seq/GRCh38_no_alt_analysis_set_GCA_000001405.15_mito_only.fasta.gz
mito_chr_name	chrM
regex_bfilt_peak_chr_name	chr[\dXY]+
blacklist	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ENCFF356LFX.bed.gz
chrsz	/hpc/group/gersbachlab/reference_data/encode_atac_seq/GRCh38_EBV.chrom.sizes.tsv
gensz	hs
bowtie2_idx_tar	/hpc/group/gersbachlab/reference_data/encode_atac_seq/bowtie2_index/ENCFF110MCL.tar.gz
bowtie2_mito_idx_tar	/hpc/group/gersbachlab/reference_data/encode_atac_seq/bowtie2_index/GRCh38_no_alt_analysis_set_GCA_000001405.15_mito_only_bowtie2_index.tar.gz
bwa_idx_tar	/hpc/group/gersbachlab/reference_data/encode_atac_seq/bwa_index/ENCFF643CGH.tar.gz
bwa_mito_idx_tar	/hpc/group/gersbachlab/reference_data/encode_atac_seq/bwa_index/GRCh38_no_alt_analysis_set_GCA_000001405.15_mito_only_bwa_index.tar.gz
tss	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ataqc/ENCFF493CCB.bed.gz
dnase	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ataqc/ENCFF304XEX.bed.gz
prom	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ataqc/ENCFF140XLU.bed.gz
enh	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ataqc/ENCFF212UAV.bed.gz
reg2map	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ataqc/hg38_dnase_avg_fseq_signal_formatted.txt.gz
reg2map_bed	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ataqc/hg38_celltype_compare_subsample.bed.gz
roadmap_meta	/hpc/group/gersbachlab/reference_data/encode_atac_seq/ataqc/hg38_dnase_avg_fseq_signal_metadata.txt
```
### Input JSON
JSON file with information about the reference used, configuration parameters (e.g. single-end or paired-end, auto-detection of adapter for trimming, etc.). Here is an example:

```json
{
    "atac.pipeline_type": "atac",
    "atac.genome_tsv": "/hpc/group/gersbachlab/reference_data/encode_atac_seq/hg38.tsv",
    "atac.fastqs_rep1_R1": [
        "/work/aeb84/collab/20250910_Morgan/data/MC-L452_S1_R1_001.fastq.gz"
    ],
    "atac.fastqs_rep1_R2": [
        "/work/aeb84/collab/20250910_Morgan/data/MC-L452_S1_R2_001.fastq.gz"
    ],
    "atac.paired_end": true,
    "atac.auto_detect_adapter": true,
    "atac.enable_xcor": true,
    "atac.title": "MC-L452_S1",
    "atac.description": "ATAC-seq on MC-L452_S1"
}
```

## Run
Using the references and inputs previously defined, everything is ready to launch the master `caper` job. Here is an example for the preferred option to run the pipeline in DCC, with `singularity`. Please check the important [gotchas](#gotchas) section at the end of this reference for important details to avoid common issues.

```sh
# Activate mamba environment with `caper` installed and enabled
source /hpc/group/gersbachlab/software/miniforge3/bin/activate \
    /hpc/group/gersbachlab/aeb84/mamba_envs/encode_atac

# Create a subfolder for the pipeline intermediate and final outputs
mkdir -p /work/aeb84/collab/20250910_Morgan/runs/${SAMPLE}
cd /work/aeb84/collab/20250910_Morgan/runs/${SAMPLE}

# Configure environment to use singularity
export SINGULARITY_TMPDIR='/cwork/aeb84/tmp'
export SINGULARITY_CACHEDIR='/cwork/aeb84/apptainer'

SAMPLE="MC-L452_S1"
INPUT_JSON="/work/aeb84/collab/20250910_Morgan/metadata/atac_seq_metadata.${SAMPLE}.json"
caper \
    hpc submit \
    /hpc/group/gersbachlab/aeb84/software/atac-seq-pipeline/atac.wdl \
    -i "${INPUT_JSON}" \
    --singularity \
    --leader-job-name atac_${SAMPLE}_master \
    --db in-memory

```

## Outputs
Finally, if everything goes well, you should see an `atac` folder under the appropriate runs subfolder. Under it, there should be a directory with a UUID with the outputs of the pipeline. To simplify and aggregate all the outputs under a single folder, the `croo` command line utility is used. Also included there is a nice utility to create a TSV file with QC stats in tabular format. Here is an example of how that would be done:

```sh
# Activate mamba environment with `croo` installed and enabled
source /hpc/group/gersbachlab/software/miniforge3/bin/activate \
    /hpc/group/gersbachlab/aeb84/mamba_envs/encode_atac

cd /work/aeb84/collab/20250910_Morgan/runs/${SAMPLE}/atac
for OUTDIR in *;
do
    croo --out-dir ${OUTDIR}/outputs ${OUTDIR}/metadata.json  # Create and populate an `outputs` directory 
    qc2tsv ${OUTDIR}/outputs/qc/qc.json > ${OUTDIR}/qc.tsv  # Create TSV file with QC stats
done
```

## Gotchas

### Create your pipeline outputs alongside the inputs
When using the `--singularity` argument with `caper` to containerize your runs, there is a [reported issue](https://github.com/ENCODE-DCC/atac-seq-pipeline/issues/420#issuecomment-3349097131) when FASTQ files are found outside of the directory tree where the intermediate outputs are generated. Just make sure to create and specify the run subfolder next to the inputs directory. Using symlinks **might** not work, so keep that in mind. If creating the pipeline outputs along the inputs is not a possibility, you might need to remove the `--singularity` argument in the `caper` command, which forces you to use a conda environment with all the pipeline dependencies installed. Reach out if this is the case, as some folks in lab might be able to share a conda environment for you to use.
