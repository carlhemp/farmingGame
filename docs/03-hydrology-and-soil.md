# 03 — Hydrology & Soil Mechanics

## 1. Three Water Stores

Water in *Frontier Farming Simulator* is kept in three places:

1. **Surface water (2D surface layer):** One water depth per column, in $1/64\text{"}$ units (doc 02, section 5). This covers rain on the ground, puddles, sheet flow, creeks, ditches, and ponds of any depth.
2. **Pore water (`poreWater`, 4 bits per voxel, $0\text{–}15$):** Capillary water held in soil pores.
   * `0`: Air-dry soil (below wilting point; plants cannot extract moisture).
   * `10`: Field capacity (ideal agricultural moisture; gravitational water has drained).
   * `14–15`: Full saturation (no soil aeration; risk of root rot and liquefaction).
3. **Room water:** One water level per enclosed room (cellars, dugouts, wells). See section 7.

```
          Rainfall
              │
              ▼
   ┌─────────────────────┐   Flow between columns
   │ Surface water depth │ ◄──────────────────────►  (virtual pipes, section 3)
   │ (2D, 1/64" units)   │
   └──────────┬──────────┘
              │  Infiltration: k(clayPct, sandPct, compaction, isFrozen)
              ▼
   ┌─────────────────────┐
   │ poreWater (voxels)  │  (Field Capacity = 10, Saturation = 15)
   └──────────┬──────────┘
              │  Hourly step
     ┌────────┴─────────┐
     ▼                  ▼
Deep Percolation    Lateral Seepage (Darcy's Law)
(Toward Bedrock)    (Springs, Ditch Cuts, Cellar Walls)
```

There are no per-voxel `freeWater` or `fillHeight` fields. The old `matterState = Liquid-Only` is retired.

---

## 2. The Surface Layer

Each column's water surface is:

$$\text{Water Surface} = \text{ground} + \text{depth}$$

Both values are integers in $1/64\text{"}$ units. Because units are fine, gentle field slopes (1 to 3%) are real slopes, not flat terraces, and water drains across them.

Awake tiles also keep flow data:

```rust
pub struct SurfaceFlow {     // only for awake 16x16 tiles
    pub out: [i32; 4],       // outflow toward N, E, S, W
    pub sediment: u16,       // soil carried in the water (erosion)
}
```

---

## 3. One Surface Water Step

Each step does the following for every awake column:

1. **Rain and snowmelt:** Add the rainfall for this step, plus any melting snow or ice (doc 11), to `depth`. Columns under a roof get no rain; the roof sheds it to the columns along its edges (the drip line).
2. **Infiltration:** Move water from `depth` into the top soil voxel's `poreWater`, at the rate the soil allows (section 4). Only that one voxel is touched.
3. **Flow (virtual pipes):** For each of the 4 neighbors, update the outflow from the difference in water surface:
   $$\text{out}_d \mathrel{+}= C \times (\text{surface}_{\text{self}} - \text{surface}_{d}), \quad \text{floored at } 0$$
4. **Clamp:** If the outflows add up to more water than the column holds, scale them all down so that
   $$\sum \text{out} \le \text{depth}$$
   A column can never send water it does not have, so water never goes negative and is never created.
5. **Update depths:**
   $$\text{depth}_{\text{new}} = \text{depth} - \sum \text{out} + \sum \text{in}$$
6. **Erosion:** Flow speed is outflow divided by depth. Fast water over bare soil picks up sediment and lowers `ground`; slow water drops it (section 6).

This "virtual pipes" method is a common one for terrain water and erosion. It is smooth, works at any depth, and gives flow speed for free.

### 3.1 Runoff Only Where Rain Beats the Soil
Water only runs off where rain falls faster than the soil can take it in. Sandy, loose, dry soil absorbs most storms, so those columns never start flowing. Runoff happens on clay, packed, frozen, or soaked ground.

* While rain is lighter than infiltration everywhere in a tile, the tile skips the flow step. It just adds water to the top soil voxels.
* Only tiles with standing water or runoff stay awake. Creeks stay awake too, but they are a thin ribbon of the map.

### 3.2 Slowed-Down Time Step
Real shallow-water physics on a $6\text{"}$ grid needs steps of a fraction of a game second to stay stable. That is far too expensive.

At normal speed a game day takes about 20 real minutes (doc 06), so real runoff across a field would finish in under one real second anyway. What matters to the player is where water ends up, not the splash on the way. So surface water runs on a deliberately slowed, stable step: about one step per game minute with a few sub-steps, with $C$ small enough that a column never swings back and forth.

### 3.3 Determinism
All surface water math uses fixed-point integers. Saves and replays then give the same result on every machine and with any number of threads.

---

## 4. Infiltration & Subsurface Seepage

### 4.1 The Infiltration Throttle
Surface water moves into the top soil voxel at a rate $k$ set by soil texture and bulk density:
$$k = f(\text{sandPct}, \text{clayPct}, \text{compaction}, \text{isFrozen})$$

* **High Sand / Low Compaction:** Fast infiltration. Sandy loam absorbs sudden cloudbursts without pooling.
* **High Clay / High Compaction (Hardpan):** Very low infiltration ($k \approx 0$). Water pools on the surface for hours, creating mud and runoff.
* **Frozen Soil (`isFrozen = 1`):** Infiltration is blocked ($k = 0$). Spring snowmelt runs off across the frozen top layer.

### 4.2 Percolation & Darcy Seepage (hourly)
Water inside the soil moves on the slow hourly step, in the 3D voxel grid:
* **Deep Percolation:** Water above field capacity (`poreWater > 10`) drains downward toward bedrock.
* **Lateral Seepage:** When water hits a barrier below (dense clay subsoil or bedrock), it moves sideways along the hydraulic gradient.
* **Hillside Springs:** Water moving sideways through permeable topsoil comes out onto the surface (into the surface layer) where the hill profile cuts across the water table.
* **Ditch Drainage:** An open ditch lowers the local water table, because neighboring soil voxels seep their extra `poreWater` into the ditch's surface water.

---

## 5. Chunk Borders & Conservation

### 5.1 Surface Tiles (2D)
* A $16 \times 16$ tile has $16 \times 4 = 64$ edge columns.
* Outflow that crosses a tile edge is written into that tile's **outbound edge buffer**.
* At the start of the next phase, each tile adds up its incoming arrivals in a single sweep (doc 06).

### 5.2 Subsurface Sub-Chunks (3D)
* Seepage can cross any of a sub-chunk's 6 faces ($256$ cells each), including up and down into the sub-chunks above and below.
* The same clamp and edge-buffer rules apply, so water is conserved across every border, including for deferred chunks catching up.

### 5.3 Grid Shape
Flow uses 4 neighbors. That spreads water in slightly diamond-shaped patterns. This is fine for gameplay. Diagonal neighbors can be added later if it looks wrong.

---

## 6. Overland Runoff & Erosion

* **Sheet Erosion:** On bare, unrooted ground (`hasRoots = 0`), fast surface flow picks up sediment, lowering `ground` and carrying away `organicMatter`.
* **Deposition:** Where the slope flattens and flow slows, sediment settles, raising `ground` and forming fertile bottomland.
* **Root Protection:** Columns whose top voxel has `hasRoots = 1` (prairie grass, clover, tree roots) resist erosion.
* **Crossing a voxel boundary:** When `ground` drops below or rises above a whole $6\text{"}$ voxel, the voxel grid is updated (a soil voxel removed or added).

---

## 7. Room Water (Cellars, Dugouts, Wells)

Enclosed spaces are rooms (doc 04). Each room has one water level.

* Water enters a room through open doorways, from surface water at the door, and by seepage from the soil walls on the hourly step.
* Water leaves by seeping back into dry soil walls or out the doorway once the level reaches it.
* A flooded cellar shows as a water level in the room, and stored goods below that level get wet (doc 08).

---

## 8. Continuous World Boundary Hydrology

A 120-acre claim exists inside a 500-acre continuous world:

```
    [ Upstream Frozen Chunks ]
                │  (Seeded Fixed Inflow)
                ▼
  ══════════ Boundary ══════════
    [ Active Player Claim ]
      - Plowing, ditches, crops
      - Dynamic water table
  ══════════ Boundary ══════════
                │  (Slope-Governed Outflow)
                ▼
   [ Downstream Frozen Chunks ]
   (Regional River / Aquifer Sink)
```

1. **Boundary Inflow (Upstream):** Streams and hillside runoff entering the claim flow in at rates calculated deterministically from the world seed's coarse watershed map.
2. **Boundary Outflow (Downstream):**
   * **Gravity Drainage (Sink):** If a border column's water surface is higher than the frozen ground next to it, the water leaves the farm into the regional sink.
   * **Backwater & Ponding:** If the land past the border is higher, water cannot cross and ponds on your land, creating a wetland unless you ditch around it.
3. **Parcel Purchase:** When adjacent land is bought, its chunks wake with baseline moisture from the world seed and connect to your existing streams and ditches.

---

## 9. Soil Mechanics & Plastic Deformation

### 9.1 Compaction & Traffic
* Freshly moldboard-plowed soil starts at `compaction = 0` (loose, high infiltration).
* Settler foot traffic, draft animal hooves, and wagon wheels pack soil down toward `compaction = 15` (hardpan).
* Wheel ruts lower `ground` slightly and pack the soil, so they channel rainwater into fast rivulets that speed up erosion.

### 9.2 Plastic Lateral Bulge
Under heavy loads (loaded wagons, stone foundations, brick chimneys), saturated clay soil (`clayPct > 10`, `poreWater > 12`) deforms:
* The loaded column's `ground` sinks.
* If an open neighbor exists (a furrow or ditch bank), that neighbor's `ground` rises, so furrow walls slough in.

### 9.3 Landslide Shear Trigger
On steep slopes:
* When heavy rain drives `poreWater` to `14–15`, pore pressure cancels internal friction.
* If the slope lacks root anchorage (`hasRoots = 0`), shear strength drops below the gravitational load.
* The engine sets `isSliding = 1`, and the saturated soil mass cascades downslope, burying downhill crops and exposing raw subsoil.
