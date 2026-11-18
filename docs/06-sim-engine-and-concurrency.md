# 06 — Simulation Engine & Concurrency (Rust)

## 1. Rust Architecture & Crate Separation

The engine is built in native **Rust** for exact memory layouts, cheap bitfield packing, and race-free multi-core parallelism.

The project keeps simulation and presentation apart in a Cargo workspace:

```
                  ┌──────────────────────────────┐
                  │      frontier_sim_core       │
                  │   (Pure Headless Library)    │
                  │  - 64-bit Voxel Sub-Chunks   │
                  │  - 2D Surface Layer          │
                  │  - Hydrology & Chemistry     │
                  │  - Agents, Jobs & Building   │
                  │  - 2-Phase Rayon Ticks       │
                  └──────────────┬───────────────┘
                                 │
           ┌─────────────────────┴─────────────────────┐
           ▼                                           ▼
┌──────────────────────────────┐            ┌──────────────────────────────┐
│       frontier_sim_tui       │            │     frontier_sim_render      │
│  (Terminal Frontend)         │            │  (Future GPU/ASCII Window)   │
│  - ratatui / crossterm       │            │  - wgpu / glyph shaders      │
│  - Modal DF-style keybinds   │            │  - Smooth pixel scrolling    │
│  - 7-ft surface slice ASCII  │            │  - Shaded terrain lighting   │
└──────────────────────────────┘            └──────────────────────────────┘
```

* **No Leaky Abstractions:** `frontier_sim_core` does not import terminal, window, or input crates. It runs headless for automated tests (doc 09).

---

## 2. Time Scale

These are **working assumptions**, to be tuned once real costs are measured (milestone M3).

* **Base tick:** 1 game second.
* **Normal speed (1x):** 1 game day takes about 20 real minutes. That is $86,400 / 1,200 = 72$ ticks per real second.
* **Speed controls:** Pause (`Space`), 1x Normal (`1`, 72 ticks/s), 2x Fast (`2`, 144 ticks/s), 5x Harvest Speed (`3`, 360 ticks/s, a day in about 4 real minutes).
* **Short calendar:** Days are real length, but seasons are short: about 12 days each, so a year is about 48 days. A full year takes about 16 real hours at 1x, and a 3-year crop rotation fits in a few long play sessions at 2x to 5x. Crop growth and the climate curve are tuned to the season, not to real crop days (doc 05). Soil heat lag and frost depth are tuned to match. Food eaten, firewood burned, and work done per season all shrink by the same amount, so farms stay in balance but cover fewer acres (doc 13, section 2).

### 2.1 Night Setting
Night handling is a player setting, because night is also a good time to plan.

| Setting | What happens at night |
| :--- | :--- |
| **Play through** (default) | Night runs at the chosen speed. You can watch, plan, and designate while settlers sleep. |
| **Fast nights** | When every settler is asleep, the game switches to a high speed until dawn, then drops back to your speed. |
| **Skip nights** | When every settler is asleep, the game jumps to dawn, running the sim in fewer, bigger steps. |

* Designations, stockpile settings, and building plans can be made at any time, including while paused. Settlers pick up new jobs when they wake.
* Pressing a speed key or pause during a fast or skipped night stops it.

### 2.2 Multi-Rate Simulation

| Subsystem | Runs Every | Game Time | What It Computes |
| :--- | :---: | :---: | :--- |
| **Agents & Jobs** | 1 tick | 1 second | Settler, animal, and wagon movement; tool work; hauling. |
| **Surface Water** | 60 ticks | 1 minute (4 sub-steps) | Rain, infiltration, surface flow, erosion (doc 03). |
| **Weather** | 600 ticks | 10 minutes | Rain rate, wind, air temperature, snow (doc 11). |
| **Subsurface Water** | 3,600 ticks | 1 hour | Percolation, Darcy seepage, room water. |
| **Soil Heat** | 3,600 ticks | 1 hour | Temperature bands, frost line, freezing and thawing. |
| **Crop Biology** | Daily (at dawn) | 24 hours | Root water uptake, N-P-K use, growth; grass regrowth and drying (doc 14). |
| **Structures** | Event + daily | 24 hours | Support checks, rot, snow load, settling (doc 04). |

