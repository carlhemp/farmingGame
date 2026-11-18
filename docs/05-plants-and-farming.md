# 05 — Plants, Forestry & Agronomy

## 1. Soil Chemistry & Nutrient Dynamics

The $6\text{"}$ voxel memory layout reserves 16 bits for agricultural chemistry. Bit widths here must match doc 02, which is the source of truth.

| Nutrient / Property | Field | Bits / Range | Agronomic Function | Depletion & Restoration |
| :--- | :---: | :---: | :--- | :--- |
| **Nitrogen (N)** | `nitrogen` | 3 bits (`0..7`) | Drives vegetative stem, leaf, and biomass growth. | Depleted heavily by corn/wheat. Fixed naturally by legumes (clover, peas). |
| **Phosphorus (P)** | `phosphorus` | 2 bits (`0..3`) | Stimulates root expansion, early vigor, and seed yield. | Slowly weathered from bedrock; restored via bone meal and manure. |
| **Potassium (K)** | `potassium` | 2 bits (`0..3`) | Governs cold hardiness, stalk stiffness, and drought tolerance. | Restored via wood ash, composted crop residue, and manure. |
| **Soil pH** | `soilPh` | 3 bits (`0..7`) | pH $4.5$ (acidic) to $8.0$ (alkaline) in $0.5$ steps. Gates nutrient uptake. | Buffered by organic matter; raised with agricultural lime or wood ash. |
| **Organic Matter** | `organicMatter` | 3 bits (`0..7`) | Humus density. Darkens soil, increases water-holding capacity. | Depleted by bare tilling; built up through cover crops and compost. |
| **Soil Temperature** | `soilTemp` | 3 bits (`0..7`) | Temperature band (doc 02, table 4.2). Regulates spring germination. | Warms slowly with sun exposure; insulated by snow cover in winter. |

### 1.1 Nutrient Availability Gating
Soil nutrients are not accessible if soil pH or moisture is outside viable biological ranges:
* **Acidic Soil ($\text{pH } < 5.5$):** Phosphorus binds tightly to iron/aluminum, starving crops even if `phosphorus` is high.
* **Alkaline Soil ($\text{pH } > 7.5$):** Phosphorus binds to calcium; micronutrients precipitate out.
* **Severe Drought (`poreWater < 3`):** Plants cannot draw dissolved ions through root membranes, causing wilting regardless of soil richness.

---

## 2. Crop Growth Cycles & Rotations

Crops are stored as records in the world-level `plantTable` and referenced by above-ground stem/foliage voxels (`matterState = 4 [Plant Mass]`).

Growth is tuned to the short calendar (about 12-day seasons, doc 06), not to real crop days. Corn takes most of one summer season. Winter wheat is sown in autumn and harvested early the next summer.

A crop voxel is a **patch of crop**, not one plant. Wheat is sown broadcast at many plants per square foot, so one $6\text{"}$ voxel of wheat stands for a small patch.

> **Open question (doc 09):** A 40-acre field is about 7 million crop voxels. One `plantTable` record each may be too much memory. The other option is a packed crop state per voxel in a separate array, with records kept only for trees and large plants.

```
  [ Seed Sown ] ──> [ Germination ] ──> [ Vegetative ] ──> [ Flowering ] ──> [ Harvest Ready ]
         │                  │                  │                  │                  │
         ▼                  ▼                  ▼                  ▼                  ▼
    Requires moist     Gated by soil      Demands high N    Demands high P    Must harvest
    topsoil (PW 8-10)  temp band (2 or 3) and moisture      and sunlight      before frost
```

### 2.1 Growth Stages
1. **Seed Stage:** Placed in topsoil by settlers. Dormant until soil warms enough for that crop: `soilTemp >= 2` ($40^\circ\text{F}$) for wheat and oats, `soilTemp >= 3` ($50^\circ\text{F}$) for corn and beans.
2. **Vegetative Growth:** Rapidly extracts Nitrogen (N) and `poreWater`. Expands root depth into lower soil voxels (`hasRoots = 1`).
3. **Flowering / Ear Setting:** Consumes Phosphorus (P) and Potassium (K). Stalks reach full height ($6\text{–}8\text{ ft}$ for frontier field corn).
4. **Maturity / Harvest:** Stalks turn golden brown. Crop yield is locked based on cumulative water and nutrient stress experienced during the season.
5. **Killing Frost / Rot:** If unharvested when ground freezes (`isFrozen = 1`), crop dies and collapses into decaying organic debris.

### 2.2 Frontier Crop Profiles & Agronomic Catalog

Growth durations are scaled to the short calendar (~12 days per season, ~48 days per year; doc 06). Days below are game days.

| Crop | Type | Sowing Window | Germination Band | Growth Days | N Use / Fix | Water Need | Frost Tolerance | Main Product & Yield |
| :--- | :--- | :--- | :---: | :---: | :---: | :---: | :--- | :--- |
| **Field Corn (Maize)** | Grain | Mid Spring | Band 3 ($50^\circ\text{F}$) | 10 days | Depletes N heavily (-3) | High (PW 8–12) | Tender (dies at $28^\circ\text{F}$) | Ears of corn; staple calorie & hog feed |
| **Winter Wheat** | Grain | Early Autumn | Band 2 ($40^\circ\text{F}$) | 18 days (overwinters) | Moderate (-1) | Moderate (PW 6–10) | Hardy (survives under snow) | Wheat sheaves → grain flour & straw |
| **Spring Oats** | Grain | Early Spring | Band 2 ($40^\circ\text{F}$) | 8 days | Moderate (-1) | Moderate (PW 6–10) | Cool-hardy (tolerates light frost) | Threshed oats; horse & settler feed |
| **Red Clover** | Legume / Forage | Spring or Autumn | Band 2 ($40^\circ\text{F}$) | 6 days | **Fixes N (+2)** | Moderate (PW 7–10) | Very hardy | Hay & green manure; grazing |
| **Field Peas** | Legume | Early Spring | Band 2 ($40^\circ\text{F}$) | 7 days | **Fixes N (+1)** | Low–Mod (PW 5–8) | Light frost tolerant | Dried peas; soup & winter stores |
| **Potatoes** | Tuber | Mid Spring | Band 2 ($40^\circ\text{F}$) | 9 days | Modest (-1) | Mod–High (PW 8–11) | Tops freeze; tubers safe underground | Seed potatoes & food tubers; heavy yield |
| **Turnips** | Brassica / Root | Late Summer | Band 2 ($40^\circ\text{F}$) | 5 days | Low (-1) | Moderate (PW 6–9) | Very hardy (harvest after frost) | Winter livestock feed & table filler |
| **Pumpkins / Squash** | Cucurbit | Late Spring | Band 3 ($50^\circ\text{F}$) | 10 days | Moderate (-2) | High (PW 8–12) | Very tender | Long-keeping winter squash |
| **Flax** | Fiber / Oil | Early Spring | Band 2 ($40^\circ\text{F}$) | 8 days | Moderate (-1) | High (PW 8–11) | Cool-hardy | Bundled flax straw for spinning linen |

---

## 3. Agronomic Workflows & Harvesting

### 3.1 Broadcast vs. Hill Planting
* **Drill / Broadcast (Small Grains, Clover):** Wheat, oats, and clover are broadcast thickly. Settlers drag a harrow across the plowed field to cover the seed. One $6\text{"}$ voxel represents a dense cluster of stalks.
* **Check-Row / Hill Planting (Corn, Squash):** Field corn and squash are planted in distinct "hills" spaced 3–4 feet apart (6–8 voxels). This allows cross-cultivation with a horse/ox-drawn hoe to eradicate prairie weeds between rows.

### 3.2 Crop Storage Architecture Decision
To prevent millions of entity records in 40-acre fields:
* **Packed Crop Grid:** Crop fields do **not** allocate individual `plantTable` heap records per voxel. Instead, each column's surface/soil record tracks crop type (5 bits), growth stage (3 bits), and cumulative stress (4 bits) packed into a dedicated 2-byte field in awake agricultural sub-chunks.
* **Large Entities Only in `plantTable`:** Only trees, perennial shrubs, and specialized structural plants maintain discrete global IDs and records in `plantTable[entityId]`.

### 3.3 Threshing & Processing
* **Small Grains (Wheat, Oats):** Cut with a sickle or grain cradle (doc 17). Bundled into shocks to dry in the field. Brought into the barn for indoor winter flailing with a wooden flail to separate grain from chaff, followed by winnowing.
* **Corn:** Ears are snapped by hand directly off stalks in late autumn, thrown into a farm wagon, and hauled to the slatted corn crib for dry air curing.

### 3.4 Hay Making & Curing Workflow
1. **Mowing:** In mid-summer, meadow grass or red clover is designated and cut using a scythe.
2. **Curing:** The cut grass rests on the ground. Its `grass_dry` field (doc 02) rises under warm summer sun and wind (doc 11). If rain falls during curing, `moisture_rot` accumulates and hay quality degrades.
3. **Cock & Stack:** Once `grass_dry > 200`, settlers rake hay into mounds (shocks/cocks), haul it by wagon, and pack it into an elevated hay barrack or barn loft for winter livestock feed (doc 14).

---

## 4. Trees, Wind Torque & Forestry

Trees are massive multi-voxel entities stored in `plantTable[entityId]` with parameters: species, age, height, trunk diameter, canopy volume, and root depth.

```
                  Wind Force (Speed x Canopy Area)
                         ═════════════════>
                              ┌─────────┐
                              │  Canopy │
                              └────┬────┘  Applied at Canopy Height
                                   │       ───> Bending Torque (M)
                                   │
                              ┌────┴────┐
                              │  Trunk  │  Resisting: Trunk Wood Shear Strength
                     ─────────┴─────────┴───────── Ground Level
                              ┌────┬────┐
                              │  Roots  │  Resisting: Root Anchorage
                              └─────────┘  (Weakened when poreWater is saturated)
```

### 4.1 Wind Damage Mechanics
High frontier winds (summer thunderstorms, prairie blizzards) apply dynamic torque to trees:
$$\text{Wind Force } (F) = \text{Wind Speed}^2 \times \text{Canopy Area} \times \text{Canopy Density}$$
$$\text{Bending Torque } (\tau) = F \times \text{Canopy Center Height}$$

The engine compares bending torque $\tau$ against two resisting forces:
1. **Trunk Strength:** Governed by trunk diameter and species (dense White Oak vs. soft White Pine).
2. **Root Anchorage:** Governed by root depth, lateral root spread, and soil moisture (`poreWater`).

### 4.2 Uprooting vs. Trunk Snap
* **Wet Soil Toppling (Uprooting):** When heavy rains saturate the soil (`poreWater = 14–15`), the muddy soil loses its grip on roots. Root anchorage fails before the trunk snaps. The entire tree uproots, lifting a root-wad and leaving a large crater in the soil.
* **Dry Soil Snapping:** In dry, compacted soil, roots hold firm. If storm winds exceed trunk strength, the tree snaps low on the trunk.

### 4.3 Tree Felling & Timber Harvesting
* **Axe Chopping:** Settlers designated to fell trees strike the base of the trunk with axes, reducing trunk integrity over several game minutes.
* **Rigid-Body Felling:** When cut, the tree detaches, calculates a fall angle based on wind and cut direction, and falls as a rigid multi-segment body.
* **Voxel Stamping:** Upon hitting the ground, the fallen trunk restamps into harvestable timber voxels (`matterState = 5 [Loose Item]`, `materialId = oak_timber`), ready for settlers to buck into logs and haul to the woodpile.
