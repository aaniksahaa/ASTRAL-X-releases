# ASTRAL-X

## A Fundamental Computational Redesign for Scalable Coalescent-Based Species Tree Inference

> **Current availability:** ASTRAL-X release archives are available from this
> repository's [Releases page](../../releases). We plan to open-source ASTRAL-X
> soon.

ASTRAL-X is a complete algorithmic redesign of the ASTRAL framework for highly
scalable, statistically consistent species tree inference from collections of
gene trees. Compact data representations, memory-efficient search-space
construction, and GPU-accelerated computation preserve ASTRAL's quartet-based
optimization and statistical guarantees while making analyses with hundreds of
thousands of taxa practical.

ASTRAL-X reconstructed a 300,000-taxon species tree within only
12 hours using 100 GB of memory, and inferred the evolutionary history of 9,524
angiosperm species in just 16 minutes.

An NVIDIA CUDA GPU is strongly recommended, particularly for large datasets,
and is the primary high-performance execution path. CPU execution remains
available as a reliable fallback for compatibility and smaller analyses. The
release archive includes its own Java runtime and automatically uses CUDA when
it is available.

## One-time setup

Keep the complete extracted application directory together: the launcher needs
the accompanying `bin/` and `lib/` directories.

Download `astralx-1.0.0-linux-x86_64.tar.gz` from the
[Releases page](../../releases). The archive is self-contained. Replace
`/path/to/downloaded/` below with the archive's actual location, then run this
setup block. It installs ASTRAL-X under `~/.local/opt/` and adds the launcher to
your user `PATH` without requiring `sudo`:

```bash
ASTRALX_INSTALL_ROOT="$HOME/.local/opt/astralx/1.0.0"
mkdir -p "$ASTRALX_INSTALL_ROOT"
tar -xzf /path/to/downloaded/astralx-1.0.0-linux-x86_64.tar.gz \
  -C "$ASTRALX_INSTALL_ROOT"
ASTRALX_DIR="$(realpath "$ASTRALX_INSTALL_ROOT/astralx-1.0.0-linux-x86_64")"
mkdir -p "$HOME/.local/bin"
ln -sfn "$ASTRALX_DIR/astralx" "$HOME/.local/bin/astralx"
grep -qxF 'export PATH="$HOME/.local/bin:$PATH"' "$HOME/.bashrc" || \
  printf '%s\n' 'export PATH="$HOME/.local/bin:$PATH"' >> "$HOME/.bashrc"
export PATH="$HOME/.local/bin:$PATH"
```

### Check the installation

Verify that the launcher is available:

```bash
astralx --version
```

## Quick start

After the one-time setup, ASTRAL-X can be run from any directory:

```bash
astralx -i /path/to/gene_trees.tre -o /path/to/output_species_tree.tre
```

Input and output may be relative to the current directory or given as absolute
paths. For a directly runnable example, change to the extracted application
directory and use the included 37-taxon dataset:

```bash
astralx -i example/all_gt_37.tre -o example/out_astralx_37.tre
```

The defaults are `--auto`, `--search-space S1`, and
`--intersection-method I2`: ASTRAL-X tries CUDA first, safely falls back to CPU,
uses the smallest search space, and uses prefix-sum intersections.

For a broader search on incomplete gene trees using cross-tree recombined
transitions as well as tree-local ones:

```bash
astralx -i /path/to/gene_trees.tre -o /path/to/output_species_tree.tre \
  --search-space S2 --intersection-method I2
```

Use `astralx --help` for the complete option list and `astralx --diagnose` to
check the packaged runtime, native libraries, driver, and GPU selection without
loading a dataset.

## Search-space presets

`--search-space` controls how broadly ASTRAL-X explores candidate species-tree
topologies. Choose `S1`, `S2`, or `S3`; bare numbers such as
`--search-space 2` are also accepted. “Complete” below means that missing taxa
are inserted into incomplete gene trees while constructing the search space.
Quartet weights are still calculated from the original gene trees.

| Preset | Search space | What it enables |
|---|---|---|
| **S1** | Incomplete, local | Uses topology candidates found directly within each original gene tree. Fastest and the default. |
| **S2** | Complete, full | Completes incomplete gene trees, constructs a distance-based guide tree, and combines compatible candidates across trees. Recommended broader search. |
| **S3** | Exhaustive | Includes everything in S2, then adds consensus-derived candidates, denser nearest-neighbour groups, remaining consensus-polytomy resolutions, large-polytomy handling, and resolutions derived from polytomous input gene trees. |

Moving from S1 to S3 progressively broadens the candidate topology set. A
larger preset can increase runtime and memory substantially and is not
guaranteed to change the inferred tree.

Most analyses only need one search-space preset. Individual search controls are
also available for specialized workflows; `astralx --help` lists them. When
options are combined, they are applied from left to right. A later individual
option can refine a preset, while a later preset selects its complete predefined
configuration.

## Intersection methods

Select an intersection method with `--intersection-method` (short form `--im`).
All four methods compute the same quartet weights; they differ only in their
memory use and performance strategy.

| Method | Name | Typical use |
|---|---|---|
| **I1** | Smaller-side traversal | Low auxiliary memory; directly walks the smaller side of each intersection. |
| **I2** | Prefix sum | General-purpose default with constant-time range intersections. |
| **I3** | Simple tree walk | Lean traversal that is often useful with very large full-search candidate sets. |
| **I4** | Bitset | Popcount-based path, usually strongest for smaller taxon sets and many gene trees. |

For example, `--im I3` and
`--weight-intersection-method simple-tree-walk` are equivalent; both forms are
accepted.
