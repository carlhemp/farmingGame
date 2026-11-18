# 13 - Settlers, Jobs & Needs

The player never controls a settler directly. The player says **what** should be done (designations, stockpiles, standing orders, priorities). The settlers decide **who** does it and **when**, and look after their own needs (doc 01, section 2).

Values marked *starting value* are for tuning in milestone M7.

---

## 1. Principles

1. **Always explain:** The player can always see what a settler is doing and why, and why a job is not getting done ("No axe available", "Too wet to plow", "Nobody allowed to do masonry").
2. **No job per voxel:** Big designations are handed out in pieces (section 4.3). Plowing an acre is about 174,000 voxels; it must never become 174,000 jobs.
3. **Deterministic:** Agents update one at a time in ID order. Choices break ties by job ID. Random values come from a hash of (seed, agent, tick). Same seed and inputs give the same day.
4. **Cheap:** A family is a handful of agents. Pathfinding is the main cost, so paths are cached and reused.

---

## 2. The Scale Rule (short calendar)

The year is about 48 days (doc 06). Each day is real length, so a settler does a real day's work in a day. That shrinks **everything per season and per year** by the same amount, about 7.6 times:

* Work done in a season (acres plowed, rails split).
* Food the family eats in a year.
* Firewood burned in a winter.
* Feed animals need over a winter.

Because these all shrink together, the farm stays in balance. It just covers fewer acres than a real farm. Roughly, a field that fed a real family for a year becomes about one-eighth the size in game. A family might crop about 5 acres where a real one cropped 40.

