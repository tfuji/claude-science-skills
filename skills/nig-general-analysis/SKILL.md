---
name: nig-general-analysis
description: >
  How to run jobs on the NIG supercomputer (DDBJ / National Institute of
  Genetics, Japan) GENERAL ANALYSIS zone. Use this whenever the user wants to
  run bioinformatics work on the NIG/DDBJ supercomputer, mentions "遺伝研スパコン",
  "nig-a003" or another general-zone login node, SLURM partitions like `epyc`
  or `rome`, the `/usr/local/biotools` Singularity/apptainer image collection,
  or the DDBJ pre-installed databases under `/usr/local/resources` and
  `/usr/local/shared_data`. Covers three things: (a) submitting SLURM jobs,
  (b) running any of the thousands of pre-built bioinformatics tools via
  apptainer, and (c) using the pre-installed sequence/annotation databases.
  This is the GENERAL analysis zone only — the personal (human) genome
  analysis zone is a physically separate network; use nig-personal-genome
  for that.
---

# NIG supercomputer — General Analysis zone

The NIG supercomputer (run by DDBJ at the National Institute of Genetics,
Mishima, Japan) is a SLURM cluster. This skill covers the **general analysis
zone** — the open zone for non-personal-genome work. It is reached through
general-zone login nodes (e.g. `nig-a003`, hostname `a003`).

> The **personal genome analysis zone** (controlled-access human data) is a
> completely separate network and is not reachable from the general zone —
> different login nodes, different SLURM, different databases. For that zone
> use the `nig-personal-genome` skill.

Everything here is designed so you rarely need to install anything: tools come
as pre-built container images, and the major public databases are already on
the shared filesystem. The three sections below map to the three things you
do on this cluster: **(a) run SLURM jobs**, **(b) run tools from
`/usr/local/biotools`**, **(c) use the pre-installed databases**.

When dispatching from Claude Science, the host is registered as an SSH compute
target (typically `ssh:nig-a003`). Read `compute_details({provider:'ssh:nig-a003'})`
first — it accumulates verified partition/account/path facts across sessions —
then follow the `remote-compute-ssh` skill for the submit → wait → harvest
mechanics. This skill supplies the NIG-specific content that goes *inside* the
job script.

---

## (a) SLURM — partitions, accounts, submitting

The scheduler is SLURM. Jobs are submitted with `sbatch`; interactive shells
with `srun`. From Claude Science you normally submit through
`host.compute.create('ssh:nig-a003').submit_job(...)`, which wraps `sbatch` —
put the `#SBATCH` directives at the top of the `command` script.

**Partitions (general zone).**

> **Always trust the live `sinfo` output over any table — including the one
> below and the official docs.** The general zone lends compute nodes out to
> labs, so partitions appear, disappear, and change size over time. The
> published documentation lags reality; the machine is the source of truth.
> Run `sinfo -s` (and `scontrol show partition <name>` for account gating)
> at the start of a session and target what is actually up.

```bash
sinfo -s                                  # live partitions + node availability
scontrol show partition epyc              # limits / AllowAccounts for one partition
sacctmgr -n show assoc user=$USER format=account,partition   # what you may use
```

The **baseline** open partitions (per DDBJ docs, subject to the caveat above):

| Partition | Node type | Hardware | Baseline scale |
|-----------|-----------|----------|-------|
| `login`   | Interactive node | HPC CPU-opt Type 1 (AMD EPYC 9654, 192 cores/node, 8 GB/core) | 3 nodes / 576 cores |
| `epyc`    | **Batch (default)** | HPC CPU-opt Type 1 (AMD EPYC 9654, 192 cores/node, 8 GB/core) | 12 nodes / 2304 cores |
| `rome`    | Batch | HPC CPU-opt Type 2 (AMD EPYC 7702, 128 cores/node, 4 GB/core) | 9 nodes / 1152 cores |
| `short.q` | Batch (short jobs) | HPC CPU-opt Type 1 (AMD EPYC 9654, 192 cores/node) | 1 node / 192 cores |
| `medium`  | Batch (large memory) | Memory-opt Type 2 (AMD EPYC 9654, 192 cores/node, 16 GB/core) | 2 nodes / 384 cores |

`login` is the interactive node for development and small, short jobs — fine
for quick interactive work but **not** for heavy batch compute. `epyc` is the
standard batch partition; use `rome` for more cores-per-node throughput,
`medium` when you need more memory per core (16 GB/core), and `short.q` for
short jobs. Reality can differ from this table — e.g. a partition listed here
may be temporarily withdrawn, and `sinfo` will show the current truth. If
`short.q` is not in `sinfo`, it is simply not up right now; use `epyc`.

