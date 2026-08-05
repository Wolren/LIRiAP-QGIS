<div align="center">

# LIRiAP

QGIS processing algorithms for computing the largest inscribed rectangle in arbitrary polygons.

[![Wiki](https://img.shields.io/badge/Documentation-Wiki-blue)](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki)
[![License: GPL v3](https://img.shields.io/github/license/GIS-Inscribed-Geometry/LIRiAP-QGIS)](https://www.gnu.org/licenses/gpl-3.0)
[![Last commit](https://img.shields.io/github/last-commit/GIS-Inscribed-Geometry/LIRiAP-QGIS)](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/commits/master)
[![Issues](https://img.shields.io/github/issues/GIS-Inscribed-Geometry/LIRiAP-QGIS)](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/issues)
[![Code size](https://img.shields.io/github/languages/code-size/GIS-Inscribed-Geometry/LIRiAP-QGIS)]()
[![Python](https://img.shields.io/badge/Python-3.9+-blue?logo=python)](requirements.txt)
[![QGIS](https://img.shields.io/badge/QGIS-3.16+-green?logo=qgis)](https://www.qgis.org/)
[![Release](https://img.shields.io/github/v/release/GIS-Inscribed-Geometry/LIRiAP-QGIS)](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/releases/latest)

</div>

## Problem statement

Given an input polygon, find a large interior rectangle that may be rotated (concave polygons and polygons with holes supported). The pack implements four algorithm families:

1. **Approximation family**: maximize area quickly, without strict containment certification. Good for finding candidates.
2. **Skeleton family**: BCRS-free solver using medial-axis skeleton decomposition for seed generation, with SDF-guided expansion and containment certification.
3. **BCRS family**: containment certification plus SDF-guided boundary expansion. The most accurate solver, especially with `top_k=3`. The `top_k=1` variant offers a speed-accuracy trade-off that rivals approximation speed while maintaining certified containment.
4. **Axis-Aligned family**: exact fixed-axis solve with vertex-coordinate precision.

## Examples

![Axis-aligned](media/Axis-aligned.png)

### Approximation (less vs denser grid)

![Approximation result](media/Approximate.png)

![Approximation (improved candidate)](media/Approximate_better.png)

### BCRS (Boundary-Coordinate Raster Solve with SDF expansion)

![BCRS result](media/BCRS.png)

![BCRS result (zoom)](media/BCRS_zoom.png)

---

## Potential uses

- **Suitability analysis**: search candidate locations for building or infrastructure placement by finding the largest feasible rectangular footprint inside constrained parcels (e.g., houses, warehouses, solar arrays, staging pads, retention structures) while respecting parcel boundaries and holes/exclusions.
- **Remote sensing**: derive stable interior rectangular patches for spectral sampling, calibration windows, texture statistics, and object-level summaries where centroid or full-polygon sampling is noisy.
- **Dynamic cartographic label placement**: place labels in the largest interior rectangle instead of using only centroid or bounding box, improving readability in concave polygons and polygons with holes. An axis-aligned version could be fast enough to handle this task.
- **Other scenarios**: map tiling anchors, drone landing-zone preselection, interior ROI extraction for QA workflows, and standardized shape descriptors for downstream analytics.
- **Computer vision**: find maximum rectangular regions of interest within arbitrary shaped detection masks.
- **Game development**: calculate valid placement areas for rectangular game objects within complex terrain polygons.

The fewer features a layer has, the denser the grid can be while maintaining reasonable accuracy.

### Potential for other algorithms

The ideas in this pack could also produce solutions for other contained shapes, and the reverse problem: finding positions for inscribed polygons in a rectangle in a way that maximizes used space.

## At a glance

The `top_k` parameter acts as a speed / accuracy slider:
- `top_k=3` (default): explores multiple candidates for maximum accuracy.
- `top_k=1`: skips candidate ranking and uses the single best angle. 2-3x faster with modest accuracy loss (~1-2% fill rate drop).

For BCRS on large datasets, use `top_k=1` with `N_WORKERS=1`: this gives the best speed / accuracy ratio. BCRS Fast with `top_k=1` rivals approximation speed while returning certified contained results.

| Family        | Primary objective                            | Strict containment               | Boundary expansion |
| ------------- | -------------------------------------------- | -------------------------------- | ------------------ |
| Approximation | Fast area-focused search                     | No                               | No                 |
| Skeleton      | BCRS-free skeleton-guided solver             | Yes (unless fallback is enabled) | Yes (SDF)          |
| BCRS          | Certified contained search + fit improvement | Yes (unless fallback is enabled) | Yes (SDF)          |
| Axis-Aligned  | Exact fixed-axis solve                       | Yes (vertex-coordinate)          | N/A                |

Best execution mode by algorithm (290 and 5406 are the feature counts of the two benchmark datasets):

| Algorithm                     | Best mode @290 | Best mode @5406 |
| ----------------------------- | -------------- | --------------- |
| Approximation Standard        | 12w            | 12w+chunk       |
| Approximation Fast            | 12w            | 12w+chunk       |
| Skeleton                      | 1w             | 1w              |
| BCRS                          | 1w             | 1w              |
| BCRS Fast                     | 1w             | 1w              |
| Axis-Aligned LIR              | 1w             | 1w              |

## Shared components

All algorithms in `LIRiAP_pack` follow the same structure:

1. **Input normalization**: read polygon geometry; for multipolygons, use the largest part.
2. **Angle candidates**: extract likely orientations from polygon edge directions, with a fallback sweep.
3. **Rectangle solve in rotated frame**: solve axis-aligned rectangle candidates on a rotated polygon to recover rotated solutions in map coordinates.
4. **Refinement and checks**: apply finer search and containment-related adjustments (depending on variant).
5. **Output**: write rectangle geometry and metrics (area, angle, ratio, and variant-specific diagnostics).

## Algorithms

| Algorithm                                               | What problem it solves                                       | Containment semantics                                                                                   | Expansion semantics                                                |
| ------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| Approximation Standard                                  | Fast area-focused approximation                              | Not certified; rectangle can violate containment in difficult cases                                     | No expansion stage                                                 |
| Approximation Fast                                      | Same as Approximation Standard with lower overhead execution | Not certified; same semantics as Standard                                                               | No expansion stage                                                 |
| Skeleton                                                | BCRS-free solver using medial-axis skeleton decomposition    | Certified contained when strict mode succeeds; optional best-effort fallback                            | SDF-guided boundary expansion                                      |
| BCRS (Boundary-Coordinate Raster Solve)                 | Full contained-plus-expansion solve                          | Certified contained when strict mode succeeds; optional best-effort fallback                            | SDF-guided boundary expansion                                      |
| BCRS Fast (Boundary-Coordinate Raster Solve, optimized) | Same as BCRS with prioritized execution                      | Same certified / best-effort semantics as BCRS                                                          | SDF-guided boundary expansion                                      |
| Axis-Aligned LIR                                        | Exact fixed-axis solve                                       | Exact (vertex-coordinate precision)                                                                     | N/A                                                                |

## Setting semantics

- `ALWAYS_RETURN` (BCRS / Skeleton):
  - `False`: strict certification only; features may return no rectangle if strict containment cannot be certified.
  - `True`: returns best-effort fallback when strict certification fails (`best_effort=1`), so the strict guarantee is no longer universal.
- `USE_BUFFER` + `BUFFER_VALUE` (BCRS / Skeleton): applies an additional containment margin in map units (usually reducing area to increase margin from boundaries/holes).
- `MAX_RATIO`: constrains the admissible rectangle aspect ratio; a tighter cap can reduce max area.
- `TOP_K`: number of candidate angles to refine. Higher is more accurate, lower is faster.
  - `top_k=1` gives 2-3x speed with ~1-2% fill rate loss.
  - `top_k=3` (default) explores multiple candidates for maximum fill rate.
- `GRID_*`, `ANGLE_STEP`: search density controls; change the result quality / runtime trade-off but not solver family semantics.
- `N_WORKERS`, `USE_CHUNKING`, `AUTO_INSTALL_NUMBA`: runtime / performance controls only; they do not change geometric guarantees.

## Processing benchmark (default settings)

All runs assume default algorithm parameters and Numba installed. 290 and 5406 are the feature counts of the two benchmark datasets.

Benchmarked with:

- i5-12400F
- 32GB DDR4 RAM

### Baseline profile (N_WORKERS=1, USE_CHUNKING=False)

| Profile | Algorithm              | TOP_K | Time @ 290 (s) | Time @ 5406 (s) | Scale ratio |
| ------- | ---------------------- | ----- | -------------- | --------------- | ----------- |
| P1      | Approximation Standard | 1     | 7.13           | 127.25          | 17.85x      |
| P2      | Approximation Fast     | 1     | 6.98           | 125.93          | 18.04x      |
| P3      | Skeleton               | 3     | 24.64          | n/a             | n/a         |
| P4      | Skeleton               | 1     | 8.41           | n/a             | n/a         |
| P5      | BCRS                   | 3     | 26.33          | n/a             | n/a         |
| P6      | BCRS                   | 1     | 10.82          | n/a             | n/a         |
| P7      | BCRS Fast              | 3     | 15.54          | n/a             | n/a         |
| P8      | BCRS Fast              | 1     | 7.21           | n/a             | n/a         |
| P9      | Axis-Aligned LIR       | n/a   | 11.81          | 120.24          | 10.18x      |

**Speed/accuracy trade-off:** dropping from `top_k=3` to `top_k=1` gives 2-3x speed with ~1-2% fill rate loss, making BCRS Fast with `top_k=1` competitive with approximation times while maintaining certified containment.

### Parallel profile (N_WORKERS=12, USE_CHUNKING=False)

| Profile | Algorithm              | TOP_K | Time @ 290 (s) | Time @ 5406 (s) | Scale ratio |
| ------- | ---------------------- | ----- | -------------- | --------------- | ----------- |
| P1      | Approximation Standard | 1     | 5.97           | 112.30          | 18.81x      |
| P2      | Approximation Fast     | 1     | 5.90           | 108.43          | 18.38x      |
| P3      | BCRS                   | 3     | 29.38          | n/a             | n/a         |
| P4      | BCRS                   | 1     | 12.10          | n/a             | n/a         |
| P5      | BCRS Fast              | 3     | 19.25          | n/a             | n/a         |
| P6      | BCRS Fast              | 1     | 8.73           | n/a             | n/a         |
| P7      | Skeleton               | 3     | 33.02          | n/a             | n/a         |
| P8      | Skeleton               | 1     | 10.97          | n/a             | n/a         |
| P9      | Axis-Aligned LIR       | n/a   | 14.83          | 158.53          | 10.69x      |

> BCRS, BCRS Fast and Skeleton perform better serially: per-feature overhead exceeds the gain from multiple workers.

### Parallel + chunking profile (N_WORKERS=12, USE_CHUNKING=True)

| Profile | Algorithm              | TOP_K | Time @ 290 (s) | Time @ 5406 (s) | Scale ratio |
| ------- | ---------------------- | ----- | -------------- | --------------- | ----------- |
| P1      | Approximation Standard | 1     | 6.04           | 109.76          | 18.17x      |
| P2      | Approximation Fast     | 1     | 5.90           | 108.43          | 18.38x      |
| P6      | Axis-Aligned LIR       | n/a   | 14.91          | 157.89          | 10.59x      |

### Fill rate benchmark (290 features, default parameters)

Fill rate = rectangle area / polygon area x 100%. Higher is better.

| Algorithm | Mean% | Median% | Min% | Max% | Std% |
|-----------|-------|---------|------|------|------|
| Axis-Aligned LIR | 35.87 | 35.43 | 3.41 | 91.69 | 17.08 |
| Skeleton | 54.96 | 52.58 | 8.12 | 97.52 | 21.21 |
| BCRS Fast (top_k=3) | 55.27 | 53.24 | 7.62 | 97.52 | 20.09 |
| BCRS Fast (top_k=1) | ~53.5 | ~51.5 | ~7.5 | ~97.5 | ~21.0 |
| BCRS (top_k=3) | 55.74 | 53.81 | 7.68 | 97.52 | 20.16 |
| BCRS (top_k=1) | ~54.0 | ~52.0 | ~7.5 | ~97.5 | ~21.0 |

**Key findings:**

- Skeleton and BCRS families achieve ~55% fill rate vs ~36% for axis-aligned.
- BCRS marginally leads in mean/median (+0.5-1% over Skeleton).
- All methods reach similar max fill (~97.5%), indicating a ceiling on difficult polygons.
- Dropping from `top_k=3` to `top_k=1` reduces fill rate ~1.5% but cuts runtime 2-3x.

## Installation & Usage

### Option 1: As Script Folder (Quick Testing)

1. Copy the `LIRiAP_pack` folder to your QGIS script folder:

   - Windows: `C:\Users\<username>\AppData\Roaming\QGIS\QGIS3\profiles\default\processing\scripts\`
   - Linux: `~/.local/share/QGIS/QGIS3/profiles/default/processing/scripts/`
   - macOS: `~/Library/Application Support/QGIS/QGIS3/profiles/default/processing/scripts/`
2. Open QGIS.
3. Open the Processing Toolbox (`Processing` -> `Toolbox`).
4. Search for "LIRiAP": the algorithms appear under "Scripts" -> "LIRiAP".

### Option 2: As a Plugin Provider (Recommended for Regular Use)

1. Copy the entire repository (or create a symlink) to your QGIS plugins folder:

   - Windows: `C:\Users\<username>\AppData\Roaming\QGIS\QGIS3\profiles\default\python\plugins\LIRiAP\`
   - Linux: `~/.local/share/QGIS/QGIS3/profiles/default/python/plugins/LIRiAP/`
   - macOS: `~/Library/Application Support/QGIS/QGIS3/profiles/default/python/plugins/LIRiAP/`
2. Ensure the folder contains:

   - `LiRiAP_provider/` (the QGIS plugin)
   - `LIRiAP_pack/` (the algorithm pack)
3. Open QGIS.
4. Go to `Plugins` -> `Manage and Install Plugins`.
5. Enable "LIRiAP" (it should appear in the list).
6. Algorithms appear in Processing Toolbox under the "LIRiAP" group.

### Running an Algorithm

1. Open Processing Toolbox (`Processing` -> `Toolbox`).
2. Navigate to **LIRiAP** (or search for the specific algorithm name).
3. Double-click an algorithm (e.g., "Approximation Standard").
4. Select:

   - **Input layer**: your polygon layer.
   - Adjust parameters as needed (grid resolution, angle step, etc.).
5. Click **Run**.

### Dependencies

- **Required**: NumPy, SciPy, Shapely.
- **Optional**: Numba (for JIT acceleration: significantly speeds up computations).

Numba will be auto-installed if `AUTO_INSTALL_NUMBA` is enabled, or install manually:

```
pip install numba
```

## Tech stack

| Component | Role |
| --------- | ---- |
| QGIS (3.16-4.99) | Processing framework |
| Python 3.9+ | Implementation language |
| NumPy | Array math and grid operations |
| SciPy | SLSQP optimization for boundary fitting |
| Shapely | Geometry representation |
| Numba | Optional JIT acceleration |

## Files

| File | Purpose |
| ---- | ------- |
| `LIRiAP_pack/*_algorithm.py` | QGIS Processing wrappers (parameters, execution, output fields, help text) |
| `LIRiAP_pack/*_worker.py` | Geometry solvers independent of the QGIS/Qt runtime |
| `LIRiAP_pack/sdf_oracle.py` | SDF oracle used by BCRS and Skeleton expansion |
| `LIRiAP_pack/event_emitter.py` | Progress / event emission helpers |
| `LIRiAP_pack/numba_bootstrap.py` | Optional Numba bootstrap helper |
| `LIRiAP_pack/help_descriptions.py` | Shared right-panel algorithm descriptions |
| `tests/*.py` | Unit tests for bootstrap safety, edge cases, and tuning-constant guardrails |
| `LiRiAP_provider/` | QGIS plugin provider wrapping the pack |
| `visualisation/` | Visualization helpers for algorithm output |
| `sync_to_provider.sh` / `sync_to_provider.ps1` | Packaging sync scripts |

## Limitations

- **Approximation family is not containment-certified**: rectangles can violate the polygon boundary in difficult cases. Use the Skeleton or BCRS families when certified containment matters.
- **Multipolygon handling**: for multipolygon inputs, only the largest part is processed.
- **Strict-mode emptiness**: with `ALWAYS_RETURN=False`, BCRS and Skeleton may return no rectangle for features where strict containment cannot be certified. `ALWAYS_RETURN=True` (default) relaxes this with a best-effort fallback.
- **Grid resolution bounds the search**: `GRID_*` and `ANGLE_STEP` set the search density; results are approximate except for the Axis-Aligned family, whose exactness is relative to the input vertex coordinates.

## Documentation

Detailed documentation is available in the [GitHub Wiki](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki):

- [Home](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Home) - Overview and quick start
- [Algorithms](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Algorithms) - Family comparison
- [Approximation](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Approximation) - Approximation algorithm details
- [Skeleton](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Skeleton) - Skeleton algorithm details
- [BCRS](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/BCRS) - BCRS algorithm details (SDF expansion)
- [Axis-Aligned](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Axis-Aligned) - Exact axis-aligned solver
- [Complexity](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Complexity) - Formal complexity analysis
- [Foundations](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Foundations) - Geometric background
- [Parameters](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Parameters) - Full parameter reference
- [Folder Layout](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Folder-Layout) - Code structure
- [Usage](https://github.com/GIS-Inscribed-Geometry/LIRiAP-QGIS/wiki/Usage) - Programmatic API usage

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening issues or pull requests.

## License

Distributed under the [GNU General Public License v3.0](LICENSE).
