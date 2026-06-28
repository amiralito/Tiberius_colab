# Tiberius Gene Prediction + BUSCO (Colab)

Batch *ab initio* gene prediction on Google Colab using
[Tiberius](https://github.com/Gaius-Augustus/Tiberius) — a deep-learning gene finder
(CNN + biLSTM + differentiable HMM) — with built-in **BUSCO** completeness assessment.
Point it at a genome (or a whole folder of them) and it returns per-genome GFF3 + CDS +
protein FASTA, a prediction stats table, and BUSCO scores, all from a single notebook with
form-field inputs.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/<your-username>/<your-repo>/blob/main/Tiberius_gene_prediction.ipynb)

## Pipeline

```
genome FASTA  →   Tiberius            →   GFF3 + CDS + protein   →   BUSCO
(one or many)     (CNN + biLSTM +         (gzipped, per-genome        (proteins and/or
                   differentiable HMM)     subfolders)                 genome mode)
```

## What it does

- Runs Tiberius over **one or many genomes** (recursive folder scan, upload, or URL).
- Names outputs after each genome and prefixes gene/transcript IDs (`tiberius_`).
- **Gzips** all outputs and sorts them into `gff/`, `cds/`, `protein/`, `log/`.
- Builds a per-genome **stats table** (mRNA, proteins, CDS, exons, exons/mRNA, protein-length distribution, total CDS).
- Scores completeness with **BUSCO** in proteins and/or genome mode and **compiles a summary table** across all genomes.
- All code cells are collapsed to **form fields** (`cellView: form`); mount Google Drive to persist inputs/outputs.

## Requirements

- Google Colab with a **GPU runtime** (`Runtime → Change runtime type → GPU`).
- **T4 (16 GB) is sufficient**; an **A100 / L4** is much faster on large genomes.
- No version pinning needed — Colab's default stack (**Python 3.12, TensorFlow 2.19**) already satisfies Tiberius (`python ≥ 3.12`, `tensorflow ≥ 2.17,<2.21`).
- Genome FASTA only (`.fa/.fasta/.fna`, optionally `.gz`). No RNA-seq or protein evidence required.

## Quick start

1. Open the notebook in Colab and select a **GPU runtime**.
2. Run **cell 1** (runtime check), **cell 2** (install Tiberius), **cell 3** (verify GPU).
3. **Cell 4** — optionally mount Drive and set the output folder (`WORKDIR`).
4. **Cell 5** — pick a model, set options, and choose the genome input mode.
5. **Cell 6** — collect/stage the genome(s); **cell 7** — run Tiberius; **cell 8** — view the stats table.
6. **Cells 9–12** — configure and run BUSCO, then compress the results.

## Notebook cells

