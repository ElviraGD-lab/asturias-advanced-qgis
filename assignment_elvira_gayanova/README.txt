ADVANCED QGIS - ASSIGNMENT
Calculate Highway Statistics, built on Asturias (Spain) data

Elvira Gayanova, September 2026


WHAT IS IN THIS ARCHIVE

  assignment_elviragayanova.model3
      The assignment model. Same file as submitted through the form.

  assignment_asturias.gpkg
      One GeoPackage containing everything needed to run the model:
      input layers, result layers, styles, the QGIS project and the
      embedded model. Open the project inside it and the model is
      already in Processing Toolbox > Project models.

  hexbin_density.model3
      Section 3 model, rebuilt as a general point-density tool.
      Includes the extra step from challenge 3.1.1.

  docs/METHODOLOGY-asturias-state-highways-EN.md
      Write-up of sections 1 and 2 on Asturias data: data sources,
      processing chain, results, errors found and how they were
      diagnosed, and the assumptions behind the numbers.

  screenshots/
      assignment_model_complete.png       the model with feature counts
      assignment_state_network_map.png    state network per concejo
      assignment_regional_network_map.png regional network per concejo
      assignment_independence_test.png    model run in a project set to
                                          EPSG:4326, ellipsoid None
      model_hexbin_with_max.png           hexbin model, challenge 3.1.1
      hexbin_density_asturias.png         density of place names
      hexbin_max_density_cell.png         the densest cell


HOW TO OPEN

  1. Open QGIS.
  2. Browser panel > navigate to assignment_asturias.gpkg > expand it.
  3. Right-click "assignment_asturias" (QGIS icon) > Open Project.
  4. The model is in Processing Toolbox > Project models > asturias >
     Calculate Highway Statistics.
  5. Run it with:
       Roads Layer            = asturias_major_roads
       Admin Boundaries Layer = asturias_concejos
       Highway Type           = State network or Regional network


ABOUT THE DATA

  The model is built on Asturias, not on the Karnataka course data,
  and selects road networks by the Spanish "ref" numbering:

      State network      A-, AP-, N-    (A-8, AP-66, N-630)
      Regional network   AS-            (Principality of Asturias)

  Run on Indian data it returns an empty table, as NH / SH codes are
  not present there.

  Source: OpenStreetMap, Geofabrik extract for Asturias, 2026-09-15,
  ODbL 1.0. (c) OpenStreetMap contributors.


WHAT THE MODEL DOES

  Extract by expression   select roads by "ref", based on the enum
  Reproject roads         to EPSG:25830 (ETRS89 / UTM zone 30N)
  Reproject boundaries    to EPSG:25830
  Sum line lengths        road length inside each municipality
  Refactor fields         output table: concejo, total_length (km)

  Both layers are reprojected inside the model on purpose, so lengths
  come out in metres whatever the project CRS and ellipsoid are. This
  was tested in a project set to EPSG:4326 with the ellipsoid at
  "None / Planimetric" - the results did not change. Without this the
  same run returns zeros, because lengths are measured in degrees and
  disappear after dividing by 1000.

  The selection pattern is (^|;)(A|AP|N)- rather than ^(A|AP|N)- .
  Some OSM segments carry two road numbers in one "ref" field, for
  example AS-242;N-630, where a regional and a state road share one
  carriageway. Anchoring only to the start of the string misses 14
  such segments, 1.6 km, all of them in the concejo of Lena.


RESULTS

  State network      1294 km, present in 42 of 78 concejos
                     longest: Valdes 111 km, Villaviciosa 95,
                     Llanes 87, Lena 79

  Regional network   2845 km, present in 77 of 78 concejos
                     longest: Tineo 147 km, Siero 118, Gijon 99
                     the only concejo with no AS- road at all is
                     San Tirso de Abres

  The two networks complement each other. The state network is
  transit: the coastal A-8 and the A-66 corridor through the Pajares
  pass, which leaves 36 municipalities with nothing. The regional
  network is connectivity and covers almost the whole region. Tineo,
  the largest concejo by area, has no state road and the longest
  regional network of all.


CHECKS

  The state network figures were verified against the same
  calculation done by hand in section 1 of the course, municipality
  by municipality: identical everywhere except Lena, where the model
  finds the 14 extra segments described above.