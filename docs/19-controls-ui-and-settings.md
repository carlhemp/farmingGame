# 19 — Controls, UI Reference & Settings

This is the single reference for every key, menu, and setting in *Frontier Farming Simulator*. Docs 07 and 13 describe individual systems; this doc lists where everything is and what it does.

The interface follows the *Dwarf Fortress* modal keyboard style: a small number of keys do different things depending on the active mode, and the bottom bar always shows the available keys for the current mode.

---

## 1. The Modal System

The interface has one **active mode** at a time. Pressing a mode key enters it; `Esc` backs out one level or returns to Normal mode.

| Mode | Enter With | Purpose |
| :--- | :---: | :--- |
| **Normal** | `Esc` (from any mode) | Move the cursor, pan the camera, inspect voxels |
| **Designate** | `d` | Mark areas for settler work |
| **Build** | `b` | Place walls, fences, floors, doors, cellars |
| **Stockpile** | `p` | Designate and configure storage zones |
| **Zone** | `z` | Mark pastures, pens, and garden beds |
| **Workshop** | `w` | Queue crafting jobs at a bench |
| **Town** | `t` | View town stock, prices, and send a settler |
| **Work Tab** | `W` (shift) | Set settler work priorities |
| **Job List** | `j` | View all open jobs and their status |
| **Alert Log** | `a` | View and dismiss alerts |
| **Settings** | `O` (shift-o) | All game and difficulty settings |
| **Pause Menu** | `q` | Save, quit, load, main menu |

---

## 2. Global Keys (Any Mode)

These work regardless of the active mode.

| Key | Action |
| :--- | :--- |
| `Space` | Pause / unpause |
| `1` | Speed: 1× Normal (1 game-second tick, ~20 real min per day) |
| `2` | Speed: 2× Fast (~10 real min per day) |
| `3` | Speed: 5× Harvest Speed (~4 real min per day) |
| `Esc` | Cancel current mode / back out one level / clear active anchor |
| Arrow keys or Numpad | Move cursor (Numpad `1..9` supports 8-way movement; `hjkl` vi-keys supported as setting) |
| `Shift` + Arrow / Numpad | Fast leap: move cursor 10 cells in direction |
| `Tab` | Toggle Z-Slice Inspection Mode (doc 07, section 2) |
| `<` / `>` | In Z-Slice mode: step down / up one 6" voxel level (`Shift+,` / `Shift+.`) |
| `+` / `-` | Zoom: cycle through Detail → Homestead → Valley Overview (doc 07, section 3) |
| `*` | Jump camera to the nearest active alert |
| `F5` | Quick-save (manual save slot) |

---

## 3. Normal Mode

Cursor movement, camera pan, and voxel inspection. No mode indicator is shown; the bottom bar says **NORMAL**.

| Key | Action |
| :--- | :--- |
| Arrow / Numpad | Move cursor one cell (camera pans automatically at viewport borders) |
| `Shift` + Arrow / Numpad | Fast leap (10 cells) |
| `Enter` | Inspect the voxel under the cursor (opens the right-side inspect panel) |
| `f` | Follow: camera tracks the selected settler or animal |
| `u` | Jump to: open a list of settlers and animals; select one to jump to them |

### 3.1 The Inspect Panel

When a voxel is selected (or whenever the cursor rests on one, in the right-side panel — doc 07, section 5), the panel shows:

```
SELECTED VOXEL
Pos: (428, 612, 18)
Matter: Soil — Topsoil
Clay: 13%  Sand: 60%  Silt: 27%
Organic Matter: 5 / 7
Compaction: 3 / 15  (Loose)
Moisture: 10 / 15  (Field Capacity)
N: 6/7   P: 2/3   K: 3/3
pH: 6.5  Temp: Band 4 (60–70°F)
Flags: hasRoots, isExposedSurface
Entity: Wheat patch [Stage: Vegetative]
```

For built voxels, the panel shows `materialId`, `durability`, and support status. For agents, it shows the settler or animal sheet (section 7).

---

## 4. Designate Mode (`d`)

