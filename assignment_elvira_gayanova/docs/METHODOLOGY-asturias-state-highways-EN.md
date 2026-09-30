# Measuring the Spanish State Road Network in Asturias

**Advanced QGIS (Spatial Thoughts) — Section 1 exercises reproduced on own data**

Author: Elvira Gayanova
Date: 2026-09-20 (updated 2026-09-22: assumption A6 added)
Software: QGIS 4.2, macOS
Course reference: Advanced QGIS by Ujaval Gandhi, Section 1 — Processing Toolbox (Exercise 1.1, Challenge 1.1.1, Exercise 1.2, Challenge 1.2.1)

---

## 1. Purpose

The course exercises measure the length of the national highway network in Karnataka, India. This document reproduces the same workflow on the autonomous community of **Asturias, Spain**, using independently sourced OpenStreetMap data, and reports:

- the processing chain and every parameter used,
- the results obtained,
- the discrepancies found between intermediate results and how each was diagnosed,
- the assumptions that qualify the final numbers.

The reason for reproducing the course on different data is that course datasets are pre-cleaned. Real extracts are not, and the failures encountered here are the substance of the exercise.

**Headline result:** the Spanish state road network (`A-`, `AP-`, `N-`) within the administrative boundary of Asturias measures **1293.87 km** in OpenStreetMap as of 2026-09-15, distributed across 42 of the 78 *concejos* (municipalities).

---

## 2. Data acquisition

### 2.1 Source

| File | Source | Data as of |
|---|---|---|
| `asturias_osm_free_2026-09-15.gpkg` | Geofabrik free extract, Asturias (`.gpkg.zip`) | 2026-09-15T20:20:37Z |
| `asturias_osm_2026-09-15.osm.pbf` | Geofabrik, Asturias (`.osm.pbf`) | 2026-09-15, same nightly update |

Licence: OpenStreetMap data, Open Database License (ODbL) 1.0. Attribution required on any published result: **© OpenStreetMap contributors**.

### 2.2 Deviation from the course: no QuickOSM plugin

The course downloads OSM data through the QuickOSM plugin. On 2026-09-16 the QGIS Official Plugin Repository was reported as `unavailable` in QGIS 4.2, and the repository URL did not resolve in a browser either. Data was therefore downloaded directly from Geofabrik, which is also the ultimate source of the course data.

### 2.3 Two extracts, two purposes

The free `.gpkg` extract carries pre-built thematic layers (roads, railways, waterways) but its administrative-area layer (`gis_osm_adminareas_a_free`) exposes only `code`, `fclass` and `name`, with **no administrative level**. The `.osm.pbf` retains the original OSM tags `boundary` and `admin_level`, which are required to isolate municipalities. Both files were therefore used: the `.gpkg` for roads, the `.pbf` for boundaries.

---

## 3. Data preparation

### 3.1 Road network — `asturias_major_roads` (14 847 features)

Extracted from `gis_osm_roads_free` with *Extract by expression*:

```
"fclass" IN ('motorway', 'trunk', 'primary', 'secondary', 'tertiary')
```

**Assumption A1.** Link roads (`motorway_link`, `trunk_link`, …) are excluded. They are short junction ramps; including them would inflate network length without adding distinct roads.

**Observation.** OSM stores a road as many separate features, split at junctions and wherever an attribute changes (bridge, speed limit, number of lanes). A single road such as A-66 is therefore hundreds of features. Any length calculation must sum segments, not count features.

### 3.2 Municipalities — `asturias_concejos` (78 features)

Extracted from the `multipolygons` layer of the `.pbf`. In Spain `admin_level = 8` denotes a municipality; in Asturias these are called *concejos*.

The first extraction returned **80** features against an official count of **78**.

**Cause.** The Geofabrik extract is cut with a buffer around the region, so small neighbouring municipalities that touch the border are included in full.