`sinfo` also lists many **lab-owned** partitions named `<labname>-c<N>` (e.g.
`kumamoto-c768`, `rikagaku-c192`, `nakayama-c576`, `tokai-c64`). Each is gated
by `AllowAccounts=<labname>-group` and is usable only if you belong to that
group. Don't target them unless the user says they have access.

**Account.** The general-zone account is typically `general_analysis`. Confirm
what the user's account is with `sacctmgr -n show assoc user=$USER
format=account,partition`. In a job script set it with `#SBATCH --account=...`
only if the cluster requires it (many general jobs run without an explicit
account).

**A minimal job script:**

```bash
#SBATCH --partition=epyc
#SBATCH --cpus-per-task=8
#SBATCH --mem=16G
#SBATCH --time=2:00:00

# ... apptainer exec commands go here (see section b) ...
```

Time limits are generous (partitions allow very long walltimes), but always
set `--time` to something realistic so the scheduler can backfill your job.
Ask for `--cpus-per-task` / `--mem` that match the tool; most alignment and
phylogenetics tools are happy with 4–16 cores and a few GB.

Useful commands: `squeue --me` (your queue), `sinfo -s` (partition state),
`sacct -j <jobid>` (accounting for a finished job), `scancel <jobid>`.

---

## (b) Tools — `/usr/local/biotools` via apptainer

**This is the key idea: you almost never install a bioinformatics tool on
NIG.** The cluster ships the entire BioContainers collection as pre-built
Singularity/apptainer images. Per the official DDBJ docs this is **over 2,000
distinct analysis programs and, counting versions, more than 90,000 Singularity
image files** under `/usr/local/biotools/`. Do **not** try to list them all —
know the layout and search for the one you need. For a tool's usage details,
consult the BioContainers registry / the tool's own documentation.

**Layout.** Images live under:

```
/usr/local/biotools/<first-letter>/<tool>:<version>
```

The `<first-letter>` is the first character of the tool name. So MAFFT images
are under `/usr/local/biotools/m/`, trimAl under `t/`, IQ-TREE under `i/`, and
so on. Each tool usually has many versioned image files
(e.g. `mafft:7.310--hdbdd923_7`).

**Find a tool / its versions** (cheap — scoped to one letter directory):

```bash
ls /usr/local/biotools/m/ | grep -i '^mafft:'      # all MAFFT versions
ls /usr/local/biotools/i/ | grep -i '^iqtree'      # iqtree and iqtree2
```

Pick the newest version tag (last line of a sorted list) unless the user needs
a specific one.

**Run a tool.** `apptainer` (v1.x) is on the default PATH — no `module load`
needed. The image *is* the executable environment; call the tool inside it
with `apptainer exec`:

```bash
IMG=/usr/local/biotools/m/mafft:7.310--hdbdd923_7
apptainer exec "$IMG" mafft --auto input.fasta > aligned.fasta
```

`singularity exec` also works (`singularity-ce` 4.x is installed) and is
interchangeable with `apptainer exec` here — the official DDBJ examples use
`singularity exec`. For a tool you call repeatedly in a script, an alias keeps
lines short, e.g. `alias singR="singularity exec /usr/local/biotools/r/r-base:3.5.1 R"`.

**Binding paths.** apptainer auto-mounts your home and usually the current
working directory. If a tool needs to read the pre-installed databases or a
scratch path that isn't auto-mounted, bind it explicitly with `-B`:

```bash
apptainer exec -B /usr/local/resources:/usr/local/resources \
  /usr/local/biotools/b/blast:2.14.0--pl5321h6f7f691_0 \
  blastp -query q.fasta -db /usr/local/shared_data/blastdb/nr -out hits.tsv -outfmt 6
```

Keep all inputs and outputs under the job workdir (or a path you bind) so
Claude Science's harvester can pull the results back.

**A complete general-zone job script** (MAFFT → trimAl → IQ-TREE, chained
through three images):

```bash
#SBATCH --partition=epyc
#SBATCH --cpus-per-task=8
#SBATCH --mem=16G
#SBATCH --time=4:00:00

B=/usr/local/biotools
MAFFT=$B/m/$(ls $B/m/ | grep '^mafft:7.310' | tail -1)
TRIMAL=$B/t/$(ls $B/t/ | grep '^trimal:1.4.1' | tail -1)
IQTREE=$B/i/$(ls $B/i/ | grep '^iqtree:' | tail -1)

apptainer exec "$MAFFT"  mafft --auto input.fasta > aln.fasta
apptainer exec "$TRIMAL" trimal -in aln.fasta -out aln.trim.fasta -automated1
apptainer exec "$IQTREE" iqtree -s aln.trim.fasta -m MFP -bb 1000 -nt AUTO
ls -lh aln.trim.fasta.* aln.fasta
```

