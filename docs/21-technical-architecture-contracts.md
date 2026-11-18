# 21 — Technical Architecture Contracts & Concurrency Protocols

This document defines the strict software contracts, concurrency patterns, and coordinate primitives needed before writing code. While `docs/01` through `docs/20` define the *behavior and simulation rules*, this document defines the *Rust types, memory transfers, and synchronization protocols* that prevent borrow-checker conflicts and architectural rewrites.

---

## 1. Concurrency Protocol: The Two-Phase Tile Edge Buffer

`doc 03` and `doc 06` specify a two-phase Rayon parallel execution model for surface water and subsurface seepage. In Rust, naive cross-tile communication will fail the borrow checker if Thread A working on Tile $(X, Y)$ attempts to write directly into the memory of neighboring Tile $(X+1, Y)$ while Thread B is executing on it.

### 1.1 The Double-Buffered Inbound/Outbound Contract

To guarantee $100\%$ race-free parallel execution without mutexes, spinlocks, or unsafe code, all edge communication uses **Double-Buffered Outbound Slots**:

```
                         PHASE 1: PARALLEL COMPUTE
   awake_tiles.par_iter_mut()
   ┌────────────────────────────────┐     ┌────────────────────────────────┐
   │ Tile (X, Y)                    │     │ Tile (X+1, Y)                  │
   │  - Reads local cells + Snapshots│    │  - Reads local cells + Snapshots│
   │  - Computes outflows           │     │  - Computes outflows           │
   │  - Writes ONLY to OWN          │     │  - Writes ONLY to OWN          │
   │    local outbound[East]        │     │    local outbound[West]        │
   └────────────────────────────────┘     └────────────────────────────────┘
                                  │         │
                                  ▼         ▼
                     BARRIER: RAYON JOIN (Zero Sync Cost)
              Pointer / Slice Swap: Outbound becomes Inbound
                                  │         │
                                  ▼         ▼
                          PHASE 2: PARALLEL APPLY
   awake_tiles.par_iter_mut()
   ┌────────────────────────────────┐     ┌────────────────────────────────┐
   │ Tile (X, Y)                    │     │ Tile (X+1, Y)                  │
   │  - Reads arrivals from:        │     │  - Reads arrivals from:        │
   │    West neighbor's East slot   │     │    Tile (X, Y)'s East slot     │
   │  - Applies net mass changes    │     │  - Applies net mass changes    │
   └────────────────────────────────┘     └────────────────────────────────┘
```

### 1.2 Data Structures

```rust
/// Exactly 16 columns along a chunk border (one 8 ft side)
pub const CHUNK_EDGE_LEN: usize = 16;

#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct EdgeCellArrival {
    /// Water mass in 1/64" volume units entering the neighbor column
    pub water_volume: u32,
    /// Soil sediment carried in the flow (for erosion deposition)
    pub sediment: u16,
}

/// Outbound buffer owned privately by each surface tile.
/// Written exclusively during Phase 1 by the tile's owning Rayon thread.
#[derive(Clone, Debug, Default)]
pub struct TileEdgeBuffers {
    pub north: [EdgeCellArrival; CHUNK_EDGE_LEN],
    pub south: [EdgeCellArrival; CHUNK_EDGE_LEN],
    pub east:  [EdgeCellArrival; CHUNK_EDGE_LEN],
    pub west:  [EdgeCellArrival; CHUNK_EDGE_LEN],
}

pub struct SurfaceTile {
    pub coords: (i32, i32),
    pub cells: Box<[SurfaceCell; 256]>, // 16x16 columns
    pub flow: Option<Box<[SurfaceFlow; 256]>>, // only allocated if awake
    pub outbound: TileEdgeBuffers,
}
```

### 1.3 Execution Invariant
1. **Phase 1 (Compute):** A tile's thread mutates *only* its own `cells`, `flow`, and `outbound` buffers. It reads neighboring border cells from a read-only snapshot or read-only edge array taken at tick start.
2. **Phase 1 $\rightarrow$ Phase 2 Barrier:** A lightweight index map or pointer table provides read-only access to each tile's neighbors' `outbound` buffers.
3. **Phase 2 (Apply):** A tile's thread reads the appropriate edge array from each of its 4 cardinal neighbors (e.g., its North boundary reads the South outbound buffer of its northern neighbor). It mutates only its own local `cells`.

---

## 2. Core $\leftrightarrow$ Presentation Boundary (`frontier_sim_core` vs `frontier_sim_tui`)

`frontier_sim_core` is a pure headless library. It has no dependency on `ratatui`, `crossterm`, `winit`, or terminal I/O. Presentation communicates across a clean boundary using **View Snapshots** and an **Input Command Queue**.

