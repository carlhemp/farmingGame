# 07 — Camera, ASCII Rendering & Terminal UI

## 1. The 7-Foot Terrain-Conforming Camera Slice

In traditional top-down grid games (e.g., standard *Dwarf Fortress*), viewing the world via a flat horizontal Z-plane cuts undulating hills into concentric elevation rings. On a frontier farm with $256\text{ ft}$ of rolling hills and creek ravines, a flat slice would display the roof of a valley cabin while simultaneously showing the deep bedrock cellar of an uphill neighbor.

To solve this, *Frontier Farming Simulator* introduces the **7-Foot Terrain-Conforming Surface Slice**:

```
                       7-Foot View Envelope
                       ┌────────────────────┐
                       │ Canopy (Clipped)   │
  Roof (Clipped)       │                    │
  ┌────────────┐       │                    │
══╡░░░░░░░░░░░░╞═══════╡░░░░░░░░░░░░░░░░░░░░╞══════ Top of 7-Ft Envelope
  │ Barn First │       │ Pioneer (@)        │
  │ Floor Room │       │ Wheat Crop (\|/)   │
  │ Soil / Rug │       │ Topsoil / Furrows  │
══╡════════════╡═══════╡════════════════════╞══════ Generated Ground Height
  │ Cellar     │       │ Subsoil & Roots    │
  └────────────┘       └────────────────────┘
```

