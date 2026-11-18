# 02 — World Scale, Voxels & Memory Layout

This doc is the single source of truth for the voxel bit layout. Other docs refer to fields by name and must match the table in section 4.

## 1. World Scale & Spatial Geometry

* **Total Footprint:** 500 Acres ($\approx 4,667\text{ ft} \times 4,667\text{ ft} \approx 1,422\text{ m} \times 1,422\text{ m}$).
* **Voxel Resolution:** $6\text{"} \times 6\text{"} \times 6\text{"}$ ($15.24\text{ cm}$) cubic voxels.
  * Planar grid: $9,334 \times 9,334 \approx 87.1\text{ million}$ vertical columns.
* **Vertical Coordinates:**
  * World Z is **absolute**: a `u16` in $6\text{"}$ steps (up to $32,767\text{ ft}$).
  * A typical valley has about $256\text{ ft}$ of relief ($512$ Z-levels).
  * There is no separate per-column `baseElevation`. The exact ground height of each column lives in the 2D surface layer (section 5), in fine units.
* **Alignment:** Sub-chunks are aligned to absolute world coordinates $(X, Y, Z)$. Neighboring voxels across chunk borders share the same Cartesian coordinates.
* **Continuous Seeded World & Claims:**
  * The entire 500-acre map is generated deterministically from a single master world seed.
  * Topography, ridgelines, and streams cross property lines seamlessly.
  * A **Claim** (initial size $\approx 120\text{ acres}$) is an ownership boundary mask. Chunks within the claim can run full simulation; chunks outside stay frozen terrain with fixed boundary inflow and slope-governed outflow.

---

## 2. Storage Model: Keep Only What Changed

The world generator is a pure function of `(world seed, generator version, position)`. Anything it can produce does not need to be stored.

| Group | Fields | What changes them | Stored? |
| :--- | :--- | :--- | :--- |
| **From the seed** | `materialId`, `clayPct`, `sandPct` | Only digging, building, erosion, or silt deposits | No, regenerated on demand until changed |
| **Changed by the player** | `compaction`, `organicMatter`, `hasRoots`, `isDesignated` | Plowing, foot and wagon traffic, crops, manure | Only where different from the seed |
| **Changed by the sim** | `poreWater`, `soilTemp`, `isFrozen`, N, P, K, pH | Weather, crops, time | Full detail while awake; compressed difference while asleep |

* **Awake sub-chunks** are held in memory as dense arrays (section 3).
* **Sleeping sub-chunks** that differ from the seed are saved as a compressed difference: a small palette per sub-chunk, then run-length encoding down each column. Soil comes in layers, so this compresses well.
* **Untouched sub-chunks** store nothing. They are regenerated when needed.
* **Generator version:** Every save records the generator version. Changing the generator code changes the land it produces, so released versions are frozen and old saves keep using theirs. Doc 12 covers the save file itself.

```rust
pub enum SubChunk {
    UniformAir,
    UniformBedrock(u8 /* materialId */),
    Procedural,                 // untouched: regenerate from the seed, nothing stored
    Compressed(Box<[u8]>),      // asleep: palette + run-length difference from the seed
    Dense(Box<[Voxel; 4096]>),  // awake: 32 KB on the heap
}
```

---

## 3. Sparse 3D Sub-Chunks

* **Column Chunks:** $16 \times 16$ horizontal columns ($8\text{ ft} \times 8\text{ ft}$ planar footprint).
* **Vertical Sub-Chunks:** $16 \times 16 \times 16$ voxel cubes ($8\text{ ft}$ cubes, $4,096$ voxels).
* **Size:** A dense sub-chunk is $4,096 \times 8\text{ bytes} = 32\text{ KB}$.
  * That fits in the L1 data cache on some CPUs (AMD Zen 4 has 32 KB) and in L2 on all current desktop CPUs (Intel P-cores and Zen 5 have 48 KB L1; Apple M-series P-cores have 128 KB).
  * The point is good locality, not "zero cache misses": the data still has to be loaded, and updates also read neighbor data.
* **Wording used in the docs:** "chunk" means a $16 \times 16$ column chunk. "Sub-chunk" means one $16^3$ cube. The 2D surface layer uses "tile" for one $16 \times 16$ block of columns.

---

## 4. Core Voxel Memory Layout (64-Bit / 8-Byte `u64`)

Surface water and micro-relief are **not** stored in voxels. They live in the 2D surface layer (section 5). That frees the bits for a 16-bit `entityRef`.

| Bit Range | Width | Field Name | Value Range | Simulation & Rendering Semantics |
| :--- | :---: | :--- | :--- | :--- |
| `0..2` | 3 bits | `matterState` | `0..7` | `0`=Air, `1`=Soil, `2`=Bedrock/Stone, `3`=Built Structure, `4`=Plant Mass, `5`=Loose Item, `6`=Snow/Ice, `7`=Reserved |
| `3..7` | 5 bits | `materialId` | `0..31` | Subtype ID (e.g. `topsoil`, `subsoil`, `peat`, `granite`, `limestone`, `oak_timber`, `sod_brick`, `pine_rail`) |
| `8..11` | 4 bits | `clayPct` | `0..15` ($6.67\%$ steps) | Soil cohesion, plasticity, water retention, red/ochre color shift. For built voxels, see 4.3 |
| `12..15` | 4 bits | `sandPct` | `0..15` ($6.67\%$ steps) | Hydraulic conductivity ($k$), porosity, pale/tan color tint. For built voxels, see 4.3 |
| `16..18` | 3 bits | `organicMatter` | `0..7` | Humus density (8 stages); darkens soil RGB toward black-brown, buffers fertility |
| `19..22` | 4 bits | `compaction` | `0..15` | Bulk density (`0`=tilled/loose, `15`=hardpan). Throttles infiltration and alters glyphs |
| `23..26` | 4 bits | `poreWater` | `0..15` | Absorbed capillary water (`10`=field capacity, `15`=full saturation / anoxia trigger) |
| `27..29` | 3 bits | `nitrogen` (N) | `0..7` | Available soil Nitrogen; depleted by heavy feeders (corn), fixed by legumes |
| `30..31` | 2 bits | `phosphorus` (P) | `0..3` | Available Phosphorus (Depleted, Low, Adequate, High); seed/fruit setting |
| `32..33` | 2 bits | `potassium` (K) | `0..3` | Available Potassium (Depleted, Low, Adequate, High); stem rigidity, disease resistance |
| `34..36` | 3 bits | `soilPh` | `0..7` | pH $4.5$ to $8.0$ in $0.5$ steps; gates nutrient uptake |
| `37..39` | 3 bits | `soilTemp` | `0..7` | Temperature band (table 4.2); dictates frost line and germination |
| `40..55` | 16 bits | `entityRef` | `0..65,535` | Index into the sub-chunk's entity lookup list (`0` = none) |
| `56..63` | 8 bits | `flags` | 8 booleans | Dedicated simulation flags (4.1) |

### 4.1 The 8 Simulation Flags (`flags`, Bits 56..63)
1. `isFrozen` (Bit 56): Soil is frozen solid; infiltration drops to zero; soil cannot be tilled.
2. `hasRoots` (Bit 57): Plant roots bind this voxel, providing shear resistance against landslides and water erosion.
3. `isSliding` (Bit 58): Saturated soil shear failure triggered; actively cascading downslope.
4. `isLoadBearing` (Bit 59): Built voxel that is part of a support path (doc 04).
5. `isExposedSurface` (Bit 60): Top voxel of the column, in contact with the surface layer and the air.
6. `isDesignated` (Bit 61): Tagged by player for settler job queue (plow, sow, fell, dig, build).
7. `isOnFire` (Bit 62): Active flame or burning ember; damages structural durability, consumes plant mass, and spreads to flammable neighbors unless stopped by bare dirt firebreaks.
8. `sunExposure` (Bit 63): Direct overhead sunlight (`1`) vs. canopy/building shadow (`0`); gates photosynthesis and speeds surface evaporation.

### 4.2 Soil Temperature Bands (`soilTemp`)

| Band | Range | Meaning |
| :---: | :--- | :--- |
| `0` | below $32^\circ\text{F}$ | Frozen ground (`isFrozen` may be set) |
| `1` | $32\text{–}40^\circ\text{F}$ | Thawed but cold |
| `2` | $40\text{–}50^\circ\text{F}$ | Root cellar range; wheat and oats can germinate |
| `3` | $50\text{–}60^\circ\text{F}$ | Corn and beans can germinate |
| `4` | $60\text{–}70^\circ\text{F}$ | Warm spring soil |
| `5` | $70\text{–}80^\circ\text{F}$ | Summer soil |
| `6` | $80\text{–}90^\circ\text{F}$ | Hot surface soil |
| `7` | above $90^\circ\text{F}$ | Baked bare surface |

Air temperature is not stored per voxel. It comes from the weather state for the whole valley.

### 4.3 Layout & Arithmetic Invariants
1. **Implicit Silt:** $\text{siltPct} = 15 - (\text{clayPct} + \text{sandPct})$. Whenever soil is initialized, mixed, or eroded, values are clamped so that $\text{clayPct} + \text{sandPct} \le 15$.
2. **Built voxels reuse soil bits:** For `matterState = 3`, bits `8..15` hold `durability` (`0..255`) instead of clay and sand. Rot, fire, and axe hits lower it. Other soil fields are unused for built voxels.
3. **Root Drowning (Anoxia):** Needs no separate bit. Saturated soil (`poreWater = 15`) or standing surface water over the column (surface layer depth $> 0$) is checked by the plant on its daily tick. If waterlogged for more than 2 to 3 days, the plant suffers root damage and wilts.

---

## 5. The 2D Surface Layer

Every column in the claim has one surface record. It holds the exact ground height, surface water, snow, and pond ice, in fine units ($1/64\text{"}$), plus standing grass. Doc 03 covers how water moves through it, doc 11 covers snow and ice, and doc 14 covers grass for grazing and hay.

```rust
pub struct SurfaceCell {
    pub ground: i32,    // absolute top of ground, 1/64" units (furrows, ruts, erosion)
    pub depth: u16,     // surface water depth, 1/64" units (up to about 85 ft)
    pub snow: u16,      // snow cover depth, 1/64" units
    pub ice: u16,       // pond ice thickness, 1/64" units
    pub grass: u8,      // standing grass amount, 0..255 (grazing, hay, fire fuel)
    pub grass_dry: u8,  // how dry the grass is, 0..255 (hay curing, fire danger)
}
```

* **This is per column, not per voxel.** The voxel stays 64 bits (section 4). A column has about 26 voxels in the soil band (about 208 bytes) and one 12-byte surface record, so the surface layer adds about 6%. It is a separate array, so it does not change the 32 KB sub-chunk.
* **Why not store these in voxels:** grass, snow, and surface water exist once per column, at the top. Giving every voxel bits for them would waste those bits in all the voxels below the surface.
* Fine units let gentle field slopes exist. A 1% slope drops about $0.06\text{"}$ per voxel, which is about 4 units. A 4-bit field in $0.375\text{"}$ steps would turn that slope into flat terraces.
* Plowing, ruts, and erosion change `ground`. The voxel grid only changes when the ground crosses a whole $6\text{"}$ voxel boundary.
* Extra flow data (outflows, sediment) exists only for awake tiles (doc 03).

---

## 6. Memory Footprint (estimates, to be measured in milestone M3)

For the 120-acre claim ($\approx 20.9\text{ million}$ columns, $\approx 81,700$ column chunks):

| Item | Estimate | Notes |
| :--- | :--- | :--- |
| Surface layer | $\approx 250\text{ MB}$ | 12 bytes per column, always resident |
| All soil-band sub-chunks, if dense | $\approx 6.5\text{ GB}$ | 2 to 3 sub-chunks per column chunk. This is why they are not all kept dense |
| Awake sub-chunks (about 5% of the claim) | $\approx 330\text{ MB}$ | Dense, 32 KB each |
| Sleeping changed sub-chunks | a few hundred MB at most | Compressed differences; depends on how much land has been worked |

---

## 7. World-Level Entity Tables & Chunk Proxies

Structures, trees, fences, and item stacks routinely cross chunk boundaries.

* **Global Permanent ID:** Every multi-voxel plant, item stack, door, or container has a permanent world ID.
* **Sub-Chunk Lookup Proxy:** Each sub-chunk keeps a compact lookup list of the global IDs touching its volume.
* **Voxel `entityRef`:** The 16-bit `entityRef` indexes into the sub-chunk's lookup list (`0` = none, up to $65,535$ entities per sub-chunk).
* **Mobile agents are not in the voxel grid:** Settlers, animals, and wagons live in an agent table with a position and a footprint (doc 01). Moving them never rewrites voxels or lookup lists.
* **Rooms:** Enclosed air spaces are found by flood fill and kept in a room table (doc 04). Rooms hold shelter, warmth, and cellar water.
* **Decoupled Scheduling:** Entities run on their own schedules, independently of chunk sleep states. A sleeping chunk never freezes half of a barn.
