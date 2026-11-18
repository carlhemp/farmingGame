# 14 - Animals

Animals are the farm's engines, its larder, and a daily duty. Oxen break the prairie and haul the timber. Cows, hogs, and chickens feed the family and bring in butter and egg money. All of them need feed, water, and shelter, every day, all winter.

Values marked *starting value* are for tuning in milestone M7 and later.

---

## 1. Animals Are Agents

Like settlers (doc 13), animals live in the agent table with a position and a footprint. They are not stored in the voxel grid.

```rust
pub struct Animal {
    pub id: AgentId,
    pub species: Species,      // Ox, Horse, Cow, Bull, Hog, Chicken, Sheep, Dog, wild kinds
    pub name: Option<String>,  // oxen and cows usually have names
    pub age_days: u16,
    pub sex: Sex,
    pub pos: AgentPos,
    pub health: Health,
    pub condition: u8,         // body condition, 0..100 (thin to fat)
    pub hunger: u8,
    pub thirst: u8,
    pub stamina: u8,           // for working animals
    pub trained: bool,         // oxen to the yoke, horses to harness
    pub home: Option<ZoneId>,  // assigned pasture or pen
    pub state: AnimalState,    // Grazing, Penned, Working(RigId), Led, Fleeing, ...
    pub pregnant_until: Option<u32>, // game day
}
```

---

## 2. The Animals

| Animal | Footprint (voxels) | Main Use | Feed | Notes |
| :--- | :---: | :--- | :--- | :--- |
| **Ox** | about $6 \times 16$ | Plowing, hauling, logging | Grass, hay | Slow, strong, steady, cheap to keep. The main draft animal on a new farm |
| **Horse** | about $4 \times 16$ | Faster hauling, riding to town | Grass, hay, grain | Faster but needs grain and costs more. Comes later |
| **Milk cow** | about $5 \times 14$ | Milk, butter, calves | Grass, hay | Gives milk after calving. Butter sells in town |
| **Hog** | about $4 \times 8$ | Pork, lard | Corn, scraps, acorns and nuts in the woods | Roots up ground; good at escaping |
| **Chicken** | about $1 \times 2$ | Eggs, meat | Grain, bugs, scraps | Needs a coop at night; prey for foxes and hawks |
| **Sheep** | about $3 \times 6$ | Wool, mutton | Grass, hay | Later; wolves love them |
| **Dog** | about $2 \times 5$ | Warns of wolves, helps drive stock | Scraps | Barks at night when predators come |

Clearance (doc 01): small animals use the person size class or smaller. Cows, oxen, and horses use the ox class. A yoked team with a plow or wagon uses the wagon class.

---

## 3. Draft Work and Rigs

### 3.1 Teams
* Oxen work in **yoked pairs**. A team is one or more pairs.
* A settler drives the team. The **Animals** skill (doc 13) sets how well: straighter furrows, fewer balks, less tiring.
* Oxen walk about $2\text{ mph}$ when working (*starting value*), slower than a person. Horses are faster.
* Young steers must be **trained** to the yoke before they can work. That takes time and the Animals skill. The family starts with a trained team.

### 3.2 Rigs
When a settler hitches a team to an implement, the settler, team, and implement move as **one rig**, with one combined footprint and speed:

| Implement | Use | Notes |
| :--- | :--- | :--- |
| **Breaking plow** | First plowing of prairie sod | Needs several yoke of oxen to do well |
| **Walking plow** | Plowing broken ground | One team |
| **Harrow** | Smoothing plowed ground before sowing | One team |
| **Wagon** | Hauling (doc 08) | Can bog down in mud |
| **Travois / skid** | Dragging logs out of the woods (doc 08) | |

### 3.3 Pulling Power
How fast a rig works depends on the team's pulling strength against how hard the ground resists:

$$\text{Work rate} = \text{base rate} \times \frac{\text{team strength}}{\text{ground resistance}}$$

* **Ground resistance** rises with dense prairie roots, clay, compaction, and wet soil (doc 03).
* **Team strength** depends on how many pairs, their condition, their stamina, and the driver's skill.
* **Breaking sod** is the hardest job on the farm. One family team breaks it slowly. Historically, settlers often **hired a breaking crew** with a big team and a heavy plow. The town can offer that (doc 01).

### 3.4 Working Animals Need Rest
Working animals tire like settlers do. They rest at midday, need water in heat, and work poorly when thin or sick.

---

## 4. Feeding and Water

### 4.1 Grazing
* Grazing animals eat the standing **grass** in the surface layer (doc 02, section 5).
* Grass grows back over the growing season, faster with rain and warmth (doc 11).
* **Overgrazing** wears grass down to bare ground. Bare ground loses root protection (`hasRoots`), so it erodes in storms (doc 03), and hooves pack it (`compaction`). Pasture size and rotation matter.
* Hogs turned into the woods in autumn eat acorns and nuts (mast), which saves corn.

### 4.2 Winter Feed
* When grass is gone or under snow (doc 11), animals eat stored **hay** and **grain**.
* Hay is meadow or prairie grass cut, cured dry, and stacked in summer (doc 01). Curing uses the grass dryness value in the surface layer; rain on cut hay spoils it.
* **Feed math follows the scale rule** (doc 13, section 2): animals eat a real day's feed each day, and winter is about 12 days long. Roughly, an ox eats 20 to 30 lb of hay a day (*starting value*).
* Running out of feed in late winter is a classic frontier disaster: animals grow thin, then sicken, then die.

### 4.3 Water
Animals drink from the creek, ponds, or a trough. Filling the trough is a standing order (doc 13, section 8.2). In winter, ice must be broken (doc 11).

---

## 5. Pastures, Pens, and Shelter

