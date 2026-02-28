# OneWeb Constellation Tracker — Changelog

## v16 — Constellation Trails, Catalog Preload & UI Cleanup (2026-02-28)
**Backup:** `index_v15_tlegraph.html`

### Added
- **Constellation trail lines**: Per-group orbital trail visualization toggled via a new orbit-icon button on each catalog row. Renders one full orbital period per satellite as `THREE.LineSegments` with additive blending. Adaptive sampling: Starlink (200 sampled, 40 steps), OneWeb (all, 60 steps), GPS (all, 120 steps). Async computation with batched yields to keep the UI responsive. Trails auto-recompute on time progression (throttled: 4s at 1x, 1s at >100x) and immediately on reference frame switch. Coexists with single-satellite orbit paths.
- **Background catalog preload**: All CelesTrak catalog groups now load automatically in the background 2 seconds after app launch (2 parallel streams, 300ms stagger). Groups load hidden — clicking a preloaded group in the Catalog panel toggles it on instantly with no fetch delay. `loadCatalogGroup()` accepts `{ showOnLoad: false }` to support preloading without making satellites visible.
- **HD satellite point textures**: All satellite dot textures upgraded from 64x64 to 128x128 canvas with improved radial gradient (tighter bright core at 15%, smoother mid-falloff at 35%/60%, cleaner edge). Applies to OneWeb, GEO, and shared catalog point clouds.
- **TLE age field**: Satellite detail panel shows TLE age with colour coding (green <3 days, yellow 3-7 days, red >7 days).
- **TLE age graph**: Clickable fleet-wide TLE age distribution histogram with oldest TLE list.

### Changed
- **GEO Sats button removed** (experimental): Removed the `#toggle-geo` button from the controls bar. GEO satellite visibility is now exclusively controlled via the Catalog panel's Eutelsat GEO row, reducing control clutter.
- **Compact toggle buttons**: Stars & SAA (experimental) / GEO & SAA (index) paired side-by-side in a half-width row, freeing vertical space in the controls bar.
- **TT&C panel z-index**: Lowered from 10 to 8 so TT&C countdown panels render behind the Catalog panel when overlapping.
- **Orbital plot UTC parsing**: Start/end time inputs now parsed as UTC via `inputToUTC()` helper.
- **Oldest TLE list scrollbar**: Custom webkit scrollbar styling (4px, transparent track).

### Technical Details
- `_constellationTrails` object stores per-group trail state: `LineSegments`, geometry, material, enabled/dirty/computing flags, and recompute timestamps.
- Trail computation uses SGP4 propagation with `satellite.propagate()`, supporting both ECEF and ECI reference frames.
- Buffer reuse: `buildTrailLineSegments()` reuses existing Float32Array if large enough, only reallocates when buffer is too small.
- Integration points: `toggleCatalogGroup()`, `_toggleBuiltinOneweb()`, `_toggleBuiltinGeo()`, `setReferenceFrame()`, and "All Off" handler all sync trail visibility and dirty state.
- Background preload uses sequential chained loading with `loadNext()` callback pattern, 2 concurrent streams.

---

## v15 — TLE Age Graph & Oldest TLE List (2026-02-28)
**Backup:** `index_v15_tlegraph.html`

### Added
- **TLE age distribution graph**: Clickable TLE age stat opens a fleet-wide histogram popup showing the distribution of TLE ages across all OneWeb satellites.
- **Oldest TLE list**: Scrollable list of the satellites with the oldest TLEs, sorted by age descending.

---

## v14 — Plane Folders, Formation Badges & Alarm Reliability (2026-02-28)
**Backup:** `index_v14_planes_alarms.html`

### Changed
- **Plane sub-folders**: Replaced SUPERSMASH / MONSTER MASH sub-folders inside ONEWEB with Plane 1–12 (and Unassigned) sub-folders. Satellites are grouped by orbital plane; all plane folders start collapsed.
- **Formation badges**: Formation status now shown as inline badges on satellite list items — green "SS" for SUPERSMASH, purple "MM" for MONSTER MASH — instead of moving satellites between folders.
- **Formation field in detail panel**: New "Formation" field between Plane and Follow button shows SUPERSMASH (green) or MONSTER MASH (purple) for satellites in a formation.
- **Formation search result count**: Searching "supersmash" or "monster mash" now shows a count header (e.g. "SUPERSMASH — 14 satellites").
- **Follow button full-width**: Follow button now spans both columns of the detail grid.
- **Plane renumbering**: Plane classification now uses reference satellites (0102→9, 0027→10, 0015→11, 0017→12) to assign correct plane numbers; remaining planes numbered 1–8 in RAAN order.