**Detection method.** Not visual inspection — the count was compared against the official figure, and all 80 names were compared against the official list of Asturian concejos. Two features did not belong:

| osm_id | Name | Region |
|---|---|---|
| 344822 | Negueira de Muñiz | Galicia (Lugo), borders Ibias |
| 345453 | Tresviso | Cantabria, borders Cabrales |

Final expression:

```
"boundary" = 'administrative' AND "admin_level" = '8' AND "osm_id" NOT IN ('344822', '345453')
```

Exclusion is by `osm_id`, not by name: OSM ids are unique, whereas names may repeat, exist in two languages or differ by accent. Four names that look suspicious were verified and retained: *Samartín del Rei Aurelio* (Asturian form of San Martín del Rey Aurelio), *San Martín de Oscos*, *Peñamellera Alta* and *Peñamellera Baja* (two distinct concejos).

Note that `admin_level` and `osm_id` are **text** fields in the `.pbf`, hence quoted as `'8'` and `'344822'`.

### 3.3 Regional boundary — `asturias_boundary` (1 polygon)

Built with *Dissolve* (no dissolve field) from the corrected 78 concejos. An earlier version built from the 80-feature layer wrongly included parts of Galicia and Cantabria and was rebuilt. This layer later served as the independent check described in section 6.

---

## 4. Exercise 1.1 — total length of the state network

### 4.1 Defining "state network" in Spain

The course distinguishes two classes, NH and SH, by the `ref` field. Spain has a three-tier structure visible in the same field:

| Tier | `ref` prefix | Examples |
|---|---|---|
| State network (Ministerio de Transportes) | `A-`, `AP-`, `N-` | A-8, A-66, AP-66, N-632 |
| Regional network (Principado de Asturias) | `AS-` | AS-381, AS-II |
| Municipal roads | concejo prefixes | GJ-10, GJ-81 (Gijón), GI-6 |

**Important distinction.** Road class (`fclass`) and road ownership (`ref` prefix) are independent attributes. `GJ-81` is tagged `fclass = motorway` but is a municipal road, not a state motorway. Selecting by `fclass` alone would answer a different question.

**Trap.** A pattern testing only the first letter `A` also matches the entire `AS-` regional network. The hyphen must be part of the pattern.

### 4.2 Processing chain

CRS of inputs: `OGC:CRS84` (WGS 84, longitude/latitude, degrees). Project CRS left unchanged — the course does not require a projected CRS here, and all lengths are computed ellipsoidally.

| # | Algorithm | Parameters | Output |
|---|---|---|---|
| 1 | Extract by expression | `regexp_match("ref", '^(A\|AP\|N)-')` on `asturias_major_roads` | 3385 features |
| 2 | Add geometry attributes | *Calculate using* = **Ellipsoidal calculations** | field `length` (metres) |
| 3 | Field calculator (algorithm) | `length_km` = `"length" / 1000`, Decimal (double) | field `length_km` |
| 4 | Basic statistics for fields | field `length_km` | `outputs/state_highways_statistics.html` |
| 5 | Refactor fields | keep `osm_id`, `fclass`, `name`, `ref`, `length_km` | `asturias_state_highways` |

Regex components: `^` anchors to the start of the string; `(A|AP|N)` matches one of the three prefixes; the literal `-` separates `A-` from `AS-`.

Steps 2 and 3 use the Processing **algorithm** *Field calculator*, not the field-calculator button in the attribute table. The algorithm produces a new layer and leaves the input untouched, which keeps the chain reproducible.

### 4.3 Result

| Measure | Value (km) |
|---|---|
| **Sum** | **1664.0941** |
| Count | 3385 segments |
| Mean | 0.4916 |
| Median | 0.1634 |
| Minimum | 0.00203 (2.03 m) |
| Maximum | 18.2005 |
| Standard deviation | 0.9410 |
| Coefficient of variation | 1.9141 |

