# Frontier Farming Simulator

A deep, deterministic 6-inch voxel farming and homesteading simulation, planned in Rust. Set on the American frontier, *Frontier Farming Simulator* combines realistic soil chemistry, hydrology, and structural mechanics with the colony management and designation tools of *Dwarf Fortress*.

**Status:** design stage. There is no code yet. See [docs/09-build-plan.md](docs/09-build-plan.md) for the build order.

---

## Core Premise & Architecture Highlights

* **True 6" Voxel Resolution:** The world is modeled as $6\text{"} \times 6\text{"} \times 6\text{"}$ cubic voxels, enabling plow furrows, wagon wheel ruts, post-hole fencing, and small crop patches.
* **64-Bit Packed Voxels (`u64`, 8 Bytes):** Each voxel packs matter state, material, soil texture (clay, sand, silt), organic matter, compaction, pore water, nutrients (N, P, K), pH, temperature band, a 16-bit entity reference, and 8 simulation flags.
* **2D Surface Layer:** Exact ground height and surface water depth per column, in $1/64\text{"}$ units. Gentle field slopes drain properly, creeks and ponds can be any depth, and rain only touches one cell per column.
* **Keep Only What Changed:** Untouched land is regenerated from the world seed. Sleeping chunks are saved as compressed differences. Only awake sub-chunks ($32\text{ KB}$ each) are held in full.
* **Autonomous Settlers (Dwarf Fortress Style):** You issue designations (plow field, fell timber, build sod wall, harvest wheat). Pioneer family members pathfind, haul physical resources, manage stamina and hunger, and do the work. Agents have real width, so doors and gates must fit them.
* **Creative Voxel Building:** No fiddly CAD pieces. Drag walls, fences, floors, and cellars voxel by voxel. Support spreads up from the ground and runs out with distance, depending on the material (sod, fieldstone, pine, oak). Unsupported voxels fall.
* **Hydrology:** Surface water flows between columns by water surface height, soaks in at a rate set by soil texture and compaction, and seeps through the soil by Darcy's Law on a slower step. Water is never created or lost at chunk borders.
* **Continuous Seeded World & Claims:** The entire 500-acre valley is generated deterministically from a single seed. Land claims are ownership boundaries; unowned land stays frozen terrain with fixed boundary inflow and slope-governed outflow.
* **7-Foot Terrain-Conforming Camera:** Rather than cutting flat horizontal planes through hills, the camera shows a 7-foot band above each column's generated ground height, so fences, crops, pioneers, and first-floor interiors are visible without roofs in the way.
* **Modular Rust Architecture:** A headless simulation library (`frontier_sim_core`) run in parallel with Rayon, separate from the terminal frontend (`frontier_sim_tui`).

---

## Technical Architecture Documentation

* [docs/architecture-summary.md](docs/architecture-summary.md)
  *Overview diagrams, risk list, and design status.*

1. [docs/01-design-pillars.md](docs/01-design-pillars.md)
   *Core fantasy, autonomous settler labor model, agent size and clearance, game loops, and the unsimulated town market.*

2. [docs/02-world-and-voxels.md](docs/02-world-and-voxels.md)
   *Spatial coordinates, storage model, sub-chunks, the 64-bit voxel layout (source of truth), the 2D surface layer, memory estimates, and entity tables.*

3. [docs/03-hydrology-and-soil.md](docs/03-hydrology-and-soil.md)
   *Surface water flow, infiltration, Darcy seepage, room water, conservation at borders, erosion, and soil mechanics.*

4. [docs/04-voxel-building-and-structures.md](docs/04-voxel-building-and-structures.md)
   *Free voxel construction, designation tools, support spreading, material reach, collapse, rooms, and foundation settling.*

5. [docs/05-plants-and-farming.md](docs/05-plants-and-farming.md)
   *Crop growth stages, N-P-K uptake, pH gating, tree root anchorage, wind toppling, and timber harvesting.*

6. [docs/06-sim-engine-and-concurrency.md](docs/06-sim-engine-and-concurrency.md)
   *Crate layout, time scale, multi-rate steps, sleep scheduling, two-phase Rayon execution, and determinism.*

7. [docs/07-camera-and-ui.md](docs/07-camera-and-ui.md)
   *7-foot terrain-conforming camera, which glyph wins, Z-slice mode, zoom levels, soil colors, glyphs, and the Ratatui TUI.*

8. [docs/08-logistics-stockpiles-and-inventory.md](docs/08-logistics-stockpiles-and-inventory.md)
   *Stockpiles, loose items, containers, hauling, and spoilage.*

9. [docs/09-build-plan.md](docs/09-build-plan.md)
   *Milestones, testing approach, docs still to write, and open questions.*

10. [docs/10-world-generation.md](docs/10-world-generation.md)
    *Coarse map, stream paths, soil profiles by landform, trees and stones, starting conditions, and choosing a claim.*

11. [docs/11-weather-and-climate.md](docs/11-weather-and-climate.md)
    *Climate targets, the short calendar, weather regimes, year types, snow and ice, drying, frost, wind, fire, hail, tornadoes, forecasts, and weather difficulty settings.*

12. [docs/12-save-file-format.md](docs/12-save-file-format.md)
    *Save folder layout, regions, what is saved and what is rebuilt, autosave, crash safety, one-life mode, versions, and sharing worlds.*

13. [docs/13-settlers-jobs-and-needs.md](docs/13-settlers-jobs-and-needs.md)
    *The scale rule, settlers, jobs and area jobs, work priorities, needs, skills, work rates, daily routine, rest day, pathfinding, health and illness, and the family over time.*

14. [docs/14-animals.md](docs/14-animals.md)
    *Oxen teams and rigs, pulling power, grazing and winter feed, pastures and shelter, animal health, breeding and products, and wild animals.*

15. [docs/15-fire.md](docs/15-fire.md)
    *Grass fuel load and dryness, prairie fire spread and wind direction, firebreaks and back-fires, building and chimney fire, settler response, fire as a land-clearing tool, and fire difficulty settings.*

16. [docs/16-town-and-economy.md](docs/16-town-and-economy.md)
    *Travel time and costs, the arrival calendar, stock and prices, selling grain and farm goods, the bumper-crop problem, the land office, hiring hands and breaking crews, town growth (mill, blacksmith, railroad, telegraph), and neighbors.*

17. [docs/17-tools-crafting-and-workshops.md](docs/17-tools-crafting-and-workshops.md)
    *Tool list and iron dependencies, condition and wear, breaking and re-hafting, the tool shed, splitting and hewing, sod cutting, fieldstone collection, the carpenter's bench and its outputs, skill and quality, and what must come from town.*

18. [docs/18-food-cooking-and-preservation.md](docs/18-food-cooking-and-preservation.md)
    *Food types and nutrition, daily needs by role, meals and cooking fuel, variety and spirits, the kitchen garden, autumn butchering, salting and smoking, churning, drying, and food storage.*

19. [docs/19-controls-ui-and-settings.md](docs/19-controls-ui-and-settings.md)
    *Modal keyboard controls, designate/build/stockpile/zone/workshop keys, inspect panel, work tab, job and alert lists, new-game flow, and consolidated difficulty/gameplay settings.*

20. [docs/20-neighbors-and-events.md](docs/20-neighbors-and-events.md)
    *Seeded valley settlement progression, neighbor profiles, mutual aid and barn-raisings, itinerant peddlers and drovers, letters and migrating kinfolk, cholera outbreaks, and locust swarms.*

21. [docs/21-technical-architecture-contracts.md](docs/21-technical-architecture-contracts.md)
    *Rayon double-buffered edge exchange protocol, Core-to-TUI WorldView and CommandQueue interfaces, AgentPos ground-binding coordinate rules, and EntityRef lifecycle transitions.*

---

## Historical Notes & Archives

Earlier design discussion notes and draft specifications are in the [archive/](archive/) directory. They are kept for history and may disagree with the current docs.
