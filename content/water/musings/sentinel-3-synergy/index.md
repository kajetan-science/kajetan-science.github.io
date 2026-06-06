---
title: Sentinel-3 Satellite SYNERGY
date: 2026-05-21T00:00:00.000Z
summary: >-
  Two complementary instruments fly together on Sentinel-3 to enable retrievals
  neither could do alone. How OLCI and SLSTR work, what the SYNERGY product
  family delivers, how the operational ground segment co-registers them, and
  what is needed to reproduce the whole thing on HEALPix.
tags: []
jupyter: volcan
---


The Sentinel-3 mission carries two optical instruments side by side --- **OLCI** and **SLSTR**. Each has its own product chain. The interesting one is the **SYNERGY (SY)** chain that produces user-level products from the *combined* observations of both instruments. SYNERGY is the only Sentinel-3 product family that exists because of multi-instrument fusion.

This musing covers, in order: what the two instruments measure, what the SYNERGY product family delivers, why the operational ground segment needs a sub-pixel co-registration step, how that step works, and what the team's [`legacy-converters`](https://github.com/EOPF-DGGS/legacy-converters) library would need to gain to deliver SYNERGY-equivalent products on HEALPix.

An acronym reference table lives at the bottom.

------------------------------------------------------------------------

## 1. The product family at a glance

The Copernicus ground segment turns OLCI and SLSTR Level-1B into five user-facing products plus one internal collocation product.

| ID | Resolution | Bands / channels | Purpose | Audience |
|---------|--------|--------------------------|-----------------------|-------|
| **SY_1_MISR** | (internal) | Correspondence grids only, no data | Geometric seed for L2 processors | Internal |
| **SY_2_SYN** | 300 m | 16 OLCI + 5 SLSTR nadir + 5 SLSTR oblique = **26** | Surface directional reflectance + aerosol over land | Users |
| **SY_2_VGP** | 1 km | 4 synthesised VGT bands (B0, B2, B3, MIR) | TOA reflectance --- SPOT-VEGETATION continuity | Users |
| **SY_2_VG1** | 1 km | Same 4 bands | Daily maximum-NDVI composite | Users |
| **SY_2_V10** | 1 km | Same 4 bands | 10-day maximum-NDVI composite | Users |
| **SY_2_AOD** | 4.5 km | 6 channels (1 OLCI + 5 SLSTR) | Global aerosol optical depth | Users |

### Why two instruments are needed

Each L2 product needs something neither instrument can supply alone:

-   **SY_2_SYN** needs OLCI's narrow VIS/NIR bands (spectral fineness) **and** SLSTR's SWIR (water-vapour and cirrus correction) **and** SLSTR's two views (atmospheric / surface separation).
-   **VGP / VG1 / V10** need OLCI for B0/B2/B3 (synthesised from narrow VIS/NIR bands) **and** SLSTR's S5 SWIR for the MIR band.
-   **SY_2_AOD** needs OLCI's narrow 440 nm band **and** SLSTR's five solar bands **and** the dual-view geometry.

SYNERGY is not "two instruments stuck together". It is the *only* way to compute these products from Sentinel-3.

------------------------------------------------------------------------

## 2. The two instruments

### 2.1 OLCI --- a hyperspectral colour imager

-   **Spectral coverage:** 400--1020 nm (visible to near-infrared), **21 narrow bands** (Oa01--Oa21), ~5--40 nm bandwidth each.
-   **Spatial resolution:** ~300 m (Full Resolution `OL_1_EFR`) or ~1 200 m (Reduced Resolution `OL_1_ERR`).
-   **Built as five push-broom camera modules** side by side. Each camera has its own CCD array with the spectral dimension on one axis and the across-track spatial dimension on the other. Each across-track element of the CCD is a **detector** --- one physical pixel of the chip looking through the camera optics at a specific off-nadir angle. The along-track sampling is provided by orbital motion at 44 ms per scan line. The exact detector count per camera should be looked up in the OLCI L1 Product Format Specification; the unified product has **4 865 columns** across-track after the 5 cameras are stitched.
-   **Tilted ~12° off-nadir** along-track to avoid sun glint over ocean.
-   **L1B output stays in sensor geometry** with per-pixel `(lon, lat)` tags. No map projection is ever applied.

