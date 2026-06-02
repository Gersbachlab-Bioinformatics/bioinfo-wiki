# CLEANSER

[CLEANSER](https://github.com/Gersbachlab-Bioinformatics/CLEANSER/) is a Bayesian model for guide assignment in CRISPR single-cell screens (CRISPRi/CRISPRa). It processes multimodal single-cell data (`.h5mu` format) and assigns guides to cells using a probabilistic framework built on Stan.

- [Container](#container)
- [Standalone run with Singularity](#standalone-run-with-singularity)
- [Running with SLURM (sbatch)](#running-with-slurm-sbatch)
- [Key parameters](#key-parameters)
- [Gotchas](#gotchas)

## Container

A pre-built Singularity image is available on DCC at:

```
/hpc/group/gersbachlab/aeb84/apptainer/ghcr.io-gersbachlab-bioinformatics-cleanser-1.2.1.img
```

This image bundles CLEANSER and its Stan/CmdStan dependencies, so no conda environment or separate installation is needed.

## Standalone run with Singularity

Below is a template to run CLEANSER interactively or in a SLURM job. Adjust `WORKDIR` and the `cleanser` arguments to match your data.

```bash
# Directory containing your input .h5mu file (will also hold outputs)
WORKDIR=/path/to/your/workdir

# Temp directory for Stan intermediate files (needs to be writable from inside the container)
TMPDIR_HOST=/work/${USER}/tmp
mkdir -p "${TMPDIR_HOST}" "${WORKDIR}/tmp"

singularity exec \
    --no-home \
    --pid \
    -B "${TMPDIR_HOST}" \
    -B "${WORKDIR}" \
    /hpc/group/gersbachlab/aeb84/apptainer/ghcr.io-gersbachlab-bioinformatics-cleanser-1.2.1.img \
    /bin/bash -c "
        export CMDSTAN=/root/.cmdstan/cmdstan-2.36.0
        export PATH=\$PATH:\$CMDSTAN/bin
        export TMPDIR='${WORKDIR}/tmp'
        cd ${WORKDIR}
        cleanser \
            -i your_input.h5mu \
            --posteriors-output your_output.h5mu \
            --modality guide \
            --crop-seq \
            --output-layer guide_assignment \
            -t 4
    "
```

### Singularity flags explained

| Flag | Purpose |
|------|---------|
| `--no-home` | Prevents mounting your `$HOME` directory inside the container, avoiding conflicts with user-level configs |
| `--pid` | Runs the container in a new PID namespace; avoids process ID collisions |
| `-B <path>` | Bind-mounts a host directory into the container so CLEANSER can read inputs and write outputs |

### Environment variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `CMDSTAN` | `/root/.cmdstan/cmdstan-2.36.0` | Points to the CmdStan installation baked into the image |
| `PATH` | `...:$CMDSTAN/bin` | Makes the `stanc` and `cmdstan` binaries available |
| `TMPDIR` | `./tmp` (relative to `WORKDIR`) | Tells Stan where to write temporary compilation artifacts |

## Running with SLURM (sbatch)

To submit CLEANSER as a batch job on DCC, save the following as a `.sh` script and submit it with `sbatch run_cleanser.sh`.

```bash
#!/bin/bash
#SBATCH --job-name=cleanser
#SBATCH --account=gersbachlab
#SBATCH --mem=64G
#SBATCH --cpus-per-task=4
#SBATCH --time=4:00:00
#SBATCH --output=cleanser_%j.out
#SBATCH --error=cleanser_%j.err

# Directory containing your input .h5mu file (will also hold outputs)
WORKDIR=/path/to/your/workdir

# Temp directory for Stan intermediate files
TMPDIR_HOST=/work/${USER}/tmp
mkdir -p "${TMPDIR_HOST}" "${WORKDIR}/tmp"

singularity exec \
    --no-home \
    --pid \
    -B "${TMPDIR_HOST}" \
    -B "${WORKDIR}" \
    /hpc/group/gersbachlab/aeb84/apptainer/ghcr.io-gersbachlab-bioinformatics-cleanser-1.2.1.img \
    /bin/bash -c "
        export CMDSTAN=/root/.cmdstan/cmdstan-2.36.0
        export PATH=\$PATH:\$CMDSTAN/bin
        export TMPDIR='${WORKDIR}/tmp'
        cd ${WORKDIR}
        cleanser \
            -i your_input.h5mu \
            --posteriors-output your_output.h5mu \
            --modality guide \
            --crop-seq \
            --output-layer guide_assignment \
            -t \${SLURM_CPUS_PER_TASK}
    "
```

Note that `-t` is set to `$SLURM_CPUS_PER_TASK` so it automatically matches the allocated cores. Adjust `--time` based on the size of your dataset.

## Key parameters

| Parameter | Description |
|-----------|-------------|
| `-i` | Input `.h5mu` file |
| `--posteriors-output` | Output `.h5mu` file with guide assignment posteriors |
| `--modality` | Modality key in the MuData object that holds guide counts (typically `guide`) |
| `--crop-seq` | Enable CROPseq mode (one guide per cell model) |
| `--output-layer` | Name of the layer written to the output file (e.g. `guide_assignment`) |
| `-t` | Number of threads for Stan sampling; tune based on available CPUs |

## Gotchas

### Bind all directories that CLEANSER needs to read or write

Singularity only sees directories you explicitly bind with `-B`. If your input file or the TMPDIR lives outside `WORKDIR`, add a separate `-B` for it. A common symptom of a missing bind is a `FileNotFoundError` or a silent hang at the Stan compilation step.

### TMPDIR must be writable

Stan compiles its model at runtime and writes intermediate files to `TMPDIR`. Make sure the directory exists on the host **before** launching the container and that the bind mount resolves correctly inside it.
