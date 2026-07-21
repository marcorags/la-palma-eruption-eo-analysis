# Methodology

This document describes the analytical methodology implemented in:

```text
notebooks/01_lapalma_analysis_final.ipynb
```

The workflow starts from the validated 204-band unified Earth Observation datastack produced or checked by:

```text
notebooks/00_build_unified_datastack.ipynb
```

Data provenance, harmonization, and reproducibility boundaries are documented separately in [`DATA_PIPELINE.md`](DATA_PIPELINE.md). Methodological and interpretative cautions are collected in [`LIMITATIONS.md`](LIMITATIONS.md).

## Methodological objective

The project is structured as an interpretable multi-stage Earth Observation workflow rather than as a single lava-classification task.

```text
Validated unified EO datastack
              │
              ▼
Optical and SAR change diagnostics
              │
              ▼
Random Forest organization of Sentinel-2 spectral evidence
              │
              ▼
Observed RF and SWIR timing anchors
              │
              ▼
Constrained affected-surface timing reconstruction
              │
              ▼
Land-cover, terrain, and infrastructure-exposure assessment
```

Three concepts remain separate throughout the analysis:

1. **Spectral evidence**: optical, SWIR, and SAR signals associated with eruption-related surface change.
2. **Assigned affected-surface timing**: observed or interpolated dates inside the accepted final footprint.
3. **Final affected area**: the professor-provided `lava_reference` domain used for impact and exposure summaries.

The Random Forest does not define the final footprint, and the timing reconstruction does not extend beyond it.

## 1. Analysis-ready inputs and spatial domains

The main analytical inputs are:

```text
data/processed/unified/LaPalma_unified_datastack_EPSG32628_10m.tif
data/metadata/LaPalma_unified_datastack_band_metadata.csv
```

Before analysis, the notebook verifies:

- raster shape: `582 × 1090`;
- CRS: `EPSG:32628`;
- resolution: 10 m;
- raster bands: 204;
- metadata rows: 204;
- Sentinel-2 dates: 25;
- Sentinel-1 dates: 5;
- presence of the expected `2021-12-14` Sentinel-2 observation.

Band selection is metadata-driven rather than based on hard-coded raster-band positions.

### 1.1 Core masks

The principal domains are:

| Domain | Role |
|---|---|
| `core_valid_mask` | Common valid support for the static and auxiliary data used in the analysis |
| `landcover_analysis_mask` | Valid domain for WorldCover-based summaries |
| `lava_reference` | Accepted final affected-domain footprint |
| `S2_pre_post_valid_mask` | Valid optical pre/post comparison support |
| `S1_pre_post_valid_mask` | Valid SAR pre/post comparison support |
| Date-specific Sentinel masks | Finite observations for each acquisition |

The final reconstruction domain is:

```text
final_lava_domain = lava_reference & core_valid_mask
```

A missing or invalid observation is never interpreted automatically as absence of affected surface change.

## 2. Sentinel-2 feature construction

For every Sentinel-2 date, the notebook reads:

```text
B2, B3, B4, B8, B11, B12
```

and derives the following features using safe division:

\[
\mathrm{NDVI} = \frac{B8-B4}{B8+B4}
\]

\[
\mathrm{NBR} = \frac{B8-B12}{B8+B12}
\]

\[
\mathrm{NDMI} = \frac{B8-B11}{B8+B11}
\]

\[
\mathrm{BSI} =
\frac{(B11+B4)-(B8+B2)}
     {(B11+B4)+(B8+B2)}
\]

\[
\mathrm{heat\_index} =
\frac{B12-B8}{B12+B8}
\]

\[
\mathrm{swir\_slope} = B12-B11
\]

`heat_index` is numerically the negative of NBR. It is treated as a SWIR/NIR contrast proxy, not as a physical surface-temperature measurement.

All ratio features are clipped to `[-1, 1]`, and pixels with unstable or invalid denominators remain `NaN`.

## 3. Exploratory optical and SAR change evidence

The first stage tests whether the eruption is visible through simple and interpretable change products.

### 3.1 Optical pre/post comparison

The optical comparison uses:

```text
Pre-eruption:  2021-08-26
Post-eruption: 2022-01-03
```

Differences are calculated as:

```text
post - pre
```

for:

```text
NDVI
NBR
NDMI
BSI
heat_index
```

Statistics are summarized separately inside and outside `lava_reference` using the relevant validity mask. The summaries include:

- pixel count;
- mean;
- median;
- 5th percentile;
- 95th percentile.

These comparisons provide diagnostic context. They do not define the final affected footprint.

### 3.2 Optical Change Vector Analysis

Optical Change Vector Analysis measures the magnitude of pre/post displacement in a non-redundant feature space:

```text
NDVI
NDMI
BSI
heat_index
```

For pixel \(x\), the score is:

\[
\mathrm{CVA}(x)=
\sqrt{
\sum_k
\left[f_k^{post}(x)-f_k^{pre}(x)\right]^2
}
\]

NBR is excluded because `heat_index = -NBR`; including both would duplicate the same B8/B12 information in the Euclidean magnitude.

### 3.3 Sentinel-1 pre/post comparison

The main SAR comparison uses:

```text
Pre-eruption:  2021-08-23
Post-eruption: 2022-01-08
```

The notebook evaluates:

```text
dVV_db
dVH_db
dVH_minus_VV_db
```

where differences are again calculated as `post - pre`.

SAR evidence is interpreted as sensitivity to surface and structural change, not as a class-specific lava signal.

### 3.4 Sentinel-1 temporal features

For every available Sentinel-1 date, the notebook derives:

```text
VV_db
VH_db
VH_minus_VV_db
RVI
```

RVI is computed after converting VV and VH from dB back to linear values:

\[
\mathrm{RVI} =
\frac{4\,VH_{lin}}
     {VV_{lin}+VH_{lin}}
\]

Adjacent acquisition intervals are compared using the VV/VH Euclidean change norm:

\[
\Delta_{\mathrm{SAR}} =
\sqrt{(\Delta VV_{db})^2+(\Delta VH_{db})^2}
\]

Median and upper-percentile changes are summarized inside and outside the accepted final reference footprint.

### 3.5 Conservative SAR first-change support

A separate Sentinel-1 support map records the first date on which a pixel exceeds a conservative change threshold relative to the first SAR acquisition.

For each later Sentinel-1 date:

1. the VV/VH Euclidean change norm is computed relative to the baseline;
2. the threshold is set to the 95th percentile of the score outside `lava_reference`;
3. pixels exceeding the threshold receive their first-change date;
4. previously assigned pixels retain the earlier date.

The threshold is not optimized against the lava mask. The map is retained as independent temporal support and is not used as the authoritative daily reconstruction.

## 4. Diagnostic change-score benchmark

Continuous optical, SAR, and combined scores are compared against `lava_reference` inside their valid analysis domains.

### 4.1 Candidate scores

The benchmark includes:

- `d_heat_index`;
- vegetation-loss score `-dNDVI`;
- optical CVA magnitude;
- absolute and signed VV/VH changes;
- change in `VH_minus_VV_db`;
- absolute and signed RVI changes;
- VV/VH SAR change norm;
- a combined optical-CVA/SAR score.

Duplicate mathematical formulations are removed. In particular, NBR decrease is omitted because it is equivalent to `d_heat_index`.

### 4.2 Combined optical/SAR score

Optical CVA and SAR change norm are independently normalized using their 2nd and 98th percentiles and clipped to `[0, 1]`.

The combined diagnostic is:

\[
S_{\mathrm{combined}} =
0.5\,S_{\mathrm{CVA,norm}}
+
0.5\,S_{\mathrm{SAR,norm}}
\]

This equal weighting is used only as a diagnostic comparison.

### 4.3 Evaluation metrics

For each score, the notebook reports:

- ROC AUC;
- best F1;
- intersection over union;
- precision;
- recall;
- specificity;
- false-positive rate;
- balanced accuracy;
- confusion counts.

The best F1 threshold is selected from up to 200 thresholds spanning the 1st–99th percentile range of the valid score.

These metrics quantify agreement with the accepted project reference. They are not independent field validation.

### 4.4 Optional unsupervised diagnostic

An optional five-cluster K-means analysis is fitted in the change-feature space:

```text
d_heat_index
-dNDVI
optical CVA
final SAR change norm
absolute RVI change
```

Features are standardized, and a maximum of 80,000 pixels is used for fitting. The cluster with the strongest overlap with `lava_reference` is reported for diagnostic comparison only.

K-means does not contribute to RF training, timing reconstruction, or impact assessment.

## 5. Random Forest spectral-evidence model

The Random Forest organizes Sentinel-2 observations into four pragmatic spectral-evidence classes:

```text
active_lava
dry_lava
cloud
other
```

These are not independently observed semantic classes.

### 5.1 RF features

The nine model features are:

```text
B2
B3
B4
B8
B11
B12
heat_index
NDVI
swir_slope
```

Sentinel-1 variables are not included as RF features because SAR changes are not sufficiently class-specific in this workflow.

### 5.2 Pseudo-label rules

Pseudo-labels are generated only from selected dates where each class is expected to be relatively clear.