| Cell | Step |
|------|------|
| 1 | Check runtime — GPU & Python |
| 2 | Install Tiberius (source install; keeps Colab's GPU TensorFlow) |
| 3 | Verify the GPU is visible to TensorFlow; list available model configs |
| 4 | Storage — mount Drive & set `WORKDIR` |
| 5 | Prediction parameters & genome input |
| 6 | Collect genome(s); optional hardmasking |
| 7 | Run Tiberius on all genome(s) → gzipped outputs into `gff/ cds/ protein/ log/` |
| 8 | Per-genome prediction stats table → `prediction_summary.tsv` |
| 9 | BUSCO options (mode toggles + lineage) |
| 10 | Install BUSCO (+ genome-mode tools if enabled) |
| 11 | Run BUSCO + compile `busco_summary.tsv` |
| 12 | Compress & save BUSCO run directories |

## Inputs & parameters (cell 5)

**Genome source** (`GENOME_SOURCE`)
- `drive_dir` — run on **every** FASTA found recursively under `GENOME_DIR`.
- `drive_file` — a single genome already on Drive (`GENOME_PATH`).
- `url` — download a genome from `GENOME_URL`.
- `upload` — upload from your machine.

**Model** (`MODEL_CFG`) — top-level models are **unmasked**:
`angiosperms`, `vertebrates`, `fungi`, `insecta`, `diatoms`, `chlorophyta`, and the `mammalia_*` variants.
The dropdown also lists the **superseded, softmasking-aware** clade models
(`monocotyledonae`, `eudicotyledons`, `angiosperms_softmasking`, `diatoms_softmasking`,
`insecta_softmasking`, `lepidoptera`, and the masked fungal clades). You can also type any
config name or path (`allow-input` is on).

**Other**
- `ID_PREFIX` — prefix for gene/transcript IDs (default `tiberius_`; carries into FASTA headers).
- `HARDMASK_REPEATS` — convert softmasked (lowercase) bases to `N` before prediction. Auto-skipped if a softmasking model is selected.
- `BATCH_SIZE` — `auto`, or set `2`/`4` if the GPU runs out of memory.
- `WANT_CODING`, `WANT_PROTEIN` — also write CDS / protein FASTA.

## Models & softmasking

Tiberius ships **unmasked** clade models (current default) and **softmasking-aware** clade
models (kept under `model_cfg/superseded/`). NCBI RefSeq genomes are softmasked (repeats in
lowercase). The unmasked models **ignore** that mask; on high-repeat genomes (e.g. many large
monocot/grass genomes) this can lead to an unexpectedly low gene count. Two ways to handle it:

1. **Hardmask** the input — enable `HARDMASK_REPEATS` (replaces lowercase with `N`) and keep an unmasked model.
2. **Use a softmasking model** — pick the clade-matched `superseded/*` model, which consumes the lowercase mask directly (leave `HARDMASK_REPEATS` off).

Both usually recover the expected gene count; run BUSCO both ways once and compare.

## Outputs

All written under `WORKDIR` (Drive when mounted, else the ephemeral session):

```
WORKDIR/
├── tiberius_predictions/
│   ├── gff/        <genome>_gff.gff3.gz
│   ├── cds/        <genome>_cds.fasta.gz
│   ├── protein/    <genome>_protein.fasta.gz
│   └── log/        <genome>.log
├── prediction_summary.tsv      # per-genome: mRNA, proteins, CDS, exons, protein-length stats
├── busco_results/              # one zip per BUSCO run
└── busco_summary.tsv           # per-genome C/S/D/F/M, mode, lineage
```

## BUSCO (cells 9–12)

- **proteins mode** — scores the predicted annotation; needs only HMMER (fast).
- **genome mode** — scores the assembly via miniprot; independent of the annotation. Needs bbtools + miniprot and is CPU-bound, so it is much slower (consider an HPC for whole genomes).
- **Lineage** — pick from plant / fungal / algal / generic `eukaryota` datasets, or choose `other` and type any dataset (`busco --list-datasets`). All genomes in a batch use the same lineage, so group same-clade genomes per run.
- BUSCO always runs in a **local** directory (`/content/busco_runs`) because it uses symlinks that the Drive FUSE mount does not support; cell 12 compresses each run into Drive.

## Notes & gotchas

- **OOM on the GPU** → set `BATCH_SIZE` to `2` or `4` (T4 ≈ 4; 24 GB ≈ 8; 80 GB ≈ 16).
- **GPU not detected after install** → `Runtime → Restart session`, then re-run cells 1–3 (no reinstall).
- **Edit a hidden cell** → cell menu (⋮) → *Show code*.
- **Genome BUSCO is slow** on Colab cores — fine for scaffolds, better on HPC for whole genomes.
- Results persist on Drive when mounted; otherwise the save cells offer downloads.

## Citation

If you use this notebook, please cite Tiberius and the tools it depends on.

- **Tiberius** — Gabriel L, Becker F, Hoff KJ, Stanke M. *Tiberius: end-to-end deep learning with an HMM for gene prediction.* Bioinformatics. 2024;40(12):btae685. doi:10.1093/bioinformatics/btae685
- **Tiberius (multi-clade models)** — Gabriel L, Brůna T, Kaur A, et al. *Accurate ab initio gene prediction in eukaryotes with Tiberius in multiple clades.* bioRxiv. 2026. doi:10.64898/2026.04.24.720536
- **BUSCO** — Manni M, Berkeley MR, Seppey M, Simão FA, Zdobnov EM. *BUSCO Update: Novel and Streamlined Workflows along with Broader and Deeper Phylogenetic Coverage for Scoring of Eukaryotic, Prokaryotic, and Viral Genomes.* Mol Biol Evol. 2021;38(10):4647–4654. doi:10.1093/molbev/msab199
- **miniprot** (BUSCO genome mode) — Li H. *Protein-to-genome alignment with miniprot.* Bioinformatics. 2023;39(1):btad014. doi:10.1093/bioinformatics/btad014
- **OrthoDB** (BUSCO lineage datasets) — Kuznetsov D, Tegenfeldt F, Manni M, et al. *OrthoDB v11: annotation of orthologs in the widest sampling of organismal diversity.* Nucleic Acids Res. 2023;51(D1):D445–D451. doi:10.1093/nar/gkac998
- **HMMER** — hmmer.org · **BBTools/BBMap** — Bushnell B., sourceforge.net/projects/bbmap

```bibtex
@article{gabriel2024tiberius,
  author  = {Gabriel, Lars and Becker, Felix and Hoff, Katharina J. and Stanke, Mario},
  title   = {{Tiberius}: end-to-end deep learning with an {HMM} for gene prediction},
  journal = {Bioinformatics},
  volume  = {40},
  number  = {12},
  pages   = {btae685},
  year    = {2024},
  doi     = {10.1093/bioinformatics/btae685}
}

@article{gabriel2026clades,
  author  = {Gabriel, Lars and Br{\r{u}}na, Tom{\'a}{\v{s}} and Kaur, Asees and Krishnan, Anish and Ortmann, Felix and Salamov, Asaf and Talbot, Samuel and Becker, Felix and Krieg, Richard and Wheat, Christopher W. and Grigoriev, Igor V. and Stanke, Mario and Hoff, Katharina J.},
  title   = {Accurate ab initio gene prediction in eukaryotes with {Tiberius} in multiple clades},
  journal = {bioRxiv},
  year    = {2026},
  doi     = {10.64898/2026.04.24.720536}
}

@article{manni2021busco,
  author  = {Manni, Mos{\`e} and Berkeley, Matthew R. and Seppey, Mathieu and Sim{\~a}o, Felipe A. and Zdobnov, Evgeny M.},
  title   = {{BUSCO} Update: Novel and Streamlined Workflows along with Broader and Deeper Phylogenetic Coverage},
  journal = {Molecular Biology and Evolution},
  volume  = {38},
  number  = {10},
  pages   = {4647--4654},
  year    = {2021},
  doi     = {10.1093/molbev/msab199}
}
```

## License & acknowledgements

This notebook is a convenience wrapper. Tiberius, BUSCO, miniprot, HMMER, and BBTools are the
work of their respective authors and carry their own licenses — please consult each upstream
repository. Model weights are downloaded from the Tiberius project servers at runtime.