Interpretation: the median is three times smaller than the mean, so the distribution is strongly right-skewed — half of all segments are shorter than 164 m, while the total is driven by a minority of long rural stretches. The 2 m minimum is a stub at an interchange, a direct consequence of OSM splitting lines at every node. A coefficient of variation near 2 is normal for an OSM road network: urban areas generate thousands of short segments, motorways a few long ones.

### 4.4 Side observation

*Add geometry attributes* also produced two fields not discussed in the course text:

- `straightdis` — straight-line (chord) distance between the first and last vertex, in metres;
- `sinuosity` — `length / straightdis`.

Values of exactly 1 mark perfectly straight segments; a motorway curve gave 1.079. In surveying terms, `length` is the traverse measured along its legs, `straightdis` is the closing line, and sinuosity is their ratio. Not required here, but relevant to corridor analysis where terrain matters.

---

## 5. Challenge 1.1.1 — features without a name

**Task.** Produce a layer excluding features whose `name` is empty.

**First attempt.** *Extract by attribute* with the operator `is not null`, as in the course solution. It returned **3385 of 3385** — nothing was filtered.

**Diagnosis.** The statistics report from step 4 already contained the answer: `NULL (missing) values: 0`. There are no true NULLs in the layer. The Geofabrik GeoPackage writes missing OSM tags as **empty strings** (`''`), not as NULL.

NULL means "no value exists"; `''` means "a value exists and contains zero characters". They are visually identical in an attribute table and behave differently in every comparison. The course dataset (`karnataka.gpkg`) contains true NULLs, which is why the course solution works there and fails here.

**Working solution.** *Extract by expression* on `asturias_state_highways`:

```
"name" is not null and "name" != ''
```

The `is not null` clause is retained although this layer has none: future extracts may contain them.

**Result.** `challenge_1_1_1_named_state_highways` — **2906** features. **479** removed, i.e. **14.2 %** of the state network carries a road number (`ref`) but no name in OSM.

---

## 6. Exercise 1.2 — network length per concejo

### 6.1 Method

*Sum line lengths* takes a polygon layer and a line layer and writes two fields to each polygon: `COUNT`, the number of intersecting lines, and `LENGTH`, the total length of line **inside** that polygon. Lines crossing a boundary are split, so each municipality receives only the portion that physically lies within it.

Input lines are `asturias_state_highways` — the full 3385-feature layer, **not** the challenge output. Whether a road is named in OSM has no bearing on its length; using the filtered layer would have discarded 479 real segments.

### 6.2 Second degrees/metres failure and the general rule behind it

The first run produced `LENGTH` values of 0.217, 0.313, 1.269 — impossible for a municipality crossed by 196 road segments.

**Cause.** *Sum line lengths* has **no** *Calculate using* parameter. It takes its measurement method from the project: **Project → Properties → General → Measurements → Ellipsoid**, which was at its default `None / Planimetric`. Planimetric measurement in a geographic CRS measures in **degrees**.

**Verification before changing anything.** Summing the visible non-zero values and extrapolating to 78 rows gave ≈15. Dividing the exercise 1.1 result by that figure: 1664.09 ÷ 14.95 ≈ 111 km per unit — the length of one degree of latitude. The unit was therefore degrees, confirmed arithmetically rather than assumed.

**Fix.** Project ellipsoid set to **WGS 84 (EPSG:7030)** and the algorithm re-run. `LENGTH` then returned 20 149.95, 31 285.45, 111 278.21 — metres.

**Confirmation.** For the same feature, 111 278.21 m ÷ 1.26973° ≈ 87 640 m per degree, consistent with the length of a degree at latitude 43° for a mixed-bearing network (81 km per degree of longitude, 111 km per degree of latitude).

**General rule extracted.** *If a Processing algorithm offers no parameter controlling how distances are computed, it inherits the project's ellipsoid setting.* This must be verified before running, not after. The same trap in exercise 1.1 was avoided only because *Add geometry attributes* has its own parameter, which overrode the project default.

