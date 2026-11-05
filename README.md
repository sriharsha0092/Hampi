# Hampi and four other heritage sites: does a site look different from its surroundings in free satellite data?

A learning project (Google Earth Engine, Python, PostGIS). **It is not a site-detection pipeline.** It is a baseline test: for five known sites, does the area of the site differ measurably from the ring of land around it, and how large are such differences at random places where nothing is known? I wrote it to learn Earth Engine and the basics of remote-sensing statistics.

**Short answer:** no robust difference. After controlling for land cover, the site-versus-ring differences at Hampi sit within the range seen at matched random places (judged for shrub; bare has too few reference points). The test can only detect large differences, and a null result cannot show that a method would find unknown sites. A post-hoc check at smaller radii gives a weak hint (mostly in radar at Hampi), which cannot be separated from the monument compound itself looking different.

## Design

**Sites.** Five sites with a 300 m circle each: `Hampi_baseline`, `Pattadakal`, `Ramappa`, `Hire_Benakal` and `Warangal_central`. Two are flagged: `Hire_Benakal` (the circle sits on the village, not on the tomb cluster, so its rows are unreliable) and `Warangal_central` (urban; only 250 pixels inside and 262 in the ring survived the natural-land mask, below the 300 minimum for the ring, so it has no rows in the results tables and only shows up in the PostGIS queries). So the results cover four sites, not five. Coordinates are in `SITES` in the notebook.

**Ring.** The band between 500 m and 1,000 m from the site centre.

**Data.** Sentinel-2 surface reflectance, 2021-01-01 to 2023-06-06, masked with Google Cloud Score+ (clear-sky probability of at least 0.60), median composite. Sentinel-1 radar (VV, VH; same dates), SRTM elevation and slope, and ESA WorldCover v100 for land cover. Ten features: NDVI, SAVI, NDMI, BSI, SWIR1, seasonal NDVI (October to December minus February to April), VV, VH, elevation, slope. Pixels are sampled at 10 m (up to 4,000 per zone, seed 42) and restricted to natural land (shrub, grass, bare).

**Round 1.** Whole circle against the ring, natural land only.

**Round 2.** The same pixels, but compared within land-cover class (shrub, grass, bare), added after round 1 showed large differences that looked like land-cover differences. Cells with fewer than 60 pixels per class in either zone are left blank.

**Effect sizes.** Cohen's d (site minus ring, pooled standard deviation) is the main effect size. The Jeffries-Matusita distance (JM, scale 0 to 2) was added later as a second one. Mann-Whitney p-values are computed but not interpreted: with thousands of spatially autocorrelated pixels they are meaningless.

**Matched random reference points (the yardstick).** Each of three parent sites (`Hampi_baseline`, `Pattadakal`, `Ramappa`) gets 6 random points 5 to 20 km away with the same radius and ring (18 in total, seed 2027). A real site only stands out if its |d| exceeds the 90th percentile (`ref_p90`) of |d| at the random places, and only features with at least 8 usable reference points are judged. In round 1, 14 of the 18 points were usable (the other four had too few natural-land pixels after the mask). A reference summary was also computed within land-cover class for round 2: 11 reference points were usable for shrub and 7 for bare, so bare falls under the minimum of 8 and is not judged.

**PostGIS.** All sampled pixels are loaded into a PostGIS table with a spatial index, with three example queries (mean NDVI by zone, distance bands from the Hampi centre, land-cover mix by zone) and a `pg_dump`.

## Results

**Round 1** (whole circle against the ring) showed large |d| at several sites, mostly in elevation, slope, seasonal NDVI and radar. That is what a land-cover difference looks like, which is why round 2 was added.

**Round 2** (within land-cover class): at Hampi, shrub is the only class with enough reference points to judge (11), and all ten features are within the range seen at matched random places (the largest is SWIR1, d = 0.58 against a random-places p90 of 1.00). Bare has only 7 usable reference points, so it isn't judged; its |d| values at Hampi are 0.32 or smaller. Grass has too few pixels at Hampi (34 inside the circle) and is left blank. The reference points show how large |d| gets by chance, so the test only detects large differences.

