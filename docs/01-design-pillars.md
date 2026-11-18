# 01 — Design Pillars & Core Gameplay

## 1. Core Vision & Setting
*Frontier Farming Simulator* is an uncompromising agronomic and homesteading simulation set on the 19th-century American frontier. Unlike casual farming games where crops grow on fixed timers and terrain is an immovable tilemap, this simulator treats the earth as a living, physically accurate medium. 

Every furrow cut by a plow alters surface drainage; every wagon driven over wet clay packs the soil into an impermeable hardpan; every tree planted draws nitrogen and anchors hillsides against erosion. The game blends the granular physics of high-fidelity environmental modeling with the emergent colony management of *Dwarf Fortress*.

---

## 2. Settler Labor Model (Autonomous Pioneers)

The player does not directly control an avatar or execute actions by clicking tools on the ground. Instead, the homestead is operated by an **autonomous pioneer family / settler group** working through a **designation and job-queue system**.

```
  Player Designations (TUI)
  [ Plow Field ]  [ Fell Trees ]  [ Build Cabin ]  [ Harvest Corn ]
           │
           ▼
     Homestead Job Queue ─── Priority Scheduler
                                    │
                  ┌─────────────────┴─────────────────┐
                  ▼                                   ▼
        Settler: Jedediah                   Settler: Margaret
        - Health: 100%                      - Health: 100%
        - Stamina: 42% (Tired)              - Stamina: 88% (Energetic)
        - Inventory: [Iron Axe, Log]        - Inventory: [Seed Bag (Wheat)]
        - Current Job: Haul Timber          - Current Job: Sow Field
```