---

## (c) Pre-installed databases

The DDBJ keeps large public databases on the shared filesystem, so you should
point tools at these paths instead of downloading anything. As with tools,
**know the roots and look inside — don't enumerate everything.**

**Two roots:**

- `/usr/local/resources/` — the DDBJ archive and public mirrors:
  - `mirror/` and `mirror_database/` — mirrors of NCBI (`ftp.ncbi.nih.gov`),
    EBI (`ftp.ebi.ac.uk`), UniProt, PDB/PDBj, RefSeq, H-Invitational.
  - `bioproject/`, `biosample/`, `dra/` (DDBJ Sequence Read Archive),
    `gea/` (Genomic Expression Archive), `metabobank/`, `trad/`, `reference/`.
- `/usr/local/shared_data/` — analysis-ready databases and references:
  - `blastdb/` — pre-formatted BLAST databases (nr, nt, etc.).
  - `genomes/`, `refseq/`, `uniprot/`, `pdb/` — sequence/structure references.
  - `dfast/` (prokaryotic annotation), `db/`, `dblink/`, `sra/`, `rdf/`.
  - `public-human-genomes/` — `GRCh38` and `CHM13` reference assemblies
    (open reference genomes; note *individual* human genotype data lives in
    the personal-genome zone, not here).

**Discover what's actually present** (versions change over time):

```bash
ls /usr/local/shared_data/blastdb/           # which BLAST DBs are formatted
ls /usr/local/resources/mirror_database/     # which mirrors exist
ls /usr/local/shared_data/genomes/           # reference genomes on hand
```

**Using them.** Point the tool's `-db` / reference argument directly at the
path, and bind the root into the container if you're running via apptainer:

```bash
# BLAST against the shared nr, no download needed
apptainer exec -B /usr/local/shared_data:/usr/local/shared_data \
  /usr/local/biotools/b/$(ls /usr/local/biotools/b/ | grep '^blast:' | tail -1) \
  blastp -query q.fasta -db /usr/local/shared_data/blastdb/nr \
         -num_threads 8 -outfmt 6 -out hits.tsv
```

`BLASTDB` is **not** set by default, so always give the full DB path (or
`export BLASTDB=/usr/local/shared_data/blastdb` at the top of your script).

---

## Quick recall

- **Tools**: `/usr/local/biotools/<letter>/<tool>:<ver>` → `apptainer exec <img> <cmd>`. Search, don't enumerate.
- **Databases**: `/usr/local/resources/` (archive + mirrors) and `/usr/local/shared_data/` (blastdb, genomes, refseq, uniprot, pdb…). `ls` the root you need; bind with `-B`.
- **SLURM**: `sinfo -s` is the source of truth (docs lag; nodes get lent out). Default batch partition `epyc`; account usually `general_analysis`; never heavy compute on `login`.
- **From Claude Science**: `compute_details` first, then `remote-compute-ssh` for submit/wait/harvest; put `#SBATCH` + apptainer lines in the job `command`.
- This is the general zone; controlled human-genome work → `nig-personal-genome`.

---

## Official documentation

The DDBJ maintains docs (Japanese, with an English toggle). Treat them as
**background/orientation, not ground truth** for live cluster state: the
published tables can lag reality (nodes get lent to labs, partitions change).
For anything that must be current — which partitions are up, which DBs are
present, which tool versions exist — query the machine (`sinfo`, `ls`) and
prefer that. Use these pages to understand *concepts* and *layout*:

- Hardware (2025): https://sc.ddbj.nig.ac.jp/guides/hardware/hardware2025/
- SLURM partitions (general zone): https://sc.ddbj.nig.ac.jp/guides/using_general_analysis_division/ga_slurm_partition/
- Apptainer / Singularity: https://sc.ddbj.nig.ac.jp/guides/software/Container/Apptainer/
- BioContainers images: https://sc.ddbj.nig.ac.jp/guides/software/Container/BioContainers/
- SLURM usage (batch/parallel/array/interactive): https://sc.ddbj.nig.ac.jp/guides/software/JobScheduler/Slurm/
- General-analysis-zone guide index: https://sc.ddbj.nig.ac.jp/guides/using_general_analysis_division/
