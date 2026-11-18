# 10 - World Generation

The whole 500-acre valley comes from one world seed. Generation runs in two stages, the way Dwarf Fortress does it: a **coarse map** made once when the world is created, then **detail** made on demand for any column or sub-chunk the game needs.

Values marked *starting value* are for tuning in milestone M1.

---

## 1. Rules

1. **Pure function:** Detail is a pure function of `(world seed, generator version, coarse map, position)`. There is no stored randomness.
2. **Coarse map wins:** The detail stage only adds variation inside the limits the coarse map sets. Fine detail never contradicts it. A creek on the coarse map is a creek in the detail, in the same place.
3. **Cheap to redo:** Untouched land is never saved (doc 02). Regenerating a sub-chunk must be fast, so most work is done once per column, not once per voxel.
4. **Same on every machine:** Floating-point math is allowed only with basic operations (`+ - * /` and `sqrt`), which give the same result everywhere. No `sin`, `cos`, `exp`, or `pow` from the system math library inside the generator. Random values come from a hash of (seed, position, purpose).
5. **Versioned:** Every save records the generator version (doc 02). Changing generator code means a new version number.

---

## 2. Target Landscape

**Decision:** the default valley is **tallgrass prairie with wooded creek valleys**, like 1850s eastern Kansas, Nebraska, or Iowa:

* Rolling prairie uplands with deep, dark soil, hard to break with a plow because of the thick sod.
* A creek winding through a bottomland of rich, flood-prone soil.
* Strips of oak, hickory, walnut, and cottonwood along the creek and on steeper slopes. Timber is limited, which makes sod houses and fieldstone matter.
* A few wet low spots, and limestone showing on bluffs and ridges.

---

## 3. Stage A: The Coarse Map

### 3.1 Grid
* One coarse cell is $32\text{ ft}$ ($64$ voxels) on a side. That matches the Valley Overview zoom (doc 07).
* The valley is $146 \times 146 \approx 21,300$ cells.
* The coarse map is saved in the world file. At about 32 bytes per cell it is under 1 MB.

### 3.2 Layers

| Layer | Stored As | Used For |
| :--- | :--- | :--- |
| **Elevation** | Height in fine units | Shape of the land |
| **Streams** | Smoothed polylines with flow rate, width, depth | Creek channels, boundary inflow (doc 03) |
| **Landform** | Enum: upland, slope, bottomland, wetland, ridge | Picks the soil profile (section 4.2) |
| **Bedrock** | Depth and type (limestone, sandstone, shale) | Where soil ends; fieldstone sources |
| **Water table** | Depth below ground | Starting `poreWater`; springs |
| **Vegetation** | Enum and density: prairie, savanna, woodland, riparian woods, wetland | Grass, trees, starting `organicMatter` |
| **Microclimate** | Small temperature offset | Warm south slopes, frost pockets in low spots |

Regional climate (rain, seasons, temperature) is the same for the whole valley and belongs to the weather doc. The coarse map only adds local offsets.

### 3.3 Steps
1. **Valley shape:** Start from a broad valley trough that enters on one map edge and leaves on another. Add layered noise at several scales for ridges, side draws, and rolling uplands. Relief is about $100\text{–}250\text{ ft}$ (*starting value*).
2. **Fill pits:** Fill closed low spots so water can always drain, except where a wetland is wanted. Leave a few shallow pits on purpose as wetlands.
3. **Flow:** Compute flow direction and flow accumulation for each cell. Cells with enough upstream area become streams. The main creek also carries the fixed inflow from the upstream map edge.
4. **Stream paths:** Turn stream cells into smoothed polylines and add gentle meanders. Store width and depth from flow rate: about $3\text{–}10\text{ ft}$ wide and $1\text{–}3\text{ ft}$ deep for the main creek (*starting values*).
5. **Landform:** Classify each cell from slope, height above the nearest stream, and flow accumulation.
6. **Bedrock and water table:** Bedrock is shallow on ridges and bluffs and deep under bottomland. The water table is near the surface in bottomland and wetlands, and deep on uplands.
7. **Vegetation:** Choose from landform, water table, and aspect. Woods along streams and on steep north-facing slopes, prairie on uplands, savanna between, wetland plants in wet low spots.
8. **Boundary inflow:** Record flow rates where streams and runoff cross into each parcel. Doc 03 uses these for frozen-land borders.

The coarse map is made once and never changes. There is no erosion or weather simulation on it over time.

---

## 4. Stage B: Detail (on demand)

