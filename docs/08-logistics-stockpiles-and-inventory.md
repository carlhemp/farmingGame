# 08 — Logistics, Stockpiles & Inventory (DF-Style)

## 1. The Dwarf Fortress Stockpile Paradigm

*Frontier Farming Simulator* handles physical materials through **designated stockpile zones, containers, and autonomous hauling jobs**, directly inspired by *Dwarf Fortress*.

Nothing teleporting or existing in an abstract global ledger: if you want to frame a barn roof with pine rafters, a settler must pick up a pine beam from a timber pile, carry it across the barnyard, and hoist it into position.

```
       [ Designated Stockpile Zone ]
       (Keyboard: p -> Wood -> Filter: Pine)
                      ▲
                      │  Autonomous Haul Job
                      │  (Pioneer Caleb + Oxen Cart)
                      │
       [ Harvest Site / Felled Tree ]
       (Loose Pine Logs on Muddy Ground)
```

### 1.1 Stockpile Types & Designation
Players designate 2D surface areas on the ground or across indoor floors (`p` $\rightarrow$ drag rectangle):

| Stockpile Type | Hotkey | Accepted Goods | Ideal Location | Special Mechanics |
| :--- | :---: | :--- | :--- | :--- |
| **Timber Yard** | `p` $\rightarrow$ `w` | Felled logs, hewn beams, fence rails, firewood cords. | Near woodlot or sawmill. | Firewood cures/seasons faster when stacked off damp soil. |
| **Stone & Masonry** | `p` $\rightarrow$ `s` | Fieldstones, limestone blocks, quarried rock. | Near quarry or building site. | Extremely heavy; requires wheelbarrows or carts. |
| **Granary & Grain Crib** | `p` $\rightarrow$ `g` | Threshed wheat, shelled corn, ear corn, oats. | Elevated dry crib or barn. | Must be roofed; rain exposure causes rapid mold/spoilage. |
| **Root Cellar (Food)** | `p` $\rightarrow$ `f` | Potatoes, turnips, salt pork, smoked bacon, cider. | Subterranean excavated dugout. | Kept cold by deep soil thermal mass (wall soil at `soilTemp <= 2`, $50^\circ\text{F}$ or below). |
| **Seed Bank** | `p` $\rightarrow$ `e` | Seed bags (corn, wheat, clover, peas, squash). | Dry pantry or homestead chest. | Small volume; stored in cloth/leather sacks inside bins. |
| **Refuse & Manure** | `p` $\rightarrow$ `r` | Crop residue, spoiled grain, stable manure, wood ash. | Downwind pit / compost heap. | Decomposes into rich humus (`organicMatter`) and fertilizer. |

### 1.2 Granular Stockpile Filters & Routing
* **Item & Quality Filters:** Stockpiles can be fine-tuned via menus (e.g., a cellar stockpile set to *only* accept Salt Pork and Seed Potatoes).
* **Give / Take Links:** Players can link stockpiles in logistical chains:
  * *Example:* A temporary "Field Harvest Dump" by the cornfield is set to **Give To** the "Main Homestead Corn Crib", prompting settlers with wagons to bulk-transfer corn at the end of the workday.

---

## 2. Physical Loose Items in the Voxel Grid

Loose items resting on the ground are physical entities represented in the voxel grid:
* **Voxel Representation:** `matterState = 5 (Loose Item)`.
* **Entity Pointer:** `entityRef` indexes the owning chunk’s lookup list, pointing into the global `itemTable[itemId]`.

### 2.1 Item Data Structure (`itemTable[itemId]`)
```rust
pub struct Item {
    pub id: u32,
    pub item_type: ItemType,      // Log, Plank, GrainSack, Tool, Foodstuff
    pub material_id: u8,          // WhiteOak, RedPine, Fieldstone, Iron
    pub weight_lbs: u16,          // Governs settler carrying speed & stamina drain
    pub volume_cu_in: u32,        // Governs container capacity
    pub quantity: u16,            // Stack size (e.g. 50 ears of corn, 1 beam)
    pub condition: u8,            // 0..100% durability / wear
    pub moisture_rot: u8,         // 0..15; rises if left in rain or flooded cellar
    pub container_id: Option<u32>,// Present if stored inside a barrel/crate
}
```

---

