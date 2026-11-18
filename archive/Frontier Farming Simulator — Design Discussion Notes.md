# **Frontier Farming Simulator — Design Discussion Notes**

Companion notes to *Frontier Sim Architecture.md*, captured from a voice discussion on November 18, 2026\. Covers feasibility, world generation, trees, structures, game loops, and the town.

## **1\. Feasibility and Memory Budget**

* A fully dense 500-acre grid at 6" resolution is about 44.6 billion voxels. At 10 bytes each, that is roughly **446 GB**, so a dense grid is off the table.  
* Storing only real soil (about 6 ft of soil plus the 7 ft surface slice) brings the full 500 acres to roughly **17–23 GB**. Chunks can stream to disk so only the area near the player is resident.  
* A 120-acre starting claim is about a quarter of the footprint: roughly **4–6 GB** for soil layers, with CPU load dropping proportionally.  
* **CPU is the harder limit than memory.** If a quarter of the farm is awake, that is around 700 million voxel updates per tick. Realistic target: a few percent of the world awake at once, with a tick budget.  
* Far-from-player chunks can run at coarser simulation or stay frozen.

## **2\. Resolution: 6" vs 1 ft**

* Going to 1 ft cuts voxel count by 8x (about 2–3 GB for the full world) and cuts CPU by the same factor. This is Gemini's argument.  
* 6" is better for what makes a farm feel like a farm: fence posts, one plant per voxel, wheel ruts, tilled rows.  
* **Decision: stay at 6" as the base resolution** for storage, rendering, building, and plants.  
* **Optional later optimization:** run water and chemistry on coarser blocks (1–2 ft), with each coarse cell governing several fine voxels. Requires downsample, coarse physics, and push-back to fine voxels. Carry extra values (minimum permeability, ditch presence) so narrow features like ruts and ditches still steer flow.  
* Build the fine-grained version first, behind a clean interface where the physics reads and writes through a small set of functions, so a coarse version can be swapped in later (estimated a few weeks of work, not a rewrite).

## **3\. Water Simulation**

* A stream and irrigation lines are a permanent source of awake chunks, since flowing water never sleeps like dry prairie.  
* Model the stream and ditches as a **flow network** (graph of segments with flow rates) rather than per-voxel moving water. Only the thin ribbon of soil beside them gets full infiltration and seepage treatment.  
* Run subsurface water on a **slower tick** (hours or days of in-game time) than surface rain and runoff.  
* Cap how many chunks can wake per tick, so a flood or breached dam catches up gradually instead of waking half the map at once.

## **4\. Claims and Expansion**

* Players start with a claim (around 120 acres, in the style of Go West) and can purchase more land later.  
* Chunks that do not exist cost nothing, so only owned land is generated and simulated.  
* Neighboring land runs as cheap, frozen terrain that only feeds water in at the borders, and is promoted to full simulation when purchased.  
* **Open decision:** one continuous world with ownership layered on top (keeps rivers and hills consistent across parcels) versus generating each parcel fresh at purchase. Also open: whether upstream and downstream neighbors matter, or the farm is effectively its own island.

## **5\. World Generation**

* Dwarf Fortress-style approach: a **coarse world map first**, then detailed generation of the chosen site.  
* Fully **deterministic and seeded**: every detail is a pure function of one world seed plus coordinates. No stored randomness. Layered noise (such as simplex) at multiple scales.  
* The coarse map acts as constraints (elevation, river paths, soil type); the detail pass only adds variation within them, so fine detail never contradicts the coarse map.  
* Store only the coarse map and the player's changes. Untouched land regenerates on demand, which keeps saves small.  
* **Recommended coarse layers (richer option):** elevation and rivers, soil type, rainfall and temperature, vegetation cover (prairie, forest, wetland).  
* These layers feed voxel fields directly: soil type seeds clay, sand, and organic matter; rainfall seeds water tables; temperature seeds soil temperature and frost line.  
* Generate the coarse map once and treat it as fixed. Do not simulate weather systems or erosion across the coarse map over time.

## **6\. Plants and Trees**

* A tree is a record in the **plant side-table** (species, age, height, trunk diameter, canopy size and density, root depth). Voxels point back to it via `entityRef`.  
* Voxels are stamped from the description when needed for rendering, collision, or an axe hit.  
* **Wind damage** works on the object: force is wind speed times canopy area, applied at canopy height, giving a bending torque at the base.  
* Compare that torque to two strengths: **trunk strength** (diameter and species) and **root anchorage** (root depth and soil saturation via `poreWater`). Wet soil is when trees topple.  
* If the trunk is weaker, it snaps, usually low on the trunk or at a weak point such as a fork or rot. If the roots are weaker, the whole tree uproots.  
* A fallen tree becomes a loose object of a few segments with simple rigid-body physics, then re-stamps voxels where it lands.

## **7\. Structures and Collapse**

* Players place structures voxel by voxel, but behind the scenes connected pieces are grouped into a **structure record**, modeled as a graph of posts, beams, wall panels, and joints, each with material, size, and strength.  
* Collapse is a **load-flow problem**: each piece carries its own weight plus what rests on it and passes load to supports and down to the ground. If a piece exceeds its strength, it fails, load redistributes, and the check repeats. That cascade produces a falling beam followed by the roof.  
* Use simplified hand-calc statics, not full finite element analysis: tributary loads, bending and shear checks against span tables, and reactions passed down to posts. Tabulate allowable spans and capacities per material and size.  
* **Event-driven checks:** recompute when something changes (new beam, snow load, rotted joint), plus a slow background sweep of a small slice of structures each frame to catch decay and weather.  
* **Selective escalation:** run a small truss or frame solver only when a structure is near failure or something dramatic happens.  
* Foundations press on the soil voxel below, so wet clay under a heavy footing can settle and crack the structure.

## **8\. Game Loops**

* **Daily loop:** check weather and soil, plant, water, repair, tend animals, and see the world respond.  
* **Seasonal loop:** plan what to plant where, rotate crops so nitrogen recovers, and get the harvest in before frost.  
* **Long loop:** expand the claim, upgrade from a sod house to a real barn, and recover from setbacks.  
* The core appeal is **building, shaping the land, and iterative improvement**, where the player's actions matter. Failures should trace back to player choices (corn three years in a row depletes nitrogen; skipped drainage floods the cellar).  
* Tension between thrift and ambition: a frontier winter punishes overextension.

## **9\. Town and Trade**

* The town is **not simulated**: a place the player travels to from the claim, similar to the trade depot in Dwarf Fortress.  
* Model it as a market with a few levers: a calendar of arrivals, a stock level per item, and a price that rises as stock drops. Arriving too late in the season means no seed corn.  
* The town **develops over time**, tied to progress: more settled claims or goods sold unlock new shops (blacksmith, mill, rail depot).  
* Trips to town are a real decision, since a day traveling is a day not working the land.

## **10\. Development Approach**

* Side project, built mostly through Antigravity on the free tier, so progress will be gradual and limited by usage caps.  
* Build in slices. Sparse chunks and deterministic generation mean the whole system never has to work at once.

## **11\. Open Questions and Next Steps**

* Continuous world versus per-parcel generation; whether upstream and downstream neighbors matter.  
* Memory budget for the maximum number of awake chunks.  
* Check field widths against dense cases: N, P, K at 3 bits (8 levels each), and `entityRef` at 16 bits (about 65,000 entities per chunk), for example in a dense orchard or large barn.  
* Draft the structures section in detail: load rules and a starter material table (oak, pine, sod).  
* Sketch the mapping from coarse map layers to voxel fields.  
* Flesh out the game loop and town sections.