Designations and zones follow the classic **Dwarf Fortress Two-Corner Anchor** model. No mouse dragging is required:

```
Step 1: Move cursor to Corner A ──> Press [Enter] to anchor first point
Step 2: Move cursor to Corner B ──> Bounding box highlights dynamically
Step 3: Press [Enter] again     ──> Designation confirmed across area
                                    (Press [Esc] before step 3 to cancel anchor)
```

After pressing `d`, select a sub-key to pick the task:

| Sub-key | Designation | Geometry | Notes |
| :---: | :--- | :--- | :--- |
| `p` | **Plow** area | Rectangle (2 corners) | Settlers cut furrows across the bounded area (doc 05) |
| `s` | **Sow** area | Rectangle (2 corners) | Prompts for seed selection from stockpile |
| `h` | **Harvest** area | Rectangle (2 corners) | Settlers reap ready crops; prioritized nearest the crib |
| `t` | **Fell trees** | Rectangle or single cell | Marks trees within the bounding box for felling (doc 05) |
| `c` | **Dig / excavate** | 3D box (2 corners + `<` / `>`) | Anchor corner A, step Z with `<` or `>`, anchor corner B (doc 04) |
| `o` | **Cut sod** | Rectangle (2 corners) | Settlers cut sod bricks from living prairie (doc 17, section 2.2) |
| `r` | **Clear rocks** | Rectangle (2 corners) | Collects loose surface fieldstones to stone stockpile (doc 17) |
| `b` | **Burn** strip | Line or rectangle (2 corners) | Controlled firebreak burn; ignited from downwind (doc 15) |
| `x` | **Cancel** designation | Rectangle (2 corners) | Clears all pending designations in the selected box |

After choosing a sub-key, the status bar displays:
`[Mode: Plow] Move to corner 1 and press [Enter] | [Esc] Cancel`

Once corner 1 is anchored, the bar displays dimensions in real feet and voxels:
`[Anchor at (420, 610)] Dimensions: 16x24 ft (32x48 voxels) | [Enter] Confirm | [Esc] Reset Anchor`

Each designation defaults to Priority `3` (`1`=Urgent, `5`=Low). Pressing `1..5` while bounding changes the priority for that designation.

---

## 5. Build Mode (`b`)

Building uses the same Two-Corner Anchor workflow:

| Sub-key | Build Type | Geometry | Notes |
| :---: | :--- | :--- | :--- |
| `w` | **Wall** | Line or hollow rectangle | Prompts for material: Sod, Fieldstone, Pine, Oak (doc 04) |
| `f` | **Fence** | Line (2 points) | Prompts for material: Split Rail, Sod, Fieldstone |
| `r` | **Floor / Roof deck** | Rectangle (2 corners) | Prompts for flooring/roofing material |
| `p` | **Pillar** | Single column (`Enter`) | Places vertical grounded support pillar (doc 04, section 2) |
| `d` | **Door / Gate** | Line within wall/fence | Prompts for width (person 3 vox, ox 6 vox, wagon 12 vox; doc 01) |
| `x` | **Remove** structure | Rectangle (2 corners) | Settlers dismantle built voxels; salvageable materials hauled to pile |

---

## 6. Stockpile Mode (`p`)

Press `p`, then anchor Corner 1 with `Enter`, move cursor, and anchor Corner 2 with `Enter`. Then choose the accepted goods category:

| Sub-key | Stockpile Type | Accepts |
| :---: | :--- | :--- |
| `w` | **Timber Yard** | Logs, beams, rails, firewood |
| `s` | **Stone & Masonry** | Fieldstone, dressed stone, sod bricks |
| `g` | **Granary / Grain Crib** | Threshed wheat, shelled corn, oats |
| `f` | **Food (Root Cellar)** | Potatoes, salt pork, smoked meat, butter, cider |
| `e` | **Seed Bank** | Seed bags (all varieties) |
| `r` | **Refuse / Manure** | Crop residue, stable manure, wood ash, spoiled grain |
| `t` | **Tool Shed** | All tools |
| `x` | **Remove** stockpile zone | Clears zone; items remain in place |