### 2.2 SLSTR --- a dual-view radiometer with thermal

-   **Spectral coverage:** 555 nm to 12 µm, **11 channels** spanning VIS, NIR, SWIR and thermal infrared.
-   **Two views per scan:** nadir + an oblique view tilted ~55° forward along-track.
-   **Two output grids:** 500 m for solar/NIR/SWIR bands (S1--S6), 1 km for thermal and fire channels (S7--S9, F1, F2).
-   **Stripe A vs Stripe B:** SWIR has two physically parallel detector "stripes" with their own optical chain. Stripe A is primary, B is redundant. Both are carried in L1B; the SY processor picks A when both are valid.
-   **Self-calibration** capability inherited from (A)ATSR: the same ground patch is observed at two viewing angles ~55° apart, with different atmospheric path lengths. Subtracting the two gives an atmospheric correction with minimal external input --- a famously accurate technique for sea-surface temperature.
-   **L1B output (`SL_1_RBT`) also stays in sensor geometry** with per-pixel `(lon, lat)`.

### 2.3 OLCI vs a phone camera, in one paragraph

A phone camera uses a Bayer-pattern colour-filter array with three broad, overlapping filters centred at ~470 / 540 / 610 nm and an NIR-cut filter to block infrared. OLCI replaces those three crayons with twenty-one highlighters: 21 narrow bands, mostly non-overlapping, deliberately positioned over physical features --- the chlorophyll absorption peak at 670 nm, the vegetation red edge between 705 and 760 nm, the oxygen-A absorption line at 762 nm, water-vapour absorption bands near 900 and 940 nm. A phone smushes those features into the "red" channel and loses them; OLCI samples each one with a dedicated band. Same kind of optical imaging, very different spectroscopy.

### 2.4 Where each band sits in the EM spectrum

    wavelength →   400nm    700nm  1000nm  1500nm   2500nm    5000nm    10 µm    13 µm
                    │        │       │       │        │         │         │        │
    ────────────────┴────────┴───────┴───────┴────────┴─────────┴─────────┴────────┴──
    region:       ◄─VIS─►◄──NIR──►◄─────SWIR──────►◄─MWIR/MIR─►   ◄──TIR/LWIR──►
                                                                      (thermal emission)

    OLCI:    ╞══════════════════════╡
             (21 narrow bands, 400–1020 nm, sun-reflected only)
             Oa01 ······ Oa11 ······ Oa17 ····· Oa21
             400 nm     710 nm     865 nm    1020 nm

    SLSTR:    ┊ S1   S2  S3 ┊         S5    S6          S7              S8  S9
             555  659  865         1610  2250        3740           10850 12000
                                                      +F1 (fire)    +F2 (fire)
                              S4(1375 cirrus)

### 2.5 Spectral overlap between the two

Yes, in VIS-NIR. Three SLSTR S-bands sit in OLCI's range:

| OLCI band | Centre | SLSTR pair | Centre | Overlap quality |
|--------------|--------|--------------|--------|-----------------------------|
| Oa06 | 560 nm | S1 | 555 nm | Close (5 nm offset) |
| Oa08 | 665 nm | S2 | 659 nm | Close (6 nm offset) |
| **Oa17** | **865 nm** | **S3** | **865 nm** | **Exact match** ←------ the GCP pair |

Beyond NIR, SLSTR has no OLCI counterparts (the SWIR bands S4--S6 and the thermal channels S7--S9, F1, F2). That asymmetry is precisely what makes the combination *valuable* rather than redundant.

------------------------------------------------------------------------

## 3. Geometry --- do they see the same ground?

Mostly yes, with a twist for the oblique view.

                             ┌──────────────────────────────────┐
                             │   SLSTR nadir swath ≈ 1 420 km   │
                             └──────────────────────────────────┘
                                ┌───────────────────────────┐
                                │   OLCI swath ≈ 1 270 km   │   (tilted ~12° off-nadir
                                └───────────────────────────┘    for sun-glint avoidance)
                                   ┌──────────────────────┐
                                   │ SLSTR oblique swath  │     (~55° along-track, observes a
                                   │       ≈ 750 km       │      different patch at the same instant)
                                   └──────────────────────┘
                             ←——————— across-track ———————→

