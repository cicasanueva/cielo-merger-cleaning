<div align="center">

# CIELO Merger-Tree Cleaning Pipeline

[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.19342289-c9b8e8?style=flat-square)](https://doi.org/10.5281/zenodo.19342289)
[![License: MIT](https://img.shields.io/badge/License-MIT-b8e8d0?style=flat-square)](LICENSE)
[![Python](https://img.shields.io/badge/Python-≥_3.8-b8d4e8?style=flat-square)](https://www.python.org/)
[![Pipeline](https://img.shields.io/badge/Pipeline-4_steps-f0d4b8?style=flat-square)](#pipeline-structure)

**Author:** Catalina Casanueva-Villarreal (cicasanueva@uc.cl) · **Version:** 2.0 · **License:** MIT

</div>

---

## Overview

This repository contains a Python pipeline to detect, characterize, and clean merger trees for central galaxies in cosmological zoom-in simulations. It is designed to work with simulations based on the GADGET-3 N-body/SPH code that use the AMIGA Halo Finder (AHF; Knollmann & Knebe 2009) merger tree format.

The main goal of the pipeline is to produce a physically consistent, duplicate-free catalogue of baryonic merger events for each central galaxy. The key contribution is the **overlap-based cleaning algorithm** (Step 4), which classifies each merger according to its orbital history and mass-transfer efficiency, then applies sequential particle-ID overlap filters to consolidate spurious tree entries arising from the fragmentation of a single infalling object into multiple subhaloes by the structure finder.

---

## Pipeline Structure

The pipeline runs in four sequential steps:

| Step | Script | Description |
|---|---|---|
| 1 | `step1_CM_r200.py` | Computes the centre of mass (CM) and virial radius R₂₀₀ along the main progenitor branch of each central galaxy |
| 2 | `step2_all_mergers_tree.py` | Detects all merger branches that physically merge into the central's main branch using the AMIGA tree |
| 3 | `step3_baryonic_mergers_caract.py` | Characterizes baryonic and dark matter content at infall, and measures mass transfer fractions to the z=0 central |
| 4 | `step4_depuracion_mergers_trees.py` | Core cleaning: classifies mergers into six groups, detects particle-level overlaps, and applies sequential filters to produce the final catalogue |

---

## Step-by-step Description

### Step 1 — Centre of Mass and R₂₀₀

**Script:** `step1_CM_r200.py`  
**Function:** `compute_cm_r200(sim_name, snap_z0, mass_cut, galaxy_type='central')`

At z=0, all central galaxies (`SubGroupNumber = 0`) with stellar mass M★ > `mass_cut` are selected. For each central, the full main progenitor branch is traversed snapshot by snapshot.

At each snapshot:
- **Centre of mass:** Recomputed using an iterative shrinking-sphere algorithm applied to stellar particles (PartType4) belonging to the subhalo. This is applied only when the subhalo contains ≥ 100 stellar particles; otherwise the SubFind-tabulated position (`SubGroupPos`) from the HDF5 is used and `flag_cm_recomputed` is set to `False`.
- **R₂₀₀:** Computed by building an enclosed-density profile ρ(<r) using all particles of the group (gas, dark matter, and stars; PartType 0, 1, and 4). The profile is interpolated in log–log space to find the radius where ρ(<r) = 200 × ρ_crit(z).

**Output:** `Datos/CM_R200/r200_mainbranches_all_central_z0_{sim}.csv`  
One row per snapshot per central. Key columns: `uid`, `snap`, `z`, `xcm_star`, `ycm_star`, `zcm_star`, `r200_kpc`, `flag_cm_recomputed`.

---

### Step 2 — Merger Detection from the Tree

**Script:** `step2_all_mergers_tree.py`  
**Function:** `detect_mergers_from_tree(sim_name, target_subfind_id, snap_target)`

The AMIGA merger tree is loaded as a directed graph using NetworkX. The main progenitor branch of the target central (`uid_target`) is extracted via depth-first search (DFS).

All secondary branches that merge into the central's main branch are identified by computing `single_target_shortest_path` on the reversed graph. For each path from any node to `uid_target`:
- Nodes already belonging to the main branch are excluded.
- Branches that previously merged into another secondary branch are excluded so that the pre-merger object is not double-counted; only the most massive progenitor at each intermediate merger is retained.

For each valid secondary merger the following are recorded: the UID of the satellite at the last snapshot before merging (`uid_before_merge`), the merger snapshot (`snap_merge`), the corresponding redshift (`z_merge`), the main-branch node where the merger occurs (`uid_merge`), and the root node of the secondary branch in the tree (`origin_node`).

**Output:** `Datos/All_Mergers/mergers_{sim}_gx_{gxid}_summary.csv`

---

### Step 3 — Baryonic Characterization and Infall Time

**Script:** `step3_baryonic_mergers_caract.py`  
**Function:** `characterize_baryonic_mergers(sim_name, target_subfind_id, snap_target)`

For each merger detected in Step 2, this step defines the **infall time** (t_infall) and measures the baryonic content of the satellite at that moment.

**Definition of t_infall:** The last snapshot at which the physical distance between the satellite's CM and the central's CM exceeds the central's R₂₀₀ (from Step 1). The snapshot immediately after this is when the satellite is first considered to be inside the halo.
- If the satellite is always inside R₂₀₀ throughout its tracked history: `flag_always_inside_r200 = True`, and the earliest available snapshot is used as the reference.
- If the satellite never crosses R₂₀₀: `flag_never_inside_r200 = True`, and the latest available snapshot is used.

At t_infall, CMs are recomputed with the shrinking-sphere algorithm (if ≥ 100 stars are available; otherwise the HDF5 value is used), for both the satellite (`flag_CM_hdf5_tinfall_sat`) and the central (`flag_CM_hdf5_tinfall_cen`).

**Masses at t_infall:** Stellar (`mass_stars_infall`), gas (`mass_gas_infall`), and dark matter (`mass_dm_infall`) masses are read from the HDF5 at the infall snapshot.

**Mass transfer fractions:** For each component, the fraction of the satellite's infall mass that ends up in the z=0 central galaxy is computed by direct particle-ID matching. This yields `Mstell_frac_t_infall_in_central_z0`, `Mgas_frac_t_infall_in_central_z0`, and `Mdm_frac_t_infall_in_central_z0`.

Additionally, stellar particles in the z=0 central that formed from the satellite's gas — both those already existing as stars at infall (1st generation) and those that formed from the satellite's gas after infall (2nd generation) — are identified and their masses recorded.

**Flags:** `flag_had_stars`, `flag_had_gas`, and `flag_had_dm` indicate whether the satellite had each component type at any snapshot during its tracked history.

**Output:** `Datos/Baryon_Mergers/baryon_mergers_{sim}_gx_{gxid}_summary.csv`

---

### Step 4 — Merger Tree Cleaning (Core Algorithm)

**Script:** `step4_depuracion_mergers_trees.py`  
**Function:** `clean_merger_catalog(sim_name, target_subfind_id, snap_target)`

This is the core of the pipeline. Only mergers with non-zero stellar or gas mass at infall are retained. Sequential overlap filters are applied.

#### Classification into six groups

Each merger is first assigned to one of six groups based on its orbital history and mass-transfer efficiency:

| Group | Orbital history | Transfer |
|---|---|---|
| A1 | Always inside R₂₀₀ | Low (all fractions < 10%) |
| A2 | Always inside R₂₀₀ | High (at least one fraction ≥ 10%) |
| B1 | Never inside R₂₀₀ | Low |
| B2 | Never inside R₂₀₀ | High |
| C1 | Crossed R₂₀₀ at some point | Low |
| C2 | Crossed R₂₀₀ at some point | High |

The transfer threshold is 10% for all mass-fraction columns (`Mstell_frac`, `Mgas_frac`, `Mdm_frac`, `frac_star1`, `frac_star2`).

#### Particle-ID loading at t_infall

For every merger, the particle IDs of all gas (PartType0), stellar (PartType4), and dark matter (PartType1) particles belonging to the satellite subhalo at t_infall are loaded from the HDF5. This particle catalogue is the basis for overlap detection.

#### Overlap graph construction

For each pair of mergers (A, B) sharing an infall snapshot, the overlap is evaluated as follows:
1. The **dominant component** (largest mass at infall: stars, gas, or DM) is identified for merger A.
2. A strong overlap is declared if ≥ 90% of merger A's dominant-component particles also belong to merger B.
3. If a secondary component of merger A contains > 100 particles, that component must also overlap merger B at ≥ 50%.

Pairs satisfying these criteria are connected as edges in a NetworkX overlap graph. Connected components of this graph form groups of physically redundant merger events.

#### Filter 1 — C2-start conservancy

If the earliest merger within an overlap-connected group is of type C2, that C2 is conserved. All other members structurally contained within it (dominant overlap ≥ 75% and secondary overlap ≥ 50% if > 100 particles) are flagged for removal (`flag_overlap_remove_start_c2`). The disruption UID of the conserved C2 is updated to the latest disruption time in the group.

#### Filter 2 — C2-containing conservancy

If the group contains a C2 but the earliest merger is not a C2, the oldest C2 in the group is conserved and later mergers contained within it are removed (`flag_overlap_remove_contains_c2`).

#### Filter 3 — B2/A2 overlap (no C merger in group)

If the group contains only B2 and A2 mergers (no C type), the oldest B2 is conserved and any A2 mergers contained within it are removed (`flag_overlap_remove_no_c`).

#### Filter 4 — Contiguous-snapshot test

For **A2 mergers**: the dominant subhalo in the snapshot immediately before the satellite's birth snapshot is checked. If that subhalo belongs to the central's main branch, the A2 is considered a spurious fragmentation and is removed (`flag_overlap_remove_no_c`).

For **B2 mergers**: the dominant subhalo in the snapshot immediately after the satellite's disruption is checked. If it corresponds to the central, the B2 is retained as a real instantaneous merger.

**Outputs:**
- `Datos/Baryon_Mergers_Cleaned/baryon_mergers_dep_{sim}_gx{gxid}.csv` — classified catalogue with all overlap flags
- `Datos/Baryon_Mergers_Cleaned/FINAL_resumen_clean_mergers_{sim}_gx{gxid}.csv` — final cleaned catalogue (flagged mergers removed, disruption UIDs updated)

---

## Key Parameters and Thresholds

| Parameter | Value | Description |
|---|---|---|
| Minimum stars for shrinking-sphere CM | 100 | Below this, the SubFind-tabulated CM is used |
| Low-transfer threshold | 10% (0.1) | Applied to all mass-fraction columns |
| Dominant component overlap (overlap detection) | ≥ 90% | Required to declare a strong overlap between two mergers |
| Dominant component overlap (conservancy removal) | ≥ 75% | Required to flag a merger for removal within an overlap group |
| Secondary component overlap | ≥ 50% | Required only when the secondary component has > 100 particles |
| R₂₀₀ definition | 200 × ρ_crit(z) | Mean enclosed density criterion; all group particles used |

---

## Dependencies

- Python ≥ 3.8
- `numpy`, `scipy`, `pandas`, `h5py`, `networkx`, `astropy`, `tqdm`, `matplotlib`, `seaborn`

See `requirements.txt` for pinned versions.

---

## Data Format

Simulation snapshots must be provided in HDF5 format with the GADGET-3/CIELO particle data structure. The merger tree must be in AMIGA multiline adjacency list format. Input paths are configured via `config.py`.

For detailed column-by-column descriptions of all output files, see [SCHEMA.md](SCHEMA.md).

---

## Usage

Steps must be run in order. Each step reads the outputs of the previous one.

```python
from step1_CM_r200 import compute_cm_r200
from step2_all_mergers_tree import detect_mergers_from_tree
from step3_baryonic_mergers_caract import characterize_baryonic_mergers
from step4_depuracion_mergers_trees import clean_merger_catalog

sim_name = "LG1"
snap_z0  = 128
gxid     = 4337
mass_cut = 1e9  # M☉

compute_cm_r200(sim_name, snap_z0, mass_cut)
detect_mergers_from_tree(sim_name, gxid, snap_z0)
characterize_baryonic_mergers(sim_name, gxid, snap_z0)
clean_merger_catalog(sim_name, gxid, snap_z0)
```

Or via the command line (Step 4 includes a CLI entry point):

```bash
python step4_depuracion_mergers_trees.py --sim LG1 --gx 4337 --snap0 128
```