### Fixed
- **Alarms now use real wall-clock time**: Both TT&C and manual alarms compare against `Date.now()` instead of simulation time, so alarms always fire at the correct real time regardless of sim speed or time jumps.
- **Alarms independent of UI state**: `checkAllTtcAlarms()` now runs independently every 30 frames, no longer gated by detail panel visibility or satellite selection.
- **Removed `_liveMode` guard on alarms**: Speed slider changes and time jumps no longer disable alarm checking.
- **`_ttcAlarms` cleared on TLE reload**: `restoreAlarmsFromStorage()` now clears the alarm map before restoring, preventing stale satellite indices after array re-sorting.

---

## v13 — Orbital Element Plot Popup & GEO Camera Restore (2026-02-28)
**Backup:** `index_v13_plot.html`

### Added
- **Orbital element plot popup**: New chart button in the satellite detail panel header opens a resizable browser popup window for plotting orbital elements over time using Plotly.js.
  - **9 plottable elements**: Latitude, Longitude, Altitude, Velocity, Inclination, Eccentricity, RAAN, Argument of Latitude, Semi-major Axis — each with a distinct colour and toggleable checkbox.
  - **Satellite dropdown**: Switch between any satellite without closing the popup. Time range auto-adjusts to ±1 orbital period.
  - **Time range controls**: Start/end datetime-local inputs with Update button. Default range centres on current simulation time.
  - **Multi-axis plotting**: Y-axes grouped by unit (°, km, km/s, dimensionless) with left/right placement.
  - **Export PNG**: Downloads chart as 1200x600 PNG via Plotly.
  - **Export CSV**: Downloads all propagated data (all 9 elements) as a CSV file.
  - **Dark theme**: Matches the main app's colour scheme (#0a0e14 background, cyan/orange/green accents).
  - **Fully self-contained**: Popup loads its own satellite.js and Plotly.js from CDN. Satellite data (satrec objects) injected onto the popup window. No `window.opener` dependency — works reliably from `file://` protocol.
- **GEO toggle camera restore**: Toggling GEO satellites off now animates the camera back to the position it was in before the GEO zoom-out. Previously the camera stayed at the wide GEO viewing angle.

### Technical Details
- Popup uses `function.toString()` injection pattern: `_popupAppMain()` defined in the main module, serialised to string, injected as a `<script>` element in the popup. This ensures all code (Plotly.react, satellite.propagate, DOM manipulation) executes in the popup's own window context, avoiding Plotly.js cross-context rendering issues ([plotly/plotly.js#702](https://github.com/plotly/plotly.js/issues/702)).
- CDN script loading chain: satellite.js → Plotly.js → app script, with `onerror` handlers for user feedback on load failure.
- Osculating orbital elements computed from ECI state vector at 30-second intervals (same math as `updateDetailPanel`).
- GEO camera state saved in `_geoPreViewPos` / `_geoPreViewTarget` before zoom-out, restored via `animateCameraToVizView()` on toggle off.

---

## v12 — Multi-Satellite TT&C Alarms, GEO Info Panel & Search (2026-02-28)
**Backup:** `index_v12_alarms_geo.html`

### Added
- **Multi-satellite TT&C alarms**: Replaced single-boolean alarm with a `Map`-based system supporting simultaneous alarms on multiple satellites. Alarms persist across satellite deselection.
- **Alarm management panel**: Integrated into the header stats bar (replaces Planes count). Shows alarm count with gold accent when active. Click to open panel listing all alarmed satellites with remove buttons.
- **Alarm toast notification**: Top-center popup when a satellite is within 60s of a TT&C pass. Requires manual dismissal (Dismiss button). Continuous looping alarm sound until dismissed.
- **Per-station per-pass firing**: Alarms fire independently for Svalbard and Inuvik passes using composite `stationIdx:aosMs` keys in a Set.
- **localStorage alarm persistence**: Alarms saved as NORAD IDs to localStorage, restored on page load. Survives browser refreshes.
- **SVG alarm icon**: Custom line-drawing alarm clock in the detail panel header (grey when off, orange when active).
- **GEO satellite info panel**: Click any GEO satellite dot or label to see a gold-accented info panel showing satellite name, orbital slot, services, and description. Comprehensive `GEO_SAT_INFO` lookup covering all Eutelsat positions.
- **GEO search integration**: Type "GEO" or "Eutelsat" in the search bar to list all GEO satellites. Selecting one zooms to it, opens the info panel, and auto-enables GEO visibility.
- **Clickable GEO labels**: GEO satellite names on the globe are now clickable (opens info panel without adjusting view).
- **GEO selection highlight**: Selected GEO satellite name turns white and bold, dot enlarges with a white highlight sprite. Cleared on deselection.
- **GEO toggle camera view**: Toggling GEO satellites on animates to a wide view showing the full geostationary ring.
- **Orbital parameter follow-mode exit**: Clicking any orbital parameter visualization (Inc, RAAN, AoL, Ecc) now disables follow mode.

