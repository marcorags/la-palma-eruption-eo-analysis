# Data Pipeline

This document describes the provenance, harmonization, validation, and downstream use of the Earth Observation data employed in the La Palma 2021 eruption project.

The repository documents a **downstream, analysis-ready EO workflow**. It does not reproduce every upstream acquisition and sensor-specific preprocessing operation from raw Sentinel products.

> [!IMPORTANT]
> The public repository contains neither the canonical 204-band unified datastack nor the preprocessed Sentinel-1, Sentinel-2, DEM, WorldCover, and professor-provided lava-extent inputs required to reconstruct it.
>
> The tracked repository files are therefore not sufficient to execute either notebook end-to-end. Users must provide either the canonical datastack or the complete set of required preprocessed inputs locally.

## Pipeline overview

```text
Raw and externally prepared EO inputs
Sentinel-2 · Sentinel-1 · Copernicus DEM · ESA WorldCover · Lava_Extent
                              │
                              ▼
Upstream acquisition and sensor-specific preprocessing
Copernicus Browser / Copernicus Data Space · Google Earth Engine · SNAP
NetCDF and GeoTIFF export · course workflow steps
                              │
                              ▼
Preprocessed local inputs
not distributed through GitHub
data/processed/ · data/auxiliary/
                              │
                              ▼
notebooks/00_build_unified_datastack.ipynb
Grid definition · alignment · resampling · scaling · masks · metadata
                              │
                              ▼
Validated 204-band analysis-ready datastack
GeoTIFF not distributed · metadata CSV tracked
                              │
                              ▼
notebooks/01_lapalma_analysis_final.ipynb
Optical/SAR diagnostics · RF spectral evidence · constrained timing
Land-cover, terrain, and optional infrastructure exposure assessment
```

## Reproducibility boundary

The repository documents reproducibility from the **preprocessed-input and harmonization stage**, but the required preprocessed inputs are not distributed.

The implemented harmonization and downstream-analysis logic can be inspected in the notebooks, but it can be executed only after the required data have been supplied locally.

| Workflow stage | Status in this repository |
|---|---|
| Raw Sentinel product discovery and download | Documented as upstream provenance, not reproduced |
| Google Earth Engine data preparation | Documented as upstream provenance, scripts not included |
| SNAP sensor-specific preprocessing | Documented as upstream provenance, full graph and parameters not included |
| Export of preprocessed NetCDF and GeoTIFF inputs | Required locally; files and complete export workflow not distributed |
| Multi-source harmonization to a common grid | Implemented in `00_build_unified_datastack.ipynb`; executable only with local inputs |
| Datastack construction and validation | Implemented in `00_build_unified_datastack.ipynb`; canonical GeoTIFF not distributed |
| Optical/SAR analysis, RF, timing reconstruction, and exposure assessment | Implemented in `01_lapalma_analysis_final.ipynb`; executable only with the local datastack |

The upstream stages followed the broader AI and Big Data for Earth Observation course workflow. However, complete product identifiers, the original Google Earth Engine script, the full SNAP processing graph, all operator parameters, and the complete set of intermediate artifacts are not available in this repository. The documentation therefore avoids claiming full raw-data-to-result reproducibility.

## 1. Source data and upstream preparation

### 1.1 Sentinel-2 optical data

The project uses 25 Sentinel-2 dates and six spectral bands:

```text
B2, B3, B4, B8, B11, B12
```

The local harmonization inputs consist of:

- one temporal NetCDF stack containing 23 eruption-period dates;
- one additional pre-eruption observation dated `2021-08-26`;
- one additional post-eruption observation dated `2022-01-03`.

Canonical local paths:

```text
data/processed/s2_stack/S2_LaPalma_stack_EPSG32628_10m.nc
data/processed/s2_extra/S2_20210826.nc
data/processed/s2_extra/S2_20220103.nc
```

The 23-date temporal stack is already organized on the project reference grid. The two additional observations are separately aligned during datastack construction.

Sentinel-2 acquisition and upstream preparation involved the course workflow and tools such as Google Earth Engine and Copernicus services. The exact acquisition queries, product identifiers, export settings, and complete preprocessing history are not fully reproduced in this repository.

### 1.2 Sentinel-1 SAR data

Five preprocessed Sentinel-1 acquisitions are used:

```text
2021-08-23
2021-09-28
2021-10-22
2021-12-03
2022-01-08
```

Canonical local paths:

```text
data/processed/s1/S1_20210823.nc
data/processed/s1/S1_20210928.nc
data/processed/s1/S1_20211022.nc
data/processed/s1/S1_20211203.nc
data/processed/s1/S1_20220108.nc
```

Each file contains positive linear variables named:

```text
Amplitude_VV
Amplitude_VH
```

The NetCDF files retain spatial-reference information exported from the upstream processing workflow. The harmonization notebook reads the CRS and image-to-map transform from the NetCDF metadata before alignment.

The complete Sentinel-1 SNAP processing chain is not included. In particular, the repository does not independently establish the full calibration, filtering, terrain-correction, or product-generation history of every source file.

### 1.3 Copernicus DEM and derived terrain layers

The terrain source is the **Copernicus DEM GLO-30**, also referred to as Copernicus WorldDEM-30. The product is provided by Airbus Defence and Space GmbH under the Copernicus Programme of the European Union and ESA, with attribution also involving DLR.

The original project-derived terrain file was:

```text
LaPalma_CopernicusDEM_GLO30_Topography_EPSG32628_v5.tif
```

For the repository-relative workflow it is referenced using the canonical local name:

```text
data/auxiliary/DEM.tif
```

This is not the untouched native DEM product. It was prepared before datastack construction by:

- cropping it to the La Palma study area;
- reprojecting it to `EPSG:32628`;
- deriving four raster layers:
  1. elevation;
  2. slope;
  3. aspect;
  4. hillshade.

The native distribution variant and the specific download portal cannot be established with certainty from the recovered project history. The documentation therefore does not claim whether the original file was the DGED or DTED variant.

### 1.4 ESA WorldCover

ESA WorldCover 2021 provides the categorical land-cover layer used for landscape and exposure summaries.

Canonical local path:

```text
data/auxiliary/ESA_WorldCover.tif
```

The source raster is aligned to the project grid during datastack construction using nearest-neighbour resampling to preserve categorical class codes.

### 1.5 Professor-provided lava extent

The `Lava_Extent` dataset was provided directly by the course professor as auxiliary project data.

It is stored locally inside:

```text
data/auxiliary/auxiliary_data.nc
```

The source contains the lava-extent variable and one-dimensional longitude and latitude coordinates. During harmonization, the notebook interprets finite values as the accepted final affected footprint and aligns the layer from geographic coordinates to the project grid using nearest-neighbour resampling.

The resulting `lava_reference` layer is used as the accepted final affected-domain reference. It is not presented as an independently reconstructed footprint or as the output of a physical lava-flow simulation.

### 1.6 Optional OpenStreetMap vectors

Optional building and road data are used only by the downstream infrastructure-exposure section:

```text
data/ancillary/osm/osm_buildings_lapalma_exposure_aoi.gpkg
data/ancillary/osm/osm_roads_lapalma_exposure_aoi.gpkg
```

These vectors are not included in the unified GeoTIFF. If unavailable, the main downstream workflow remains executable and produces placeholder exposure outputs.

## 2. Local preprocessed inputs

The harmonization notebook expects the following local structure:

```text
data/
├── processed/
│   ├── s2_stack/
│   ├── s2_extra/
│   ├── s1/
│   └── unified/
├── auxiliary/
├── ancillary/
│   └── osm/
└── metadata/
```

Large NetCDF, GeoTIFF, GeoPackage, and binary products are intentionally excluded from Git and are not provided through this repository.

## 3. Common reference grid

The Sentinel-2 23-date temporal stack defines the target grid for the unified datastack.

Validated target properties:

| Property | Value |
|---|---:|
| CRS | `EPSG:32628` |
| Resolution | 10 m |
| Height | 582 pixels |
| Width | 1090 pixels |
| Affine transform | `(10, 0, 211470, 0, -10, 3171670)` |
| Pixel area | 100 m² |
| Pixel area | 0.01 ha |

Before any optional candidate build, the notebook verifies the Sentinel-2 reference stack against the expected CRS, shape, transform, resolution, band inventory, and date inventory.

## 4. Datastack harmonization

The harmonization logic is implemented in:

```text
notebooks/00_build_unified_datastack.ipynb
```

The notebook is a **consolidation and alignment step**, not a complete raw-data preprocessing pipeline.

### 4.1 Sentinel-2 handling

The main Sentinel-2 temporal stack already defines the target grid and is copied without further spatial resampling.

The main stack follows a nominal reflectance scale-factor-10,000 convention. The two additional pre/post Sentinel-2 files contain unit-scale reflectance and are:

1. spatially aligned to the common grid using bilinear resampling;
2. multiplied by `10,000` to match the main-stack numerical convention.

For each date, the six reflectance bands are written in the order:

```text
B2, B3, B4, B8, B11, B12
```

A date-specific Sentinel-2 validity mask is computed as the joint finite-data support of all six bands. These masks identify finite observations; they are not guaranteed cloud-free masks.

### 4.2 Sentinel-1 handling

For each Sentinel-1 NetCDF:

1. the source CRS and transform are read from the NetCDF spatial metadata;
2. `Amplitude_VV` and `Amplitude_VH` are aligned to the common grid using bilinear resampling;
3. positive values are transformed using the project convention:

```text
10 × log10(value)
```

4. the results are stored as `VV_db` and `VH_db`;
5. a date-specific validity mask is computed from their joint finite support.

The available metadata identify the inputs as positive, amplitude-labelled linear values but do not independently verify that they are calibrated sigma-nought products. The conversion is retained because it is the validated convention used by the project, while the provenance ambiguity is stated explicitly.

### 4.3 Terrain handling

The four terrain bands are aligned to the 10 m grid using bilinear resampling:

```text
elevation
slope_degrees
aspect_degrees
hillshade
```

Because the local terrain raster has an approximately 30 m source resolution, this step is a true interpolation to the finer reference grid.

Aspect is a circular variable, so direct bilinear interpolation may be imperfect near the 0°/360° boundary. The notebook preserves the original validated project rule rather than replacing it during repository curation.

### 4.4 WorldCover handling

WorldCover is aligned using nearest-neighbour resampling. This preserves its categorical class codes and avoids creating artificial intermediate land-cover values.

### 4.5 Lava-reference handling

The professor-provided `Lava_Extent` variable is georeferenced from its longitude and latitude coordinates and aligned to `EPSG:32628` using nearest-neighbour resampling.

Two distinct layers are produced:

- `lava_reference`: binary accepted final affected footprint;
- `lava_valid_mask`: spatial coverage of the auxiliary lava source.

These layers have different meanings and are kept separate.

## 5. Generated masks

Validity and analysis-domain masks are computed inside the harmonization notebook rather than supplied as external files.

### Sensor-specific masks

- 25 Sentinel-2 validity masks;
- 5 Sentinel-1 validity masks.

### Static and domain masks

- `lava_valid_mask`;
- `landcover_valid_mask`;
- `dem_valid_mask`;
- `core_valid_mask`;
- `landcover_analysis_mask`.

`core_valid_mask` combines valid lava-source coverage, land-cover support, and terrain support.

### Pre/post comparison masks

- `S2_pre_post_valid_mask`;
- `S1_pre_post_valid_mask`;
- `S1_S2_pre_post_valid_mask`.

The optical pre/post comparison uses:

```text
2021-08-26 → 2022-01-03
```

The SAR pre/post comparison uses:

```text
2021-08-23 → 2022-01-08
```

## 6. Unified datastack construction

The canonical analysis-ready outputs are:

```text
data/processed/unified/LaPalma_unified_datastack_EPSG32628_10m.tif
data/metadata/LaPalma_unified_datastack_band_metadata.csv
data/metadata/README_datastack.md
```

The GeoTIFF contains 204 `float32` bands:

| Component | Bands |
|---|---:|
| Sentinel-2 reflectance | 150 |
| Sentinel-1 VV/VH dB layers | 10 |
| Static layers | 6 |
| Sentinel-2 validity masks | 25 |
| Sentinel-1 validity masks | 5 |
| Static and analysis-domain masks | 5 |
| Pre/post comparison masks | 3 |
| **Total** | **204** |

GeoTIFF configuration:

- `float32` data type;
- `NaN` NoData;
- DEFLATE compression;
- tiled storage;
- 256 × 256 pixel blocks;
- `BIGTIFF=IF_SAFER`.

Each raster band receives a canonical band description. A companion metadata CSV records:

- one-based band index;
- canonical band name;
- category;
- source;
- date;
- variable;
- resampling rule;
- units;
- description;
- scaling information.

Derived analytical variables are deliberately excluded from the datastack. Spectral indices, change scores, RF predictions, timing maps, and impact products are generated downstream.

## 7. Candidate-build policy

The public notebook is configured by default as:

```python
BUILD_CANDIDATE_OUTPUTS = False
```

