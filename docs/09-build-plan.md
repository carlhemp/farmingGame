# 09 - Build Plan & Milestones

The full design is far too big to build at once. This plan builds it in slices. Each milestone ends with something that runs and can be tested. The sparse, seeded world means the whole system never has to work at once.

The **first playable** version is the end of M6: a small farm you can plow, plant, water, and harvest by designation, with no settlers yet.

---

## Milestones

| # | Milestone | What gets built | Done when |
| :---: | :--- | :--- | :--- |
| **M0** | Workspace & voxel basics | Cargo workspace (`frontier_sim_core`, `frontier_sim_tui`). `Voxel` pack/unpack for every field in doc 02. `SubChunk` enum. Fixed-point unit types. | Round-trip tests pass for every field and every edge value. |
| **M1** | World generation & viewer | Coarse map and detail generation (doc 10) for a small test map (about 6 acres, $512 \times 512\text{ ft}$). TUI viewer with soil colors, pan, zoom, glyph rules (doc 07), and the inspect panel. | Same seed gives the same map every time. The viewer runs smoothly. |
| **M2** | Surface water | 2D surface layer, rain, infiltration, virtual-pipe flow, single-threaded first (doc 03). Rain comes from simple scripted test weather until the full generator in M9. | Puddles fill low spots, a 1% slope drains, and total water only changes by rain in, boundary out, and infiltration. |
| **M3** | Tiles, sleep & threads | Tiles and sub-chunks wake and sleep. Rayon two-phase steps. Scale up to a 120-acre claim. | Same result with 1 thread and with all threads. Real memory and step costs measured, and the day length in doc 06 tuned. |
| **M4** | Save & load | Compressed differences from the seed, generator version, save folder and regions, autosave, one-life mode (docs 02, 12). | Save, quit, load gives an identical world. Save size measured on a worked claim. |
| **M5** | Subsurface & soil | Hourly percolation, Darcy seepage, soil heat bands, freezing and thawing, erosion and deposition. | A ditch lowers the nearby water table. Ground freezes in winter and thaws in spring. |
| **M6** | Crops (first playable) | One crop (corn) with water and N stress, then wheat and clover rotation. Designations happen instantly, with no settlers. | Corn three years running drops yields. Clover brings them back. |
| **M7** | Settlers & jobs | One settler, footprint and clearance map, two-level pathfinding, jobs and area jobs, work priorities, basic needs, hauling, basic stockpiles, a team of oxen and a plow rig (docs 01, 08, 13, 14). | A settler plows a designated field and hauls the harvest to a stockpile. |
| **M8** | Building | Free voxel building, support spreading, collapse, rooms (doc 04). | The 24 ft barn example behaves as written. A closed cabin becomes a room. |
| **M9** | Weather & seasons | Weather generator (doc 11), snow, frost, wind; the short-calendar year loop; the night setting (play through, fast, skip); forecasts and the forecast accuracy setting; hail and tornadoes. | A full 48-day year runs with plausible weather and the seasonal loop in doc 01. |
| **M10+** | The rest | Trees and wind damage, more crops, more animals, predators, and grasshopper plague (doc 14), wagons, fire, the family over time, the town. | One at a time, each with its own test. |

---

## Testing Approach

The simulation core is headless (doc 06), so most testing needs no UI.

* **Unit tests:** voxel packing, fixed-point math, generator determinism.
* **Conservation tests:** total water in a closed test map stays exactly the same over thousands of steps.
* **Determinism tests:** the same seed and inputs give the same world with 1 thread and with all threads.
* **Golden-seed snapshots:** a few fixed seeds whose results are saved. Any change that alters them must be on purpose.
* **Agronomy scenarios:** scripted runs such as "corn three years running" with expected yield trends.

---

## Docs Still to Write

| Doc | Covers | Needed by |
| :--- | :--- | :--- |

---

## Settled Decisions (Locked for Coding)

1. **Parcels & Claim Size (doc 10, doc 02):** 40-acre parcels ($1,320 \times 1,320\text{ ft}$, $165 \times 165$ chunks of $8\text{ ft}$). The starting claim is 3 adjacent parcels (120 acres).
2. **Stream Topography (doc 10):** A single main creek traversing the valley plus 1–2 seeded side springs/draws. No initial post-generation erosion pass; mathematical spline and noise meanders provide natural drainage.
3. **The Scale Rule & Cropping Footprint (doc 13):** Physical work rates remain realistic. A subsistence family naturally cultivates ~3 to 5 acres of crops, with the rest of the 120-acre claim supporting pastures, hay meadows, woodlots, and fire buffers. **No artificial game limits:** ambitious players can scale up to commercial multi-parcel farming by hiring breaking crews, buying extra oxen, and employing seasonal labor.
4. **Draft Animal & Contractor Progression (doc 14, doc 16):** Oxen are the baseline draft animal. Horses are available in town from Year 1 but cost \$80–\$120. Hired breaking crews are available in town from M7 to break heavy prairie sod for cash (\$2–\$4/acre).
5. **Crop Storage Architecture (doc 05):** Agricultural fields use a packed 2-byte grid per column in awake sub-chunks (crop type, growth stage, stress). Individual heap records in `plantTable` are reserved strictly for multi-voxel trees and perennial shrubs.
6. **Concurrency & Buffer Safety (doc 21):** Rayon execution uses double-buffered outbound edge slots (`TileEdgeBuffers`) to eliminate data races and borrow-checker conflicts without mutexes.

---

## Open Questions (To Be Tuned in Milestones)

1. **Surface Flow Constant $C$ (doc 03):** To be tuned during M2 integration tests over a 1% slope to guarantee zero oscillation under the clamping rule.
2. **Awake Chunk Budget (doc 06):** Baseline cap of 2,048 awake surface tiles (~8 MB) and 1,024 3D sub-chunks (~32 MB) per step. Measured and refined in M3.
3. **Building Material Reach (doc 04):** Starting values (sod 1, fieldstone 8, pine 16, oak 32 voxels) to be tested in the M8 creative building prototype.
4. **Disease Dispersion (doc 13):** Starting with simple distance/slope well-risk calculation. Contaminant plume tracing through groundwater deferred to M10+ if needed.