### 6.1 Stockpile Configuration

Pressing `Enter` on an existing stockpile opens its config panel:

* **Filter:** restrict accepted item types and materials (e.g., pine rails only).
* **Give To:** link this stockpile to send goods to another (doc 08, section 1.2).
* **Take From:** link to pull goods from another.
* **Max items:** cap how much is stored here.

---

## 7. Zone Mode (`z`)

Zones define areas for animals and gardens, without directing settler labor directly. Use Two-Corner Anchor (`Enter` $\rightarrow$ move cursor $\rightarrow$ `Enter`):

| Sub-key | Zone Type | Notes |
| :---: | :--- | :--- |
| `p` | **Pasture** | Grazing zone for assigned animals (doc 14, section 5) |
| `n` | **Pen** | Enclosed zone for penned animals |
| `g` | **Kitchen Garden** | Small growing zone tended by the cook settler (doc 18, section 6) |
| `x` | **Remove** zone | |

After anchoring the zone rectangle, the config panel opens to assign animals (by species or by name) or confirm garden bed assignment.

---

## 8. Workshop Mode (`w`)

Move the cursor to a carpenter's bench voxel and press `w` to open the workshop queue for that bench.

The panel lists available recipes, required materials, and quantities in nearby stockpiles. Select a recipe, set a quantity, and confirm. The job enters the homestead queue.

Doc 17, section 2.4 lists all available recipes.

---

## 9. Town Menu (`t`)

Opens the town panel (doc 16). Available any time, but buying requires sending a settler.

Sub-panels:
* **Stock & Prices** — current seasonal stock, buying and selling prices.
* **Land Office** — available 40-acre parcels with terrain previews; click to buy (settler must travel).
* **Hire** — available hired hands and their skills and day rate.
* **Breaking Crew** — availability and price per acre.
* **Send Settler** — designate which settler goes, what to sell, what to buy, and any special errands.

---

## 10. Work Tab (`W`)

The work tab is a grid of settlers (rows) × job types (columns). Numbers `1`–`5` set priority; `-` means never. Press `Enter` to edit a cell, arrow keys to navigate.

| Column | Covers |
| :--- | :--- |
| **Farming** | Plow, sow, hoe, harvest, thresh |
| **Animals** | Drive oxen, milk, feed, butcher |
| **Woodcutting** | Fell, buck, split rails and firewood |
| **Carpentry** | Bench crafting, building timber, tool repair |
| **Masonry** | Fieldstone, sod brick, chimneys, cellars |
| **Hauling** | Move items to and from stockpiles |
| **Cooking** | Meals, preserving, smoking, churning |
| **Chores** | Fetch water, tend garden, feed chickens, general errands |
| **Hunting** | Send settler hunting for game |

Doc 13, section 4.4 describes how the scheduler uses these priorities.

---

## 11. Settler & Animal Panels

The right-side panel (doc 07, section 5) shows the full settler or animal sheet when one is selected with `Enter` or `f`.

**Settler sheet:**
```
Jedediah   Adult   Age 34
Health: 98%    Stamina: 62%
Needs: Fed ✓  Water ✓  Rested ~  Warm ✓  Spirits: Fair
Skills: Farming 7  Woodcutting 6  Animals 4  ...
Job: Plowing Field 2 (furrow 14 of 48)
      [Waiting on: nothing]
Holding: Moldboard Plow [condition 71%]
```

**Animal sheet:**
```
Bessie   Milk Cow   Age 2y 4mo
Health: 95%   Condition: 78%
Hunger: Low   Thirst: None
State: Grazing (Pasture 1)
Pregnant: No
Last milked: 6h ago
```

Pressing `j` from a selected settler opens a filtered job list showing only their jobs.

---

## 12. Job List (`j`)

Lists every open job, with columns: **Job**, **Assigned To**, **Status**, **Waiting On**.

