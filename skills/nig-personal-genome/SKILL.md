---
name: nig-personal-genome
description: >
  How to run jobs on the NIG supercomputer (DDBJ / National Institute of
  Genetics, Japan) PERSONAL GENOME ANALYSIS zone — the security-hardened,
  NBDC-compliant division for controlled-access human (personal) genome data.
  Use this whenever the user works with individual human genotype/genome data
  on NIG/DDBJ, mentions the "個人ゲノム解析区画", SSL-VPN / FortiClient login to
  DDBJ, a `-pg` account, GPU genome analysis with Parabricks, or the L40S GPU
  nodes (`--partition=l40s`). This zone is a PHYSICALLY SEPARATE network from
  the general analysis zone: different login path (SSL-VPN), different accounts,
  different SLURM/partitions, node-lease model, and no shared public-DB layout.
  For open, non-personal-genome work use nig-general-analysis instead. Like
  that zone it uses `/usr/local/biotools` apptainer images for tools; the
  differences are access, scheduling, GPUs, and data handling.
---

# NIG supercomputer — Personal Genome Analysis zone

The **personal genome analysis zone** (個人ゲノム解析区画) is DDBJ's
security-hardened division for **controlled-access human genome data**. It is
built to comply with the NBDC Human Data security guidelines, and it is a
**completely separate network** from the general analysis zone — you cannot see
or reach it from a general-zone node (`nig-a003` etc.), and vice versa.

> If the work does **not** involve controlled human personal-genome data, use
> the `nig-general-analysis` skill — that zone is simpler (direct SSH, shared
> SLURM, pre-installed public DBs). This skill is specifically for the
> personal-genome division and its stricter access + scheduling model.

The three functional sections mirror the general-zone skill — **(a) SLURM /
scheduling, (b) tools via `/usr/local/biotools`, (c) data & databases** — but
the specifics differ because of the security model. Read section 0 first: how
you even get in is the biggest difference.

---

## (0) Access — SSL-VPN, two-factor, `-pg` account

Unlike the general zone (direct SSH to a gateway), the personal-genome zone is
reached **only through an SSL-VPN tunnel**:

- **VPN client**: FortiClient ("FortiClient VPN-only", the free edition) must
  be installed and configured on the user's machine.
- **Two-factor auth**: connecting sends a one-time password to the account's
  registered email; you enter it as the VPN token.
- **Network lockdown while connected**: by design, when the SSL-VPN is up the
  user's machine loses general internet access (local-network traffic still
  works). From *inside* the zone, only outbound **HTTPS** to the internet is
  permitted by the firewall.
- **Account name**: the personal-genome account is the general-zone account
  name **with a `-pg` suffix** (e.g. `oogasawa` → `oogasawa-pg`). Every
  personal-genome user also gets a general-zone account; the `-pg` one is for
  this zone.
- **Gateway**: after the VPN is up you SSH to the personal-genome gateway
  (`gwa.ddbj.nig.ac.jp`), then on to your compute node (see section a).
- **Billing / application**: this zone is a **paid, node-lease service**.
  Access requires an approved account application plus a submitted usage plan
  (利用計画表), and adherence to the NBDC human-data guidelines. Even
  non-human-genome GPU work must apply here because the GPUs live in this zone.

> **Claude Science note.** Driving this zone from Claude Science requires a
> registered SSH compute target that routes *through the SSL-VPN + gateway*.
> The VPN/2FA step is interactive and per-user; it is not something the agent
> can complete unattended. If no personal-genome SSH target is configured,
> author/run against user-provided details rather than assuming general-zone
> paths carry over.

---

## (a) SLURM / scheduling — node-lease + shared GPU SLURM

The scheduling model here is fundamentally different from the general zone's
shared batch partitions:

**Two modes:**

1. **Dedicated (leased) compute nodes — the default.** For security (so users
   can't see each other's jobs), personal-genome nodes are normally handed over
   as **dedicated/occupied nodes**. You SSH from the gateway to *your own*
   leased node and work there. DDBJ can install **Grid Engine or SLURM** on the
   leased node on request, so you may submit jobs to *your own* node's
   scheduler — but there is no big shared batch pool like the general zone's
   `epyc`.

2. **Shared GPU SLURM — for the L40S GPU nodes.** The accelerator-optimized
   Type-2 nodes (NVIDIA L40S) are scarce, so they are shared through a
   **dedicated GPU SLURM scheduler**. After VPN+gateway login, hop to the GPU
   interactive node and submit there:

   ```bash
   ssh at022vm02              # GPU-SLURM interactive node (from the gateway)
   ```

**Submitting to the L40S GPU partition.** The required SLURM options:

```bash
#SBATCH --partition=l40s --account=l40s   # L40S GPU partition + account
#SBATCH --gres=gpu:N                       # N = 1..8 GPUs on the node
```

**Node-exclusivity caveat (important for GPU jobs).** On a shared GPU node,
even if your job holds all GPUs, another job can still land on the same node if
CPU cores / memory remain free. To keep a whole node to yourself, request the
GPUs **and** cap the node's resources:

```bash
#!/bin/bash
#SBATCH -t 0-10:00:00
#SBATCH --gres=gpu:4
#SBATCH --mem=384000        # take the node's full 384 GB → no other job fits
#SBATCH -J example
# ... your program ...
```

or use exclusive scheduling:

```bash
#!/bin/bash
#SBATCH -t 0-10:00:00
#SBATCH --gres=gpu:4
#SBATCH --exclusive         # CPU=48 cores, mem=192 GB, node not shared
#SBATCH -J example
# ... your program ...
```

