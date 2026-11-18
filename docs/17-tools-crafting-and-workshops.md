# 17 — Tools, Crafting & Workshops

The frontier farm makes most of what it needs from wood, stone, and soil on the land. Iron is the exception: blades, bolts, and hardware come from town, and later from the blacksmith. This doc covers what the farm can make, how tools wear and break, and what a settler needs to do it.

Values marked *starting value* are for tuning in milestone M10+.

---

## 1. Tools

Every physical task on the farm requires the right tool. A settler without a tool works much slower or not at all. Tools are items (doc 08, section 2) with a `condition` field (`0..100`).

### 1.1 Tool List

| Tool | Used For | Made On Farm? | Iron Part |
| :--- | :--- | :---: | :--- |
| **Axe** | Felling trees, bucking logs, splitting kindling | Handle only | Axe head (from town) |
| **Broad axe** | Hewing round logs into square timber | Handle only | Head (from town) |
| **Froe** | Splitting shingles and fence rails from billets | Handle only | Blade (from town) |
| **Adze** | Shaping timber, hollowing wood | Handle only | Head (from town) |
| **Hand saw** | Crosscutting lumber | No | Blade (from town) |
| **Auger** | Boring holes for pegs, joints, fence posts | Handle only | Bit (from town) |
| **Hammer / maul** | Driving pegs, fencing, masonry | Wooden maul on farm; handle only for iron hammer | Head (from town) |
| **Hoe** | Hoeing weeds, hilling corn, garden work | Handle only | Blade (from town) |
| **Spade / shovel** | Digging, sod cutting, cellar work | Handle only | Blade (from town) |
| **Scythe (grass)** | Cutting meadow hay | Handle (snath) on farm | Blade (from town) |
| **Grain cradle** | Harvesting wheat and oats | Full cradle on farm | Optional iron tip |
| **Flail** | Threshing grain indoors | Full flail on farm | None |
| **Pitchfork** | Moving hay, manure, straw | Handle only | Tines (from town) |
| **Wheelbarrow** | Hauling stone, soil, grain | Full wheelbarrow on farm | Iron wheel or ring (from town) |
| **Yoke** | Hitching oxen (doc 14) | Full yoke on farm | Iron ring and staple (from town) |
| **Plow (walking)** | Plowing broken ground (doc 14) | Beam and handles on farm | Plowshare and coulter (from town) |
| **Breaking plow** | First plowing of prairie sod (doc 14) | Beam on farm | Heavy share (from town) |
| **Harrow** | Smoothing plowed ground | Wooden frame on farm | Iron teeth (from town) |
| **Rifle / musket** | Hunting, defense | No | From town |

### 1.2 Tool Condition and Wear

Each tool item has `condition: u8` (`0..100`). Condition drops with every use tick, at a rate that depends on the work:

* **Hard work wears tools faster:** breaking sod wears a plowshare much faster than plowing already-broken ground. Chopping hard oak wears an axe faster than soft pine.
* **Rocky soil** (`materialId = fieldstone` in nearby voxels) hits plowshares and hoe blades especially hard.

| Condition | State | Effect |
| :---: | :--- | :--- |
| `80–100` | Sharp / new | Full work speed and quality |
| `50–79` | Used | Slight speed penalty; no quality loss |
| `20–49` | Dull / worn | Noticeable speed penalty; some quality loss (duller axe, rougher cut) |
| `1–19` | Near failure | Large speed penalty; risk of breaking each use |
| `0` | Broken | Cannot be used; becomes a broken item (section 1.3) |

### 1.3 Breaking

When a tool's condition reaches 0, it breaks. The iron and wood parts separate into two loose items:

* `axe_head` (iron, full condition) — can be re-hafted with a new handle.
* `axe_handle_broken` (wood, low condition) — fits only for firewood.

This models the real frontier pattern: an axe head was expensive iron, kept and re-hafted many times. The handle was cheap wood, made from the farm's own timber.

### 1.4 Tool Repair

| Part | Repaired By | Where |
| :--- | :--- | :--- |
| **Wooden handle** | Settler with Carpentry skill at the carpenter's bench | On farm |
| **Iron blade / head** | Blacksmith | Town (once blacksmith unlocks, doc 16) |
| **Plowshare** | Blacksmith | Town; before the blacksmith, buy a replacement share |
| **Full wooden tool** (flail, yoke, cradle) | Settler with Carpentry skill | On farm |

