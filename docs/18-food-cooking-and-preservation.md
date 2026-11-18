# 18 — Food, Cooking & Preservation

Food is what the farm is for. Every acre plowed, every hog penned, every barrel of salt hauled from town ultimately comes back to the question of whether the family eats through winter. Doc 08 covers spoilage. This doc covers what food exists, how much people need, how meals are made, and how the autumn harvest is turned into stores that last until spring.

Values marked *starting value* are for tuning in milestone M10+.

---

## 1. The Scale Rule

As with all yearly things (doc 13, section 2), the scale rule applies. Each person eats a real day's food each day. Because a year is about 48 days, annual food consumption is roughly one-eighth of a real year's food. A family that in real life would eat 400 lb of pork in a year eats about 50 lb in game. The numbers in this doc follow that compression. Daily quantities stay realistic; it is the totals that shrink.

---

## 2. Food Types

Foods are items in the `itemTable` (doc 08, section 2.1). Each food item has a `nutrition` value, a `variety_tag`, a `condition` (freshness), and `moisture_rot` (spoilage).

### 2.1 Staple Foods

| Food | Source | Nutrition (per unit) | Notes |
| :--- | :--- | :---: | :--- |
| **Cornmeal** | Threshed corn, ground (or milled) | Medium | The frontier staple. Cornbread, mush, johnnycakes |
| **Wheat flour** | Threshed wheat, ground or milled | Medium | Bread, biscuits. Better morale than corn alone |
| **Salt pork** | Hog butchering + salt | High | Preserved in brine in a barrel. Lasts all winter |
| **Smoked bacon / ham** | Hog butchering + smokehouse | High | Longer shelf life than salt pork; better variety |
| **Dried beans** | Harvested field beans | Medium | Easy to store, slow to cook |
| **Potatoes** | Root cellar stores | Medium | Bulk calorie; keeps well in a cold cellar |
| **Turnips** | Root cellar stores | Low–Medium | Filler; settlers grow tired of them quickly |

### 2.2 Supplemental Foods

| Food | Source | Nutrition | Notes |
| :--- | :--- | :---: | :--- |
| **Milk** | Milk cow (doc 14) | Medium | Perishable; must be used within 1–2 days or churned |
| **Butter** | Churned from milk | Medium | Keeps longer than milk; sold in town |
| **Eggs** | Hens (doc 14) | Medium | Seasonal; abundant in spring, scarce in winter |
| **Fresh vegetables** | Kitchen garden (section 5) | Low–Medium | Onions, cabbages, dried herbs, beans |
| **Wild game** | Hunting (doc 14, section 8) | High | Venison, turkey, prairie chicken; fresh only |
| **Lard** | Hog fat at butchering | Medium | Cooking fat; also preserved in crocks |
| **Cheese** | Milk + rennet | Medium | Later addition; keeps longer than milk |

### 2.3 Luxuries

Luxuries provide no extra nutrition but raise spirits significantly when present:

| Item | Source | Spirit Effect |
| :--- | :--- | :--- |
| **Coffee** | Town general store | Strong; a morning without coffee is a noticeable mood dip |
| **Sugar / molasses** | Town general store | Sweetens cornbread and porridge; children especially |
| **Salt** | Town (also needed for preservation) | Improves palatability of all food |
| **Spices** | Town; expensive | Small but meaningful variety boost |
| **Cider** | Orchard (later); also bought in town | Modest spirits boost; a barrel at harvest is a tradition |

---

## 3. Daily Food Need

Each settler has a **daily food need**, filled by meals (section 4). Needs vary by role and work intensity:

| Role | Base Daily Need | Heavy Work Day | Notes |
| :--- | :---: | :---: | :--- |
| Adult (working) | 100 units | 130 units | Plowing sod, hauling stone, felling trees |
| Adult (light work) | 80 units | — | Indoor winter work, supervision |
| Youth (10–15) | 70 units | 90 units | Growing fast; can eat more than expected |
| Child (under 10) | 50 units | — | |
| Elder | 65 units | — | Smaller appetite; needs more variety for spirits |

*Starting values.* Heavy work is defined as a job with high stamina drain (doc 13, section 5).

If a settler's food need is not met:
* **Missed one meal:** stamina recovers poorly overnight.
* **Missed two meals in a row:** work speed drops, spirits fall.
* **Genuinely starving (days without food):** health drains; death is possible for children and elders first.

---

## 4. Meals

### 4.1 Three Meals a Day

Settlers eat three times a day at set times (doc 13, section 8.1):

| Meal | Time | Size | Typical Food |
| :--- | :--- | :---: | :--- |
| **Breakfast** | Dawn | Small | Cornmeal mush, leftover cornbread, coffee |
| **Dinner** | Midday | Large | Hot meal: salt pork or game with cornbread and vegetables |
| **Supper** | Dusk | Medium | Lighter: bread, butter, cold leftovers, milk |