-   **OLCI and SLSTR nadir** overlap almost entirely at the same second. OLCI (~1 270 km) fits inside SLSTR's nadir swath (~1 420 km).
-   **OLCI's 12° off-nadir tilt** shifts its nadir slightly across-track from SLSTR's --- small geometric offset that contributes to the registration residual.
-   **SLSTR oblique** at the same instant looks ~700 km along-track from where the spacecraft is. That patch becomes the nadir target a few minutes later. So **the same ground point is sampled three times per pass**: oblique first, OLCI middle, SLSTR nadir middle.

That three-fold coverage is the scientific reason SYNERGY exists. Aerosol retrieval needs observations of the same patch from different viewing angles to disentangle surface from atmosphere; the dual SLSTR views plus OLCI provide exactly that.

------------------------------------------------------------------------

## 4. Each pixel has `(lon, lat)` --- so why is sub-pixel co-registration still needed?

Every L1B pixel carries a `(lon, lat)` tag computed by tracing the instrument line-of-sight through the orbit + attitude + DEM lookup. Each instrument's `(lon, lat)` is **accurate to roughly ~100--200 m in the absolute sense** (figures of this order, exact values are in the Mission Performance Reports). Good enough to draw maps; not good enough for sub-pixel joint retrieval.

The problem: OLCI and SLSTR compute their `(lon, lat)` through **independent geolocation chains**:

    OLCI:   orbit + OLCI-instrument alignment + DEM → (lon, lat)_OLCI
    SLSTR:  orbit + SLSTR-instrument alignment + DEM → (lon, lat)_SLSTR

Both chains have ~100--200 m absolute error, but **their error patterns are different and uncorrelated**. The *difference* between the two declared positions for the same physical ground point has a systematic spatial residual of a fraction of a pixel up to roughly one OLCI pixel.

For visual maps that is invisible. For the L2 SY_2_SYN inversion --- which reads "OLCI Oa17 here, SLSTR S3 here, both observing the same patch" --- even sub-pixel misregistration biases the retrieval into the wrong aerosol / surface combination.

So the SY-1 algorithm does what bilinear-in-lat/lon cannot: it **measures the OLCI↔SLSTR shift directly** at a sparse grid of Ground Control Points on the actual scene, smooths the measurements into a continuous deformation field, and uses that field to project SLSTR onto OLCI pixel positions.

------------------------------------------------------------------------

## 5. How the operational chain co-registers OLCI and SLSTR

### 5.1 The seven steps (P0 → M4)

The SY-1 Detailed Processing Model breaks the chain into labelled steps. Renamed to be readable:

| Step | Operation | What it does |
|-----|----------------------|----------------------------------------------|
| P0 | Temporal overlap search | Find which OLCI and SLSTR L1B files cover the same time window. |
| P1 | OLCI ingestion | Read OLCI radiances, place them on the SYN reference grid (the OLCI image grid). Subsample to a sparse candidate-GCP grid; exclude water pixels. |
| P2 | SLSTR ingestion + calibration | Read SLSTR radiances; apply the per-band, per-view refinement factors from the SY1PP ADF. |
| M1 | Imagette extraction | Around each GCP, cut a *context imagette* (OLCI radiometry) and a *search imagette* (SLSTR radiometry resampled to the search window). |
| M2 | Cross-correlation | Compute the normalised cross-correlation (or phase correlation) between context and search imagettes. The peak position gives the local shift in OLCI pixel units. Sub-pixel refinement by parabolic peak interpolation. |
| M3 | Deformation field fit | Pool the per-GCP shifts. Fit a Thin-Plate Spline. Delaunay-triangulate. Within each triangle, fit a local affine model to the TPS predictions. Evaluate at every OLCI pixel. |
| M4 | Collocation | For each OLCI pixel, fetch SLSTR data at the displaced coordinate via bilinear interpolation. Repackage OLCI bands alongside on the same grid. Apply quality flags. |

