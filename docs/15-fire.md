# 15 — Fire

Fire is a physical force on the frontier. Prairie fires sweep for miles. A chimney fire can bring down a cabin. A settler who lets a campfire escape the firebreak has to race to save the harvest. Fire is dangerous, it rewards preparation, and it can be tamed as a tool.

Doc 11 decides when lightning strikes. This doc owns everything that happens after the fire starts.

Values marked *starting value* are for tuning in milestone M10+.

---

## 1. Fire State

Fire is not stored as a separate fluid or grid. It lives in two places that already exist.

### 1.1 The `isOnFire` Flag (Bit 62, doc 02)

Any voxel that is actively burning has `isOnFire = 1`. It is set when a voxel ignites and cleared when the voxel is consumed or extinguished. Only voxels with flammable `matterState` can hold the flag:

| `matterState` | Flammable? | What burns |
| :--- | :---: | :--- |
| `4` Plant Mass | Yes | Crops, tree canopy, standing grass roots |
| `3` Built Structure | Depends on material | Pine and oak timber; not fieldstone or sod |
| `5` Loose Item | Depends | Wood items, grain sacks, straw; not stone or iron |
| `1` Soil | No | Soil itself does not burn, but surface grass does |
| `2` Bedrock / Stone | No | |
| `0` Air | No | |

Soil voxels never get `isOnFire`. Surface grass burns through the surface layer (section 2), not through voxels.

### 1.2 Surface Grass Fuel

The surface layer (doc 02, section 5) has two grass fields:

```rust
pub grass:     u8,  // 0..255 — standing grass amount; also hay fuel load
pub grass_dry: u8,  // 0..255 — how dry the grass is; 0 = green, 255 = tinder
```

These drive prairie fire. A column's **fuel load** is `grass`, and its **ignition risk** scales with `grass_dry`. A green column (`grass_dry` near 0) is very hard to ignite even with a lot of grass. A dry late-summer column (`grass_dry` near 255) with heavy grass catch quickly and burns hot.

---

## 2. Prairie Fire

### 2.1 Ignition

A prairie fire can start from:

1. **Lightning** (doc 11, section 5.9): when a thunderstorm brings a strike, the engine picks a column in or near the strike footprint. If `grass > 20` and `grass_dry > 100` on that column (*starting values*), a fire starts there.
2. **Player-set fire** (back-fire or controlled burn): the player designates a strip to burn with `d` → `b` → drag line. A settler walks to the strip and starts the fire from downwind. This is the main tool for cutting firebreaks ahead of an approaching fire (section 2.4).
3. **Escaped hearth or campfire:** a building fire that breaks out of its chimney or hearth can ignite surface grass on adjacent columns if they are dry enough (section 3.3).
4. **Fallen burning tree:** a tree with `isOnFire` that falls (doc 05, section 3.2) stamps burning timber voxels that can ignite the grass they land on.

### 2.2 Spread

Each surface fire step (runs with the weather step, every 600 ticks / 10 game minutes):

1. **For each burning column**, attempt to spread to each of its 4 neighbors:

   $$p_{\text{spread}} = k \times \frac{\text{grass\_dry}_{\text{neighbor}}}{255} \times \frac{\text{grass}_{\text{neighbor}}}{255} \times w_{\text{dir}}$$

   * $k$ is a base spread constant (*starting value*: `0.6` per step in a calm wind).
   * $w_{\text{dir}}$ is a wind multiplier: the downwind neighbor gets $w_{\text{dir}} = 2.0$; the crosswind neighbors get $1.0$; the upwind neighbor gets $0.3$ (*starting values*). Wind direction comes from the weather state (doc 11).
   * If a random value from `hash(seed, column, tick) % 1000 / 1000.0 < p_{\text{spread}}`, the neighbor ignites.

2. **Burn down the column**: each burning column loses grass each step:

   $$\text{grass\_new} = \max(0,\ \text{grass} - \text{burn\_rate})$$

   `burn_rate` is proportional to `grass_dry` (*starting value*: `grass_dry / 4` per step). When `grass` reaches 0, the column is extinguished and left as bare, scorched soil.