Dinner is the main meal and fills most of the daily need. Skipping dinner to keep working costs more in the long run.

### 4.2 Cooking as a Job

Cooking is a standing order assigned to the settler with the highest Cooking skill (doc 13, section 8.2). It is a timed job at the hearth or stove:

1. **Gather ingredients** from food stockpiles (root cellar, grain bin, salt barrel).
2. **Cook** at the hearth — a timed job consuming firewood from the fuel pile. A simple meal (cornmeal mush) takes less time than a full dinner (salt pork, cornbread, boiled turnips).
3. **Serve** — the meal is placed as a food item in the common area or on the table. Other settlers eat it when their meal time arrives.

Leftover food from a meal that is not fully eaten can be carried over to supper. Food left out more than 1 game day at warm temperatures (`soilTemp` band 4+) begins to spoil (`moisture_rot` rising).

### 4.3 Cooking Fuel

Each meal consumes firewood from the indoor fuel stack (a designated stockpile inside the cabin, separate from the main timber yard). The hearth or stove must have fuel; a settler will not cook without it. Fuel consumption scales with meal size and cook time (*starting value*: 2–5 lb of dry firewood per meal for a family of 4–6).

Green (unseasoned) firewood can be burned but produces poor heat, extends cook time, and raises the chimney fire risk (doc 15, section 3.3).

---

## 5. Variety and Spirits

Eating the same food every day wears on morale. The game tracks a **variety score**: a rolling average of how many different `variety_tag` values have appeared in recent meals.

| Recent Variety | Spirit Effect |
| :--- | :--- |
| 4+ different foods in the last 3 days | Small spirits bonus |
| 2–3 different foods | Neutral |
| Only 1 food (e.g., cornmeal mush every meal) | Spirits penalty, growing over time |
| Coffee present in breakfast | Small daily spirits bonus, independent of variety |
| Luxury item in dinner (cider, spiced dish) | Moderate spirits bonus |

The player sees variety warnings in the settler panel: *"The family is tired of cornmeal"*. The fix is to unlock more food sources — the kitchen garden, a milk cow, hunting, the smokehouse — not to stockpile more corn.

---

## 6. The Kitchen Garden

A kitchen garden is a small designated planting zone near the cabin, separate from the main crop fields. It is sown, tended, and harvested by the settler assigned to cooking or general chores.

* **Size:** typically $20 \\times 20\\text{ ft}$ to $40 \\times 40\\text{ ft}$ (*starting value*), small enough to tend by hand.
* **Crops:** onions, cabbages, turnips, dried beans, herbs (flavoring — spirits bonus), squash, and later garden peas.
* **Watering:** kitchen garden crops are more drought-sensitive than field crops. Without rain, the cook settler waters them by hand from the well, consuming 1–2 buckets per bed per day in dry weather.
* **No plow needed:** beds are hand-dug with a spade and hoe. This is accessible from the very first day of the game.
* **Yield:** small but high-variety. A good kitchen garden produces fresh vegetables through summer and stores for winter, meaningfully reducing the variety problem.

Kitchen garden crops follow the same growth stages as field crops (doc 05, section 2.1) but are stored in the root cellar stockpile rather than the granary.

---

## 7. Autumn Preservation

The autumn butcher and preservation work is the most labor-intensive cooking event of the year. It happens after the first hard freeze (doc 11, section 2.2), when cold keeps the meat from spoiling during the work.

### 7.1 Butchering

Butchering a hog or steer is a settler job requiring the Cooking skill and an Animals skill helper (to handle the animal). After the first hard freeze:

1. The player designates animals for butchering in the animal management menu.
2. A settler drives the designated animal to the butchering site (a stockpile zone near the smokehouse or cellar).
3. The settler slaughters the animal, producing: fresh meat cuts, lard/fat, organ meat, and hide (*starting values*: a mature hog yields roughly 100–150 lb of meat and 20–30 lb of lard in game units following the scale rule).
4. Fresh meat must be processed within 2–3 game days in cold weather, or it begins to spoil.

### 7.2 Salting

Salt pork and salted beef are made by packing fresh meat cuts with salt in a barrel:

* **Input:** fresh meat, salt (from town stockpile), oak barrel (doc 08, section 3).
* **Process:** settler packs the barrel, alternating layers of meat and salt. Job time is proportional to the amount.
* **Output:** salt pork or salt beef barrel, stored in the root cellar or food stockpile. Keeps through winter and into next year if the cellar stays cold.
* **Salt cost:** approximately 1 lb of salt per 5 lb of meat (*starting value*). Running out of salt before finishing the packing means some meat spoils — a painful loss.

### 7.3 Smoking

The smokehouse is a small built structure (a room, doc 04, section 6) with a fire pit inside and gaps in the wall or roof for smoke venting. Hardwood fuel (hickory or oak) produces the best smoke.