In this mode, it validates the existing canonical products without overwriting them.

When candidate generation is explicitly enabled, outputs are written only to:

```text
data/processed/unified/LaPalma_unified_datastack_EPSG32628_10m_candidate.tif
data/metadata/LaPalma_unified_datastack_band_metadata_candidate.csv
```

Candidate files are never promoted automatically. They must pass validation before any manual replacement of the canonical artifacts.

## 8. Datastack validation

The harmonization notebook validates:

- band count;
- raster shape;
- CRS;
- pixel resolution;
- affine transform;
- metadata-row count;
- sequential band indices;
- uniqueness of band names;
- exact correspondence between GeoTIFF descriptions and metadata;
- finite binary mask encoding;
- Sentinel-1 and Sentinel-2 date inventories;
- expected spectral bands and polarizations;
- presence of the `2021-12-14` Sentinel-2 observation;
- expected category counts;
- absence of downstream analytical layers;
- final lava-reference pixel count and affected area.

The accepted final reference contains approximately:

| Quantity | Value |
|---|---:|
| Affected pixels | 124,096 |
| Pixel area | 0.01 ha |
| Reference area | 1,240.96 ha |

An optional visual quality-control mosaic compares representative Sentinel-2, Sentinel-1, terrain, WorldCover, lava-reference, and core-mask layers for orientation and obvious spatial misalignment.

Detailed band-level documentation is provided in [`data/metadata/README_datastack.md`](../data/metadata/README_datastack.md).

## 9. Downstream analysis

The validated datastack and metadata CSV are consumed by:

```text
notebooks/01_lapalma_analysis_final.ipynb
```

The notebook first repeats essential structural checks and then performs the following analytical stages.

### 9.1 Optical and SAR diagnostics

- pre/post Sentinel-2 spectral-index and SWIR diagnostics;
- Sentinel-1 VV/VH temporal summaries;
- continuous optical, SAR, and combined change-score comparison;
- conservative Sentinel-1 first-change support mapping;
- optional unsupervised change-space diagnostic.

These comparisons measure diagnostic agreement with `lava_reference`; they are not independent field validation.

### 9.2 Random Forest spectral-evidence organization

A Random Forest organizes Sentinel-2 observations into four pseudo-label classes:

```text
active_lava
dry_lava
cloud
other
```

The model uses six Sentinel-2 bands plus derived heat-index, NDVI, and SWIR-slope features. It is trained from conservative rule-based pseudo-labels and evaluated with a spatial block split.

The RF is used to organize spectral evidence. It does not define an unconstrained final lava footprint, and its validation measures pseudo-label consistency rather than field accuracy.

### 9.3 Constrained affected-surface timing reconstruction

Observed RF and heat-index timing evidence is combined with:

- the accepted final lava-reference domain;
- the Tajogaite vent location;
- slope-weighted geodesic ordering;
- local frontier propagation;
- the known eruption-end constraint.

The result is a complete daily timing proxy inside the accepted final footprint. It is not a physical lava-flow simulation and not a literal daily active-lava map.

### 9.4 Impact and exposure assessment

The final lava reference, rather than RF predictions, defines the affected footprint for:

- WorldCover land-cover summaries;
- terrain summaries;
- timing-by-land-cover summaries;
- optional OSM building and road intersections.

WorldCover and OSM results are exposure proxies, not official damage statistics.

## 10. Generated downstream outputs

The analysis notebook writes selected products under:

```text
outputs/figures/
outputs/maps/
outputs/tables/
outputs/models/
```

The notebook remains the primary executable record. Selected lightweight figures, maps, and tables may be tracked in Git, while large binary models and raster arrays are excluded.

## 11. Data lineage summary

```text
Sentinel-2 acquisition and upstream preparation
                      ┐
Sentinel-1 acquisition and SNAP-based preparation
                      │
Copernicus DEM GLO-30 → terrain derivatives
                      ├──► preprocessed local inputs
ESA WorldCover 2021  │
                      │
Professor-provided Lava_Extent
                      ┘
                              │
                              ▼
00_build_unified_datastack.ipynb
                              │
          common grid · scaling · alignment
          resampling · masks · metadata · QC
                              │
                              ▼
204-band canonical datastack
                              │
                              ▼
01_lapalma_analysis_final.ipynb
                              │
          optical/SAR diagnostics
          RF spectral evidence
          constrained timing proxy
          impact/exposure assessment
                              │
                              ▼
Selected figures · maps · tables · model products
```