| Class | Selected dates | Candidate rule |
|---|---|---|
| `active_lava` | 2021-10-10, 2021-10-15, 2021-10-30, 2021-11-14, 2021-11-29 | Inside `lava_reference`, `heat_index ≥ 0.50`, and `B12 > B8` |
| `dry_lava` | 2021-12-29, 2022-01-03 | Inside `lava_reference`, `heat_index ≥ 0.20`, and `NDVI < 0.35` |
| `cloud` | 2021-10-25, 2021-11-24, 2021-12-19, 2021-12-24 | Outside `lava_reference`, `B2 ≥ 4000`, and `heat_index ≤ 0.30` |
| `other` | 2021-08-26, 2021-09-05 | Outside `lava_reference`, `B2 ≤ 3500`, and `NDVI > 0.05` |

The visible-band thresholds refer to the scale-factor-10,000 Sentinel-2 convention used by the datastack.

These rules prioritize conservative candidate quality rather than complete spatial coverage.

### 5.3 Sampling

The notebook samples up to 5,000 pixels per class.

Samples are distributed across the selected dates using per-date quotas. If fewer high-confidence candidates are available, the class remains smaller instead of relaxing the rule.

Sampling is deterministic under:

```text
SEED = 42
```

### 5.4 Spatial validation split

To reduce local spatial leakage, pixels are grouped into `50 × 50` pixel blocks. At 10 m resolution, these correspond to approximately `500 × 500 m` blocks.

A grouped split assigns:

- 75% of the sampled data to training;
- 25% to testing.

Blocks, rather than individual pixels, are separated between the two sets.

### 5.5 Model configuration

The Random Forest uses:

```text
n_estimators = 300
min_samples_leaf = 5
class_weight = balanced
random_state = 42
```

Validation outputs include:

- classification report;
- normalized confusion matrix;
- feature importance;
- train/test block summary.

The validation measures the model's ability to reproduce held-out pseudo-label structure. It does not measure independent field accuracy.

### 5.6 All-date classification

The trained RF is applied to all 25 Sentinel-2 dates.

For every valid pixel and date, the same nine features are generated and classified. The resulting temporal stack preserves `cloud` as an explicit class rather than silently treating cloud-obscured pixels as unaffected.

The all-date classifications are used as spectral-evidence layers and for timing-anchor extraction. They are not permitted to expand the final affected footprint beyond `lava_reference`.

## 6. Constrained affected-surface timing reconstruction

The reconstruction converts sparse Sentinel-2 observations into a complete daily timing proxy inside the accepted final footprint.

### 6.1 Temporal domain

The reconstruction period is:

```text
Start: 2021-09-19
End:   2021-12-13
```

No timing is assigned outside this interval.

### 6.2 Observed timing anchors

Two sources of eruption-window timing evidence are extracted independently:

1. the first RF `active_lava` prediction;
2. the first direct heat-index detection satisfying:

```text
heat_index ≥ 0.50
B12 > B8
```

Both are restricted to:

```text
final_lava_domain = lava_reference & core_valid_mask
```

When both sources detect a pixel, the earliest date is retained.

Post-eruption RF `dry_lava` predictions are stored only as final-state support. They are never interpreted as post-eruption arrival dates.

### 6.3 Vent seed and geodesic ordering

The Tajogaite vent coordinate is:

```text
longitude: -17.866111111111
latitude:   28.612777777778
```

It is transformed to `EPSG:32628` and snapped to the nearest pixel inside the final lava domain when necessary.

A slope-weighted traversal cost is defined as:

\[
C(x)=1+\frac{\mathrm{slope}(x)}{90}
\]

Minimum accumulated cost from the snapped vent is calculated inside the final domain using `MCP_Geometric` with eight-neighbour connectivity.

The resulting geodesic distance provides a topological ordering, not a physical travel-time estimate.

Compatibility between this ordering and observed timing anchors is assessed using Pearson and Spearman correlations and binned timing summaries.

### 6.4 Target cumulative affected-area curve

For every day, the notebook counts all observed timing anchors assigned up to that day.

A monotonic target cumulative pixel count is constructed from:

- one seed pixel on day 0;
- each observed cumulative increase;
- the complete final reference-domain pixel count on the eruption-end date.

Linear interpolation between these anchors produces a daily target count. The sequence is forced to be non-decreasing and never smaller than the observed cumulative evidence.

### 6.5 Frontier-based timing assignment

For each reconstruction day:

1. all directly observed pixels due by that date are assigned first;
2. the remaining number of pixels required by the cumulative target is calculated;
3. candidate pixels are selected from the current eight-neighbour frontier inside the final domain;
4. if the frontier is empty, all remaining final-domain pixels become candidates;
5. candidates are ranked by:

\[
\mathrm{score} =
d_{\mathrm{geo}}
+
20\,d_{\mathrm{future}}
-
50\,I_{\mathrm{postdry}}
\]