### 5.2 The bits worth zooming in on

#### P0 --- why temporal-overlap search

OLCI and SLSTR ground-segment chains are independent. They slice their products into time-bounded files whose UTC boundaries don't always align --- even though the two instruments observe the same ground at the same instant. P0 just does the calendar arithmetic to identify which OLCI file overlaps which SLSTR file. Pure bookkeeping.

#### P2 --- why radiometric calibration *here*

L1B radiances are already calibrated using on-board black bodies and the VISCAL solar target. The SY-1 step `P2` applies a **vicarious-calibration correction** stored in the SY1PP ADF --- a small per-band, per-view factor derived from external validation targets (deserts, snow, lunar observations) that compensates for slow instrument drift. For a HEALPix re-indexing port of an existing SAFE SY product, this correction is *already applied* and does not need to be repeated.

#### Context vs search imagette (the SentiWiki glossary, verbatim)

-   **Context imagette**: "The radiometric counterpart of the context window, obtained by extracting the OLCI channel radiometry corresponding to the context window."
-   **Search imagette**: "The radiometric counterpart of the search window, obtained by resampling the SLSTR channel radiometry to the search window."

Two imagettes per GCP, not four. The naming distinguishes **which instrument** supplies the data, not coarse vs fine. OLCI provides the template (context imagette); SLSTR provides the searchable area (search imagette); cross-correlation between the two gives the local shift.

#### M2 --- cross-correlation, NCC vs phase correlation

The output is a shift vector `(Δr, Δc)` in OLCI pixel units. Two interchangeable implementations:

-   **Normalised cross-correlation (NCC)** --- spatial-domain sliding window. Reference: any standard template-matching tutorial.
-   **Phase correlation** --- Fourier-domain method (Kuglin & Hines 1975). Same peak location, faster for large windows because of FFT, more robust to uniform brightness offsets.

Sub-pixel precision comes from parabolic interpolation on the 3×3 neighbourhood of the peak.

#### Which bands are matched against which? --- Not explicitly documented

The DPM refers to a generic "OLCI reference channel" and "SLSTR nadir reference band" and points to the SY1PP ADF for the actual band selection. The ADF contents are not in the public SentiWiki pages I've checked.

Physical reasoning narrows the choice to **Oa17 ↔ S3 at 865 nm**: only NIR is bright enough over land to give a sharp correlation peak, and Oa17 / S3 are the only OLCI / SLSTR pair with identical central wavelengths. This is the overwhelmingly likely operational choice but should not be quoted as a fact from a document I have not read. Verification path: read the SY-1 ATBD or inspect a SY1PP ADF directly.

#### M3 --- Thin-Plate Spline, then a fast piecewise-affine evaluator

The TPS interpolates a smooth function from the scattered GCP shifts. Mathematically: given shifts `dᵢ` at GCP positions `pᵢ`, find `f(p)` minimising

$$E = \sum_i \|d_i - f(p_i)\|^2 + \lambda \int \|\nabla^2 f\|^2 \, dp$$

--- a balance between data-fitting and bending-energy minimisation (the elastic energy of a thin steel plate).

-   **λ = 0** → exact interpolation through every GCP. Noisy GCPs cause local bends.
-   **λ moderate** (operational choice) → smooth approximation, robust to GCP noise.
-   **λ → ∞** → pure affine fit, no local detail.

Why TPS over alternatives:

-   C² smoothness (no kinks in the deformation field).
-   Bending-energy interpretation matches the physical intuition.
-   Handles scattered, irregular data --- no grid needed.
-   Separates rigid (affine) from non-rigid (plate-bending) deformation naturally.
-   More robust than polynomial fits (no Runge oscillation at swath edges).

Evaluating TPS directly at every OLCI pixel would be `O(N_GCP)` per pixel --- too slow. So the algorithm pre-builds a **fast evaluator**:

1.  Delaunay-triangulate the GCPs in the OLCI image plane.
2.  In each triangle, fit a local affine model to the TPS predictions at the three vertices.
3.  For every OLCI pixel, identify its enclosing triangle and apply the local affine --- `O(1)` per pixel.