* **Zones:** The player marks pasture and pen zones (`z` then drag, like stockpiles in doc 08) and assigns animals to them.
* **Fences** (doc 04) keep animals in. Weak or broken fences let them out, and loose stock tramples crops. Hogs test every fence.
* **Gates** must be wide enough for the animals and rigs that use them (doc 01).
* **Shelter:** In a blizzard or deep cold, animals left outside can freeze to death. A barn or shed that forms a room (doc 04) protects them. Chickens need a closed coop at night against predators.
* **Mud:** Hooves pack wet ground in pens and lanes (doc 03), so busy pens turn to mud in wet weather.
* **Manure:** Penned animals leave manure. Hauling it to the manure pile (doc 08) is a chore. Spread on fields it adds N and organic matter (doc 05). Barnyards and manure piles also raise the bad-water risk for wells downhill of them (doc 13, section 10.1).

---

## 6. Animal Health

* **Needs:** feed, water, rest for working animals, and shelter in severe cold.
* **Condition:** body condition rises with good feed and falls with hunger, cold, and hard work. Thin animals work slower, give less milk, and get sick more.
* **Illness and injury** (*starting list*):
  * **Bloat:** cattle turned onto wet clover can bloat, sometimes fatally. A real risk that comes with clover rotation (doc 05).
  * **Foot rot:** from standing in muddy pens.
  * **Injury:** from fences, rough ground, or wolves.
* **Death:** an animal that dies fresh can be butchered for meat if found quickly, in cold weather.

---

## 7. Breeding, Products, and Butchering

The scale rule applies (doc 13, section 2): **anything that happens per day stays real, and anything that happens per season or per year follows the short calendar.**

| Animal | Per Day (real) | Per Year (calendar) |
| :--- | :--- | :--- |
| **Milk cow** | About 1 to 2 gallons of milk while in milk; milked twice a day | One calf a year, if bred |
| **Hen** | About one egg most days in spring and summer, few in winter | Broody hens raise chicks in spring |
| **Sow** | | One or two litters a year |
| **Sheep** | | Sheared once in spring |
| **Young stock** | | Grow to working or butchering size in 1 to 3 years |

* **Breeding:** needs a male. The family may not own a bull or boar at first; a cow can be taken to a neighbor's or to town (doc 01).
* **Butter:** churning milk into butter is a chore. Butter and eggs sold in town were real frontier income.
* **Butchering:** done after the first hard freeze in late autumn, so the meat keeps (doc 11). The meat is salted, smoked, or packed into barrels (doc 08).

---

## 8. Wild Animals and Pests

Wild animals are few and simple. They wander, flee, or hunt by simple rules.

| Animal | Role |
| :--- | :--- |
| **Deer, turkey, prairie chicken** | Game. A settler with a rifle can be sent hunting for meat. |
| **Wolves, coyotes** | Take chickens, lambs, calves, and young pigs at night. Pens, coops, and a dog help. |
| **Foxes, hawks** | Take chickens. |
| **Rattlesnakes** | A rare bite while working in tall grass. |

### 8.1 Predator Danger (difficulty setting)

| Easier | **Realistic (default)** | Harder |
| :--- | :--- | :--- |
| Foxes and hawks only; no wolves or coyotes | Wolves and coyotes take poultry and young stock at night now and then, more often in hard winters. Grown cattle and oxen are rarely touched | Bolder packs: more raids, and in hard winters they go after sheep, calves, and weak grown stock |

On every setting, wolves do not hunt people. Historically, healthy wild wolves on the plains were a danger to stock, not to settlers.

### 8.2 Grasshopper Plague (difficulty setting)
Swarms of Rocky Mountain locusts swept the plains in some years, worst in 1874, eating whole crops in a few days.

* **When it comes:** in mid to late summer, as a swarm that darkens the sky.
* **What it does:** eats standing crops, garden plants, and grass in a few days, then moves on. It can leave eggs in the soil that hatch the next spring.
* **What helps:** crops already harvested are safe; root crops underground do better; smoky fires and ditches save a little; plowing in autumn breaks up the eggs.
* **Planned from the seed** like the weather (doc 11, section 4.1), so a reload does not change it.

| Easier | **Realistic (default)** | Harder |
| :--- | :--- | :--- |
| None | About once in 15 to 20 years | About once in 5 to 8 years |

The realistic rate is a rough estimate from the history of plains swarms in the 1850s to 1870s, not a record for any one valley.

### 8.3 Presets
The Gentle, Frontier, and Harsh presets in doc 11 (section 7) also set these two.

---

## 9. Starting Animals

The family arrives with (*starting values*): a trained team of 2 oxen, 1 milk cow, a few chickens in a crate on the wagon, and a dog. More animals are bought in town or raised.

---

## 10. Cost

* A farm has tens of animals at most, updated with the settlers.
* Grazing and penned animals wander locally and do not need long paths. Only animals being moved or working use full pathfinding (doc 13, section 9).
* Grass growth is part of the daily plant step (doc 06).

---

## 11. Tests

* **Plowing:** a settler hitches the oxen, plows a designated field, and the rate changes with soil and wetness as section 3.3 says.
* **Overgrazing:** too many animals on a small pasture leaves bare ground that erodes in the next storm.
* **Winter feed:** a herd with too little hay grows thin and starts to die before spring; one with enough comes through.
* **Shelter:** animals left out in a blizzard are hurt or die; sheltered ones are fine.
* **Same seed, same result:** animal behavior is deterministic like settlers.

---

## 12. Open Questions

Decided: predator danger and grasshopper plague are difficulty settings with realistic defaults (sections 8.1 and 8.2).

1. **Horses:** Available from the start in town, or only after the town grows?
2. **Breaking crews:** Should hiring a breaking crew be part of the first version, or come with the town doc?
