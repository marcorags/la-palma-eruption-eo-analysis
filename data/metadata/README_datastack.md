# La Palma Unified Earth Observation Datastack

This document describes the canonical analysis-ready datastack used by the La Palma 2021 eruption analysis workflow.

The datastack consolidates harmonized Sentinel-2, Sentinel-1, terrain, land-cover, lava-reference, validity-mask, and analysis-domain layers into one multi-band GeoTIFF on a common spatial grid.

It is generated or validated by:

```text
notebooks/00_build_unified_datastack.ipynb
```

and used as the primary input by:

```text
notebooks/01_lapalma_analysis_final.ipynb
```

> [!NOTE]
> The canonical 204-band GeoTIFF is not included in this repository. The paths documented below refer to the expected local data layout. Running the downstream notebook requires users to provide the datastack locally.

## Canonical files

The canonical datastack consists of:

```text
data/processed/unified/LaPalma_unified_datastack_EPSG32628_10m.tif
data/metadata/LaPalma_unified_datastack_band_metadata.csv
data/metadata/README_datastack.md
```

Only the metadata CSV and this documentation file are tracked in the repository. The canonical GeoTIFF remains local and is excluded from Git because of its size.

## Raster properties

| Property | Value |
|---|---:|
| Format | GeoTIFF |
| Number of bands | 204 |
| Raster height | 582 pixels |
| Raster width | 1090 pixels |
| CRS | EPSG:32628 |
| Spatial resolution | 10 m |
| Data type | `float32` |
| NoData value | `NaN` |
| Compression | DEFLATE |
| Internal tiling | 256 × 256 pixels |
| Pixel area | 100 m² |
| Pixel area in hectares | 0.01 ha |

The expected affine transform is:

```text
Affine(10.0, 0.0, 211470.0,
       0.0, -10.0, 3171670.0)
```

All bands share the same shape, CRS, transform, resolution, and spatial extent.

## Datastack composition

The 204 bands are organized as follows:

| Band range | Category | Count |
|---|---|---:|
| 1–150 | Sentinel-2 reflectance | 150 |
| 151–160 | Sentinel-1 VV/VH dB layers | 10 |
| 161–166 | Static layers | 6 |
| 167–191 | Sentinel-2 validity masks | 25 |
| 192–196 | Sentinel-1 validity masks | 5 |
| 197–201 | Static and analysis-domain masks | 5 |
| 202–204 | Pre/post comparison masks | 3 |
|  | **Total** | **204** |

The GeoTIFF band descriptions match the `band_name` values in the metadata CSV exactly.

## Sentinel-2 reflectance bands

The datastack contains six Sentinel-2 bands for each of 25 acquisition dates:

```text
B2, B3, B4, B8, B11, B12
```

Band names follow this pattern:

```text
S2_YYYYMMDD_BAND
```

For example:

```text
S2_20210826_B2
S2_20210826_B3
S2_20210826_B4
S2_20210826_B8
S2_20210826_B11
S2_20210826_B12
```

The Sentinel-2 dates are:

```text
2021-08-26
2021-09-05
2021-09-10
2021-09-20
2021-09-25
2021-09-30
2021-10-05
2021-10-10
2021-10-15
2021-10-20
2021-10-25
2021-10-30
2021-11-04
2021-11-09
2021-11-14
2021-11-19
2021-11-24
2021-11-29
2021-12-04
2021-12-09
2021-12-14
2021-12-19
2021-12-24
2021-12-29
2022-01-03
```

The main temporal stack follows the project convention of reflectance values stored at a nominal scale factor of 10,000. Values are not assumed to be strictly limited to the interval 0–10,000 because of the source products and upstream preprocessing convention.

The additional pre-eruption and post-eruption observations, dated `2021-08-26` and `2022-01-03`, originate as unit-scale reflectance and are multiplied by 10,000 during harmonization to match the main stack convention.

## Sentinel-1 layers

The datastack contains VV and VH layers for five Sentinel-1 acquisitions.

Band names follow these patterns:

```text
S1_YYYYMMDD_VV_db
S1_YYYYMMDD_VH_db
```

The Sentinel-1 dates are:

```text
2021-08-23
2021-09-28
2021-10-22
2021-12-03
2022-01-08
```

The source variables are named:

```text
Amplitude_VV
Amplitude_VH
```

Positive linear input values are transformed using the project convention:

```text
10 × log10(value)
```

The resulting variables are stored as `VV_db` and `VH_db`.

The available source metadata identify the original inputs as positive, amplitude-labelled linear values, but do not independently establish that they are calibrated sigma-nought products. The dB transformation is retained as the validated project convention, with this provenance ambiguity made explicit.

## Static layers

The six static layers are:

| Band | Source or meaning | Units or encoding |
|---|---|---|
| `land_cover` | ESA WorldCover 2021 | Categorical class code |
| `lava_reference` | Accepted final affected-domain footprint | Binary 0/1 |
| `elevation` | DEM band 1 | metres |
| `slope_degrees` | DEM band 2 | degrees |
| `aspect_degrees` | DEM band 3 | degrees |
| `hillshade` | DEM band 4 | source-derived hillshade value |

ESA WorldCover is aligned using nearest-neighbour resampling.

The terrain layers are aligned using bilinear resampling. The source DEM has a 30 m spatial resolution and is resampled to the 10 m target grid. Because aspect is circular, bilinear interpolation may be imperfect near the 0°/360° boundary; this limitation is retained from the validated project workflow.

## Lava-reference layer

`lava_reference` represents the accepted final affected-domain footprint used by the downstream analysis.

It is not a generic validity mask and should not be interpreted as an independently validated physical lava-flow simulation.

The canonical layer contains:

| Property | Value |
|---|---:|
| Affected pixels | 124,096 |
| Pixel area | 0.01 ha |
| Total reference area | 1,240.96 ha |