This piecewise-affine is an *approximation to the TPS within each small triangle*, not a regridding of OLCI. The deformation field lives on the OLCI grid; OLCI pixels themselves do not move.

#### M4 --- what "collocate" means here

M4 uses the deformation field to bilinearly interpolate SLSTR data at the displaced coordinate corresponding to each OLCI pixel. The 1 km SLSTR fields (meteorology, LST) get nearest-neighbour interpolation instead, because they are either categorical or already smoothed.

The phrase in the docs *"collocate all SLSTR channels and OLCI bands onto the OLCI reference grid"* sounds as if OLCI is also being resampled. **It isn't.** OLCI bands are already on the OLCI reference grid; "collocate" there means "package together on the common grid", a no-op for OLCI. The deformation field's one-way arrow is OLCI → displaced SLSTR coordinates; data flows the other way (SLSTR values into OLCI pixel slots).

------------------------------------------------------------------------

## 6. What makes Sentinel-3 unusually friendly to HEALPix

Both OLCI L1B and SLSTR L1B end in **sensor geometry with per-pixel `(lon, lat)` tags**. This is the opposite of Sentinel-2, where L1C / L2A are forced onto a UTM tile grid (with all the zone-overlap pathology that creates). The Sentinel-3 legacy chain hands you a non-projected sensor raster and a pair of geolocation arrays.

HEALPix integration is then a **pixel-to-cell lookup**: for every pixel, compute its HEALPix cell ID from `(lon, lat)` on the WGS84 ellipsoid. No projection to fight, no UTM zone boundaries, no tile-stitching overhead.

`legacy-converters` exploits this today. Two variants:

-   **Cell-point at HEALPix Level 29** (`settings/sentinel3.py`) --- each sensor pixel maps bijectively to one cell at ~12 mm cell edge. Lossless.
-   **PSF deconvolution at sensor-matched levels** (`settings/sentinel3_psf.py`) --- Level 15 (~199 m) for OLCI EFR / LFR (300 m), Level 14 (~398 m) for SLSTR S-bands (500 m), Level 13 (~796 m) for SLSTR F/I-bands (1 km) and OLCI ERR / LRR (1 200 m).

Both strategies work for **the individual L1B products**, OLCI alone or SLSTR alone. Neither produces SYNERGY.

------------------------------------------------------------------------

## 7. What the legacy converter doesn't yet do for SYNERGY

| Capability | In `legacy-converters` today? |
|-----------------------------------------|-------------------------------|
| Cell-point Level-29 ingestion of OLCI L1B | ✓ `settings/sentinel3.py` |
| Cell-point Level-29 ingestion of SLSTR L1B | ✓ `settings/sentinel3.py` |
| PSF ingestion at sensor-matched levels | ✓ `settings/sentinel3_psf.py` |
| Nearest-neighbour for categorical quality fields | ✓ |
| Bilinear for tie-point geometry / meteorology | ✓ |
| Settings file for SY_2_SYN at Level 15 | ✗ --- to be created |
| Settings file for SY_2_VGP / VG1 / V10 at Level 13 | ✗ --- to be created |
| Settings file for SY_2_AOD at Level 11 | ✗ --- to be created |
| STAC entries in `dggest` for SAFE SY products | ✗ --- to be added |
| GCP-driven OLCI↔SLSTR co-registration in HEALPix coordinates | ✗ --- only needed for the fully-HEALPix-native variant |
| Multi-resolution single-root Zarr writer | ✗ --- converter currently writes one group per invocation |

The gap is uneven. **The re-indexing port** (ingesting a SAFE SY product as-is, just re-mapping pixels into HEALPix cells) is mainly a configuration job: four settings dictionaries plus STAC plumbing. **The fully HEALPix-native variant** (re-implementing the SY-1 GCP / cross-correlation / TPS chain in HEALPix coordinates) is a larger algorithmic piece --- interesting but not required for a first deliverable.

------------------------------------------------------------------------

## 8. Where to start

Smallest useful step:

1.  Write `legacy_converters/settings/sentinel3_synergy.py` mirroring the structure of `sentinel3_psf.py`. Four group-settings dictionaries: SY_2_SYN (L15), SY_2_VGP (L13), SY_2_VG1 (L13), SY_2_V10 (L13), SY_2_AOD (L11).
2.  Add STAC entries for SAFE SY products to [`dggest`](https://github.com/EOPF-DGGS/dggest) so that `Scene.convert_to_healpix(...)` works end-to-end.
3.  Write a benchmark notebook comparing the HEALPix output against the operational SY_2_SYN on a pilot orbit and on the Mediterranean Basin regional case proposed in the GRID4EARTH plan.

That gets `SW-07` (the SOW's S3 SYN DGGS processor) to a defensible v0.1. The operational SY-1 co-registration is preserved bit-for-bit --- `legacy-converters` is just re-indexing the operational product onto HEALPix.

Once validated, the more ambitious goal --- replacing the OLCI image grid with HEALPix Level 15 throughout, and re-implementing the GCP / cross-correlation / TPS chain to operate on HEALPix-indexed data --- becomes a well-scoped second iteration with an expected payoff of sharper outputs than the operational baseline (the PSF deconvolution at the OLCI and SLSTR re-indexing stages removes the dominant blur source that the operational chain leaves in).

------------------------------------------------------------------------

## 9. Bottom line

Sentinel-3 carries two complementary instruments: OLCI is a hyperspectral colour imager with 21 narrow bands in the visible and NIR; SLSTR is a broader-spectrum dual-view radiometer adding SWIR and thermal infrared. Their L1B products both end in sensor geometry with per-pixel `(lon, lat)` --- unusually friendly to HEALPix.

`legacy-converters` already re-indexes both instruments individually onto HEALPix in two variants. What it cannot yet do is **SYNERGY**, because SYNERGY adds two things: sub-pixel OLCI ↔ SLSTR co-registration (a real algorithmic step) and a family of L2 products at three resolutions (a configuration matter). For the immediate re-indexing port the operational SY-1 co-registration can be inherited as-is; the configuration work is four settings dictionaries. A fully HEALPix-native re-derivation is a worthwhile second iteration.

------------------------------------------------------------------------

## Appendix --- acronyms used here

| Acronym | Meaning |
|-------------|-----------------------------------------------------------|
| ADF | Auxiliary Data File (NetCDF files holding processor configuration) |
| ATBD | Algorithm Theoretical Basis Document |
| BT | Brightness Temperature |
| DEM | Digital Elevation Model |
| DGGS | Discrete Global Grid System |
| DPM | Detailed Processing Model |
| EO | Earth Observation (the OLCI L1 processing mode) |
| ECMWF | European Centre for Medium-Range Weather Forecasts |
| FOV | Field Of View |
| GCP | Ground Control Point |
| HEALPix | Hierarchical Equal Area isoLatitude Pixelisation |
| ISP | Instrument Source Packet |
| L0 / L1A / L1B / L2 | Processing levels (raw → calibrated counts → calibrated radiance → geophysical product) |
| LST | Land Surface Temperature |
| LUT | Look-Up Table |
| NCC | Normalised Cross-Correlation |
| NEDT | Noise-Equivalent brightness-temperature Difference |
| NIR | Near-Infrared |
| OLCI | Ocean and Land Colour Instrument |
| PDU | Product Dissemination Unit |
| PFS | Product Format Specification |
| PSF | Point Spread Function |
| RBT | Radiance and Brightness Temperature (the SLSTR L1B product designator) |
| SDR | Surface Directional Reflectance |
| SLSTR | Sea and Land Surface Temperature Radiometer |
| SWIR | Short-Wave Infrared |
| SY / SYN | SYNERGY (Sentinel-3 product family combining OLCI and SLSTR) |
| SY1PP | SY-1 Processing Parameters (the relevant ADF) |
| TIR / LWIR | Thermal Infrared / Long-Wave Infrared |
| TOA / TOC | Top of Atmosphere / Top of Canopy |
| TPS | Thin-Plate Spline |
| VISCAL | Visible Calibration target (on-board SLSTR) |
