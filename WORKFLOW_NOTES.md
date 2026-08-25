# Workflow Notes & Tips

Practical notes from running each notebook, including gotchas and parameter choices.

---

## smallbaselineApp_aria.ipynb — MintPy + ARIA GUNW

### Data access
- Requires `awscli` configured with credentials, or a Zenodo download (slower)
- Bounding box used: `37.25 38.1 -122.6 -121.75` (San Francisco Bay, Track 42)
- Pre-staged data is at `s3://asf-jupyter-data-west/unavco2022/SanFranSenDT42.tar.gz`

### Key config parameters (SanFranSenDT42.txt)
```
mintpy.reference.lalo           = 37.69, -122.07    # stable rock reference pixel
mintpy.troposphericDelay.method = pyaps             # ERA5 via PyAPS (change from 'no' for real analysis)
mintpy.networkInversion.weightFunc = var            # variance-based weights (better than 'no')
```

> **Note:** The notebooks use simplified settings (`weightFunc = no`, `troposphere = no`) for speed.
> For publication-quality results, enable ERA5 tropospheric correction and variance weighting.

### Unwrapping error correction
MintPy uses triplet consistency — if the sum of phases around a closed triangle of interferograms is non-zero, an integer-cycle error exists. The `correct_unwrap_error` step identifies and corrects these.

### GNSS validation
MintPy can compare InSAR LOS velocities directly to projected GNSS velocities. Load GNSS from UNR or UNAVCO using `save_gnss.py` and `view.py --ref-gnss`.

---

## dolphin-isce2.ipynb — Dolphin with ISCE2 SLCs

### Dataset
- New Orleans, ascending track
- ISCE2 coregistered SLCs from AWS: `s3://earthscope-insar2025/dolphin-isce2/NewOrleans.tar.gz`

### Important config parameters
```yaml
strides: {x: 6, y: 3}         # ~30 m output from 5 m ISCE2 products
half_window: {x: 25, y: 25}   # SHP search window — larger = smoother but slower
interferogram_network:
  max_bandwidth: 3             # only form pairs within 3 acquisitions of each other
```

### Outputs to check
| File | Description |
|---|---|
| `timeseries/*.tif` | Per-date unwrapped LOS displacement |
| `velocity.tif` | Linear LOS velocity |
| `temporal_coherence.tif` | Quality metric — mask below 0.6–0.7 |
| `ps_mask.tif` | Persistent scatterer mask |
| `amplitude_dispersion.tif` | Used to identify PS pixels |

---

## dolphin-new-orleans-cslcs.ipynb — Dolphin with OPERA CSLCs

### Data search & download
- Requires NASA Earthdata credentials in `~/.netrc`
- Uses `opera_utils.download.search_cslcs()` with `check_missing_data=True` to ensure consistent burst coverage

### Burst IDs for New Orleans (ascending track 165)
```python
burst_ids = ["T165-352434-IW2", "T165-352435-IW2"]
```

### Multi-burst stitching
Dolphin automatically stitches burst-wise interferograms before unwrapping when multiple burst IDs are provided. Use `--n-parallel-bursts 2` to process them in parallel.

### Water mask
```bash
aws s3 cp s3://earthscope-insar2025/dolphin-opera-cslcs/water_mask.tif ./
```

---

## opera-disp-02-timeseries.ipynb — OPERA DISP-S1

### Product features
- Frame 11116 covers Central California, descending geometry
- Solid Earth Tide and ionospheric corrections are included in the files but **not applied by default**
- Reference date "moves" forward in production — must use `disp-s1-reformat` to fix the cumulative displacement

### Fixing the moving reference date
```bash
opera-utils disp-s1-reformat \
    --reference-method BORDER \
    --no-apply-solid-earth-corrections \
    --output-name opera-subset-CA-stack.nc \
    --input-files opera-subset-CA/OPERA_L3_DISP-S1*.nc
```

### Quality masking
```python
strict_mask = (ds.temporal_coherence[-1] > 0.75) | (ds.phase_similarity[-1] > 0.6)
```
Using `|` (OR) rather than `&` keeps pixels that are good in either metric, improving coverage.

### Coordinate conversion
The OPERA data is in UTM — use `pyproj.Transformer` or `opera_utils.disp.DispProductStack.lonlat_to_rowcol()` to convert lat/lon to pixel coordinates without warping the entire dataset.

---

## General Tips

### Phase sign convention
In MintPy and OPERA DISP-S1, **positive LOS displacement = motion toward the satellite** (range shortening). In ISCE2/Dolphin raw outputs, the sign may differ — always check the metadata.

### ERA5 tropospheric correction
ERA5 requires `PyAPS` (`pip install pyaps3`) and ECMWF account credentials. It is the most accurate option for non-mountainous terrain. For mountainous terrain, the phase-elevation regression (`mintpy.troposphericDelay.method = height_correlation`) can work better.

### Masking pipeline
A recommended masking order for MintPy outputs:
1. Temporal coherence > 0.7
2. Water mask (from geometry file)
3. Connected component intersection
4. Layover/shadow mask