Surface water deliberately runs slower than real water. At 1x speed, real runoff across a field would finish in under one real second, so only where the water ends up matters (doc 03, section 3.2).

### 2.3 Rough Cost Check (estimate)
In a heavy storm with runoff on 20% of the 120-acre claim, about 4 million columns run the surface water step. At 1x that is 4 sub-steps every 0.83 real seconds, or roughly **20 million cell updates per real second**. That is easy on a modern multi-core CPU. At 5x it is about 100 million, which is still workable. Doing the same in 3D voxels would cost about 26 times more.

---

## 3. Active-Zone & Sleep Simulation

Running 500 acres of soil in 3D every tick would mean billions of voxel updates. The engine only runs what is active. The 2D surface tiles and the 3D sub-chunks wake and sleep **separately**.

```
 [ Sleeping Prairie ]  [ Sleeping Prairie ]  [ Sleeping Prairie ]  (Zero CPU Cost)
 [ Sleeping Prairie ]  [ AWAKE (Farm)     ]  [ AWAKE (Farm)     ]  (Rayon Par Iter)
 [ Sleeping Prairie ]  [ AWAKE (Farm)     ]  [ Sleeping Prairie ]
```

### 3.1 Surface Tiles (2D)
* **Wake:** standing water or runoff in the tile (rain falling faster than the soil absorbs it), water arriving from a neighbor tile, or a player change to the ground (plowing, digging, building).
* **Light rain** does not wake a tile. It is added straight into the top soil voxels.
* **Sleep:** no surface water anywhere in the tile.

### 3.2 Sub-Chunks (3D)
* **Wake:** for the hourly subsurface step, when the top soil got wetter than field capacity, when seepage arrives from a neighbor, or when a settler digs, builds, or plants in it.
* **Sleep:** moisture changes have settled and no jobs are active. A sleeping sub-chunk is stored as a compressed difference from the seed (doc 02).

### 3.3 Awake Cap & Deferred Catch-Up
* The scheduler caps how many tiles and sub-chunks can be awake per step, to protect the frame rate.
* If a flood or breached dam would wake hundreds at once, the extra ones go into a deferred queue with a "time owed" counter.
* Deferred units catch up later with fewer, bigger steps. The clamp rule keeps water conserved. Bigger steps are less accurate in how water moves, which is acceptable for catch-up.

---

## 4. Two-Phase Parallel Execution (`rayon`)

Each step has two phases. The unit of work is a surface tile for surface water, and a sub-chunk for subsurface water and heat.

```
               Phase 1: Compute (parallel, per tile or sub-chunk)
       awake_units.par_iter_mut() across all CPU cores
       ┌────────────────────────┐    ┌────────────────────────┐
       │ Tile (X, Y)            │    │ Tile (X+1, Y)          │
       │ - Rain & infiltration  │    │ - Rain & infiltration  │
       │ - Outflows (clamped)   │    │ - Outflows (clamped)   │
       │ Writes edge buffers ───┼───>│ Writes edge buffers    │
       └────────────────────────┘    └────────────────────────┘
                                    │
                                    ▼
               Phase 2: Apply (parallel, per tile or sub-chunk)
             - Add up arrivals from neighbor edge buffers
             - depth_new = depth - Σout + Σin
```

### Phase 1: Compute
* Each unit reads its own data plus a **read-only snapshot** of its neighbors' edge cells from the previous step.
* It computes outflows clamped to the water each cell holds ($\sum \text{out} \le \text{depth}$).
* Outflow that crosses an edge is written to the unit's own outbound edge buffer.

### Phase 2: Apply
* Each unit adds up the arrivals from its neighbors' edge buffers and updates its cells.
* Moving from phase 1 to phase 2 is the only wait point in a step.

### Determinism
Same seed and same inputs must give the same world, with any number of threads:
* All simulation math is fixed-point integer.
* A unit's result depends only on its own data and read-only snapshots, never on thread order.
* Random numbers come from a hash of (world seed, position, tick), not from a shared generator.
* Simulation code never depends on `HashMap` iteration order.
