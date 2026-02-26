# OneWeb Constellation Tracker — Changelog

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