### 6.3 Result

`state_highways_by_concejos` — 78 features with `COUNT`, `LENGTH` (m) and `length_km`.

| Measure | Value (km) |
|---|---|
| **Sum** | **1293.8727** |
| Count | 78 concejos |
| Mean | 16.5881 |
| Median | 2.8995 |
| Maximum | 111.278 |
| Standard deviation | 25.9438 |
| Coefficient of variation | 1.5640 |

The report gives `Unique values: 43` for 78 features with `Majority: 0`, so **36 concejos contain 0 km** of state network and **42 contain some**. This is consistent with the region's geography: the state network follows the coastal A-8, the A-66 / AP-66 / N-630 corridor through the Pajares pass, and a handful of N-roads. Western and eastern mountain concejos are served by the regional `AS-` network, which is outside the selection.

---

## 7. Discrepancy analysis and validation

### 7.1 The discrepancy

| Source | Result |
|---|---|
| Exercise 1.1 — whole layer | 1664.09 km |
| Exercise 1.2 — sum over 78 concejos | 1293.87 km |
| **Difference** | **370.22 km (22.2 %)** |

A difference of this size cannot be explained by rounding or by slivers between polygons. Two competing explanations were possible:

- **H1.** The missing length lies outside the administrative boundary of Asturias, inside the buffer with which Geofabrik cuts its extracts — the same buffer that produced 80 concejos instead of 78.
- **H2.** The concejo polygons do not tile the region without gaps, and road segments fall into the gaps.

The two hypotheses call for opposite responses: H1 is a documented property of the data, H2 is a defect requiring geometry repair. They were therefore separated by experiment rather than by judgement.

### 7.2 The test

*Sum line lengths* was re-run with the same line layer against `asturias_boundary` — a **single** polygon covering the whole region, so gaps between concejos cannot exist by construction.

| Run | Polygons | LENGTH (m) | COUNT |
|---|---|---|---|
| Per concejo | 78 polygons | 1 293 872.737 | — |
| Whole region | 1 polygon | 1 293 872.737 | 3313 |

**The two results are identical to six decimal places.**

### 7.3 What the test proves

1. **The concejo layer tiles the region without gaps.** Had gaps existed, the sum over 78 parts would be smaller than the measurement over the single dissolved polygon. Exact agreement excludes H2. As a by-product this also validates `asturias_boundary`, since the dissolved outline captures exactly what its parts capture.

2. **The missing 370.22 km lies outside Asturias.** It is present in `asturias_state_highways` but beyond the administrative boundary: sections of A-8 continuing into Cantabria and Galicia, A-66 and N-630 beyond the Pajares pass towards León, and roads in the border strip.

3. `COUNT` fell from 3385 to **3313**, so 72 segments lie entirely outside the region, and further segments are clipped by the boundary. Length falls proportionally more than feature count, as expected when long trans-border segments are cut.

### 7.4 Which figure to use

Both numbers are correct; they answer different questions.

| Figure | Meaning | Use |
|---|---|---|
| 1664.09 km | State network within the bounds of the Geofabrik extract | Exercise 1.1 result; not a regional statistic |
| **1293.87 km** | **State network within the administrative boundary of Asturias** | **The figure to report** |

---

## 8. Assumptions and limitations

Every figure in this document is qualified by the following, which are properties of the data rather than errors in the processing:

**A1.** Link roads (`motorway_link` and similar) are excluded from the source network.

**A2.** A dual carriageway is stored in OSM as **two separate lines**, one per direction, and both are counted. Motorway kilometres are therefore counted approximately twice relative to an administrative road inventory.

**A3.** Urban sections of `N-` roads (former *travesías*, now often transferred to municipal ownership) retain their `N-` number in OSM and are included by the `ref`-based selection.

**A4.** Selection is by the `ref` prefix, which reflects road numbering, not the current legal owner of the road. A road transferred between administrations may keep its old number in OSM.