This is good for the game: the worked land is small enough to watch and to simulate at $6\text{"}$ detail, and the 120-acre claim leaves lots of room to grow into. The other choice, speeding up field work, is listed in section 13. This is still an open decision.

---

## 3. The Settler

```rust
pub struct Settler {
    pub id: AgentId,
    pub name: String,
    pub age_years: u8,
    pub role: FamilyRole,        // Adult, Youth, Child, Elder, HiredHand
    pub pos: AgentPos,           // voxel position plus fine offset (doc 01 footprint)
    pub health: Health,          // injuries, illness, frostbite
    pub stamina: u8,             // 0..100
    pub needs: Needs,            // section 5
    pub skills: Skills,          // section 6
    pub work_settings: WorkSettings, // section 4.4
    pub hands: Option<ItemId>,   // what they are holding
    pub carrying: Option<ItemId>,// sack, basket, or wheelbarrow load
    pub job: Option<JobStep>,    // section 4
}
```

* **Footprint:** a person is about $3 \times 2$ voxels (doc 01).
* **Walking speed:** about $3\text{ mph}$ (about 9 voxels per second) on firm, flat ground (*starting value*). Slower when carrying a heavy load, in mud, in snow, uphill, and when tired.
* **Family at start:** 2 adults and 1 to 3 children (*starting value*). Children can do light chores (fetch water, feed chickens, gather kindling) and get stronger as they grow.

---

## 4. Jobs

### 4.1 Where Jobs Come From

| Source | Examples |
| :--- | :--- |
| **Designations** (docs 01, 04) | Plow, sow, harvest, fell trees, dig, build |
| **Stockpiles** (doc 08) | Haul items into stockpiles; move goods along give and take links |
| **Standing orders** (section 8) | Keep firewood stocked, keep the water barrel full, feed and milk the animals, cook meals |
| **Emergencies** | Fight a fire, get to shelter from a blizzard or tornado, help an injured settler |

### 4.2 A Job Is a List of Steps
Each job breaks into steps, and each step is checked again just before it starts:

```
Fell oak #4127:
  1. Walk to tool shed            (reserve the axe)
  2. Pick up axe
  3. Walk to oak #4127
  4. Chop                         (progress saved if interrupted)
  5. Tree falls                   (doc 05)
  6. Job done; new job created: "Buck and haul oak #4127"
```

* **Reservations:** A settler reserves the tool, the materials, and the target before walking. Two settlers never go for the same axe or the same log. Reservations are released if the job is dropped or the settler is hurt.
* **Progress is kept:** If a job is interrupted (dinner, rain, nightfall), its progress stays with the job, and anyone can pick it up later.

### 4.3 Area Jobs
Plowing, sowing, harvesting, and digging cover big areas. Each designation is **one area job** that hands out small pieces:
* A settler takes one piece, such as one furrow run of about 16 voxels, or one $8 \times 8\text{ ft}$ patch to sow (*starting values*).
* Pieces are handed out in a sensible order: plow furrows run back and forth along the field, harvest starts nearest the crib.
* Several settlers can work the same field on different pieces.

### 4.4 Choosing a Job
When a settler is free (or every 10 game seconds while working on something low priority, *starting value*), they pick the best job they are allowed to do:

1. **Emergencies** always come first.
2. **Urgent needs** come next (section 5).
3. Otherwise, the job with the best **work priority** for that settler wins. Ties go to the nearest job, then the higher skill, then the lower job ID.

**Work priorities** are set per settler and per kind of work, in a grid like RimWorld's work tab:

| | Farming | Animals | Woodcutting | Carpentry | Masonry | Hauling | Cooking | Chores |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Jedediah | 1 | 2 | 1 | 3 | — | 4 | — | 4 |
| Margaret | 1 | 1 | — | — | — | 3 | 1 | 2 |
| Caleb (12) | 3 | 2 | — | — | — | 2 | — | 1 |

`1` is highest. `—` means never. Each designation also has its own priority (1 to 5), so "harvest the wheat before the hail" can jump the queue.

### 4.5 What Stops a Job
* **Darkness:** most outdoor work stops at night (sunrise and sunset come from the calendar). Lamplight work indoors is fine.
* **Weather:** heavy rain stops plowing and harvesting; lightning stops felling; blizzards stop everything outdoors.
* **Ground:** plowing waits while the soil is too wet (`poreWater > 12`) or frozen (`isFrozen`).
* **Missing things:** no tool, no materials, no path. The job shows the reason and waits.

---

## 5. Needs

| Need | Filled By | When It Gets Urgent | If Ignored |
| :--- | :--- | :--- | :--- |
| **Food** | Meals: breakfast, a big midday dinner, supper | Missed a meal | Weak, slow work, then starving |
| **Water** | Well, spring, creek, water barrel | A few hours without in hot work | Heat sickness, faster in summer |
| **Rest** | Sleep at night; short rests when stamina is low | Stamina low, or very late | Slow, clumsy, more injuries |
| **Warmth** | A room with a hearth or stove; good clothing | Feels-like temperature too cold (doc 11) | Frostbite, then death in severe cold |
| **Spirits** | Shelter, varied food, rest days, family, a good harvest | Long overwork, poor food, a death in the family | Slower work, more mistakes |

* **Stamina** drains with hard work, faster for heavy jobs (plowing sod, felling, hauling stone) and in heat. It refills with rest and sleep.
* **Spirits** are kept simple: one value with a few causes shown to the player. There is no deep mood system.

---

## 6. Skills

Skills run from 0 to 10 and grow with practice (*starting values*):

| Skill | Covers |
| :--- | :--- |
| **Farming** | Plowing, sowing, hoeing, harvesting, threshing |
| **Animals** | Driving oxen, milking, feeding, butchering |
| **Woodcutting** | Felling, bucking, splitting rails and firewood |
| **Carpentry** | Timber building, tools, wheelbarrows, furniture |
| **Masonry** | Fieldstone, sod brick, chimneys, cellars |
| **Cooking** | Meals, preserving, smoking, salting |
| **Weather** | Forecasts (doc 11 open question) |

Higher skill means faster work and better results (straighter furrows, fewer wasted seeds, stronger walls). Age matters too: children are slower and tire fast; elders know a lot but have less stamina.

---

## 7. Work Rates

Rough historical rates, to be checked and then tuned (*starting values*):

| Task | Rate per working day | Notes |
| :--- | :--- | :--- |
| Breaking prairie sod | about 1 to 2 acres | Breaking plow and several yoke of oxen. The first plowing of prairie is the hardest work on the farm |
| Plowing broken ground | about 1 to 2 acres | Walking plow and one team |
| Harvesting wheat by cradle | about 2 acres | Plus binding and shocking |
| Felling a tree | from minutes to hours | By trunk diameter and wood (doc 05) |

Each rate is changed by skill, tool quality, stamina, weather, and the soil itself: wet clay slows plowing, and dense sod slows it more (doc 01).

---

## 8. Daily Routine and Standing Orders

### 8.1 Default Day
| Time | Activity |
| :--- | :--- |
| Dawn | Wake, morning chores (animals, water, fire), breakfast |
| Morning | Work |
| Midday | Dinner, the main meal |
| Afternoon | Work |
| Dusk | Evening chores, supper |
| Evening | Indoor work by lamplight (mending, shelling corn, threshing in winter), then sleep |

The player can change the routine for each settler, for example to let a youth sleep later.

### 8.2 Standing Orders
Standing orders are rules the family follows without new designations (*starting values*):
* Keep at least 3 days of firewood by the house.
* Keep the water barrel full.
* Feed and water animals at dawn and dusk; milk cows twice a day.
* Cook meals from stored food.
* Put tools back in the shed at the end of the day.

### 8.3 Rest Day (setting)
A weekly rest day, as many frontier families kept a Sabbath. It is a setting: on by default, and the player can turn it off or pick the day.

* Only chores go on: animals, water, fire, and meals. No field work or building.
* Spirits rise, and stamina fully recovers.
* Emergencies still come first: a fire, a flood, or a storm about to flatten ripe wheat.
* The cost is real: with 12-day seasons, a rest day takes almost 2 days out of each season. That is part of the trade-off.

---

## 9. Pathfinding

* **Two levels** (summary, risk 4): a coarse graph of connected $8\text{ ft}$ chunks for long trips, and fine steering on the $6\text{"}$ grid near the agent.
* **Clearance map:** each size class (person, ox, wagon) only goes where it fits (doc 01).
* **Path cost:** mud, snow, slopes, and deep water cost more. Walking through standing crops costs extra, so settlers use lanes and field edges. Every footstep adds a little `compaction` (doc 03), so well-used routes become packed paths on their own.
* **Doors and ladders** connect rooms and levels.
* **Caching:** common routes (house to well, house to barn) are cached and only recomputed when something changes along them.

---

## 10. Health, Injury, and Death

* **Injuries** come from falls, axe slips, being kicked by an ox, collapsing structures (doc 04), and falling trees (doc 05). They heal with rest; bad ones heal slowly and may leave a lasting limp or weakness.
* **Cold and heat** (doc 11): frostbite, freezing, heat sickness.
* **Illness:** see 10.1.
* **Death** is permanent. The family grieves (spirits drop), and the work falls to everyone else. In one-life mode (doc 12) there is no going back.

### 10.1 Illness
Frontier illness is in. The first version works out **risk from the land and the farm layout**, using things the sim already tracks. It does not add a new "germs" value to the soil. A settler's daily chance of falling ill comes from that risk.

| Illness | Where the Risk Comes From | How the Player Lowers It |
| :--- | :--- | :--- |
| **Ague (malaria)** | Standing water that lasts several days in warm months (surface layer, doc 03), counted more the closer it is to where the family sleeps | Drain sloughs and wet ground; build the house on high ground away from them |
| **Bad-water fever** (typhoid, dysentery) | What can reach the well or spring: privies, barnyards, and manure piles **uphill** of it along the soil-water flow (doc 03), in soil that water moves through fast (sand), over a shallow water table | Dig the well uphill of the privy and barn, dig deeper, line the well, use a clean spring |
| **Winter sickness** (colds, pneumonia) | Cold, damp rooms, overwork, poor food | A warm, dry house, rest days, good stores |
| **Cholera** (rare event) | Brought by travelers or from town in outbreak years | Stay away from town during an outbreak |

* **Well risk** is worked out when a well is dug, and again whenever a privy, barn, or manure pile is built or moved nearby. A source uphill in the water's path counts heavily; one downhill barely counts.
* **The player can see it:** inspecting a well shows "Water: sweet / doubtful / bad" and the reason, for example "privy 40 ft uphill, sandy soil."
* **Effects:** ague brings fever days that come and go and leave the settler weak, but rarely kills. Bad-water fever is serious and can kill children and elders. Nursing by another settler and rest improve recovery.
* **Later option:** actually track contamination moving through the groundwater, as a small plume around each source using the hourly seepage step (doc 03). It is more realistic and more work, so it only comes in if the simple version feels fake.

---

## 11. The Family Over Time

The family changes over the years. With 48-day years (doc 06), a child grows up in about 8 to 10 game years, so this is part of the long game.

* **Children grow up:** Child, then Youth (around 10), then Adult (around 16) (*starting values*). They learn skills by working alongside adults.
* **Hired hands:** Hire help in town for a few days at harvest, for a season, or for a year. Pay is cash, room and board, or a share of the crop. Hands who are not paid or are treated badly leave.
* **Marriage:** A grown child can marry. Then either:
  * the spouse moves in, adding an adult to the household; or
  * the couple moves on. They might take up a parcel next door and become neighbors who trade and help at harvest, or they might leave the valley.
* **Births:** Married couples may have children over the years.
* **Relatives:** A letter from back east says a brother's family is coming in spring. They arrive needing shelter and food.
* **Elders:** Settlers age, slow down, and eventually die.

Most of these can be turned on or off as settings.

---

## 12. What the Player Sees

* **Settler panel** (doc 07): each settler's current job, the reason for it, and their needs.
* **Work tab:** the priority grid in section 4.4.
* **Alerts:** a settler is starving, freezing, or hurt; a job is stuck and why; a storm is coming.
* **Job list:** every open job, who has it, and what it is waiting for.

---

## 13. Open Questions

Decided: an optional weekly rest day (8.3), frontier illness worked out from the land (10.1), the family growing and changing over time (11), and animals getting their own doc before M7 (doc 14).

1. **Scale rule (section 2):** Keep real work speed and smaller fields, or speed up big field tasks so fields stay closer to real size? Still being thought over.
2. **Family order:** Which of the family changes in section 11 come first?
3. **Contamination plume:** Is the simple well-risk model enough, or should contamination be tracked through the groundwater?