The layer is used to constrain the final affected-surface timing reconstruction and to define the reference domain for diagnostic comparisons.

## Sentinel-2 validity masks

Each Sentinel-2 date has one binary validity mask named:

```text
valid_S2_YYYYMMDD
```

A Sentinel-2 pixel is valid when all six reflectance bands are finite:

```text
finite(B2, B3, B4, B8, B11, B12)
```

These masks do not remove all clouds or cloud-like observations. Cloud and active-surface ambiguity is handled explicitly in the downstream spectral-evidence workflow.

## Sentinel-1 validity masks

Each Sentinel-1 date has one binary validity mask named:

```text
valid_S1_YYYYMMDD
```

A Sentinel-1 pixel is valid when both transformed polarization layers are finite:

```text
finite(VV_db, VH_db)
```

## Static and analysis-domain masks

The datastack includes five static or analysis-domain masks:

| Mask | Meaning |
|---|---|
| `lava_valid_mask` | Valid spatial coverage of the auxiliary lava-reference source |
| `landcover_valid_mask` | Pixels with finite, positive WorldCover class codes |
| `dem_valid_mask` | Pixels with finite elevation, slope, aspect, and hillshade |
| `core_valid_mask` | Intersection of valid lava-source coverage, land cover, and terrain support |
| `landcover_analysis_mask` | Valid WorldCover domain used for exposure summaries |

`lava_valid_mask` records where the auxiliary source is spatially defined. It must not be confused with `lava_reference`, which represents the accepted affected footprint.

## Pre/post comparison masks

Three masks define consistent comparison domains:

| Mask | Definition |
|---|---|
| `S2_pre_post_valid_mask` | Valid Sentinel-2 pre/post observations intersected with the core valid domain |
| `S1_pre_post_valid_mask` | Valid Sentinel-1 pre/post observations intersected with the core valid domain |
| `S1_S2_pre_post_valid_mask` | Intersection of the Sentinel-1 and Sentinel-2 pre/post masks |

The optical comparison uses:

```text
2021-08-26 → 2022-01-03
```

The SAR comparison uses:

```text
2021-08-23 → 2022-01-08
```

## Metadata table

The companion metadata file is:

```text
data/metadata/LaPalma_unified_datastack_band_metadata.csv
```

It contains one row per GeoTIFF band and the following columns:

| Column | Description |
|---|---|
| `band_index` | One-based GeoTIFF band index |
| `band_name` | Unique canonical band name |
| `category` | Datastack component category |
| `source` | Source filename or `computed` |
| `date` | Acquisition date, when applicable |
| `variable` | Original or derived variable name |
| `resampling` | Alignment or resampling method |
| `units` | Units or measurement convention |
| `description` | Additional semantic information |
| `original_scale` | Original numerical scale, when documented |
| `applied_scale_factor` | Scale factor applied during harmonization |
| `output_scale` | Numerical convention of the harmonized output |

The expected metadata categories are:

```text
sentinel2_reflectance
sentinel1_backscatter_db
static
sentinel2_validity_mask
sentinel1_validity_mask
static_domain_mask
pre_post_mask
```

The `band_index` column must be sequential from 1 to 204, and all `band_name` values must be unique.

## Layers deliberately excluded

The unified datastack contains source-aligned variables and masks, not downstream analytical products.

The following are deliberately excluded:

- NDVI, NBR, NDMI, and BSI;
- temporal index differences;
- heat index and SWIR-slope variables;
- SAR ratios and log-ratios;
- optical, SAR, and fused change scores;
- thresholded change maps;
- Random Forest predictions;
- classified temporal stacks;
- arrival-day and daily reconstruction maps;
- land-cover or infrastructure impact maps;
- latitude and longitude raster bands;
- temporal descriptor bands.

These products are computed by the downstream notebook rather than stored in the canonical datastack.

## Validation requirements

The canonical datastack is considered valid when:

- it contains exactly 204 bands;
- its shape is exactly 582 × 1090 pixels;
- its CRS is EPSG:32628;
- its spatial resolution is 10 m;
- the metadata table contains exactly 204 rows;
- `band_index` runs sequentially from 1 to 204;
- metadata band names are unique;
- GeoTIFF band descriptions match metadata band names exactly;
- all mask bands contain only finite binary values;
- all expected Sentinel-2 and Sentinel-1 dates are present;
- all six Sentinel-2 bands exist for every Sentinel-2 date;
- VV and VH exist for every Sentinel-1 date;
- the `2021-12-14` Sentinel-2 observation is present;
- the canonical lava-reference footprint contains approximately 124,096 pixels;
- derived analytical layers are absent;
- category counts match the expected 204-band composition.

The validation suite is implemented in:

```text
notebooks/00_build_unified_datastack.ipynb
```

## Candidate builds and canonical promotion

The public harmonization notebook does not overwrite the canonical datastack by default:

```python
BUILD_CANDIDATE_OUTPUTS = False
```

When candidate generation is explicitly enabled, outputs are written to:

```text
data/processed/unified/LaPalma_unified_datastack_EPSG32628_10m_candidate.tif
data/metadata/LaPalma_unified_datastack_band_metadata_candidate.csv
```

Candidate products are not promoted automatically. They must pass the validation suite before any manual replacement of the canonical files.

## Intended use

This datastack is the analysis-ready foundation for:

- optical and SAR change diagnostics;
- Random Forest organization of spectral evidence into pseudo-label classes;
- constrained affected-surface timing reconstruction;
- land-cover and terrain characterization;
- optional OpenStreetMap infrastructure-exposure assessment.

It should not be interpreted as a collection of raw satellite products or as a complete reproduction of the upstream Sentinel acquisition and SNAP preprocessing workflow.