## 3. Containers & Storage Vessels

Without containers, storing 2,000 ears of corn would clutter 2,000 separate $6\text{"}$ voxels across the floor. **Containers solve tile explosion** by aggregating multiple items into a single voxel:

| Container | Construction Material | Typical Contents | Capacity / Role |
| :--- | :--- | :--- | :--- |
| **Oak Cask / Barrel** | Oak Staves + Iron Hoops | Pickled beef, salt pork, cider, lard, water. | Airtight & watertight; protects against flood water (surface or room water). |
| **Woven Basket** | Willow / Ash Splints | Fresh apples, squash, hand-picked corn ears. | Lightweight; used during manual field harvesting. |
| **Burlap / Cloth Sack** | Woven Flax / Hemp | Grain seed, flour, dried beans, cornmeal. | Keeps seeds sorted; permeable (must stay dry). |
| **Wooden Storage Chest** | Pine Boards + Iron Latch | Iron tools, nails, horse tack, pioneer clothing. | Settler personal and tool storage inside cabins. |

### 3.1 Container Stacking & Placement
* A single $6\text{"} \times 6\text{"}$ voxel can hold one standard barrel or basket.
* When stored in a designated stockpile, settlers automatically nest seed sacks into bins or stack barrels up to 2 levels high ($12\text{"}$ vertical).

---

## 4. Hauling Logistics, Stamina & Transport

Moving hundreds of pounds of timber, stone, and grain is central to the frontier experience:

```
                  Hauling Progression
  ┌────────────────────────────────────────────────────────┐
  │ 1. Bare Hands   ──> Heavy, slow, high stamina drain    │
  │ 2. Wheelbarrow  ──> Single settler, moderate loads     │
  │ 3. Oxen Travois ──> Skids heavy logs across fields     │
  │ 4. Farm Wagon   ──> Massive capacity, creates ruts     │
  └────────────────────────────────────────────────────────┘
```

1. **Bare Hands:** 
   * A settler carries one medium item (e.g., a $40\text{ lb}$ seed sack or small pine post). 
   * Attempting to carry a green oak log ($150+\text{ lbs}$) severely penalizes movement speed and rapidly depletes stamina.
2. **Wheelbarrows:**
   * Built at the carpenter’s bench (`matterState = Built Structure`).
   * Allows a single settler to transport heavy stones or several bags of grain at full walking speed.
3. **Oxen Draft & Travois:**
   * A draft ox hooked to an A-frame wooden travois can drag full-length tree trunks directly out of the forest to the timber stockpile.
4. **The Farm Wagon:**
   * Carries up to $2,000\text{ lbs}$ (dozens of bushels of wheat, multiple barrels, or entire building frames).
   * **Terrain Feedback:** Driving a loaded wagon across soft, waterlogged clay (`poreWater > 12`) packs the earth into deep wheel ruts (`compaction = 15`, lower ground), and runs the risk of getting bogged down in mud.
   * **Width:** A wagon needs a gap about $6\text{ ft}$ (12 voxels) wide. Gates, lanes, and barn doors must be built wide enough for it (doc 01).

---

## 5. Spoilage, Weathering & Thermal Preservation

Frontier food and materials degrade if neglected:

* **Rain & Flood Spoilage:**
  * Loose wheat or flour sacks left in an open-air stockpile during rain, or sitting in surface or room water, absorb moisture.
  * Within 48 game hours, damp grain develops fungal mold (`moisture_rot = 15`), rendering it inedible and turning it into compost refuse.
* **The Root Cellar Advantage:**
  * Warm summer surface temperatures ($80^\circ\text{F}+$) rot fresh potatoes and turnips in days.
  * In an underground cellar excavated $6\text{–}8\text{ ft}$ beneath the earth, the high thermal inertia of the soil keeps a steady cool temperature (`soilTemp` band 2, $40\text{–}50^\circ\text{F}$).
  * Root crops stored in dark, cool cellars remain edible throughout the entire winter and into next spring.
* **Firewood Seasoning:**
  * Freshly felled green timber has high internal moisture. Burning green wood in the hearth produces heavy creosote, poor heat, and noxious smoke.
  * Stacked in a covered outdoor timber yard, cordwood loses moisture over 1–2 seasons, becoming seasoned firewood.