```
Job                          Assigned   Status        Waiting On
Plow Field 2 (furrow 14)     Jedediah   In Progress   —
Haul wheat → Main Crib       —          Waiting       No settler free
Build Fence Ln 7 vox 4-18    Caleb      In Progress   —
Cure smokehouse (day 2/4)    —          Running       Time
Town trip (return day 3)     Margaret   Traveling     —
```

Pressing `Enter` on a job opens it: reassign settler, cancel, or change priority.

---

## 13. Alert Log (`a`)

Alerts appear in the top bar as a flashing indicator. The alert log shows the full history.

| Priority | Examples |
| :--- | :--- |
| **Emergency** | Fire at [location], Blizzard approaching, Settler injured, Animal escaping |
| **Urgent** | Settler starving, Settler freezing, Job stuck (with reason), Food stores low |
| **Info** | Town trip returned, Crop ready to harvest, Chimney needs sweeping, Animal pregnant |

Pressing `*` jumps the camera to the most recent emergency alert. `Enter` on an alert acknowledges it; it moves to the history.

---

## 14. New Game Flow

### 14.1 World Creation

1. **World Seed:** type a seed string or press `Enter` for a random one. The seed also appears as a shareable world code (doc 12, section 8).
2. **World Name:** name the save.
3. **Difficulty Preset:** Gentle / Frontier / Harsh (section 15). Individual settings can be changed after.
4. **One-Life Mode:** on or off (doc 12, section 5.4). Cannot be changed later.
5. **Generate** — the coarse map is built (a few seconds). The game moves to claim selection.

### 14.2 Claim Selection (Valley Overview)

The player sees the full 500-acre valley in the Valley Overview zoom (doc 07, section 3). Parcels are highlighted as the cursor moves over them.

**Parcel preview panel (right side):**
```
Parcel [1, 2]   40 acres
Terrain:   Rolling upland prairie
Soil:      ★★★★☆  Silt loam, deep topsoil
Water:     ★★★☆☆  Seasonal creek on south edge
Timber:    ★★☆☆☆  Light woodland along creek
Stone:     ★☆☆☆☆  Some fieldstone on north ridge
Price:     $120 (included in starting grant)
```

The player selects **3 adjacent parcels** (the starting claim). Diagonals count as adjacent. Press `Enter` on each parcel to toggle selection; a preview of the combined 120 acres appears. Confirm with `Enter` when 3 are selected.

The family arrives in early spring at the claim edge with a wagon, 2 trained oxen, 1 milk cow, a few chickens, a dog, tools, seed, and food for a few weeks. The game begins.

---

## 15. Settings

Opened with `O` (shift-o). All settings are grouped here. Settings can be changed at any time during play (except one-life mode).

### 15.1 Presets

Three presets set all danger settings at once. Individual settings can be changed after applying a preset.

| Preset | Description |
| :--- | :--- |
| **Gentle** | Easier weather and danger; less punishing; good for learning |
| **Frontier** (default) | Realistic settings based on historical records |
| **Harsh** | More frequent and severe dangers; unforgiving |

### 15.2 Gameplay Settings

| Setting | Options | Default | Doc |
| :--- | :--- | :--- | :--- |
| **Night handling** | Play through / Fast nights / Skip nights | Play through | 06 |
| **Rest day** | On / Off | On | 13 |
| **Rest day of week** | Day 1–7 of each week | Day 7 | 13 |
| **Family events** | Full / Births & Deaths Only / Off | Full | 13 |
| **Autosave frequency** | Every dawn / Every 2 dawns / Every 3 dawns | Every dawn | 12 |
| **Autosave slots** | 3 / 5 | 3 | 12 |

### 15.3 Danger Settings

Each danger setting has a realistic default and easier / harder options. The active preset is shown; changing any individual setting moves the preset indicator to "Custom."