**A5.** Results reflect OSM completeness as of 2026-09-15. 14.2 % of state-network segments have no `name` tag, which indicates the general level of attribute completeness in this extract.

**A6.** Some segments carry **several road numbers in one `ref` field**, separated by a semicolon, where two roads share one carriageway (found during section 2 in Lena: `AS-242;N-6…`). The selection pattern `^(A|AP|N)-` tests only the **first** number, so a segment whose state number is not listed first is **not selected**. The state network is therefore slightly **under-counted** on shared sections; the size of the effect has not been measured. A pattern that tests every number in the field would be `regexp_match("ref", '(^|;)(A|AP|N)-')`.

Because of A2 and A3 (partly offset by A6), the figure of 1293.87 km exceeds the official length of the state network in Asturias (approximately 1000 km). It is correct for the stated selection criterion and should not be presented as an administrative inventory figure.

---

## 9. Errors encountered, and how they were found

Three processing errors occurred. None was detected by the software — all three produced valid-looking output. Each was caught by an external check, which is the transferable lesson.

| # | Symptom | Cause | Detection |
|---|---|---|---|
| 1 | `Sum 0.0149`, `Max 0.000163` for a road network | *Add geometry attributes* left at *Calculate using* = `Layer CRS`; measured in degrees | Order-of-magnitude check: a regional network cannot be 15 m long. Confirmed by 0.0149 × 1000 × 111 km/° ≈ 1664 km |
| 2 | `Sum 5 730 805`, `Min 1`, `Max 3385` | *Basic statistics for fields* left at its default field, `fid` | 3385 × 3386 ÷ 2 = 5 730 805 — the sum of row numbers. Also caught by the report's own first line, `Analyzed field:` |
| 3 | `LENGTH` of 0.217 and 1.269 per municipality | *Sum line lengths* has no measurement parameter; project ellipsoid was `None / Planimetric` | Same order-of-magnitude reasoning as #1, before changing any setting |

Error 1 additionally hid inside the saved layer: `length_km` held degrees ÷ 1000 and displayed as `8.759093915…e-06`, which was read as kilometres because the attribute column was too narrow to show the `e-06` suffix.

### Working rules adopted

1. Check the **order of magnitude** of any computed length or area against physical reality before accepting it.
2. Values of the form `0.0000…` in a length or area field indicate computation in degrees. One degree in Asturias is 81–111 km.
3. Widen an attribute column before reading a number — scientific notation hides in truncated cells.
4. Read the `Analyzed field:` line of a statistics report before using any number from it.
5. If an algorithm offers no parameter controlling how distances are computed, check the project's ellipsoid setting first.
6. When two results that should agree do not, design a test that distinguishes the candidate explanations rather than choosing the more comfortable one.

---

## 10. Deliverables

All layers are stored in `data/asturias.gpkg`.

| Layer / file | Features | Content |
|---|---|---|
| `asturias_major_roads` | 14 847 | source road network, five `fclass` values |
| `asturias_concejos` | 78 | municipalities, neighbouring-region features removed |
| `asturias_boundary` | 1 | dissolved regional outline |
| `asturias_state_highways` | 3385 | state network with `length_km` |
| `challenge_1_1_1_named_state_highways` | 2906 | state network excluding unnamed features |
| `state_highways_by_concejos` | 78 | `COUNT`, `LENGTH` (m), `length_km` per concejo |
| `state_highways_by_concejos_formatted` | 78 | `concejo`, `total_length` (rounded km) |
| `outputs/state_highways_statistics.html` | — | statistics report for exercise 1.1 |
| `solutions/Processing_Tools_Checkpoint1.qgz` | — | project at the end of exercise 1.1 |
| `solutions/Processing_Tools_Checkpoint2.qgz` | — | project at the end of exercise 1.2 |

Naming follows the course convention with the region substituted: *karnataka* → *asturias*, *districts* → *concejos*, *national_highways* → *state_highways*.
