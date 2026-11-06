# User guide

SingleCell_VDJ provides end-to-end pipelines for **single-cell full-length V(D)J (TCR/BCR)**
data derived from Singleron's sCircle<sup>TM</sup> kit.

## Installation

### Docker (recommended)
```
docker pull quay.io/singleron-rd/singlecell_vdj:v2.1.1
docker run --rm -it -v /path/to/data:/data quay.io/singleron-rd/singlecell_vdj:v2.1.1 /bin/bash
```

### Conda + pip
```
git clone https://github.com/Chenjunjie1996/SingleCell_VDJ.git
cd SingleCell_VDJ
conda install mamba
mamba create -n SingleCell_VDJ -y --file conda_pkgs.txt
conda activate SingleCell_VDJ
pip install .
```

## Pipelines

Each assay exposes two kinds of commands:

- `multi_{assay}`: generate per-sample job scripts (`shell` mode) or an SJM job file (`sjm` mode). **Recommended.**
- `celescope {assay} {step}`: run a single step of a single sample (advanced usage).

| assay | assembler | 
|---|---|
| [flv_CR](./assay/multi_flv_CR.md) | Cell Ranger VDJ |
| [flv_trust4](./assay/multi_flv_trust4.md) | TRUST4 |


## flv_CR vs flv_trust4

Both pipelines start from the same sCircle VDJ libraries, but differ in the assembler and
downstream steps. The overall flow is:

```
read QC  →  assembler  →  cell calling  →  scRNA matching  →  report
 barcode    └─ see below
 convert
```

| | flv_CR | flv_trust4 |
|---|---|---|
| assembler | **Cell Ranger VDJ** (external software, must download separately) | **TRUST4** (bundled with the conda env, no extra install) |
| prep step | Runs `convert` to produce 10x-style fastq (`R1` + `R2`) that Cell Ranger expects | No convert — runs directly on Singleron fastq |
| assemble step | Runs `cellranger vdj`. A **refine pass** runs by default after Cell Ranger: barcodes marked `is_cell=False` but carrying multiple productive chains are re-assembled to recover real cells Cell Ranger missed. Disable with `--not_refine`. | Splits candidate reads into 4 barcode-based chunks (run in parallel via multiprocessing, TRUST4 itself is single-threaded), then merges results. Disable chunking with `--not_split`. |
| downstream steps | `summarize` → `match` (explicit step) → `mapping` (overlay on scRNA t-SNE) | `summarize` (also does cell calling) → `annotation` (matches to scRNA and renders the html report; no separate `mapping` step) |
| cell calling | Cell Ranger's built-in cell calling + optional refine pass | Custom cell calling with three knobs: `--diffuseFrac`, `--expected_target_cell_num`, and `--target_cell_barcode` + `--target_weight` for target-cell weighting |
| scRNA integration | Two explicit steps: `match` writes `matched_*` files, then `mapping` overlays on t-SNE | Single `annotation` step — restricts VDJ results to matched scRNA cells and renders the report |
| reference | Download the **Cell Ranger VDJ reference** (GRCh38/GRCm38 from 10x); point `--ref_path` to the unpacked directory | Bundled TRUST4 references inside the conda env; pick with `--ref` (`hg38`, `hg19`, `GRCm38`) |

## Typical workflow

Two chained steps:
1. Run **scRNA library** with [CeleScope](https://github.com/singleron-RD/CeleScope), which
   produces a matched directory (e.g. `sample1_rna/`).
2. Run one of `multi_flv_CR` / `multi_flv_trust4`. **Column 4 of the mapfile must point to
   the matched scRNA directory.**