### Fixed
- **Duplicate alarm firing**: Rounded AOS timestamps to nearest 10 seconds in fired-key generation, preventing recomputed passes with slightly different timestamps from triggering duplicate alarms.

### Technical Details
- `_ttcAlarms` Map keyed by satellite index, each entry tracks `firedForAos` Set, cached `passes` array, and `lastPredictTime`.
- Pass predictions cached per alarmed satellite, recomputed only on >60s sim-time drift.
- Alarm sound: Web Audio API triple-tone chime (660/880/1100 Hz), looping every 2.5s via `setInterval`.
- GEO raycasting: Separate hit detection for GEO point cloud in `onMouseClick` and `processMouseHover`, with `isGeoSatVisible()` occlusion check.
- GEO list items appended asynchronously when GEO TLE data loads (separate from LEO `buildSatList`).

---

## v11 — Eutelsat GEO Satellite Fleet (2026-02-27)
**Backup:** `index_v11_geo.html`

### Added
- **Eutelsat GEO satellites**: Fetches TLE data from CelesTrak (`NAME=EUTELSAT`), filters to station-kept geostationary sats (inclination < 0.5°, mean motion 0.99–1.01 rev/day), and renders them as gold/amber point cloud at ~33 scene units (35,786 km altitude).
- **GEO toggle button**: New "GEO Sats" toggle at the top of the ground station controls panel. Gold accent (#ffc107), hollow-circle icon. Off by default.
- **GEO orbit ring**: Thin dashed gold circle in the equatorial plane at GEO altitude, providing a visual reference for the geostationary belt. Toggles with the GEO button.
- **GEO satellite labels**: Screen-space HTML labels for each Eutelsat satellite showing name and orbital slot (e.g. "HOTBIRD 13F · 13.0°E"). Gold text, behind-globe occlusion, same positioning pattern as ground station labels.
- **ECI/ECEF frame support**: GEO positions and labels update correctly in both reference frames. Positions force-refresh on frame switch.

### Technical Details
- CORS proxy fallback chain identical to OneWeb TLE fetch (corsproxy.io → codetabs.com).
- GEO position updates run every 30th frame at 1x speed (GEO sats barely move), scaling to every frame at >100x.
- Point size 0.4 (larger than OneWeb's 0.22) since GEO sats are further from camera at default zoom.
- Reuses existing allocation-free helpers (`geoToVec3Into`, `_geoOut`, `_camToStation`, etc.) for zero-GC label projection.

---

## v10 — Excluded Satellite Styling, Plane Selection & UX Polish (2026-02-26)
**Backup:** `index_v10_exclusions.html`

### Added
- **Excluded satellite red styling**: The 3 hardcoded exclusions (0013, 0050, 0618) now render as red dots instead of blue, with red hover colour and red tracking halo when selected.
- **Neutral point texture**: Satellite point cloud texture changed from cyan-tinted to white/neutral gradient so vertex colours control hue entirely (enables red excluded sats).
- **Hover expand sprite**: Hovering over a satellite shows a larger glowing sprite at its position for better visibility. Skipped when the satellite is already selected.
- **Unassigned satellite search**: Typing "unassigned" in the search box shows a red-dotted "Unassigned" group item plus all plane 0 satellites. Clicking it highlights them with glow (no disc or camera pan since they're not in a coherent plane).
- **Plane selection from detail panel**: Clicking the Plane field in the satellite info box selects that plane.
- **Plane disc overlay**: Semi-transparent cyan disc rendered in the orbital plane when a plane is selected (6% opacity).

### Changed
- **Satellite list sort order**: Satellites now sorted by OneWeb number (extracted from name) instead of NORAD ID.
- **Eccentricity info popup**: Moved from inline in the detail panel to a separate floating popup window.
- **Camera minimum distance**: `controls.minDistance` set to `EARTH_RADIUS + 0.5` to prevent camera going inside Earth.
- **Follow camera distance preservation**: Re-normalises camera position after lerp to prevent zooming in at high speeds.
- **AoL camera angle**: Now matches the inclination camera's tilted perspective exactly.
- **Plane camera**: Uses inclination-style tilt, picks whichever side is closer to current camera, always looks slightly down.
- **RAAN camera**: Views from 89° latitude (nearly directly above) instead of 75°.
- **Orbital viz toggle**: No more view jolt when rapidly toggling on/off — `cameraAnim` cancelled on clear, frame restore skipped during immediate re-open.
- **`clearOrbitalViz()` called on plane select**: Any active orbital visualisation is cleared when selecting a plane.

### Fixed
- **Inclination camera side**: Always positions toward the arc's concave side (vizLabelAnchor direction) instead of sometimes flipping to the wrong side.
- **`anPoint` redeclaration**: Fixed duplicate `const anPoint` in `showAolViz` that caused SyntaxError.

---

## v9 — Orbital Visualisations, Osculating Elements & Accuracy Fixes (2026-02-26)
**Backup:** `index_v9_vizfields.html`

### Added
- **Argument of Latitude (AoL) visualisation**: Clickable field in detail panel opens live-updating 3D arc from ascending node to satellite position in the orbital plane. Label follows satellite in real time.
- **Eccentricity visualisation**: Clickable field shows elliptical orbit with min/max altitude markers, circular reference orbit, and Earth focus point. Markers placed using SGP4 + WGS84 geodetic sampling (360 points) so they match the live altitude readout exactly.
- **Live-updating eccentricity labels**: Min/max altitude tags re-sample the orbit every 0.5s to stay in sync with the evolving osculating orbit.
- **Mobile landscape CSS**: `@media (max-height: 440px) and (orientation: landscape)` with 220px side panels, compact header, slim clock bar.
- **Dismissible detail panel**: Close button hides the panel without deselecting the satellite — orbit path and tracking persist. Re-clicking the same satellite re-shows the panel.
- **Selected satellite real-time propagation**: At 1x speed, the selected satellite is propagated every frame (not every 4th) for monitoring-grade accuracy.

### Changed
- **All orbital readouts use osculating elements**: Inclination, eccentricity, RAAN, AoL, and period are computed from the ECI state vector (position + velocity) each frame, matching the SGP4-propagated orbit exactly.
- **Perigee/apogee markers**: Placed at true min/max geodetic altitude positions (accounting for WGS84 oblateness), not Keplerian perigee/apogee. Labels read "MIN ALT" / "MAX ALT".
- **Detail panel field order**: RAAN and AoL moved above Period and NORAD ID.
- **Altitude field**: Removed blue accent colour.
- **dt capped at 0.1s**: Prevents massive time jumps when returning from a background tab.

### Fixed
- **`periGeo` redeclaration**: Variable name collision between Three.js SphereGeometry and geodetic conversion caused a SyntaxError preventing the app from loading.
- **Inclination camera angle**: Now views the inclination arc head-on (was 90° off).
- **ECI frame guard**: Toggling away from ECI auto-closes inclination/RAAN/AoL visualisations that require the inertial frame.

---

## v8 — South Atlantic Anomaly Overlay & Pan Controls (2026-02-26)
**Backup:** `index_v8_saa.html`

### Added
- **SAA zone overlay**: Semi-transparent radiation heatmap over the South Atlantic Anomaly region, rendered as a subdivided mesh on the Earth's surface with a custom `ShaderMaterial`. Gradient runs red (peak radiation interior) → yellow → green (outer boundary), computed per-pixel from distance to nearest boundary point. Boundary smoothed via Catmull-Rom spline interpolation (18 control points → 144 smooth vertices).
- **SAA toggle button**: New "SAA Zone" toggle in the ground station controls panel, following the TT&C/Gateway button pattern. Red accent colour (#ff5252) with hazard connotation. Off by default on load.
- **Camera panning**: Enabled `OrbitControls.enablePan` — right-click drag (or two-finger trackpad) now pans the view off-center.

### Technical Details
- SAA boundary: 18 lat/lon control points tracing the ~10² protons/cm²/s flux contour at LEO altitude, centered ~27°S 49°W.
- Mesh: triangle fan from centroid, recursively subdivided 3 levels (64 sub-triangles per fan segment), all vertices projected onto sphere at `EARTH_RADIUS * 1.005`.
- Per-vertex `aT` attribute: angular distance to nearest of 144 boundary points, normalized 0 (at edge) to 1 (deepest interior). Drives the fragment shader colour/opacity gradient.
- Boundary outline: green `THREE.Line` from spline-smoothed points.
- Attached to `earthGroup` — automatically rotates with Earth in ECI mode.

---

## v7 — Performance Optimisation & Behind-Earth Occlusion (2026-02-26)
**Backup:** `index_v7_performance.html`

### Fixed
- **Per-frame object allocations**: Eliminated ~680+ `new THREE.Vector3` / `Raycaster` / `Quaternion` allocations per frame by promoting to reusable module-level objects (`_satPos`, `_velDir`, `_camToStation`, `_earthSphere`, etc.).
- **Mousemove raycasting**: Raycasts now run once per frame via a dirty flag instead of on every mousemove event (was 100+/sec during mouse movement).
- **Orbit path material leak**: `disposeLine()` now disposes both geometry and material.
- **Duplicate propagations**: `updateDetailPanel()` reuses cached `sat.eciX/Y/Z` instead of re-propagating. Tracking arrow uses position deltas (`sat._prevX/Y/Z`) instead of propagating a future step.
- **Ground station label jitter**: Style writes skipped when position moves less than 0.5px; world positions cached (stations don't move).
- **Tooltip DOM thrashing**: `innerHTML` only rewritten when content changes; positioning uses GPU-composited `transform` instead of `left`/`top`.
- **Detail panel DOM lookups**: All 12 `getElementById()` calls replaced with pre-cached `$detail*` references.

### Added
- **Behind-Earth occlusion**: Satellites on the far side of the globe can no longer be hovered or clicked. `isSatVisible()` tests ray-sphere intersection before accepting a hit. Both hover and click iterate through raycast results to find the nearest visible satellite.
- **Allocation-free `geoToVec3Into()`**: Writes into a reusable Vector3 for the hot `updateSatellitePositions()` loop (~600 sats).

### Changed
- **Adaptive satellite propagation**: Frequency scales with time multiplier — every 4th frame at 1x (~15/sec), every 2nd at 10–100x, every frame at >100x.
- **Adaptive orbit path resolution**: Step count reduced at >50x speed (400→150 tracking, 200→80 selection). Minimum rebuild interval raised to 3 frames.
- **`computeSunDirection` / `computeMoonDirection`**: Return reusable Vector3 instead of allocating new ones.

---

## v6 — Day/Night Brightness Tuning & Data Source Indicator (2026-02-26)
**Backup:** `index_v6_lighting2.html`

### Fixed
- **Day-side brightness**: Raised ambient floor from 0.3x to 0.8x with gentler 0.6x diffuse ramp (peaks 1.4x). Oceans and land now always appear brighter during the day than at night, even at glancing sun angles near the terminator.
- **Night-side darkness**: Reduced base from 0.12x to 0.06x and city lights from 1.8x to 1.5x so dark side is clearly darker than day.
- **Alternate data source**: Replaced broken `allorigins.win` CORS proxy with `corsproxy.io` and `codetabs.com` fallback chain.

### Added
- **Data source indicator**: Header stats show "Primary" (green) when loaded directly from CelesTrak, or "Secondary" (orange) when a CORS proxy fallback was used.

---

## v5 — Lighting Fix & Plane Exclusions (2026-02-26)
**Backup:** `index_v5_lighting.html`

### Fixed
- **Day/night shader logic**: Corrected `mix()` argument order — `mix(night, day, dayBlend)` so `dayBlend=1` (sunlit) shows bright blue-marble and `dayBlend=0` (shadow) shows city lights. Previously swapped.
- **Day diffuse direction**: Uses `max(NdotL, 0.0)` (positive on sunlit side) instead of negated value.
- **Atmosphere glow**: Removed incorrect negation — `dot(pos, sunDir)` now correctly brightens the sunlit limb.

### Changed
- **Plane exclusion list**: Added `PLANE_EXCLUDE` array (`0013`, `0050`, `0618`) — these satellites are excluded from orbital plane classification (assigned plane 0).

---

## v4 — UI Refinements: Speed Slider, Clock Format, Bottom Actions (2026-02-26)
**Backup:** `index_v4_ui.html`

### Added
- **Speed slider**: Replaced 4 speed buttons (1x/10x/60x/5m) with an exponential slider ranging from 1x to 1200x, with a live speed label.
- **Bottom actions bar**: LIVE button moved from header stats to a shared bottom-center container alongside Reset View.

### Changed
- **Clock format**: Now shows full ISO date+time `2025-02-26  05:05:12` with UTC label centered below.
- **Day brightness boost**: Sunlit side of Earth now renders up to 1.5x texture brightness for clearer day/night contrast.
- **Reset View position**: Moved up slightly (bottom: 84px) to give more clearance from the clock bar.
- **UTC label**: Centered under the time display with 4px top margin.

---

## v3 — Realistic Day/Night Earth Rendering (2026-02-26)
**Backup:** `index_v3_daynight.html`

### Added
- **Day/night shader**: Custom `ShaderMaterial` blends a blue-marble day texture (sunlit side) with city-lights night texture (dark side) using `smoothstep` for a sharp, visible terminator.
- **Solar ephemeris (`computeSunDirection`)**: Simplified sun position from Julian date → ecliptic longitude → RA/dec, combined with `satellite.gstime()` for Earth rotation. Sun direction updates dynamically with simulation time.
- **Dynamic sun position**: `DirectionalLight` now tracks the computed sun direction instead of being fixed. Updates every frame at high time multipliers (>10x), every 10th frame at 1x.
- **Atmosphere sun modulation**: Atmosphere glow shader now brightens on the sunlit limb and dims on the dark side via `smoothstep` blend.
- **Sun toggle integration**: Toggling sun off switches Earth to flat-lit day texture (no night lights) and disables atmosphere sun modulation.

### Changed
- **Earth textures**: Now loads two textures in parallel (blue-marble day + city-lights night) instead of using the night texture for both colour and emissive maps.
- **Time speed labels**: 300x button labelled "5m" (5 minutes per second).

---

## v2 — Orbital Plane Classification & Selection (2026-02-26)
**Backup:** `index_v2_planes.html`

### Added
- **Plane classification**: Satellites classified into 12 orbital planes (Walker-Star, 15° RAAN spacing) after TLE parsing. Non-nominal satellites (wrong inclination/altitude) left unassigned (plane 0).
- **Plane search**: Typing "plane" in the search box shows all 12 plane group items. Typing "plane 7" shows the Plane 7 item plus all satellites in that plane.
- **Plane selection**: Clicking a plane item in the search results highlights all satellites in that plane with a bright cyan colour and big blue glow overlay. Other satellites dim to near-invisible.
- **Camera pan to plane (edge-on view)**: Selecting a plane auto-pans the camera to the equator, perpendicular to the orbital plane normal — so the plane's satellites appear as a vertical line seen from the side of the Earth. Picks the closest of the two possible viewing directions.
- **Plane metadata in sat list**: Each satellite's list entry now shows its plane number (e.g. `Plane 7`) in the metadata line.
- **`panToPlane()`**: Computes orbital plane normal via cross-products of satellite positions, projects onto equatorial plane, and animates camera to the edge-on viewpoint.
- **`showPlaneGlow()` / `removePlaneGlow()`**: Overlay point cloud with large soft blue glow for visually prominent plane highlighting.
- **Rotate buttons**: &minus;90° / +90° buttons appear flanking the clock when a plane is selected. Smoothly rotates the camera around the globe to view the plane from different angles.

### Changed
- **Ground station label z-index**: Lowered from 15 to 9 so station labels render behind the bottom control buttons when they overlap.
- **`restoreAllSatColors()` / `resetSatColor()`**: Now respect active plane selection (bright cyan for plane sats, deep dim for others).
- **`selectSatellite()`**: Clears any active plane selection and removes glow overlay.
- **`resetView()`**: Clears plane selection and glow on reset.

---

## v1 — Initial Release
**File:** `index.html` (original)

- 3D globe with OneWeb constellation rendered as point cloud
- Real-time TLE data from CelesTrak
- Satellite search by name / NORAD ID
- Satellite selection with orbit path, ground track, detail panel
- Camera tracking and follow mode
- TT&C and Gateway ground station markers with toggle buttons
- Sun illumination toggle
- Time controls (1x/10x/100x) and live mode
- Side panel with satellite list, collapsible
