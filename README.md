# Monterey_Canyon_MBES — DPDK027 Lower Monterey backscatter normalization

Project workspace for sector/angle backscatter normalization of the **DPDK027**
*David Packard* multibeam survey of the lower Monterey Canyon (2025–2026).

The generic, reusable processing code that this project used to carry has been
lifted into the [`mbes-tools`](https://github.com/Jeff-Beeson/mbes-tools)
library (`mbes_tools.backscatter`). This repository now holds only
**project-specific** material: configuration, geometry grids, output tables, and
the survey data itself.

## Layout

```
.env                     # DATA_ROOT -> the DPDK027 kmall directory
pyproject.toml           # depends on mbes-tools
src/monterey_canyon_mbes # (stub) project package for any project-only helpers
DPDK027/                 # 164 GB survey data + output tables/grids (gitignored)
  DPDK027_LowerMonterey/
    *.kmall *.mb261 ...                       # raw soundings
    geom_100m_bathy.grd / _sd.grd / _num.grd  # MB-System bathy + roughness grids
    geom_100m_mask_*.{tif,asc}                # geometry keep/reject masks
    SectorAngleIntensity_Table_*.csv          # generated correction tables
archive/                 # provenance only, gitignored (see below)
```

`archive/` keeps the original working scripts (`original_scripts/`, the
ancestors of the lifted `mbes_tools.backscatter` code), the old vendored
`Python_KMALL/` viewer + kmall library, and the original
`DavidPackard_Backscatter_Normalization.ipynb` notebook. Nothing there is used
by the live pipeline; it is retained for history.

## Setup

```bash
conda activate monterey-canyon-mbes
pip install -e ~/code/mbes-tools        # provides mbes_tools + the mbes-bs-* CLIs
pip install -e .                        # this project
cp .env.example .env                    # then set DATA_ROOT
```

## Pipeline (mbes-tools console scripts)

The processing tools are the `mbes-bs-*` console scripts from `mbes-tools`.
`DATA_ROOT` in `.env` supplies the default kmall input directory.

1. **Generate the sector/angle correction table.** The combined 2025/2026 table
   used the tier-1/tier-2 QC encoded in its filename
   (`..._tier1_tier2_slope8_sd20_pingqc_...`): geometry mask + sampled
   slope/bathy-SD grids + ping QC.

   ```bash
   D=DPDK027/DPDK027_LowerMonterey
   mbes-bs-table \
       --geometry-mask-grid   $D/geom_100m_mask_slope10_sd30.asc \
       --geometry-mask-crs    EPSG:32610 \
       --slope-value-grid     $D/geom_100m_slope_deg.asc   --slope-max-deg 8 \
       --bathy-sd-value-grid  $D/geom_100m_bathy_sd_m.asc  --bathy-sd-max-m 20 \
       --min-ping-valid-fraction 0.45 \
       --max-port-starboard-diff-db 6 \
       -o $D/SectorAngleIntensity_Table_tier1_tier2_slope8_sd20_pingqc_2025_2026_combined.csv
   ```

   Geometry/slope/roughness grids and masks are built from the MB-System bathy
   grids by `archive/original_scripts/make_geometry_grids_and_masks.sh`
   (gdaldem slope/roughness + gdal_calc masks). ASCII (`.asc`) versions are made
   with `gdal_translate -of AAIGrid mask.tif mask.asc`.

2. **Balance sectors (Lambertian) and export the patch CSV.**

   ```bash
   mbes-bs-gui $D/SectorAngleIntensity_Table_tier1_tier2_slope8_sd20_pingqc_2025_2026_combined.csv
   # -> "Auto Lambert Normalize" per mode, then "Export KMALL Patch Correction CSV"
   #    (schema: depthMode,rxFanIndex,txSectorNumb,correction_dB)
   ```

3. **Apply corrections to the KMALL seabed-image samples.**

   ```bash
   mbes-bs-apply \
       --input        $D \
       --corrections  $D/fine_tuned_corrections_V2.csv \
       --output-dir   $D/BSCORR \
       --qa-csv       $D/kmall_bs_correction_QA.csv
   ```

   This patches only `SIsample_desidB` inside #MRZ datagrams, preserving
   datagram length, and writes `*_BSCORR.kmall` plus a QA CSV.

## Notes

- The survey data (`DPDK027/`) and `archive/` are gitignored; only the small
  config/metadata files are tracked.
- KMALL manual depth modes (>=100) are normalized by subtracting 100; the
  calibration/patch numbering adds +1 to both depthMode and sector.