```
┌─────────────────────────────────┐
│        frontier_sim_tui         │
│  - Ratatui event loop           │
│  - Keyboard modal parsing       │
└───────┬─────────────────▲───────┘
        │                 │
Command │                 │ Read-Only Render View
Queue   │                 │ (WorldView Trait)
        ▼                 │
┌─────────────────────────┴───────┐
│        frontier_sim_core        │
│  - World State & Rayon Engine   │
│  - Voxel grids & Surface layer  │
└─────────────────────────────────┘
```

### 2.1 The Render Query Interface (`WorldView`)

The TUI queries only what is needed to render the visible terminal viewport ($W \times H$ cells, typically $120 \times 40$ characters). It never clones entire chunks or acquires locks across simulation arrays.

```rust
pub struct ViewportCell {
    pub glyph: char,
    pub fg_color: (u8, u8, u8), // 24-bit TrueColor RGB
    pub bg_color: (u8, u8, u8),
}

pub struct VoxelInspectDetail {
    pub world_pos: (i32, i32, i32),
    pub matter_state: u8,
    pub material_id: u8,
    pub soil_texture: Option<(u8 /* clay */, u8 /* sand */, u8 /* silt */)>,
    pub organic_matter: u8,
    pub compaction: u8,
    pub pore_water: u8,
    pub npk: (u8, u8, u8),
    pub soil_ph: u8,
    pub soil_temp_band: u8,
    pub flags: u8,
    pub entity_summary: Option<String>,
}

pub trait WorldView {
    /// Renders a 7-foot terrain-conforming surface slice for a bounding box of columns.
    fn query_surface_slice(&self, min_col: (i32, i32), max_col: (i32, i32)) -> Vec<ViewportCell>;

    /// Renders a flat horizontal Z-slice at a fixed Cartesian height.
    fn query_z_slice(&self, z_level: i32, min_col: (i32, i32), max_col: (i32, i32)) -> Vec<ViewportCell>;

    /// Returns deep agronomic and structural data for the voxel inspector HUD.
    fn inspect_voxel(&self, pos: (i32, i32, i32)) -> VoxelInspectDetail;

    /// Returns high-level colony status (game date, weather, speed, alerts, settler summary).
    fn get_status_summary(&self) -> ColonyStatusSummary;
}
```

### 2.2 The Player Command Queue (`DesignationCommand`)

Player input from the TUI never directly mutates simulation arrays. All key actions emit serializable commands into a thread-safe MPSC queue drained at the start of each simulation tick:

```rust
pub enum DesignationCommand {
    PlowArea { top_left: (i32, i32), bottom_right: (i32, i32), priority: u8 },
    SowArea { top_left: (i32, i32), bottom_right: (i32, i32), seed_type: u8, priority: u8 },
    FellTrees { bounds: ((i32, i32), (i32, i32)), priority: u8 },
    ExcavateVolume { corner_a: (i32, i32, i32), corner_b: (i32, i32, i32), priority: u8 },
    BuildWall { start: (i32, i32, i32), end: (i32, i32, i32), material: u8 },
    BuildFloor { bounds: ((i32, i32), (i32, i32)), z: i32, material: u8 },
    SetStockpile { bounds: ((i32, i32), (i32, i32)), category: StockpileCategory },
    SetWorkPriority { settler_id: u32, job_category: u8, priority: Option<u8> },
    SendTownTrip { settler_id: u32, orders: TownTripManifest },
}
```

---

## 3. Agent Footprint & Surface-Bound Coordinates

Agents (settlers, oxen, wagons) are not stored in the 3D voxel grid. Because movement occurs along the living, eroding, undulating landscape, agents have a specific coordinate contract.