Always confirm the live partition/account names and node hostname with
`sinfo -s` / `sacctmgr` **from inside the zone** — as in the general zone,
the running machine is the source of truth and the docs can lag.

---

## (b) Tools — `/usr/local/biotools` + Parabricks

**BioContainers via apptainer works the same as the general zone.** Tool
images live under `/usr/local/biotools/<first-letter>/<tool>:<version>` and you
run them with `apptainer exec` / `singularity exec`. See the
`nig-general-analysis` skill section (b) for the layout, search idiom, version
selection, and `-B` path binding — those apply here unchanged. (Verify the
biotools path from inside the zone; it should be present, but confirm.)

**Parabricks — the GPU genomics workhorse of this zone.** NVIDIA Parabricks
(GPU-accelerated equivalents of GATK/BWA workflows: `fq2bam`, variant callers,
etc.) is the reason most users come to the L40S nodes.

- Parabricks ships as a **Docker image from NVIDIA (free)**; on NIG you run it
  on the L40S GPU nodes (via apptainer, converting/pulling the image per DDBJ's
  Parabricks setup guide).
- Typical use: align + sort FASTQ→BAM with `pbrun fq2bam`, then GPU variant
  calling — hours-long GATK pipelines drop to ~1–2 h on L40S.
- Follow the two-step DDBJ guide: **setup** (obtain/prepare the Parabricks
  container) then **run** (`fq2bam` etc.), combined with the GPU SLURM options
  in section (a). NVIDIA's own tutorial is the reference for `pbrun` commands.

L40S GPUs have 40 GB each; they handle GATK-class genomics well and can run
AI models (e.g. AlphaFold3) within that memory budget.

---

## (c) Data & databases — controlled human data, restricted transfer

Data handling is the other big difference from the general zone.

**Getting data in/out.** With the SSL-VPN up and general internet cut off, the
paths are:

- **`scp` / `sftp`** to the personal-genome gateway `gwa.ddbj.nig.ac.jp` —
  the standard route for user↔zone transfer.
- **Aspera (`ascp`)** — for large NCBI/EBI/DDBJ transfers; install the client
  in your home. NIG's Aspera server is *not* set up for user
  uploads/downloads, so from your own machine use Archaea tools instead
  (Aspera-from-your-PC is not available). The zone-wide Aspera cap is 10 Gbps.
- **Archaea tools** — for large transfers between your PC and your zone home
  (no license speed cap); note the personal-genome Archaea-tools server may be
  listed as "in preparation" — verify availability.
- Inside the zone, only outbound **HTTPS** reaches the internet, so tools that
  fetch references over HTTPS work; arbitrary FTP/other egress does not.

**Databases.** This zone does **not** inherit the general zone's
`/usr/local/resources` + `/usr/local/shared_data` public-DB layout — it is a
separate network with its own (leased-node) filesystem. What reference data is
present depends on the node/lease. **Do not assume general-zone DB paths
exist here.** Determine what's available from inside the zone:

```bash
ls /usr/local/                 # what shared trees exist on this leased node
df -h ; ls ~                   # your home / lustre allocation
```

Controlled-access human datasets (e.g. **JGA** — the Japanese
Genotype-phenotype Archive) are handled here under the NBDC guidelines; access
to a specific dataset is governed by your data-access approval, not by a
world-readable path.

> **TODO (fill from inside the zone):** exact biotools path confirmation,
> the reference-genome / DB locations actually mounted on the leased node,
> and the live `sinfo` partition list. These require a personal-genome
> session and are intentionally left to verify rather than guessed.

---

## Quick recall

- **Separate network**: reach only via **SSL-VPN (FortiClient) + 2FA**, then SSH the gateway `gwa.ddbj.nig.ac.jp`. Account = general name **+ `-pg`**. Paid, node-lease, NBDC-compliant.
- **SLURM**: default = **dedicated leased node** (own Grid Engine/SLURM). Shared **L40S GPU SLURM** for GPU work: `ssh at022vm02` → `#SBATCH --partition=l40s --account=l40s --gres=gpu:N`; use `--mem=384000` or `--exclusive` to keep a node to yourself.
- **Tools**: `/usr/local/biotools` apptainer images (same idiom as general zone) + **Parabricks** on L40S for GPU genomics (`pbrun fq2bam`, GPU GATK).
- **Data**: `scp`/`sftp` to `gwa.ddbj.nig.ac.jp`; Aspera/Archaea for bulk; only outbound HTTPS from inside. **Don't assume general-zone DB paths** — verify on the leased node. JGA = controlled human data under NBDC access approval.
- **From Claude Science**: needs an SSH target routed through the VPN+gateway; VPN/2FA is interactive. `compute_details` first; `remote-compute-ssh` for mechanics.

---

## Official documentation

DDBJ docs (Japanese, English toggle). Treat as orientation; verify live state
from inside the zone.

- Personal-genome division index: https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/
- Overview (占有 vs shared GPU): https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/pg_overview/
- Login (SSL-VPN / FortiClient): https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/pg_login/
- CPU nodes: https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/CPU_nodes/
- L40S GPU nodes: https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/GPU_nodes_type2/
- GPU job-submission notes: https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/job_submission_notes/
- Parabricks: https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/Parabricks/
- Account application: https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/pg_application/
- Data transfer: https://sc.ddbj.nig.ac.jp/guides/using_personal_genome_division/pg_data_transfer/