Repair is a job like any other: `Repair axe handle` → settler walks to bench → crafts a new handle → re-hafts the axe. The repaired tool is restored to `condition = 90` (*starting value*; never quite as good as new from the blacksmith).

### 1.5 The Tool Shed

The tool shed is a designated indoor stockpile zone (doc 08) for tools. Putting tools back in the shed at the end of the day is a standing order (doc 13, section 8.2).

* Tools left outdoors in rain gain `moisture_rot` (doc 08) and lose condition faster.
* A tool that falls in a flooded cellar or creek takes immediate condition damage.
* The player can inspect any tool's condition from the shed stockpile list.

---

## 2. What the Farm Can Make

Crafted goods are produced by designating a workshop job. The player opens the workshop menu at the relevant bench or site (`w` → select bench → select item → set quantity), and a settler with the right skill carries out the job, consuming materials from nearby stockpiles and producing the output item there.

### 2.1 Splitting and Hewing (Outdoors, No Bench)

These jobs happen at a **splitting site**: any flat outdoor area near a timber stockpile.

| Output | Input Materials | Skill | Notes |
| :--- | :--- | :--- | :--- |
| **Fence rails** (×6) | 1 straight pine or oak log | Woodcutting | Split with froe and maul. 1 log → 6 rails (*starting value*) |
| **Firewood cords** | Logs (any species) | Woodcutting | Split with axe or maul |
| **Shingles** (×20) | 1 straight-grained pine billet | Woodcutting + Carpentry | Requires froe and shaving horse. Pine shingles last 10–15 years (*starting value*) |
| **Hewn timber** (×1) | 1 round log (oak preferred) | Woodcutting | Hewing with broad axe. Produces square-section beam for heavy framing (doc 04) |
| **Peg stock** | Small-diameter poles | Woodcutting | Cut into wooden pegs for joinery |

A **shaving horse** is a simple device (a low bench with a foot-lever clamp) made at the carpenter's bench. Without one, shingle-making is not possible.

### 2.2 Sod Cutting