### 4.1 Ground Height
For each column, in $1/64\text{"}$ units (doc 02, section 5):

$$\text{ground} = \text{smooth}(\text{coarse elevation}) + \text{detail noise} + \text{micro-relief} - \text{channel}$$

* **Smooth:** Interpolate the coarse elevation smoothly, so there are no steps at coarse cell edges.
* **Detail noise:** Gentle bumps and swales, limited to about $\pm 2\text{ ft}$ (*starting value*), so the land never leaves the coarse shape.
* **Micro-relief:** Prairie hummocks and animal paths, about $\pm 1\text{"}$.
* **Channel:** Distance from the column to the nearest stream polyline gives a channel cross-section (steep banks, flat bed). The polylines come from the coarse map, so channels line up across every chunk border.

The result before any player change is the **generated ground height**. The camera band starts from it (doc 07).

### 4.2 Soil Profile
Each column's soil is a stack of layers (horizons). The landform sets the layers; small noise varies them.

| Landform | Topsoil (A) | Subsoil (B) | Below | Starting Chemistry |
| :--- | :--- | :--- | :--- | :--- |
| **Upland prairie** | Silt loam, $12\text{–}18\text{"}$, `organicMatter` 6 | Clay loam, $2\text{–}3\text{ ft}$ | Loess or glacial till | pH about 6.5, N high, dense grass roots |
| **Slope** | Silt loam, $6\text{–}10\text{"}$, `organicMatter` 3–4 | Clay loam | Glacial till | Erodes easily once plowed |
| **Bottomland** | Silt loam, $2\text{–}4\text{ ft}$, `organicMatter` 7 | Layers of sand and silt | Alluvium | Rich, floods, high water table |
| **Wetland** | Peat / muck, $1\text{–}3\text{ ft}$ | Gray clay | Clay | Waterlogged, pH about 5.5 |
| **Woodland** | Leaf litter over silt loam, $6\text{–}10\text{"}$, `organicMatter` 4 | Clay-rich | Glacial till | pH about 5.5–6, more acidic |
| **Ridge / bluff** | Thin, $0\text{–}6\text{"}$ | Thin or none | Limestone at $2\text{–}4\text{ ft}$ | Surface stones; pH about 7.5 over limestone |

* Each layer sets `materialId`, `clayPct`, `sandPct`, `organicMatter`, `compaction`, N, P, K, and pH.
* Values are worked out **once per column**, then the column's voxels are filled layer by layer. That keeps regeneration cheap.

### 4.3 Starting Water and Temperature
* **Water:** Below the water table, `poreWater = 15`. Above it, moisture grades up to near field capacity at the surface, adjusted for the starting season.
* **Temperature:** `soilTemp` bands by depth and season (doc 02, table 4.2). The game starts in early spring: surface bands 1 to 2, deeper soil steady around band 2. Microclimate offsets warm south slopes and chill low spots.
* **Bought land:** When a parcel is bought, its chunks start from these same values, adjusted to the current season (doc 03, section 8).

### 4.4 Plants and Stones
* **Prairie:** The top soil voxels get `hasRoots = 1`, and the surface layer gets a starting `grass` amount from the vegetation layer (doc 02, section 5). Grass is not an entity.
* **Trees:** The land is split into $16 \times 16\text{ ft}$ cells (*starting value*). Each cell may hold one tree, with a chance set by vegetation density. Position, species, and age come from a hash of the cell's coordinates, so:
  * any chunk can work out which trees reach into it by checking nearby cells, out to the largest canopy radius;
  * a tree's ID is stable, made from its cell coordinates and the generator version.
* Tree records (doc 05) are created when their land is first needed by the sim. Trees on land you do not own stay procedural.
* **Fieldstones:** Glacial till and ridges get scattered surface stones as stone voxels. They are the starting source of fieldstone for building.

---

## 5. Choosing a Claim

* The valley is divided into **40-acre parcels** ($1,320\text{ ft}$ squares), like the government land survey of the time. That is a $3 \times 3$ grid of parcels across the valley, with a strip of unsurveyed land around the edge.
* A new game starts on the Valley Overview. The player picks **3 parcels** (120 acres) next to each other, with a preview of soil, water, timber, and stone, the way DF shows a site before you embark.
* More parcels can be bought later (doc 01).
* The family arrives in early spring at the claim edge with a wagon, oxen, tools, seed, and food for a few weeks.

---

## 6. Cost

* **Coarse map:** made once at world creation. Expected to take a few seconds at most.
* **Detail:** per column, a few noise lookups, a nearest-stream lookup, and a layer stack. Filling a sub-chunk is then a simple loop. The target is a few milliseconds per sub-chunk (*starting value*, measured in M1).
* **Valley Overview:** drawn straight from the coarse map, without generating detail.

---

## 7. Tests

* **Same seed, same world:** golden seeds give identical coarse maps and identical sub-chunks, on Windows and Linux.
* **Coarse map wins:** detail ground height stays within the noise limit of the smoothed coarse elevation everywhere.
* **Streams line up:** a stream channel is continuous across every chunk and tile border.
* **Water drains:** outside planned wetlands, surface water poured anywhere reaches a stream or the map edge.
* **Regeneration matches:** a sub-chunk thrown away and regenerated is identical to the first copy.

---

## 8. Open Questions

1. **Parcels:** Do 40-acre squares and a 3-parcel start feel right?
2. **One-time erosion pass:** Should world creation run a single erosion pass to make valleys look more natural? It would be done once, never over time.
3. **More than one stream:** Should larger valleys get side creeks and springs as standard, or only sometimes?
4. **Other landscapes later:** A forested eastern valley or a drier western plain could be added later as world options.