3. **A burning column heats its neighbors**: the air above a fire is hot. Neighboring columns' `grass_dry` rises by a small amount each step while they are adjacent to fire (*starting value*: +8 per step, capped at 255). This means fire dries out the grass just ahead of it, speeding spread in calm conditions.

### 2.3 Speed

At the realistic setting a prairie fire in dry late-summer grass with a 15 mph wind spreads about 1 to 2 miles per hour. At the game's $6\\text{\"}$ voxel scale, that is roughly 2 to 4 columns per 10-game-minute step. Fire can outrun a settler on foot, which is why firebreaks must be cut *ahead* of the front, not beside it. These numbers are tuned by $k$ and $w_{\text{dir}}$.

### 2.4 Stopping Fire: Firebreaks

Fire cannot cross a column where there is nothing to burn. A firebreak is any strip of bare ground:

* **Plowed strip**: `compaction > 0` and `grass = 0` from tilling. A single-furrow plow pass leaves a bare strip 1 voxel wide ($6\\text{\"}$). A double-furrowed break of 2 to 4 voxels ($12\\text{–}24\\text{\"}$) is more reliable in a strong wind (*starting value* for minimum safe width in calm conditions: 2 voxels).
* **Burned strip (back-fire)**: the player designates a burn line. A settler starts the fire at the downwind edge, so it burns *toward* the oncoming front. When the two fire fronts meet, they go out for lack of fuel. This is the classic Great Plains technique.
* **Creek or pond**: surface water columns (`depth > 0`) do not carry surface grass and block spread.
* **Plowed field or bare road**: any continuous strip of low-grass columns stops spread.

> [!TIP] The player can designate a firebreak well before fire season and leave it as a permanent plow strip around crops and buildings. This is cheaper than scrambling during an active fire.

### 2.5 Extinguishing

* **Rain**: each surface water step during rain reduces `grass_dry` on burning columns and their neighbors. If `grass_dry` drops below the ignition threshold before `grass` is consumed, the fire goes out without fully burning the column. Columns left with partial grass recover in the next growing season.
* **Settling**: on a column that is not spreading to new neighbors and is not receiving heat from burning neighbors, `grass_dry` falls with normal drying (doc 11, section 5.3 reversed) and the fire self-extinguishes.
* **Night and calm**: fire slows dramatically in still, cool night air. Many real prairie fires smoldered to extinction at dawn after racing all afternoon.

### 2.6 Aftermath

After fire passes through a column:

* `grass = 0`, `grass_dry = 0` — no fuel remains.
* The top soil voxel's `organicMatter` rises by +1 (*starting value*): ash and charcoal feed the first spring regrowth. This is historically accurate — burns often produced lush early grass the next year.
* `hasRoots` is cleared on burnt soil-surface voxels (doc 02, flag bit 57), leaving them vulnerable to erosion until roots regrow.
* In spring, `grass` regrows from the surface layer's normal daily grass tick (doc 06, section 2.2), faster on recently burned and ash-enriched ground.

---

## 3. Building Fire

Building fires are distinct from prairie fires. They burn through the voxel grid rather than the surface layer.

### 3.1 Flammable Materials

| Material | Burns? | Burn rate | Notes |
| :--- | :---: | :---: | :--- |
| Pine timber / rail | Yes | Fast | High resin, lights easily |
| Hewn oak | Yes | Slow | Dense, hard to start but burns long |
| Thatch / straw | Yes | Very fast | — |
| Loose items (grain sacks, tools, wood) | Depends | — | Wood and fibre items burn; iron and stone do not |
| Fieldstone | No | — | Hearths and foundations do not burn |
| Sod brick | No | — | Earthen walls resist fire |
| Soil / bedrock | No | — | |

### 3.2 Spread Through Voxels

A voxel with `isOnFire = 1` damages its neighbors every fire sub-tick (runs with the daily structure sweep, doc 04, section 5):

1. **Durability damage**: each burning built voxel loses `durability` (bits 8..15, doc 02) each sub-tick at the material's burn rate. When `durability` reaches 0, the voxel is destroyed, triggering a support check (doc 04, section 5). The collapsed voxels become debris (`matterState = 5`), and debris on the ground can itself be burning.
2. **Spread to neighbors**: for each of the 6 face-adjacent neighbors, a voxel can ignite if the neighbor is flammable and `isOnFire = 0`. Spread chance rises with the burning voxel's current heat (approximated as remaining `durability` consumed). Fire spreads faster upward (heat rises) than sideways or downward (*starting value*: upward spread chance `×2`).
3. **Air gap**: fire cannot jump an air voxel to reach a distant flammable voxel. This means gaps between structures matter. A barn separated from a cabin by even 2 or 3 voxels of open air is safer than one that shares a wall.

### 3.3 Hearth, Chimney, and Chimney Fire

A hearth or stove is a designated built structure voxel or a small set of fieldstone voxels with a chimney. Burning firewood inside is the controlled, safe use of fire. It is handled separately from the voxel fire system: firewood is consumed from inventory, the room's temperature rises, and no `isOnFire` flags are set on the structure under normal operation.

A **chimney fire** starts if green (unseasoned) firewood is burned. Green wood produces heavy creosote that coats the flue. Creosote can ignite. The chance of a chimney fire rises with:

* The fraction of firewood burned that is green (high `moisture_rot` items — doc 08).
* How long since the chimney was swept (a standing-order chore).

When a chimney fire occurs, the chimney voxels get `isOnFire = 1`. From there it follows the normal voxel fire rules (section 3.2). A fieldstone chimney resists burning but can transmit heat to adjacent timber. A wooden chimney (a shortcut some settlers took) burns quickly. The player is alerted immediately.

> [!CAUTION] A chimney fire at night, when settlers are asleep and the well bucket is indoors, is one of the most dangerous events on the farm. The alert wakes settlers, but there may not be time to save the cabin.

### 3.4 Crop Voxels

Plant-mass voxels (`matterState = 4`, doc 02) can burn:

* Ripe wheat or standing corn in late summer is dry enough to ignite from an adjacent prairie fire front.
* A burning crop voxel sets `isOnFire = 1`, loses `durability` (here representing plant structural integrity), and spreads to neighboring crop voxels the same way building fire spreads.
* When the voxel is consumed, the `plantTable` record for that patch is set to dead. Yield is lost.
* Root voxels (deeper soil with `hasRoots = 1`) are not burned; only above-ground plant mass burns.

---

## 4. The Settler Response

### 4.1 Alerts

When fire starts anywhere in the active claim, an alert fires immediately in the UI (doc 07): **"Fire! [Location]"**. The player can jump the camera there with a keypress.

### 4.2 Emergency Priority

Fire is an emergency (doc 13, section 4.4). All settlers drop their current jobs and respond unless the fire is small and far from the homestead.

### 4.3 Fighting Fire

Settlers fight a building fire by carrying water buckets from the well, creek, or trough to the burning structure. Each bucket delivery reduces `grass_dry` on adjacent surface columns and lowers `isOnFire` progress on burning built voxels. This is slow. Bucket brigades can save a stone cellar or slow a barn fire; they cannot stop a fully involved timber barn.

Settlers fight a prairie fire by beating the edge with burlap sacks or wet grain sacks (applying a small extinguish effect to adjacent burning columns). Again, this is slow and risky compared to firebreaks.

### 4.4 Fleeing

If the fire grows faster than settlers can fight it, settlers with no viable path to the well, or facing a fire front moving faster than they can run, switch to the **flee** state: they move away from the fire as fast as possible. Children and elders flee before adults. Animals in open pastures flee on their own (doc 14, section 6). Animals in enclosed pens cannot escape unless a gate is open.

---

## 5. Fire as a Tool

### 5.1 Clearing Land

A settler can designate an area of heavy prairie grass or dry scrub for a **controlled burn** (`d` → `b` → drag area): settlers walk a firebreak around the perimeter first, then ignite the interior from the downwind edge. This is the fastest and cheapest way to clear brush before plowing. The ash enriches the soil (section 2.6).

### 5.2 Back-fire

The player can designate a burn line ahead of an approaching prairie fire (section 2.4). Settlers read the wind and start the line at the downwind side so it burns into the oncoming fire. Timing matters: start too late and there is not enough burned ground to stop the front; start too early and the back-fire may turn and head back toward the homestead.

---

## 6. Difficulty Setting

Fire danger is a difficulty setting, following the pattern in doc 11, section 7.

| Setting | Easier | **Realistic (default)** | Harder |
| :--- | :--- | :--- | :--- |
| **Prairie fire spread** | Slower spread, narrower fronts | Section 2.2 values | Faster spread; fire can jump 1-voxel gaps in high wind |
| **Lightning ignition** | Requires very dry grass (`grass_dry > 180`) | `grass_dry > 100` (*starting value*) | `grass_dry > 60`; also ignites plowed-out stubble |
| **Chimney fire** | Only with very green wood (`moisture_rot > 12`), infrequent | Scales with creosote buildup | More frequent; sweeping the chimney is essential every winter |
| **Building fire spread** | Slower durability loss, fire cannot jump air gaps | Section 3.2 values | Embers can jump 1–2 air voxels in wind |

**Presets:** *Gentle*, *Frontier*, *Harsh* (see doc 11, section 7).

> [!NOTE] On the Realistic setting, fire is dangerous but survivable with preparation. A farm with plowed firebreaks around its fields and buildings, seasoned firewood, and a clean chimney will rarely face a catastrophic fire. The danger is a nudge toward good frontier practice, not a random punishment.

---

## 7. Performance

* **Prairie fire step** runs with the weather step (every 600 ticks). It scans only awake surface tiles that are burning or adjacent to a burning tile — a small set.
* **Building fire sub-tick** runs with the daily structure sweep (doc 04, section 5). A building fire is a rare event, so the cost is negligible outside active fires.
* **No new voxel field is added.** All fire state uses `isOnFire` (already in doc 02), `grass` and `grass_dry` (already in the surface layer), and `durability` (already in built voxels).

---

## 8. Tests

* **Spread with wind:** ignite a dry grass column; verify the fire spreads faster downwind than crosswind and barely upwind.
* **Firebreak stops spread:** a 2-voxel-wide plowed strip blocks a fire front in calm conditions.
* **Back-fire:** designate a burn line 20 columns ahead of a moving front; verify the fronts meet and both extinguish.
* **Rain extinguishes:** a moderate rain during an active fire reduces `grass_dry` and stops spread within a few steps.
* **Building fire collapse:** a burning central post loses `durability` to 0; support check triggers; roof falls.
* **Chimney fire alert:** burn green wood for several days; verify chimney fire starts and the alert fires.
* **Aftermath:** after a prairie burn, `grass = 0` on burnt columns, `organicMatter` +1 on top soil voxels, `hasRoots` cleared.
* **Same seed, same fire:** the same seed, same weather, and same ignition point produce the same fire path with any thread count.

---

## 9. Open Questions

1. **Ember spotting:** real prairie fires throw burning embers far ahead of the main front in high winds, starting spot fires. Should this be in, as a rare high-wind event? It would make the Harder setting more threatening.
2. **Ash fertilizer:** the `organicMatter +1` from burns — is that the right amount? Too generous, and players burn fields on purpose every year; too little, and the historical incentive is lost.
3. **Animal behavior in fire:** should animals in open pastures try to break through a fence to flee, or just flee within their zone? Breaking fences would be realistic (and chaotic).
4. **Smoke and visibility:** should a large fire reduce `sunExposure` on downwind columns (darkened sky, no photosynthesis), or is that too complex for the first version?