Sod bricks are cut from living prairie turf with a spade. The player designates an area of prairie (`d` → `o` for sod cut → drag rectangle). A settler with a spade cuts the top $6\\text{\"}$ of turf into blocks.

* **Input:** 1 surface column of live prairie grass with `hasRoots = 1` and `compaction < 8`.
* **Output:** 1 sod brick item placed in the nearest sod stockpile.
* **Effect on the land:** the cut column loses its grass and roots; `hasRoots = 0`, `grass = 0`. It is now bare soil, vulnerable to erosion (doc 03). Regrowth takes about one full season.
* Sod bricks dry and shrink slightly over a few days. Fresh-cut bricks are heavy and crumble more easily; aged bricks are more stable. This is flavor only in the first version (*starting value*: treat all sod bricks as equivalent).

### 2.3 Fieldstone Collection

Glacial till on ridge and slope landforms scatters surface stones (doc 10, section 4.4). The player designates a fieldstone clearing area (`d` → `r` for rock clearing → drag). A settler picks up each surface stone voxel and hauls it to the stone stockpile.

* Clearing stones from a field also removes tripping hazards and soil disruptions that would slow plowing.
* Fieldstone requires no shaping for rubble-fill or dry-laid walls. For chimneys and refined masonry, a settler with a mason's hammer dresses the stone at the stone stockpile (a simple job: input 1 rough fieldstone → output 1 dressed fieldstone, skill: Masonry).

### 2.4 Carpenter's Bench

The carpenter's bench is a built structure voxel (`matterState = 3`, `materialId = workbench`) placed inside a shelter. It unlocks carpentry jobs that require precision work:

| Output | Input Materials | Skill |
| :--- | :--- | :--- |
| **Wheelbarrow** | 2 pine boards, 1 iron wheel (from town) | Carpentry |
| **Tool handle** (axe, hoe, etc.) | 1 straight-grained hardwood billet | Carpentry |
| **Ox yoke** | 2 hewn oak billets, 2 iron rings (from town) | Carpentry |
| **Ladder** | 2 pine poles, 8 rungs | Carpentry |
| **Door** | 3 pine boards, 1 iron hinge pair (from town) | Carpentry |
| **Shaving horse** | 4 pine billets | Carpentry |
| **Furniture** (table, bench, bed frame, shelving) | Pine boards | Carpentry |
| **Barrel / cask** | Oak staves, 2 iron hoops (from town) | Carpentry |
| **Seed bin / storage chest** | Pine boards, 1 iron latch (from town) | Carpentry |

The carpenter's bench itself is built from 6 hewn pine planks by a settler with Carpentry skill; no bench is needed to build the first one (a settler can build it on the ground).

### 2.5 What Must Come From Town

Iron cannot be made on the farm. The following must be bought in town or (once the blacksmith unlocks) ordered there:

| Item | Used In |
| :--- | :--- |
| Axe head, hoe blade, spade blade, froe blade | Tool making and repair |
| Plowshare and coulter | Walking plow and breaking plow |
| Harrow teeth | Harrow |
| Nails, spikes | Building and carpentry |
| Iron hinges, latches, hooks | Doors, chests, gates |
| Iron wheel / wheelbarrow ring | Wheelbarrow |
| Iron yoke rings and staples | Ox yoke |
| Wagon hardware (axle, bolts, wheel rims) | Wagon (doc 08) |
| Saw blade | Hand saw |
| Rifle / musket | Hunting and defense |

A new farm should bring a supply of nails, spare axe heads, and at least one spare plowshare from the start, or buy them on the first town trip. Running out of nails mid-build is a real setback.

---

## 3. The Workshop as a Voxel

A workshop bench is a single built voxel. To use it, the settler must be standing adjacent to it, and the input materials must be accessible from a nearby stockpile (within pathfinding reach, doc 13, section 9).

Multiple benches can be placed side by side in a carpenter's shop. Several settlers can work at different benches simultaneously. A settler working at a bench is not available for other jobs until the craft job finishes.

---

## 4. Skill and Quality

Crafting quality scales with the settler's Carpentry or Masonry skill (doc 13, section 6).

* A low-skill settler produces functional items but slower, with slightly reduced durability.
* A high-skill settler produces items with higher starting `condition` and may produce a bonus output (an extra rail from a log, or shingles that last longer).
* A very low-skill settler attempting a complex item (a barrel, a yoke) may ruin the materials entirely and produce only scrap.

Quality differences are shown to the player as item descriptions: *rough-hewn fence rail*, *well-made fence rail*.

---

## 5. The Crafting Queue

Crafting jobs enter the homestead job queue like any other job. The player can queue multiple items (e.g., "make 40 fence rails, then 2 wheelbarrow handles") and assign priority. A settler finishes the current item before starting the next.

Materials are reserved from the stockpile when the job is claimed, so two settlers do not compete for the same last board.

---

## 6. Cost

* Crafting runs only when a settler is at a bench — no background ticks.
* No new voxel fields. Tools are items in the `itemTable` (doc 08, section 2.1) with `condition` already defined.
* The workshop bench is a single built voxel, handled by the existing structure system.

---

## 7. Tests

* **Rail splitting:** designate 10 logs for rail splitting; verify 60 fence rails appear in the timber stockpile and 10 logs are consumed.
* **Tool wear:** plow a field of dense, rooty sod; verify plowshare condition drops faster than on broken ground.
* **Tool break:** run a hoe to `condition = 0`; verify it separates into a hoe blade and a broken handle.
* **Re-hafting:** carry the broken handle to the carpenter's bench; craft a new handle; verify the hoe is restored to `condition = 90`.
* **Sod cut:** designate a prairie patch; verify sod bricks appear in the stockpile, `hasRoots = 0` and `grass = 0` on the cut columns.
* **Iron dependency:** attempt to build a wheelbarrow with no iron wheel in the stockpile; verify the job is blocked with the reason shown.
* **Multi-settler bench:** two settlers at two benches in the same shop; verify they produce independently without competing for the same materials.

---

## 8. Open Questions

1. **Plank-making:** real frontier carpentry often needed sawn planks, produced by pit-sawing a log (a two-person, slow job) or by a mill with a sash saw. Should pit-sawing be a farm job, or should planks only come from the grist mill once it has a saw blade?
2. **Tool quality tiers:** should there be distinct quality tiers (frontier-made wooden maul vs. town-bought iron hammer vs. blacksmith-forged premium tool), each with its own work-rate multiplier?
3. **Candle and soap making:** minor but historically common winter crafts. Worth including, or too small to model?
4. **Wagon building:** can a skilled carpenter build a full wagon on the farm from hewn wood and town hardware, or is a wagon always bought in town?