where:

- \(d_{\mathrm{geo}}\) is slope-weighted geodesic cost;
- \(d_{\mathrm{future}}\) penalizes assignment before a later direct observation;
- \(I_{\mathrm{postdry}}\) indicates post-eruption dry-lava support.

Lower scores are assigned first.

All remaining pixels are assigned by the final eruption-end constraint. Direct observations are then re-applied so that inferred timing does not overwrite an earlier observed date.

The coefficients `20` and `50` are project-specific heuristic priorities, not calibrated physical parameters.

### 6.6 Reconstruction provenance map

Each reconstructed pixel receives a source code:

| Code | Meaning |
|---:|---|
| 0 | Outside final lava domain |
| 1 | Direct RF or heat-index timing evidence |
| 2 | Post-eruption dry-lava support used during interpolation |
| 3 | Topology/frontier-filled timing |

This map separates direct evidence from constrained inference.

### 6.7 Daily cumulative reconstruction

For each day \(t\), the cumulative affected-surface mask is:

\[
A_t(x)=
I\left[
0 \leq \mathrm{arrival\_day}(x)\leq t
\right]
\]

restricted to the final lava domain.

The result is a complete daily cumulative timing proxy. It is not a physical lava-flow simulation and not a literal active-lava map.

## 7. Land-cover and terrain assessment

Impact area is defined from the accepted final reference rather than RF predictions:

```text
affected = lava_reference & landcover_analysis_mask
```

### 7.1 WorldCover exposure

For each ESA WorldCover class, the notebook calculates:

- affected pixels;
- affected area in hectares;
- fraction of the total affected area;
- fraction of the available class area affected.

The `Built up` class is interpreted as raster land-cover exposure, not as a count of buildings.

### 7.2 Terrain summaries

Elevation, slope, aspect, and hillshade are summarized separately for affected and non-affected pixels using:

- pixel count;
- mean;
- median;
- 5th percentile;
- 95th percentile.

Additional affected-area summaries are generated for:

**Elevation bands**

```text
0–250 m
250–500 m
500–750 m
750–1000 m
1000–1500 m
>1500 m
```

**Slope classes**

```text
0–5°
5–15°
15–30°
30–45°
>45°
```

These summaries describe terrain context and do not establish causal controls on lava propagation.

### 7.3 Timing by land-cover class

The assigned timing surface is intersected with each affected WorldCover class.

For every class, the notebook reports:

- affected pixels and hectares;
- median assigned timing day;
- 5th and 95th percentile timing day;
- median assigned calendar date.

These values inherit the assumptions of the constrained timing reconstruction.

## 8. Optional OpenStreetMap infrastructure exposure

When local OSM building and road files are available, the accepted affected mask is converted to a dissolved polygon in the project CRS.

### 8.1 Building exposure

Building polygons are:

1. reprojected to the project CRS;
2. cleaned for valid polygon geometry;
3. selected when they intersect the affected polygon.

The notebook reports:

- number of intersecting OSM buildings;
- total footprint area of those buildings;
- building-footprint area intersected by the affected polygon.

### 8.2 Road exposure

Road lines are intersected with the affected polygon.

The notebook reports:

- number of intersecting road segments;
- total intersected road length;
- intersected road length by OSM `highway` class.

If the OSM files or vector libraries are unavailable, placeholder tables are generated and the main notebook continues.

These are exposure indicators, not official damage statistics.

## 9. Reproducibility and output policy

All stochastic procedures use:

```text
SEED = 42
```

The notebook writes outputs under:

```text
outputs/figures/
outputs/maps/
outputs/tables/
outputs/models/
```

The main exported products include:

- change-diagnostic maps and metric tables;
- RF pseudo-label samples, validation metrics, model, and all-date classifications;
- observed timing evidence;
- geodesic diagnostics;
- daily cumulative affected-surface reconstruction;
- reconstruction-source map;
- land-cover and terrain summaries;
- optional OSM exposure summaries.

Selected lightweight outputs may be tracked in Git. Large raster arrays and model binaries remain excluded.

## 10. Interpretation hierarchy

The workflow should be interpreted in the following order:

1. **Optical and SAR diagnostics** identify and compare eruption-related signals.
2. **RF classes** organize date-wise Sentinel-2 spectral evidence.
3. **Observed RF and heat-index detections** provide sparse timing anchors.
4. **Geodesic/frontier constraints** fill timing gaps inside the accepted final footprint.
5. **The accepted final footprint** defines impact and exposure area.
6. **WorldCover and OSM overlays** describe exposure, not verified damage.

This hierarchy prevents diagnostic scores, RF predictions, interpolated timing, and final affected-area estimates from being treated as interchangeable products.