### 2.1 Autonomous Agent Capabilities
Each settler is a physical agent in the world. Agents are **not stored in the voxel grid**: they live in an agent table with a position and a footprint, so moving them never rewrites voxels.
* **Physical Presence & Movement:** Settlers stand $5.5\text{–}6\text{ ft}$ tall ($11\text{–}12$ vertical voxels) and pathfind across the terrain. They navigate slopes, furrows, step over low fences, climb ladders, and avoid deep water or unstable landslide zones.
* **Footprint & Clearance:** Agents take up real width, measured in voxels:

  | Agent | Footprint | Needs a gap at least |
  | :--- | :---: | :---: |
  | Person | about $3 \times 2$ voxels ($18\text{"} \times 12\text{"}$) | 3 voxels ($18\text{"}$) |
  | Ox | about $6 \times 16$ voxels | 6 voxels ($3\text{ ft}$) |
  | Farm wagon | about $12 \times 24$ voxels | 12 voxels ($6\text{ ft}$) |

  Pathfinding uses a clearance map, so each size class only goes where it fits. A $3\text{ ft}$ cabin door is about 6 voxels wide. A gate must be wide enough for the wagon.
* **Inventories & Physical Logistics:** Settlers cannot summon materials out of thin air. To build a fence, a settler must walk to a timber stockpile, pick up pine rails, carry them to the designated site, and assemble them voxel by voxel.
* **Stamina & Physical Exertion:**
  * Plowing dense, wet prairie sod requires tremendous draught force. Settlers (and oxen) deplete stamina rapidly when working compacted clay compared to light, friable sandy loam.
  * When exhausted, settlers must rest, eat, and sleep.
* **Warmth, Shelter & Health:**
  * Winter on the frontier is lethal. Settlers exposed to sub-zero soil/air temperatures suffer hypothermia and frostbite.
  * Settlers require an enclosed room (doc 04, section 6) with a hearth or stove for warmth.

### 2.2 The Designation System
The player interacts through a keyboard-driven cursor interface (classic *Dwarf Fortress* modal workflow):
* `d` $\rightarrow$ `p`: Designate area to plow.
* `d` $\rightarrow$ `s`: Designate area to sow (select crop seed from stockpile).
* `d` $\rightarrow$ `t`: Designate trees to fell.
* `b` $\rightarrow$ `w`: Designate wall placement (drag rectangle, select material).
* `b` $\rightarrow$ `f`: Designate fence line (drag line).
* `d` $\rightarrow$ `c`: Designate excavation / cellar dig.

Settlers evaluate the global job queue based on urgency, skill, proximity, and daylight, claiming jobs and executing them autonomously.

---

## 3. Game Loops

### 3.1 The Daily Loop
* **Dawn Assessment:** Check the morning weather, air temperature, soil frost depth, and surface moisture. Review what happened overnight (did heavy rain flood the cellar? did an embankment slip?).
* **Chore Allocation:** Settlers wake, eat breakfast from food stores, tend draft animals, and begin their assigned queues.
* **Physical Work & Impact:** As work proceeds, the world responds immediately:
  * Wagon wheels create compacted ruts (`compaction = 15`, slightly lower ground) that alter rain runoff.
  * Plows turn topsoil, lowering bulk density (`compaction = 0`) and creating furrows (ripples in the surface layer's ground height) that catch water.
* **Evening Recovery:** Settlers return to the homestead before nightfall, store tools in the shed, cook, and sleep.
* **Night Planning:** While the family sleeps, the player can lay out fields, buildings, and stockpiles for tomorrow. Night can play through, run fast, or be skipped; that is a setting (doc 06).

### 3.2 The Seasonal Loop
Seasons are short, about 12 days each, so a year is about 48 days (doc 06). Each day is still a full, real-length day.

* **Spring (Thaw & Planting):**
  * Soil temperature rises above the germination threshold. 
  * Melted snow saturates the soil (`poreWater = 15`), creating mud that bogs down wagons and makes plowing difficult until fields dry out to field capacity.
  * Plant early cool-season crops (wheat, oats) followed by warm-season crops (corn, squash).
* **Summer (Growth & Maintenance):**
  * Monitor soil moisture and crop nutrient depletion (N, P, K).
  * Hoe weeds, irrigate from nearby streams or dug ponds during dry spells.
  * Cut and cure meadow grass for hay to sustain livestock through winter.
* **Autumn (Harvest & Preservation):**
  * Race against the first killing frost.
  * Harvest grain and corn, haul ears to the crib, thresh wheat, and pack root crops (potatoes, turnips) into dry root cellars.
  * Plant winter cover crops or spread manure/compost to restore organic matter.
* **Winter (Survival & Maintenance):**
  * Ground freezes down to the frost line (`isFrozen = 1`). Infiltration ceases; snow blankets the land.
  * Settlers chop firewood, repair tools, mend harness, thresh grain indoors, and care for penned livestock.

### 3.3 The Multi-Year Homestead Loop
* **Sod Dugout to Heavy Timber Barn:** Begin with a crude, hand-dug sod shelter cut into a hillside. As timber is cleared and fields produce surplus grain, expand into hewn-log cabins, stone cellars, and timber-frame barns.
* **Soil Stewardship:** Avoid frontier "soil mining." Corn planted three years consecutively strips soil nitrogen, dropping yields to zero. The player must rotate legumes (clover, beans) and apply wood ash/manure to maintain long-term fertility.
* **Claim Expansion:** Accumulate capital and grain reserves to purchase adjacent 40- or 80-acre parcels from the land office, scaling from a subsistence homestead to a productive frontier farm.

---

## 4. The Town and Trade (Abstracted Market)

To keep CPU budgets focused on deep physical simulation, **the town is not simulated as a living voxel settlement**. It functions as an off-map trade depot and regional hub:

* **The Travel Trade-Off:** Traveling to town takes 1–3 full days by wagon or foot. Sending a settler and wagon to town means taking an essential laborer off the fields at a crucial point in the season.
* **The Arrival Calendar & Seasonal Stock:**
  * The town general store and livery operate on seasonal shipments (riverboats or freight wagons).
  * If the player arrives too late in spring, seed corn or plowshares may be completely sold out.
* **Dynamic Supply & Demand:**
  * Grain and pork prices fluctuate based on regional frontier harvests and transport availability.
  * Selling a bumper crop of wheat drops local prices, incentivizing diversification (wool, smoked bacon, cider).
* **Town Development:** The town develops organically over the years based on regional settlement and commerce (doc 16). Rather than arbitrary player unlock gates, businesses open independently: an independent grist mill makes flour milling cheaper and faster, a local blacksmith eliminates distant freight markups on iron tools, and eventual railroad arrival opens regional export markets. Everything physically possible on the frontier is open from Day 1 if you have the resources and physical labor to do it.
