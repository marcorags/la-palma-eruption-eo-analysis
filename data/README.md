# Data

This directory documents the local data layout required by the notebooks in this repository.

> [!IMPORTANT]
> This repository does not include the canonical 204-band unified datastack or the preprocessed Sentinel-1, Sentinel-2, DEM, WorldCover, and professor-provided lava-extent inputs required to reconstruct it.
>
> Consequently, the notebooks cannot be executed end-to-end using the tracked repository files alone. Users must provide the required data locally following the directory structure documented below.

Large Earth Observation rasters, NetCDF files, GeoPackages, and generated binary products are not distributed through GitHub. They must be obtained, reconstructed, or placed locally using the paths and filenames described below.

Subject to the availability of the required local data, the repository documents two possible execution levels:

1. Running the downstream analysis after providing the canonical unified datastack locally.
2. Rebuilding a candidate unified datastack after providing all required preprocessed Earth Observation and auxiliary inputs locally.

## Data availability

The repository does not fully reproduce the upstream data acquisition and sensor-specific preprocessing stages.

Sentinel-1 and Sentinel-2 acquisition, Copernicus Browser and Google Earth Engine processing, SNAP preprocessing, NetCDF export, and other course-workflow steps were completed outside this repository.

Execution of the documented workflow can therefore start only after the user has locally provided either:

- the validated unified datastack and its metadata table; or
- the already-preprocessed Earth Observation and auxiliary inputs required to rebuild the datastack.

All large data files are excluded from Git through `.gitignore`.

The paths described in this document therefore refer to the expected local filesystem layout, not to files available in the public GitHub repository.

## Local directory structure

The expected local structure is:

```text
data/
├── README.md
│
├── metadata/
│   ├── LaPalma_unified_datastack_band_metadata.csv
│   └── README_datastack.md
│
├── processed/
│   ├── s2_stack/
│   │   └── S2_LaPalma_stack_EPSG32628_10m.nc
│   │
│   ├── s2_extra/
│   │   ├── S2_20210826.nc
│   │   └── S2_20220103.nc
│   │
│   ├── s1/
│   │   ├── S1_20210823.nc
│   │   ├── S1_20210928.nc
│   │   ├── S1_20211022.nc
│   │   ├── S1_20211203.nc
│   │   └── S1_20220108.nc
│   │
│   └── unified/
│       └── LaPalma_unified_datastack_EPSG32628_10m.tif
│
├── auxiliary/
│   ├── DEM.tif
│   ├── ESA_WorldCover.tif
│   └── auxiliary_data.nc
│
└── ancillary/
    └── osm/
        ├── osm_buildings_lapalma_exposure_aoi.gpkg
        └── osm_roads_lapalma_exposure_aoi.gpkg
```

Only this README, the datastack metadata CSV, and selected metadata documentation are intended to be tracked by Git.

## Running the downstream analysis

The main downstream notebook is:

```text
notebooks/01_lapalma_analysis_final.ipynb
```

Its two required runtime inputs are:

```text
data/processed/unified/LaPalma_unified_datastack_EPSG32628_10m.tif
data/metadata/LaPalma_unified_datastack_band_metadata.csv
```
The GeoTIFF is not included in the repository. The downstream notebook cannot run until this file has been supplied locally at the expected path.

The companion file:

```text
data/metadata/README_datastack.md
```

documents the datastack but is not required for the numerical calculations.

The optional OSM infrastructure-exposure section uses:

```text
data/ancillary/osm/osm_buildings_lapalma_exposure_aoi.gpkg
data/ancillary/osm/osm_roads_lapalma_exposure_aoi.gpkg
```

If these files are absent, the notebook remains executable and produces placeholder infrastructure-exposure outputs.

## Rebuilding the unified datastack

The harmonization notebook is:

```text
notebooks/00_build_unified_datastack.ipynb
```

A complete rebuild requires the following preprocessed inputs.

### Sentinel-2 temporal stack

```text
data/processed/s2_stack/S2_LaPalma_stack_EPSG32628_10m.nc
```

This file contains 23 eruption-period Sentinel-2 dates and six reflectance bands:

```text
B2, B3, B4, B8, B11, B12
```

### Additional Sentinel-2 dates

```text
data/processed/s2_extra/S2_20210826.nc
data/processed/s2_extra/S2_20220103.nc
```

These provide the additional pre-eruption and post-eruption observations.

### Sentinel-1 acquisitions

```text
data/processed/s1/S1_20210823.nc
data/processed/s1/S1_20210928.nc
data/processed/s1/S1_20211022.nc
data/processed/s1/S1_20211203.nc
data/processed/s1/S1_20220108.nc
```

Each file contains the VV and VH input variables used by the harmonization workflow.

### Auxiliary layers

```text
data/auxiliary/DEM.tif
data/auxiliary/ESA_WorldCover.tif
data/auxiliary/auxiliary_data.nc
```

These provide:

- elevation, slope, aspect, and hillshade;
- ESA WorldCover land-cover classes;
- the accepted final lava-reference footprint.

By default, the harmonization notebook validates the existing canonical datastack without overwriting it:

```python
BUILD_CANDIDATE_OUTPUTS = False
```

When candidate generation is explicitly enabled, rebuilt products are written using `_candidate` filenames. Promotion of a candidate file to the canonical datastack must be performed manually after validation.

## Canonical unified datastack

The validated unified datastack has the following properties:

| Property | Value |
|---|---:|
| Raster bands | 204 |
| Height | 582 pixels |
| Width | 1090 pixels |
| CRS | EPSG:32628 |
| Spatial resolution | 10 m |
| Metadata rows | 204 |
| Sentinel-2 dates | 25 |
| Sentinel-1 dates | 5 |

Its band composition is:

| Component | Bands |
|---|---:|
| Sentinel-2 reflectance | 150 |
| Sentinel-1 VV/VH backscatter | 10 |
| Static layers | 6 |
| Sentinel-2 validity masks | 25 |
| Sentinel-1 validity masks | 5 |
| Static and analysis-domain masks | 5 |
| Pre/post comparison masks | 3 |
| **Total** | **204** |

Derived analytical products such as spectral indices, change maps, Random Forest predictions, arrival-day maps, and exposure summaries are not stored as bands in the unified datastack. They are generated downstream by the analysis notebook.

## Data provenance and usage

The data retain the licensing and attribution requirements of their original providers, including Copernicus Sentinel data, ESA WorldCover, elevation products, and OpenStreetMap contributors.

This repository distributes project code, documentation, metadata, and selected analytical outputs. It does not redistribute the complete source imagery or the full analysis-ready raster dataset.
