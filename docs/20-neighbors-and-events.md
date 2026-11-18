# 20 — Neighbors & Events

The homestead is not isolated in a void. Over decades, the 500-acre valley fills with other homesteaders, peddlers, kinfolk from back east, and visiting contractors. Like the weather (doc 11) and world topography (doc 10), all historical events, neighbor claims, and visitors are **pre-planned deterministically from the master world seed**.

Values marked *starting value* are for tuning in milestone M10+.

---

## 1. Principles of Frontier Encounters

1. **Deterministic Scheduling:** Major historical arrivals and regional events are a pure function of `hash(world_seed, event_type, year)`. A reload never alters whether a peddler visits or a letter arrives.
2. **Minimal CPU Cost:** Off-map neighbors and traveling agents are not tracked as living voxel entities. They exist as records in `state.bin` (doc 12) until physical presence is required (e.g. a wagon arriving at the claim border).
3. **Emergent Social Web:** A marriage, death, or dispute alters trade terms, work availability, and regional spirit without full interpersonal conversation trees.

---

## 2. Valley Neighbors & Settlement Progression

The 500-acre valley is split into 40-acre parcels (doc 10, section 5). The player begins with 3 parcels (120 acres). The remaining 6 surveyed parcels and surrounding unsurveyed margins gradually fill with neighboring pioneer families.

```
       500-Acre Valley Settlement Timeline (Approximate)
 Year 1 ──> Only Player Homestead arrives in Early Spring
 Year 2 ──> First neighbor family claims creek bottomland (East Parcel)
 Year 4 ──> Second family claims upland prairie (North Parcel)
 Year 6 ──> Grown pioneer children marry or establish adjacent claims
 Year 8+ ──> Valley reaches capacity (4–6 distinct homestead claims)
```

### 2.1 Neighbor Claim Profiles

Each neighboring family has a pre-determined profile generated from the world seed:

| Family | Focus / Strength | Trade Willingness | Available Livestock / Assets |
| :--- | :--- | :---: | :--- |
| **The O'Callaghans** | Heavy cattle & dairy | High | Breeding bull for hire, butter, salted beef |
| **The Brauers** | Expert carpentry & timber | Medium | Hewn oak beams, extra froes/axes, shingles |
| **The Whitakers** | Large family, grain farming | High | Threshing labor, seed wheat, extra work hands |
| **Solitary Trapper (Old Mac)** | Hunting & scouting | Low | Beaver pelts, venison, early predator warnings |

---

## 3. Neighbor Interactions & Barn-Raisings

Pioneers relied heavily on communal labor for jobs too vast for a single family.

### 3.1 Mutual Aid & Labor Exchanges
* **Work-Swapping:** A player can request 1–2 neighbor workers for a critical 2-day push (e.g., harvesting wheat before hail, or breaking sod). In exchange, the neighbor requests settler labor later in the season. Failing to repay labor ruins neighbor relations and cuts off future aid.
* **Barn-Raising:** When raising a heavy timber barn frame (doc 04) that requires raising massive bents, the player can designate a "Barn-Raising Day". Neighbors arrive for one full workday, dramatically boosting construction speed and eliminating the need for complex mechanical hoists. The host farm must supply a large midday dinner feast and plenty of cider/food (doc 18).

### 3.2 Livestock Breeding Exchanges
* **Stud Service:** Early in the game, the player lacks a bull or boar (doc 14). A settler can lead a cow to a neighbor's pasture for breeding for a small fee (\$1.00–\$2.00 or a bushel of corn).

---

## 4. Traveling Visitors & Peddlers

Visitors enter the claim from the road edge, follow the clearance map to the homestead yard, hitch their horse or wagon, and stay for 1–2 game days.

```
Road Edge ───> [ Arriving Traveler ] ───> Homestead Yard / Porch
                     │
             Trades / News / Letters
                     │
             Departs along road
```

### 4.1 Visitor Types

| Visitor | Frequency | Purpose & Interaction |
| :--- | :--- | :--- |
| **Tinsmith / Peddler** | 1–2 times per year (Summer/Autumn) | Sells small ironware, needles, pots, spices, books; buys rags and beeswax |
| **Circuit Rider (Preacher)** | Once per spring/autumn | Holds a service in the cabin; large Spirits bonus for the family (doc 13) |
| **Livestock Drover** | Late Autumn | Drives cattle/hogs to river market; willing to buy finished hogs for cash |
| **Wandering Laborer** | Seasonal | Looking for temporary room, board, and day-wage work |
| **Land Surveyor** | Rare (Year 3–5) | Verifies parcel boundary markers; resolves boundary disputes |

---

## 5. Frontier Events & Letters

Events arrive either via the post in town (doc 16), letters carried by travelers, or direct physical incidents.

### 5.1 Letters from Back East
Settlers periodically receive letters when visiting the town general store:
* **Family Migration:** *"Cousin Silas and his wife are leaving Ohio. Expect them next spring."* In spring, Silas arrives with a covered wagon, adding 2 adults to the household (or founding an adjacent claim).
* **Inheritance / Small Draft:** A distant relative passes, leaving a \$20–\$50 bank draft redeemable at the town land office.
* **Family Tragedy:** News of a relative's death; settlers experience a temporary dip in Spirits.

### 5.2 Regional Epidemics & Quarantine
* **Cholera Years (Historical 1849–1854):** In a designated outbreak year, the town posts a cholera warning. Traveling to town during an outbreak exposes settlers to severe bad-water sickness (doc 13, section 10.1). Prudent homesteaders enforce self-quarantine until the first autumn freeze clears the outbreak.

### 5.3 Prairie Locust Swarms (Grasshopper Plague)
* **Frequency:** Very rare historical event (e.g., historical 1874 Rocky Mountain Locust swarm), tied to the harsh difficulty setting or a dedicated event trigger.
* **Impact:** A dense cloud sweeps across the valley in late summer. Over 24 hours, standing field crops, garden greens, and pasture grass are stripped to the soil. Stored grain inside roofed cribs and root cellars remains untouched.

---

## 6. Seeded Event Engine

All valley events are generated at world creation using the `EventCalendar`:

```rust
pub struct EventCalendar {
    pub scheduled_events: Vec<ScheduledEvent>,
}

pub struct ScheduledEvent {
    pub year: u16,
    pub season: Season,
    pub day: u8,
    pub event_kind: EventKind, // NeighborArrival, Peddler, EpidemicWarning, Locusts, RelativeLetter
    pub handled: bool,
}
```

* Because the schedule is pure-hashed from the seed, save files only record which events have already fired (`handled = true`).
* If a blizzard or emergency occurs on the arrival day of a peddler, the visit is delayed until fair weather returns.

---

## 7. Tests

* **Deterministic Schedule:** Two games generated with seed `8817263510942` schedule the arrival of the Brauer family on the exact same year and season.
* **Visitor Road Pathing:** Spawn a peddler at the map boundary; verify pathfinding successfully navigates roads and gates without trampling crop fields.
* **Barn-Raising Dinner:** Trigger a barn-raising; verify neighbor attendance depends on sufficient food stores in the root cellar.
* **Locust Protection:** Simulate a locust swarm; verify unroofed wheat fields are stripped while grain in an enclosed granary is preserved.

---

## 8. Open Questions

1. **Neighbor Disputes:** Should disputes occur over stray cattle or water rights if the player dams a creek upstream of a neighbor?
2. **Squatters:** Should unowned parcels occasionally spawn unpermitted squatters who must be bought out or tolerated?
3. **Barter Menu:** Should trading with neighbors use a direct bartering scale (bushels of corn for timber) rather than dollars?