| Setting | Gentle | **Frontier** | Harsh | Doc |
| :--- | :--- | :--- | :--- | :--- |
| **Forecast accuracy** | Seasoned | Frontier | Frontier (no harder option) | 11 |
| **Tornado rate** | None | ~1 per 250 years | Stormy (~1/25 yr) or Tornado Alley (~1/5–10 yr) | 11 |
| **Year types** | More Normal years | Historical odds | Dry and Hard Winter twice as likely | 11 |
| **Frost dates** | Season 3 days longer, no surprise frosts | Historical dates, small surprise-frost chance | Season 3 days shorter, more surprise frosts | 11 |
| **Blizzard danger** | Day's warning, cold hurts slowly | Builds over hours, short warning | Some blizzards arrive with almost no warning | 11 |
| **Prairie fire spread** | Slower spread, narrower fronts | Historical spread rate | Faster; can jump 1-voxel gaps in high wind | 15 |
| **Lightning ignition** | Requires very dry grass | `grass_dry > 100` | `grass_dry > 60`; also ignites dry stubble | 15 |
| **Chimney fire** | Only very green wood, infrequent | Scales with creosote buildup | More frequent; sweeping essential every winter | 15 |
| **Predator danger** | No wolves or foxes | Wolves and foxes present, realistic frequency | Wolves more frequent; can breach weak fences | 14 |

### 15.4 Display Settings

| Setting | Options | Default |
| :--- | :--- | :--- |
| **Color mode** | TrueColor 24-bit / 256-color / 16-color | TrueColor |
| **Glyph style** | Standard / High-density Unicode | Standard |
| **Scroll speed** | Slow / Normal / Fast | Normal |
| **Voxel inspect** | Auto (follows cursor) / Manual (Enter only) | Auto |
| **Alert sounds** | On / Off | On |

---

## 16. Pause Menu (`q`)

| Option | Action |
| :--- | :--- |
| **Resume** | Return to game |
| **Save** | Save to a named manual slot |
| **Load** | Browse saves and autosaves |
| **Settings** | Open the settings panel (section 15) |
| **World Code** | Display the seed + generator version string for sharing (doc 12, section 8) |
| **Quit to Main Menu** | Saves first (prompts if unsaved changes) |
| **Quit to Desktop** | Saves first |

---

## 17. Quick Reference Card

```
── ALWAYS ─────────────────────────────────────────────────────
Space  Pause/unpause          1/2/3  Speed 1×/2×/5×
Tab    Z-slice mode           </>    Z-slice level down/up
+/-    Zoom                   *      Jump to alert
F5     Quick-save             Esc    Cancel / back
q      Pause menu

── NORMAL MODE ────────────────────────────────────────────────
Arrows / Numpad  Move cursor    Shift + Dir  Leap 10 cells
Enter            Inspect voxel  f            Follow settler/animal
u                Jump to settler t           Town menu

── MODES ──────────────────────────────────────────────────────
d  Designate (Enter..Enter)    b  Build (Enter..Enter)
p  Stockpile (Enter..Enter)    z  Zone (Enter..Enter)
w  Workshop                    W  Work Tab
j  Job List                    a  Alert Log
O  Settings                    q  Pause menu

── DESIGNATE (d → …) ──────────────────────────────────────────
p Plow    s Sow     h Harvest   t Fell trees
c Dig     o Cut sod r Clear rocks b Burn strip
x Cancel designation

── BUILD (b → …) ──────────────────────────────────────────────
w Wall    f Fence   r Floor/Roof   p Pillar
d Door    x Remove built voxels

── STOCKPILE (p → …) ──────────────────────────────────────────
w Timber  s Stone   g Granary   f Food
e Seed    r Refuse  t Tool Shed x Remove zone

── ZONE (z → …) ───────────────────────────────────────────────
p Pasture   n Pen   g Garden   x Remove zone
```

---

## 18. Open Questions

1. **Mouse support:** should basic mouse input (click to move cursor, drag to designate) be supported alongside keyboard? Keyboard is the primary interface, but mouse drag for large area designations is natural.
2. **Keybind remapping:** should players be able to rebind keys? The DF-style fixed layout is deliberate, but accessibility may require alternatives.
3. **Controller support:** out of scope for the first version, but worth flagging for later.
4. **Tutorial mode:** a guided first season that introduces each mode as it becomes relevant — worth building, but after the core systems are playable.