* **Build requirement:** a room of at least $6 \\times 6\\text{ ft}$ ($12 \\times 12$ voxels in plan), with a stone or brick floor, a vented roof or chimney, and a designated smoking stockpile inside. The room does not need to be perfectly airtight.
* **Process:** fresh or salted meat is hung inside the smokehouse. A settler starts and tends the fire over several days, consuming firewood.
* **Duration:** 3–5 game days of slow, cool smoke (*starting value*). The fire must not be too hot (cooking rather than curing) or too cold (insufficient smoke).
* **Output:** smoked bacon, smoked ham, or smoked venison. Condition starts at 100 and decays very slowly — a smoked ham in a dry cellar can last 1–2 game years.

### 7.4 Churning

Churning turns fresh milk into butter. It is a light indoor chore suitable for children and elders.

* **Input:** a full milk pail (from milking, doc 14, section 7) placed in the churn.
* **Process:** a settler works the churn for about 30–60 game minutes per batch (*starting value*).
* **Output:** approximately 1 lb of butter per gallon of milk (*starting value*), plus buttermilk (a low-nutrition drink item that counts as a variety tag).
* **Butter storage:** a crock or wrapped block in the root cellar keeps for weeks. Packed into a salted crock it keeps for months.

### 7.5 Drying

Simple drying requires no special structure — just dry air and a rack (a few horizontal pine poles inside the cabin or under a roof):

* **Apple slices:** dried in autumn; keep all winter; a useful variety source.
* **String beans (leather britches):** threaded and hung; a preserved vegetable for winter soups.
* **Herbs:** dried in bunches hung from rafters; flavoring for winter cooking; spirits bonus.

Drying takes 3–7 game days depending on indoor humidity and airflow. Rain outside or high indoor moisture (from a leaky roof or flooded cellar) slows drying and risks mold.

---

## 8. Food Storage Summary

| Food | Storage | Lasts |
| :--- | :--- | :--- |
| Fresh meat | Cold outdoor air (below freezing) | 2–3 game days |
| Salt pork / salt beef | Barrel in root cellar | 1+ game years |
| Smoked bacon / ham | Dry cellar or hung in smokehouse | 1–2 game years |
| Cornmeal / flour | Dry grain bin, roofed | Several months; molds if wet |
| Potatoes / turnips | Root cellar, cool and dark | Through winter into spring |
| Butter | Crock in root cellar | Weeks to months |
| Dried fruit / beans | Dry pantry shelf | 1 game year |
| Milk | Any stockpile | 1–2 game days |
| Eggs | Cool pantry | 1–2 weeks |

Spoilage details (moisture thresholds, rot progression) follow doc 08, section 5.

---

## 9. What the Player Sees

* **Food stores panel:** a summary of how many days of food each type covers, with a total. The player can see at a glance if winter stores are sufficient.
* **Meal log:** recent meals and their variety score, in the settler needs panel (doc 07).
* **Preservation alerts:** "The pork barrel is not sealed — salt is running low" or "Smoked ham is curing in the smokehouse (3 days remaining)."
* **Variety warning:** "The family is tired of cornmeal. Try hunting or starting a kitchen garden."

---

## 10. Cost

* Meals are daily events processed on the daily tick (doc 06, section 2.2).
* Preservation jobs are settler jobs like any other — no background processing.
* No new voxel fields. Food is tracked through the `itemTable` and stockpile system (doc 08).
* The smokehouse is a room (doc 04), already handled by the room system.

---

## 11. Tests

* **Daily consumption:** run a family of 4 through 12 game days; verify total food consumed matches 4 × 12 × daily need.
* **Variety penalty:** feed a family only cornmeal for 5 days; verify spirits drop and the warning appears.
* **Variety bonus:** add butter and game meat to meals; verify spirits recover and the bonus is shown.
* **Salting:** butcher a hog, salt the meat with correct salt quantity; verify a full salt pork barrel appears in the cellar.
* **Salt shortage:** butcher a hog with insufficient salt; verify only part of the meat is preserved and the rest spoils.
* **Smokehouse:** hang fresh meat in a completed smokehouse and start the fire; verify smoked output appears after the correct number of days.
* **Churning:** fill the churn with milk; verify butter and buttermilk appear after the job completes.
* **Kitchen garden drought:** no rain for 3 days; verify garden crops wilt unless a settler waters them.

---

## 12. Open Questions

1. **Cheese:** milk curdled with rennet into cheese. Worth modeling? Cheese keeps much longer than milk and adds a valuable variety tag. Requires rennet (a town good).
2. **Coffee rationing:** should the player be able to set how liberally coffee is used (save it for hard days vs. every morning)? Or is it always consumed if available?
3. **Feasts and celebrations:** a harvest feast (special meal consuming more food than usual) for a large spirits bonus — worth implementing as a player-initiated event?
4. **Food quality grades:** should there be a quality difference between, say, fresh-ground cornmeal vs. town-milled flour beyond just variety, reflected in a nutrition modifier?