### 1.1 Column-Independent Surface Evaluation
* For every $(x, y)$ column in the visible viewport, the band starts at the column's **generated ground height**: the ground as the world generator made it, before any digging or building.
* The camera shows a vertical band from that height up to $7\text{ ft}$ above it ($14$ vertical $6\text{"}$ voxels).
* **Why generated ground, not the current top voxel:**
  * Over a cabin, the top solid voxel is the roof. Starting the band there would show only roofs.
  * If digging moved the band, the view would jump around while you dig.
* **What is Visible:** Ground soil, tillage furrows, fences, full-height settlers ($5.5\text{–}6\text{ ft}$), livestock, and first-floor interior spaces.
* **What is Clipped:** Upper tree canopies, high rafters, and roof shingles that would otherwise occlude gameplay.

### 1.2 Which Glyph Wins
Each column shows one glyph. The renderer picks it in this order:
1. **Agents and items:** a settler, animal, wagon, or loose item in the band is drawn on top, so a branch at $6\text{ ft}$ never hides the settler under it.
2. **Surface water:** if the column has standing water, it is drawn over the ground.
3. **Top voxel in the band:** the highest non-air voxel inside the band.
4. **Below the band:** if the whole band is air (for example over a dug cellar), a dimmed "below" glyph shows what is under it, the way DF draws lower levels. Switch to Z-slice mode (section 2) to look inside.

---

## 2. Flat Z-Slice Inspection Mode

While the terrain-conforming slice is the default overland view, players can toggle into **Flat Z-Slice Inspection Mode** (`Tab`):
* Switches to a strict horizontal Cartesian plane.
* Step up and down in **$6\text{"}$ increments** using `<` and `>` (or scroll wheel).
* **Primary Uses:**
  * Inspecting subterranean root cellars, dug wells, and underground perched water tables.
  * Inspecting second-story barn lofts and roof structures.
  * Analyzing vertical soil profiles (A-horizon topsoil, B-horizon clay subsoil, bedrock).

---

## 3. Multi-Scale Discrete Zoom

The player can zoom seamlessly between three operational scales:

```
┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
│     1:1 Detail View     │  │   Homestead Aggregate   │  │     Valley Overview     │
│ 1 cell = 6" x 6"        │  │ 1 cell = 4 ft x 4 ft    │  │ 1 cell = 32 ft x 32 ft  │
│ - Individual furrows    │  │ - Whole cabin & barn    │  │ - 500-acre watershed    │
│ - Seedlings & postholes │  │ - Field boundaries      │  │ - Regional river system │
│ - Micro-wheel ruts      │  │ - Pasture enclosures    │  │ - Claim boundaries      │
└─────────────────────────┘  └─────────────────────────┘  └─────────────────────────┘
```

1. **Detail View (1:1):** Every terminal character represents exactly one $6\text{"} \times 6\text{"}$ voxel. Used for precise construction, tool digging, and crop inspection. Terminal cells are about twice as tall as wide, so this view looks stretched north-south. Drawing each voxel as two characters side by side fixes that, at the cost of half the view width.
2. **Homestead Aggregate View ($8 \times 8$ Voxels / $4\text{ ft}$):** Cells aggregate soil and structure states. Used for laying out multi-acre crop fields, roads, and pasture fence lines.
3. **Valley Overview ($64 \times 64$ Voxels / $32\text{ ft}$):** Displays the full 500-acre valley. Shows macro topography, watershed hydrology, forest distribution, and property claim borders.

---

## 4. Procedural Soil RGB & Surface Glyphs

### 4.1 Continuous Soil Color Interpolation
Terminal cells use 24-bit TrueColor to communicate soil composition at a glance:
* **Base Color:** Dynamically blended from soil texture:
  * Sand: Pale tan / straw (`#D2B48C`).
  * Clay: Ochre / reddish-brown (`#A0522D`).
  * Silt: Warm brown (`#8B5A2B`).
* **Organic Matter Darkening:** Higher `organicMatter` shifts the base RGB toward deep, rich black-brown (`#2A1810`).
* **Moisture Darkening:** Absorbed `poreWater` darkens the color value:
  $$\text{Rendered RGB} = \text{Base RGB} \times \left(1.0 - 0.4 \times \frac{\text{poreWater}}{15}\right)$$

### 4.2 Micro-Relief Glyph Table
When viewing the overland surface, the terminal glyph communicates physical micro-relief:

| Glyph | Representation | Physical Trigger |
| :---: | :--- | :--- |
| `.` | Flat, undisturbed soil | Ground about level with neighbors, `compaction == 4..8` |
| `~` | Tilled plow furrow | `compaction == 0`, ground ripples between neighbors |
| `_` | Packed wagon wheel rut | `compaction >= 12`, ground lower than neighbors |
| `▲`, `▼` | North/South slope gradient | Large ground height difference to the north or south |
| `◄`, `►` | West/East slope gradient | Large ground height difference to the west or east |
| `~` (Blue) | Standing surface water | Surface layer water depth $> 0$ |
| `·` (White) | Snow cover | Surface layer snow depth $> 0$; denser shading for deeper snow |
| `=` (Pale Blue) | Pond ice | Surface layer ice thickness $> 0$ |
| `"` | Prairie grass or pasture | Surface layer `grass` above a threshold; color shifts from green to tan as `grass_dry` rises |
| `*` | Sown seed / sprout | `plantTable` growth stage 0–1 |
| `|` | Mature corn stalk | `plantTable` growth stage 4 |
| `#` | Hewn timber wall | `matterState = Built Structure`, `materialId = oak` |
| `+` | Wooden cabin door | Structure door voxel |
| `@` | Pioneer settler | Active settler avatar |

---

## 5. Terminal User Interface (`ratatui`)

The terminal frontend is laid out into functional panels:

```
┌────────────────────────────────────────────────────────┬──────────────────────┐
│  FRONTIER FARMING SIMULATOR — Year 1854, Late Spring   │ HOMESTEAD STATUS     │
├────────────────────────────────────────────────────────┤ Settlers: 3          │
│                                                        │ - Jedediah  (Plowing)│
│               ..~~..~~..~~..~~                         │ - Margaret  (Sowing) │
│               ..~~..~~..~~..~~                         │ - Caleb     (Hauling)│
│               ................                         ├──────────────────────┤
│               ====+===========                         │ SELECTED VOXEL (1:1) │
│               #..............#                         │ Pos: (428, 612, 18)  │
│               #   @..........#                         │ Soil: Sandy Loam     │
│               #..............#                         │ Clay: 13%  Sand: 60% │
│               ================                         │ Humus: 5   Bulk: Low │
│                                                        │ Moisture: 10 (Field) │
│                                                        │ N:6/7 P:2/3 K:3/3    │
├────────────────────────────────────────────────────────┴──────────────────────┤
│ [1] Normal  [2] Fast  [Space] Pause  |  [d] Designate  [b] Build  [Tab] Z-Slice│
└───────────────────────────────────────────────────────────────────────────────┘
```

* **Left Viewport:** Interactive ASCII map with smooth pan and multi-scale zoom.
* **Top-Right Panel:** Settler health, current jobs, active tools, and inventory.
* **Mid-Right Panel:** Deep inspection readouts for the voxel under the cursor (chemistry, moisture, structural stress).
* **Bottom Status Bar:** Game clock, season, weather conditions, simulation speed, and modal keybind hints.
