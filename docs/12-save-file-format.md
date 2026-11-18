# 12 - Save File Format

A save stores the world seed, the coarse map, and **only what differs from what the seed would generate** (doc 02, section 2). Everything else is regenerated on load.

Values marked *starting value* are for tuning in milestone M4.

---

## 1. Goals

1. **Small:** Untouched land costs nothing. A worked claim should stay in the tens to low hundreds of MB.
2. **Fast autosave:** Only the parts of the world that changed since the last save are written.
3. **Never corrupt:** A crash or power cut during a save never damages the previous save.
4. **Exact:** Save, quit, and load gives the identical world. Running on after a load gives the same result as never having saved.
5. **Versioned:** Old saves keep loading after the game is updated.

---

## 2. Save Layout

One save is one folder, so autosave can rewrite single files instead of one huge file:

```
saves/
  Pleasant Creek/
    save.json              header (section 3)
    coarse.bin             the coarse map (doc 10)
    state.bin              time, weather, town, settings
    agents.bin             settlers, animals, wagons
    entities.bin           plants, trees, items, containers, doors
    jobs.bin               designations, job queue, stockpiles and links
    regions/
      r.0.0.bin            one region of world data (section 4)
      r.0.1.bin
      ...
  Pleasant Creek.autosave-1/
  Pleasant Creek.autosave-2/
  Pleasant Creek.autosave-3/
```

* **Regions:** $32 \times 32$ column chunks ($256\text{ ft}$ square, about 1.5 acres) each (*starting value*). A 120-acre claim is about 80 regions. Only regions that changed are rewritten.
* **Compression:** every `.bin` file is compressed (zstd or similar). Regions also use the palette and run-length encoding from doc 02 before that.
* **Encoding:** a compact binary format with explicit field order (for example `serde` with `postcard` or `bincode`). The header is plain JSON so a person can read it.

---

## 3. Header (`save.json`)

| Field | Example | Purpose |
| :--- | :--- | :--- |
| `format_version` | `1` | Save layout version (section 7) |
| `generator_version` | `1` | World generator version (doc 10); picks the generator code to use |
| `game_version` | `0.4.0` | Game build that wrote it |
| `world_seed` | `8817263510942` | The master seed |
| `save_name` | `Pleasant Creek` | Shown in the load menu |
| `game_date` | `Year 3, Summer, Day 7, 6:12` | Shown in the load menu |
| `tick` | `9871920` | Exact game time in ticks |
| `claim_parcels` | `[[1,1],[1,2],[2,2]]` | Owned 40-acre parcels (doc 10) |
| `settings` | night setting, difficulty settings | Doc 06 and doc 11 |
| `one_life` | `true` | One-life mode is on (section 5.4) |
| `written_at` | real date and time | For sorting saves |

---

## 4. What Each File Holds

### 4.1 Regions (`regions/r.X.Y.bin`)
For each column chunk in the region that differs from the seed:
* **Surface layer** (doc 02, section 5): ground height stored as a difference from the generated ground height, plus water depth, snow, ice, and grass. Most values are zero, so this compresses very well.
* **Sub-chunks:** the `Compressed` difference from the seed for each sub-chunk that has changed (doc 02, section 2). Awake `Dense` sub-chunks are compressed into the same form when saved.
* **Sub-chunk entity lookup lists** (doc 02, section 7).
* **Sleep state:** awake or asleep, and any "time owed" for deferred catch-up (doc 06, section 3.3).

Column chunks that match the seed are left out entirely. A region with nothing changed has no file.

### 4.2 Other Files

| File | Contents | Not Saved (rebuilt on load) |
| :--- | :--- | :--- |
| `coarse.bin` | The coarse map (under 1 MB) | |
| `state.bin` | Tick, calendar, current weather regime, year type, town prices and stock, town upgrades, settings | The weather plan (recomputed from the seed) |
| `agents.bin` | Settlers, animals, wagons: position, health, needs, skills, inventory, current job and path goal | Paths (recomputed) |
| `entities.bin` | Plant and tree records, items, containers, doors | Trees that are still procedural (doc 10) |
| `jobs.bin` | Designations, job queue, reservations, stockpile zones, filters, give and take links | Item lookup buckets (rebuilt from items) |
| | | Rooms (flood fill, doc 04), support values (doc 04), clearance maps (doc 01) |

