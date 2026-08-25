# InSAR Time-Series Analysis of Ground Surface Displacement

> **Self-directed research project** | Tools: ARIA-tools · MintPy · Dolphin · OPERA DISP-S1 · ISCE+

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Notebooks](https://img.shields.io/badge/Jupyter-Notebooks-orange)](notebooks/)

---

## Overview

This repository documents my end-to-end InSAR time-series processing pipeline for measuring ground surface displacement using Sentinel-1 SAR data. The project spans three complementary workflows — ARIA/MintPy SBAS, Dolphin PS/DS phase linking, and OPERA DISP-S1 ready-made displacement products — and demonstrates how each approach can be used to extract mm-level ground deformation signals.

The notebooks in this repo are adapted from the [2025 EarthScope ISCE+ Short Course](https://github.com/isceplus/2025-isceplus), which I attended/used to build a hands-on understanding of the full InSAR time-series processing chain.

---

## Key Results

| Metric | Value |
|---|---|
| Study Area | San Francisco Bay Area & New Orleans |
| Dataset | Sentinel-1 GUNW ARIA products (Track 42) |
| LOS Velocity across San Andreas Fault | **~13 mm/year** |
| Geophysical Interpretation | Consistent with Pacific–North American plate motion observed by GPS |
| Corrections Applied | ERA5 tropospheric delay, unwrapping error correction, water masking |

---

## Workflows

### 1. ARIA-tools + MintPy SBAS (`smallbaselineApp_aria.ipynb`)

The classical Small Baseline Subset (SBAS) time-series approach applied to ARIA GUNW interferogram products over the San Francisco Bay Area.

**Pipeline steps:**
- Download Sentinel-1 GUNW products via `ariaDownload.py` over bounding box `[37.25°N–38.1°N, 122.6°W–121.75°W]`
- Prepare interferogram stack using `ariaTSsetup.py` with water masking and coherence filtering
- Load into MintPy via `prep_aria.py`, generating `ifgramStack.h5` and `geometryGeo.h5`
- Run `smallbaselineApp.py` SBAS workflow:
  - `load_data` → `reference_point` → `correct_unwrap_error`
  - `invert_network` (weighted least-squares)
  - `correct_troposphere` (ERA5 atmospheric delay correction)
  - `velocity` (linear LOS velocity estimation)
- Validate against GNSS station time series

**Output:** LOS displacement time-series and velocity map showing ~13 mm/year relative velocity across the San Andreas Fault.

---

### 2. Dolphin PS/DS Phase Linking — ISCE2 SLCs (`dolphin-isce2.ipynb`)

Phase linking workflow for persistent scatterer (PS) and distributed scatterer (DS) analysis using coregistered SLC stacks from ISCE2.

**Pipeline steps:**
- Configure `dolphin_config.yaml` with half-window size, stride, and interferogram network parameters
- Run `dolphin run dolphin_config.yaml` to execute:
  - SHP (Statistically Homogeneous Pixel) detection
  - Mini-stack phase linking (EVD/EMI)
  - Interferogram network formation and unwrapping (SNAPHU)
  - Displacement time-series and velocity map inversion
- Inspect outputs: temporal coherence, amplitude dispersion (PS mask), velocity map

**Study Area:** New Orleans, LA — land subsidence monitoring

---

### 3. Dolphin PS/DS Phase Linking — OPERA CSLCs (`dolphin-new-orleans-cslcs.ipynb`)

Same phase linking workflow but driven by OPERA geocoded co-registered SLCs (CSLCs) downloaded directly from ASF, demonstrating multi-burst stitching.

**Pipeline steps:**
- Search and download OPERA CSLCs via `opera-utils` for burst IDs `T165-352434-IW2` / `T165-352435-IW2` (New Orleans, ascending track 165)
- Configure Dolphin with:
  - Spatial strides `--sx 6 --sy 3` (30 m output)
  - Max-bandwidth-3 interferogram network
  - Water mask from ASF S3 bucket
  - SNAPHU unwrapping with tiled parallelism
- Run `dolphin run` and visualize displacement time-series rasters

---

### 4. OPERA DISP-S1 Time-Series Extraction (`opera-disp-02-timeseries.ipynb`)

Extraction and analysis of the NASA OPERA Level-3 Displacement product (DISP-S1), a ready-to-use geocoded displacement dataset covering California Frame 11116.

**Pipeline steps:**
- Download OPERA DISP-S1 NetCDF products via `opera-utils disp-s1-download` (2020–2022, `--frame-id 11116`)
- Reformat with moving-reference correction using `opera-utils disp-s1-reformat` (cumulative sum, `BORDER` reference method)
- Load into `xarray`, apply temporal coherence & phase similarity masking, coarsen to 90 m
- Extract pixel-level LOS displacement time-series at specific lat/lon coordinates
- Convert to MintPy-compatible HDF5 format for further analysis

**Study Area:** Central California — fault motion and landslide displacement near the San Andreas system

---

## Repository Structure

```
insar-timeseries-analysis/
│
├── notebooks/
│   ├── smallbaselineApp_aria.ipynb       # ARIA + MintPy SBAS workflow (San Francisco)
│   ├── dolphin-isce2.ipynb               # Dolphin PS/DS with ISCE2 SLCs (New Orleans)
│   ├── dolphin-new-orleans-cslcs.ipynb   # Dolphin PS/DS with OPERA CSLCs (New Orleans)
│   └── opera-disp-02-timeseries.ipynb    # OPERA DISP-S1 time-series (Central California)
│
├── environment/
│   ├── environment.yml                   # Conda environment spec
│   └── requirements.txt                  # pip-installable dependencies
│
├── docs/
│   └── workflow_diagram.png              # End-to-end pipeline diagram
│
└── README.md
```

---

## Environment Setup

### Option A: Conda (Recommended)

```bash
conda env create -f environment/environment.yml
conda activate insar-env
```

### Option B: pip

```bash
pip install mintpy aria-tools dolphin opera-utils
```

### Key Dependencies

| Package | Purpose |
|---|---|
| `mintpy` | SBAS time-series inversion, corrections, velocity estimation |
| `aria-tools` | Download and prepare ARIA GUNW products |
| `dolphin` | PS/DS phase linking, OPERA CSLC processing |
| `opera-utils` | OPERA DISP-S1 download, reformatting, MintPy conversion |
| `isce2` / `isce3` | Underlying SAR processor for SLC coregistration |
| `snaphu-py` | Phase unwrapping |
| `xarray`, `rioxarray` | NetCDF/raster data handling |
| `gdal` | Geospatial raster I/O |

---

## Scientific Background

### What is InSAR?

Interferometric Synthetic Aperture Radar (InSAR) measures the phase difference between two SAR acquisitions of the same area taken at different times. This phase difference encodes the line-of-sight (LOS) displacement of the ground surface to sub-centimeter precision.

### SBAS vs. Phase Linking

| Method | Approach | Best for |
|---|---|---|
| **SBAS (MintPy)** | Inversion of unwrapped interferogram network | Distributed scatterers, broad deformation fields |
| **Phase Linking (Dolphin)** | Optimal phase estimation from full covariance matrix | PS/DS mixed scenes, urban + agricultural areas |
| **OPERA DISP-S1** | Pre-computed, analysis-ready product | Rapid analysis without raw SLC processing |

### Key Corrections Applied

- **Tropospheric delay (ERA5):** Atmospheric water vapor causes apparent range delay; corrected using ERA5 reanalysis weather model
- **Unwrapping error correction:** Integer-cycle phase jumps from SNAPHU are identified and corrected using triplet consistency checks
- **Solid Earth Tides / Ionosphere:** Available in OPERA DISP-S1 products (not applied by default)
- **Water masking:** Ocean and large water bodies masked using ARIA/ASF water mask layers

---

## Results & Interpretation

The MintPy SBAS velocity map over the San Francisco Bay Area reveals:

- **~13 mm/year LOS velocity gradient** across the San Andreas Fault (Hayward–Calaveras segment)
- Positive LOS velocities (motion toward satellite) on the Pacific Plate side
- Negative velocities on the North American Plate side
- Pattern is consistent with ~35–40 mm/year right-lateral strike-slip motion projected into the Sentinel-1 descending LOS geometry, in agreement with GNSS-observed plate motion

---

## Acknowledgements

Notebooks adapted from the [2025 EarthScope ISCE+ Short Course](https://github.com/isceplus/2025-isceplus) (August 18–22, 2025), instructors: Zhang Yunjun, Heresh Fattahi, Scott Staniewicz, Sara Mirzaee, and the broader MintPy / Dolphin / OPERA teams.

- [MintPy](https://github.com/insarlab/MintPy) — Yunjun et al. (2019), *Computers & Geosciences*
- [Dolphin](https://github.com/isce-framework/dolphin) — Staniewicz et al. (2024), *JOSS*
- [ARIA-tools](https://github.com/aria-tools/ARIA-tools)
- [opera-utils](https://github.com/opera-adt/opera-utils)

---

## References

- Yunjun, Z., Fattahi, H., Amelung, F. (2019). Small baseline InSAR time series analysis: Unwrapping error correction and noise reduction. *Computers & Geosciences, 133*, 104331. [doi:10.1016/j.cageo.2019.104331](https://doi.org/10.1016/j.cageo.2019.104331)

- Staniewicz, S., Mirzaee, S., Gunter, G. M., Cabrera, T. O., Havazli, E., Fattahi, H. (2024). Dolphin: A Python package for large-scale InSAR PS/DS processing. *Journal of Open Source Software, 9*(103), 6997. [doi:10.21105/joss.06997](https://doi.org/10.21105/joss.06997)