Round 1 and round 2 tables are in `outputs_cs_plus/` (`round1_results.csv`, `round2_results.csv`, `reference_point_summary.csv`). The round-2 reference summary and the radius sweep are printed in the notebook (sections 7 and 7b) but I didn't save them as CSVs.

**Radius sweep (post hoc, exploratory).** I suspected the 300 m circle dilutes the site. Using the cached pixels, I re-computed round 1 for the pixels within 100, 150, 200 and 300 m of the centre, with matched reference points at each radius. Number of the 10 features above the random-places 90th percentile (about 1 is expected by chance):

| Site | 100 m | 150 m | 200 m | 300 m |
|---|---|---|---|---|
| Hampi_baseline | 2 | 2 | 0 | 0 |
| Pattadakal | 2 | 2 | 2 | 0 |
| Ramappa | 3 | 3 | 3 | 1 |

The features above the line at 100 to 150 m are mostly correlated pairs, so they amount to about one signal per site. At Hampi that is radar (VV and VH, d = 0.75 and 0.78 at 100 m against p90 of 0.53 and 0.54; at 150 m the margin is thin). At Pattadakal it is SAVI and SWIR1 (d = 1.01 and 1.22 at 100 m against 0.81 and 1.06). At Ramappa it is seasonal NDVI, VV and VH, with large differences at every radius, i.e. a setting difference, not a core effect. Across the whole sweep (3 sites x 4 radii x 10 features = 120 comparisons) 20 exceed the line against about 12 expected by chance, with nested radii and correlated features widening the real chance spread. This is not controlled for land cover. I read it as a weak hint that is mostly radar at Hampi and mostly vegetation and dryness at Pattadakal, consistent with, but not evidence of, dilution. It does not change the main conclusion. The table is printed in section 7b of the notebook.

## Limitations

- **A null cannot show the method finds unknown sites.** Only large differences are detectable (see the reference points).
- **Standing monuments, not buried features.** Hampi is dominated by standing structures. The theory behind this kind of test is about buried or subtle features; this test does not address them.
- **The ring may contain remains.** Around a large monument complex, the ring is not necessarily empty, which makes the contrast conservative.
- **The circle may dilute the signal.** Partly examined by the radius sweep, which is post hoc and not controlled for land cover.
- **Few sites, 10 m pixels.** Five sites, one date range (2021 to 2023), and a 300 m circle is about 2,800 pixels.
- **Few reference points.** 14 of 18 were usable in round 1, and in round 2 only 11 for shrub and 7 for bare (bare isn't judged). The radius sweep uses subsets of the same pixels, so its radii are not independent.
- **Spatial autocorrelation.** Pixels within a zone are not independent; read effect sizes, not p-values.
- **`Hire_Benakal` is on the village and `Warangal_central` is urban**, as flagged above.

## Amendment log

The notebook's header records the order in which the analysis changed and why, including the cloud-mask correction (the legacy QA60 mask stops working on 2022-01-25: from February 2022 it reports every pixel as clear, in every month, so cloudy images go in unmasked, while Cloud Score+ keeps only 13 to 71% of the Hampi circle in April to December 2022. All results come from the Cloud Score+ rerun and the QA60 numbers are superseded), the added round 2, the reference points, JM and the radius sweep. Read it before reading any number.

## Reproducing

1. Open `hampi_multisite_pipeline.ipynb` in Colab (Runtime, Run all). Earth Engine needs your own Google Cloud project ID in `EE_PROJECT` (non-commercial registration); the notebook has `YOUR-PROJECT-ID` there. The notebook is saved with its outputs, so you can read every number without running it. I removed the map output and the widget state.
2. The PostGIS section needs a Linux environment with root access (Colab works) and installs PostgreSQL and PostGIS on first run.
3. Seeds are fixed in the settings cell and written to `settings_record.json`.

## AI assistance

Parts of the notebook and this README were drafted with AI assistance (Claude) while I was learning Earth Engine and the statistics. I ran the analyses, checked the outputs against the numbers above, and take responsibility for the content.

## Citation and licence

Kavali, S. H. (2026). *Hampi multi-site baseline test* [software]. Zenodo. DOI: [DOI AFTER RELEASE].

Licence: [CHOOSE BEFORE RELEASING].

Contact: [EMAIL OR GITHUB PROFILE].