### 3.1 The `AgentPos` Primitive

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct AgentPos {
    /// Discrete 2D planar position in 6" voxel columns (0..9333 across 500 acres)
    pub col: (i32, i32),

    /// Fine sub-voxel planar offset in 1/64" units (-192..+192) for smooth walking
    pub sub_col: (i8, i8),

    /// Absolute vertical base level (in 6" voxel Z levels) of the support surface
    /// (e.g. ground level, barn loft floor, or cellar floor)
    pub support_z: i32,

    /// Height above support surface in fine 1/64" units:
    /// - 0 for grounded settlers/oxen
    /// - Positive for fluttering chickens, jumping over furrows, or flying birds/predators
    pub altitude_offset: u16,

    /// Agent heading: 16 compass directions (0 = North, 4 = East, 8 = South, 12 = West)
    pub heading: u8,
}
```

### 3.2 Vertical Elevation & Support Surface Invariants
Agents are not subject to expensive continuous 3D physics engines, but support multi-story buildings and aerial states via two rules:
1. **Supporting Plane Determination:**
   - **Outdoors on Terrain:** `support_z` is set to the top of the column's solid ground (`SurfaceCell.ground`).
   - **Indoor / Multi-Story Structures:** An agent standing on a built floor or loft deck (doc 04) has `support_z` anchored to the top of that built voxel.
   - **Underground (Cellars/Wells):** `support_z` is anchored to the excavated floor level.
2. **Airborne, Roosting & Fluttering (`altitude_offset`):**
   - **Walking/Draft Animals:** `altitude_offset = 0`. Moving between columns with small elevation deltas (furrows, wheel ruts) interpolates smoothly.
   - **Chickens & Fowl:** Chickens can flutter (`altitude_offset > 0` for 1–3 seconds), allowing them to flap over low 1-voxel split-rail fences or hop onto roosting poles, coop nest boxes, or barn rafters.
   - **Aerial Predators (Hawks):** Cruise with high `altitude_offset` (50–100 ft in fine units) and swoop down into low offsets during strikes.
   - **Falls:** If an agent's supporting structure voxel is destroyed (e.g. barn loft collapse, doc 04), they fall: `support_z` drops toward the lower floor or ground, and fall impact damages health based on distance fallen.

### 3.3 Footprint & Clearance Collision
Collision is evaluated against the 2D **Clearance Map**:
* A person ($3 \times 2$ voxels) checks that all columns within their footprint have at least $7\text{ ft}$ (14 voxels) of clear air above `ground`.
* An ox team with a wagon ($12 \times 24$ voxels) requires a $6\text{ ft}$ wide path clear of solid built voxels, trees, deep water, and steep vertical drop-offs ($>1$ voxel height delta per column).

---

## 4. Entity Lifecycle & Lookup Lists

Every voxel in `doc 02` contains a 16-bit `entityRef` (`0..65535`). To guarantee that multi-voxel objects, items inside containers, and held tools never cause pointer corruption:

### 4.1 Global Entity ID Union

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub enum GlobalEntityId {
    None,
    Item(u32),       // Index into World::item_table
    Plant(u32),      // Index into World::plant_table (trees, shrubs)
    Structure(u32),  // Index into World::structure_table (doors, machines)
    Zone(u16),       // Index into World::zone_table (stockpiles, pens)
}
```

### 4.2 Sub-Chunk Lookup Proxy

Each $16 \times 16 \times 16$ sub-chunk maintains a compact vector of the `GlobalEntityId`s currently touching its volume:

```rust
pub struct SubChunkEntityProxy {
    /// Index 0 is always GlobalEntityId::None.
    /// Indices 1..N map to non-zero voxel entityRef values.
    pub lookup: Vec<GlobalEntityId>,
}
```

### 4.3 Item State Transitions

An item exists in exactly **one** of four states at any moment:

```
                      ┌────────────────────────────────┐
                      │ 1. Loose in Voxel Grid         │
                      │  - matterState = 5             │
                      │  - entityRef = proxy index     │
                      │  - item.container_id = None    │
                      └───────┬────────────────▲───────┘
                              │                │
            Picked up by      │                │ Dropped onto ground
            settler           │                │
                              ▼                │
                      ┌────────────────────────┴───────┐
                      │ 2. Held in Agent Hands         │
                      │  - Voxel entityRef cleared (0) │
                      │  - settler.hands = Some(id)    │
                      │  - item.container_id = None    │
                      └───────┬────────────────────────┘
                              │
            Packed into barrel│
            or storage chest  │
                              ▼
                      ┌────────────────────────────────┐
                      │ 3. Contained in Storage Vessel │
                      │  - Voxel entityRef is 0        │
                      │  - item.container_id = Some(C) │
                      │  - Container C holds item id   │
                      └───────┬────────────────────────┘
                              │
            Consumed, burned, │
            or rotten away    │
                              ▼
                      ┌────────────────────────────────┐
                      │ 4. Destroyed / Recycled        │
                      │  - ID freed to slot pool       │
                      └────────────────────────────────┘
```

**Invariant:** An item that is in an agent's hands, in a wagon, or inside a container **never** has a voxel `entityRef` assigned to it. It exists purely in `World::item_table` and the holding entity's inventory vector.

---

## 5. Summary: Readiness for Milestone M0

With these contracts in place:
1. `SubChunk`, `Voxel`, and `SurfaceCell` memory layouts are frozen.
2. The concurrency exchange buffer structure avoids all Rayon mutable aliasing issues.
3. The TUI and Core are decoupled behind `WorldView` and `DesignationCommand`.
4. Agent coordinates and item states have zero spatial ambiguities.

The workspace is fully ready for **Milestone M0: Cargo workspace setup and 64-bit voxel unit testing**.
