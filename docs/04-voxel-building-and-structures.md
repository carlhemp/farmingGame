# 04 — Creative Voxel Building & Structural Mechanics

## 1. Design Philosophy: Pure DF-Style Voxel Building

Building on the frontier must feel **creative, intuitive, and unfiddly**. The game rejects CAD-style component placement (rotating 3D beams, aligning joint pins, or picking trusses).

**Decision: building is free voxels.** There are no snap-together post, beam, or panel pieces, and the game does not group voxels into a piece graph.
* You drag lines, boxes, and planes across the $6\text{"}$ voxel grid.
* You pick the material (e.g., Sod Brick, Pine Rail, Fieldstone, Hewn Oak).
* Autonomous settlers haul the physical materials and assemble the voxels.
* Under the hood, a simple **support spreading** rule (section 4) decides what stands and what falls.

```
       Player Designates (TUI)
       b -> w -> Oak Timber -> Drag 16x24 ft Rectangle
                           │
                           ▼
              Homestead Construction Jobs
                           │
                           ▼
              Settler Hauls Hewn Oak Logs
                           │
                           ▼
          Voxels Assembled on 6" Grid
          (matterState = 3 [Built Structure])
                           │
                           ▼
       Engine Spreads Support From the Ground
     ┌────────────────────────────────────────┐
     │ - Resting on soil or stone?       [OK] │
     │ - Within material reach of support?    │
     │ - Footing pressure on soil?   < Limit  │
     └────────────────────────────────────────┘
```

---

## 2. Player Designation Tools

| Designation | Keyboard Shortcut | Operation | Gameplay & Structural Role |
| :--- | :---: | :--- | :--- |
| **Split-Rail Fence** | `b` $\rightarrow$ `f` | Drag 1-voxel wide line | Encloses pastures, stops livestock wandering, marks claim borders. |
| **Solid Wall** | `b` $\rightarrow$ `w` | Drag line or hollow box | Provides shelter, wind protection, and grounded support. |
| **Floor / Roof Deck** | `b` $\rightarrow$ `r` | Drag horizontal rectangle | Keeps rain out, creates upper barn lofts, provides walking surfaces. |
| **Support Pillar** | `b` $\rightarrow$ `p` | Click single vertical column | Adds a grounded support in the middle of a span. |
| **Door / Gate** | `b` $\rightarrow$ `d` | Drag line within a wall or fence | Lets agents through. Must be wide enough for the agent (person, ox, wagon; doc 01). |
| **Cellar Dig** | `d` $\rightarrow$ `c` | Drag 3D volume underground | Excavates cool root cellars and wells; produces excavated soil/stone. |

---

## 3. Autonomous Construction Workflow

1. **Designation Stamping:** Designated voxels are marked with outline glyphs (`isDesignated = 1`).
2. **Material Allocation:** The job scheduler reserves raw materials from homestead stockpiles (e.g., 24 oak logs, 80 sod bricks).
3. **Hauling & Assembly:**
   * Settlers pick up tools and materials, walk to the site, and build voxels one by one.
   * A voxel can only be placed if it would have support (section 4), so settlers build from the ground up.
   * If bad weather interrupts work, partly built walls stand exposed to wind and rain.
4. **Voxel Placement:** A finished voxel gets `matterState = 3 (Built Structure)`, its `materialId`, and full `durability` (bits `8..15`, doc 02).
5. **Room Update:** If the new voxels close off an air space, the room table is updated (section 6).

---

## 4. Support Spreading

The engine does not run finite element analysis or track loads through beams. It uses one rule: **support spreads out from the ground and runs out with distance.**

### 4.1 The Rule
* **Grounded:** A built voxel resting on soil or stone, or on a grounded built voxel directly below it, has full support.
* **Sideways:** A built voxel next to a supported voxel gets that neighbor's support minus one voxel. Its support can never be more than its own material's reach.
* **Stacked:** A built voxel resting on top of a supported (not grounded) voxel gets that voxel's support.
* **Falling:** A built voxel with no support left falls.

Put simply, **span means the distance to the nearest grounded support**, measured along built voxels.

### 4.2 Material Reach

| Material | Reach (distance to nearest grounded support) | Weight / Density | Ideal Uses | Failure Look |
| :--- | :---: | :---: | :--- | :--- |
| **Sod Bricks** | $1\text{ voxel } (6\text{"})$ | Very Heavy | Thick earthen walls, dugouts | Crumble / Shear slip |
| **Dry Fieldstone** | $8\text{ voxels } (4\text{ ft})$ | Very Heavy | Foundations, hearths, cellars, lintels | Joint rupture |
| **Pine Timber** | $16\text{ voxels } (8\text{ ft})$ | Light | Fences, rafters, interior walls | Mid-span snap |
| **Hewn Oak** | $32\text{ voxels } (16\text{ ft})$ | Heavy | Major tie-beams, corner posts | Sagging, then breaks |

Reach values are starting numbers for tuning.

### 4.3 Emergent Architecture: The 24 ft Barn
* The barn has walls on both long sides, so the middle of the roof is $12\text{ ft}$ from the nearest wall.
* **Pine roof, no posts:** $12\text{ ft}$ is more than pine's $8\text{ ft}$ reach. The middle of the roof falls.
* **Oak roof, no posts:** $12\text{ ft}$ is within oak's $16\text{ ft}$ reach. It stands.
* **Pine roof with a row of center posts:** No roof voxel is more than $6\text{ ft}$ from support. It stands.

So the player learns the real frontier trade-off: buy or hew oak, or put posts down the middle of the barn.

### 4.4 Cost
Support only needs to be recomputed near a change. The largest reach is 32 voxels, so a change only affects voxels within 32 of it. Support values are not stored in voxel bits; they are computed when needed and can be cached per room or per area.

---

## 5. Event-Driven Collapse & Cascading Failures

Support checks do **not** run every tick. They run:
* when a built voxel is added, removed, burned, or hit by an axe;
* when `durability` drops to zero from rot;
* in a slow daily sweep over a small slice of structures, for decay and snow load. Heavy snow on a roof lowers reach by a tuning amount while it lasts.

### Cascading Failure Sequence
If a central post is destroyed (by rot, fire, or axe):
1. **Recompute:** Support is spread again within reach of the change.
2. **Fall:** Voxels left with no support fall.
3. **Repeat:** Falling voxels are removed and support is spread again, until nothing else falls. This gives a falling beam followed by the roof.
4. **Debris:** Fallen voxels become loose debris (`matterState = 5 [Loose Item]`), crushing crops and injuring settlers below.

---

## 6. Rooms

A **room** is an air space fully closed off by built voxels, soil, or stone, with doors counting as closed. Rooms are found by flood fill when building or digging changes the boundary, and are kept in a room table.

Rooms are used for:
* **Shelter and warmth:** Settlers need a room with a hearth or stove in winter (doc 01).
* **Cellar water:** Each room has one water level (doc 03, section 7).
* **Storage conditions:** Root cellars stay cool because their walls are deep soil (doc 08).

---

## 7. Soil Reaction & Foundation Settling

Structures press on the soil under them:

$$\text{Footing Pressure} = \frac{\text{Weight of Built Voxels Above}}{\text{Footprint Area}}$$

The weight above a footing is the count of built voxels stacked over it, times each material's density.

* **Stable Ground:** Packed dry loam, gravel, or bedrock easily supports heavy stone chimneys and two-story barns.
* **Wet Clay Settlement:** Heavy stone built on wet, loose clay (`poreWater > 12`, `clayPct > 10`) causes plastic lateral bulge in the ground beneath (doc 03).
* **Uneven Settling:** One corner of the cabin sinks a little (its column's `ground` drops), tilting walls, jamming doors, and lowering `durability` in the stone above.
