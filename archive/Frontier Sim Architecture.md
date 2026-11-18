# Frontier Farming Simulator — Technical Architecture Specification

## 1. World Scale & Spatial Geometry
* **Total Footprint:** 500 Acres ($\approx 4,667\text{ ft} \times 4,667\text{ ft}$).
* **Voxel Resolution:** $6\text{"} \times 6\text{"} \times 6\text{"}$ cubic voxels ($9,334 \times 9,334 \approx 87.1\text{ million}$ columns).
* **Vertical Relief:** $256\text{ ft}$ standard relief ($512$ Z-levels) supported via per-column `baseElevation` offsets (`u16` in $6\text{"}$ units, expandable up to $32,768\text{ ft}$).
* **Sparse 3D Sub-Chunks:**
  * World partitioned into $16 \times 16$ column chunks ($8\text{ ft} \times 8\text{ ft}$ footprints) and $16 \times 16 \times 16$ vertical sub-chunks ($8\text{ ft}$ cubes).
  * Uniform bedrock below grade and uniform open sky above grade collapse into constant single-ID compressed blocks until modified by digging or construction.

---

## 2. Core Voxel Memory Layout (10-Byte / 80-Bit Option B)
Each active $6\text{"}$ voxel is packed into `80 bits` ($10\text{ bytes}$) to maintain CPU cache locality while preserving deep agricultural chemistry and hydrological state:

| Bit Range | Width | Field Name | Range / Values | Simulation & Rendering Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `0..2` | 3 bits | `matterState` | `0..7` | `0`=Air, `1`=Soil, `2`=Bedrock/Stone, `3`=Built Structure, `4`=Plant Mass, `5`=Loose Item, `6`=Snow/Ice, `7`=Liquid-Only |
| `3..7` | 5 bits | `materialId` | `0..31` | Subtype ID (e.g., `topsoil`, `subsoil`, `peat`, `granite`, `limestone`, `oak_timber`, `sod_brick`) |
| `8..12` | 5 bits | `clayPct` | `0..20` ($5\%$ steps) | Controls soil cohesion, water retention, plastic lateral bulge, and red/ochre color shift |
| `13..17` | 5 bits | `sandPct` | `0..20` ($5\%$ steps) | Controls hydraulic conductivity ($\text{silt} = 20 - \text{clay} - \text{sand}$) and pale/tan tint |
| `18..21` | 4 bits | `organicMatter` | `0..15` | Humus density; darkens soil RGB toward rich black-brown and buffers fertility |
| `22..25` | 4 bits | `compaction` | `0..15` | Bulk density (`0`=freshly tilled, `15`=hardpan). Throttles infiltration and alters surface glyphs |
| `26..29` | 4 bits | `poreWater` | `0..15` | Absorbed capillary/pore water (`10`=field capacity, `15`=saturated/liquefaction risk) |
| `30..33` | 4 bits | `freeWater` | `0..15` | Unbound water in open space (surface puddles, overland runoff, creek flow, flooded cellars) |
| `34..37` | 4 bits | `fillHeight` | `0..15` | Sub-voxel vertical fill ($0.375\text{"}$ increments) for smooth slopes, furrows, wheel ruts, and erosion |
| `38..40` | 3 bits | `nitrogen` (N) | `0..7` | Available Nitrogen; depleted by heavy feeders (corn), fixed by legumes |
| `41..43` | 3 bits | `phosphorus` (P) | `0..7` | Available Phosphorus; critical for root development and seed/fruit yield |
| `44..46` | 3 bits | `potassium` (K) | `0..7` | Available Potassium; restored via wood ash, compost, and mineral weathering |
| `47..49` | 3 bits | `soilPh` | `0..7` | Maps $\text{pH } 4.5$ (acidic) to $\text{pH } 8.5$ (alkaline/saline); gates nutrient availability |
| `50..53` | 4 bits | `soilTemp` | `0..15` | Thermal mass lag underground; governs frost-line depth and spring germination |
| `54..69` | 16 bits | `entityRef` | `0..65,535` | Chunk-local sparse table index (`0`=none) linking to multi-voxel Plants, Structures, or Item Stacks |
| `70..75` | 6 bits | `flags` | 6 booleans | `isFrozen`, `hasRoots`, `isSliding`, `isLoadBearing`, `isExposedSurface`, `isDesignated` |
| `76..79` | 4 bits | `reserved` | `0..15` | Alignment padding / future moisture-vapor or salinity flags |

### Sparse Entity Side-Tables
* **Plants (`plantTable[entityRef]`):** Stores species, growth stage, exact hydration, root depth, and yield. Root voxels keep normal soil bits with `hasRoots = 1` (adding shear resistance against landslides); above-ground stem/canopy voxels set `matterState = 4` and point back to `entityRef`.
* **Structures (`structureTable[entityRef]`):** Stores building/object metadata, durability, and container/thermal state. Every occupied $6\text{"}$ voxel sets `matterState = 3`, `isLoadBearing = 1`, and applies downward weight onto the soil voxel below.

---

## 3. Physics & Hydrology Engine

### Dual-Channel Water Dynamics
1. **Infiltration Throttle:** Standing or flowing `freeWater` transfers into subsurface `poreWater` at a rate strictly limited by permeability $k(\text{clayPct}, \text{sandPct}, \text{compaction})$. Heavy rainfall pools on the surface first before slowly percolating downward over hours/days.
2. **Lateral Seepage & Perched Water Tables:** Once `poreWater` exceeds field capacity (`> 10`) or hits an impermeable clay/bedrock layer below, excess water migrates laterally along hydraulic gradients (Darcy's Law), forming hillside springs and seepage along ditch cuts.
3. **Overland Runoff:** Unabsorbed `freeWater` flows across surface `fillHeight` gradients, driving sheet and rill erosion that strips topsoil (`fillHeight` and `organicMatter`) from steep, unrooted slopes and deposits silt in low basins.

### Plastic Deformation & Slope Failure
* **Lateral Compaction Flow:** Heavy loads (wagons, livestock, structural foundations) applied to wet, high-clay voxels with an unconfined adjacent cell (furrow wall, ditch bank) compress the loaded voxel downward while bulging neighboring voxels outward/upward.
* **Landslide Shear Trigger:** When `poreWater` approaches saturation (`14..15`) on steep slopes lacking `hasRoots` reinforcement, shear strength drops below gravitational load, setting `isSliding = 1` and cascading soil mass downslope.

---

## 4. ASCII Rendering & Camera System

### 7-Foot Terrain-Conforming Surface Slice
* **Default Overland View:** Rather than slicing along a rigid flat elevation plane, the renderer evaluates each $(x, y)$ column independently from `surfaceZ` up to `surfaceZ + 7 ft` ($14$ vertical voxels).
* **Visual Layering:** Captures ground soil, tillage furrows, crops, fences, first-floor barn interiors, livestock, and full-height pioneers ($5.5\text{–}6\text{ ft}$) across undulating $256\text{ ft}$ hills while clipping out high tree canopies and roofs that would otherwise occlude gameplay.
* **Elevation & Procedural Soil Styling:**
  * Base RGB interpolates continuously from `(sandPct, clayPct, organicMatter)` and darkens dynamically with `poreWater`.
  * Micro-relief (`fillHeight` + `compaction`) selects surface glyphs (`~` tilled furrows, `_` packed wagon ruts, directional slope indicators `▲`, `▼`, `/`, `\`).
* **Flat Z-Slice Inspection Toggle:** Switches to a strict horizontal Z-plane stepping in **$6\text{"}$ increments** to inspect underground root cellars, wells, perched water tables, and upper barn lofts.
* **Multi-Scale Discrete Zoom:** Supports seamless switching between **1:1 Detail View** ($1\text{ glyph} = 6\text{"} \times 6\text{"}$), **Homestead Aggregate View** ($1\text{ glyph} = 2\text{ ft} \times 2\text{ ft}$ or $4\text{ ft} \times 4\text{ ft}$), and **Valley Overview** (acre-scale topography and hydrology).

---

## 5. Language, Runtime & Concurrency Architecture (Rust)
* **Implementation Language:** **Rust** (native execution for deterministic memory layout, SIMD-friendly bit operations, and race-free multithreading).
* **Active-Zone / Sleep Simulation:**
  * Each $16 \times 16$ column chunk maintains an `isAwake` state and dirty-flag bitmask.
  * Undisturbed prairie and deep bedrock chunks sleep with zero per-tick CPU cost.
  * Rainfall, plowing, irrigation, structural loading, or boundary water inflow wakes the local chunk and its immediate neighbors.
* **Parallel Tick Execution (`rayon`):**
  * **Phase 1 (Parallel Intra-Chunk Update):** `awake_chunks.par_iter_mut()` computes vertical infiltration, thermal diffusion, root uptake, and internal soil mechanics concurrently across all CPU cores.
  * **Phase 2 (Boundary Exchange / Double-Buffered Flux):** Cross-chunk lateral `freeWater` runoff, subsurface seepage, and plastic soil displacement write into edge halo buffers and resolve deterministically before the next tick.