Rebuilding these on load is cheap, and it means they can never be saved out of step with the world.

The coarse map could be regenerated from the seed, but it is saved anyway. It is small, it loads faster, and it protects the save if the coarse map generator is ever changed by mistake.

---

## 5. How Saving Works

### 5.1 Autosave
* **When:** every game dawn (about every 20 real minutes at 1x, *starting value*) and when quitting.
* **What:** only regions marked changed since the last save, plus the small files.
* **Rotation:** 3 autosave slots, oldest overwritten first (*starting value*). Manual saves are never overwritten by autosave.

### 5.2 Saving Without Stopping the Game
1. At the end of a step, take a snapshot: copy awake `Dense` sub-chunks, changed surface tiles, and the small tables. Sleeping data is already compressed and is not changed in place, so it can be shared, not copied.
2. The game keeps running. A background thread compresses and writes the snapshot.
3. The snapshot copy is about the size of the awake set (a few hundred MB at most, doc 02), which takes a fraction of a second.

### 5.3 Never Corrupt
* Each file is written to a temporary name first, flushed to disk, then renamed over the old file. A rename is all-or-nothing on Windows and Linux.
* `save.json` is written last. It lists a checksum for every other file in the save. On load, a file that fails its checksum makes the game fall back to the newest good autosave and say so.

### 5.4 One-Life Mode (optional)
An option chosen when the world is created. It cannot be turned on or off later.

* There is one save. Manual saves and loading older saves are turned off.
* The game saves at every dawn and on quit, always over the same save.
* Crash safety still works: a hidden backup of the previous save is kept, and it is used **only** if the current save fails its checksum. It cannot be loaded by choice.
* What happens stays happened: a failed harvest, a burned barn, or a settler lost in a blizzard cannot be undone.
* If the whole family dies, the save ends. It can still be opened to look around the valley, but not played.
* The load menu marks one-life saves.

---

## 6. How Loading Works

1. Read `save.json`. Check `format_version` and `generator_version` (section 7).
2. Load `coarse.bin` and the small files.
3. Load regions near the camera and around the agents first, so the game can start. Load the rest in the background.
4. Rebuild rooms, item lookup buckets, clearance maps, and anything else in the "not saved" column.
5. Recompute the weather plan from the seed. Resume at the saved tick.

---

## 7. Versions and Old Saves

* **Generator version:** A released generator version is **frozen**. Its code is never changed. A change to world generation becomes a new version alongside the old one, and new worlds use the newest. Old saves keep using the version they were made with, so their untouched land comes back exactly as it was.
* **Format version:** When the save layout changes, the game includes a migration that upgrades older saves on load. The upgraded save is written as a new save; the original is kept.
* **Voxel bit layout:** Changing the voxel layout (doc 02) is a format change and needs a migration.
* **New fields:** New fields get a default value when loading older saves.

---

## 8. Sharing Worlds

* **World code:** the seed plus generator version, as a short string. Another player can type it in to start in the same valley with the same terrain and the same planned weather.
* **Whole save:** the save folder can be zipped and shared. It holds no personal data.

---

## 9. Size Estimates (to measure in M4)

| Part | Estimate |
| :--- | :--- |
| Header and small files | Under 1 MB |
| Coarse map | Under 1 MB |
| Agents, entities, jobs | A few MB (more if crops are records; doc 05 open question) |
| Regions, newly claimed land | Near zero |
| Regions, fully worked 120-acre claim | Tens of MB to low hundreds of MB |

---

## 10. Tests

* **Round trip:** save, load, and compare: the two worlds are identical.
* **Run-on match:** run N ticks, save, load, run M more ticks. The result equals running N + M ticks without saving.
* **Crash during save:** kill the game partway through a save. The previous save still loads.
* **Corrupt file:** damage one region file. The game notices, falls back to the newest good autosave, and says so.
* **Old saves:** saves from every past `format_version` load and upgrade correctly (keep sample saves in the test suite).
* **Frozen generator:** each released generator version still produces the same golden-seed sub-chunks.

---

## 11. Open Questions

Decided: one-life mode is an option (section 5.4).

1. **Autosave slots and timing:** Are 3 slots saved every dawn the right default?
2. **Save browser:** Should the load menu show a small Valley Overview picture of each save?
