# 16 — Town & Economy

The town is not simulated as a living voxel settlement. The CPU budget goes into the farm, not into a second colony. The town functions as an **off-map trade depot, labor market, and land office** — a destination with real costs and real constraints, not a menu.

Values marked *starting value* are for tuning in milestone M10+.

---

## 1. The Town as a Place

The town sits at the edge of the valley, reachable by road. It is shown on the Valley Overview zoom (doc 07, section 3) as a named cluster of icons, not as a voxel settlement. The player never walks through it; they send a settler and get a report when the settler returns.

```
Valley Overview (1 cell = 32 ft)

  [ Your Claim ]──road──[ Neighbor Claim ]──road──[ TOWN ]
                                                      │
                                              General Store
                                              Livery Stable
                                              Land Office
                                              (later: Mill, Blacksmith,
                                               Railroad Depot, Telegraph)
```

---

## 2. Travel

### 2.1 The Trip

A settler going to town takes a wagon (for selling or buying bulk goods) or goes on foot or horseback (for quick errands or when the wagon is in use). The trip has two legs: out and back.

| Mode | Speed | One Way | Round Trip |
| :--- | :---: | :---: | :---: |
| On foot | ~3 mph | ~1 day | ~2 days |
| Ox wagon (empty) | ~2 mph | ~1.5 days | ~3 days |
| Ox wagon (loaded) | ~1.5 mph | ~2 days | ~4 days |
| Horseback | ~5 mph | ~half a day | ~1 day |

Distances and speeds are *starting values*, tuned so that a round trip by ox wagon takes 3 to 4 days. Road condition matters: a muddy spring road after heavy rain slows the wagon further and risks getting bogged down.

### 2.2 The Cost of Going

Sending a settler to town is a real trade-off:

* The settler is away from the farm for 2 to 4 full working days.
* The wagon is away too. If it is the only wagon, hauling stops on the farm.
* In a 12-day season, losing 3 days is a quarter of the season's work.

The player must judge whether the errand is worth the absence. Going to town during planting or harvest is expensive. Going in winter, when field work is impossible anyway, is nearly free.

### 2.3 The Trip as a Job

A town trip is a settler job like any other (doc 13, section 4.2):

```
Trip to Town:
  1. Load wagon from stockpile (goods to sell)
  2. Walk / drive to town
  3. Sell goods, buy goods, run errands
  4. Load wagon with purchases
  5. Walk / drive home
  6. Unload wagon into stockpile
```

The player designates the trip from a menu: what to sell, what to buy, and any special errands (hire a hand, visit the land office, check prices). The settler carries out the plan. If a good is sold out when they arrive, they come home with the news and the original goods.

---

## 3. The Arrival Calendar

The town general store and livery stable receive seasonal shipments by riverboat or freight wagon. Stock arrives at predictable times each season, but supply is limited.

### 3.1 Seasonal Stock

| Season | What Arrives | Goes Fast |
| :--- | :--- | :--- |
| **Early Spring** | Seed corn, wheat seed, garden seed, plowshares, nails, fabric | Seed — plant late and you miss the season |
| **Late Spring** | Draft animals (oxen, horses), farm tools, wagons | Horses and good oxen pairs |
| **Summer** | Hardware, iron goods, salt, sugar, coffee, medicine | Salt (needed for preserving the autumn butcher) |
| **Autumn** | Winter clothing, traps, ammunition, almanacs | Winter goods before first snow |
| **Winter** | Very little. Some hardware and dry goods from freight sleds | — |

If the player arrives after the seasonal shipment has sold out, that good is gone until next season's boat. This is the main reason to plan trips and go early.

### 3.2 Out of Stock

When a good is sold out, the settler returns with the news. The player can wait for next season, look for an alternative (trade with a neighbor, make it on the farm), or accept the loss. Some goods — breaking-plow blades, iron axle hardware, the right seed variety — cannot be improvised.

---

## 4. Prices

### 4.1 Buying

Town prices for goods the farm needs to buy are set at the start of the year from the seed (doc 11, section 4.1 pattern: planned from `(world seed, year, item)`). They vary modestly year to year based on the regional year type (doc 11, section 4.4): a Wet year raises grain prices (less regional supply), a Dry year raises them more. A Hard Winter year raises livestock prices in spring.

| Good | Typical Price Range | Notes |
| :--- | :--- | :--- |
| Seed corn (bushel) | \$1.00–\$1.50 | *starting value* |
| Wheat seed (bushel) | \$0.75–\$1.25 | |
| Plowshare (iron) | \$2.00–\$4.00 | Wears out; needs replacing |
| Axe | \$1.00–\$2.00 | |
| Salt (100 lb) | \$1.50–\$3.00 | Needed for preserving pork |
| Ox pair (trained) | \$50–\$80 | |
| Horse | \$60–\$120 | |
| Hired hand (per day) | \$0.50–\$1.00 + board | |

Prices are shown to the player in the town menu before they commit to a purchase.

### 4.2 Selling

The farm earns cash by selling surplus goods in town. Prices the farm receives are lower than buying prices (the merchant's margin) and fall when regional supply is high.

| Good | Typical Selling Price | What Drives It |
| :--- | :--- | :--- |
| Wheat (bushel) | \$0.40–\$0.80 | Regional harvest; floods or drought raise it |
| Corn (bushel) | \$0.25–\$0.50 | Very common; price drops if everyone grows it |
| Salt pork (barrel) | \$6–\$10 | Steady; towns always need preserved meat |
| Butter (pound) | \$0.12–\$0.20 | Reliable small cash; not affected by big harvests |
| Eggs (dozen) | \$0.08–\$0.15 | Seasonal; high in spring, low in autumn |
| Wool (pound) | \$0.20–\$0.35 | Steady; not weather-dependent |
| Timber (100 bd ft) | \$0.75–\$1.50 | High if local mill is not yet built |

### 4.3 The Bumper Crop Problem

If the player harvests a large surplus of wheat and sells it all at once, the local price drops. This reflects regional supply: when everyone's harvest is good, grain is cheap. Diversifying into butter, eggs, smoked pork, and wool smooths income. This is by design — the town economy gently teaches the same crop-rotation lesson the soil chemistry teaches, from the market side.

### 4.4 Cash

Cash is tracked as a simple integer (cents). There are no gold coins to store physically; cash is an abstraction carried by the settler on a trip. The player can see how much cash the homestead has at any time.

---

## 5. The Land Office

New land is bought at the land office in town. The valley is divided into **40-acre parcels** on the government survey grid, each with a preview of terrain, soil, timber, and water (doc 10, section 5).

### 5.1 Buying a Parcel

* The player reviews available parcels in the land office menu, shown on the Valley Overview with their characteristics.
* Price per parcel is set at the start of the game from the seed. Rich bottomland costs more; poor ridge land costs less (*starting values*: \$50–\$200 per 40-acre parcel).
* Buying requires the settler to travel to town and back. The parcel activates when the settler returns: its chunks wake from frozen terrain with baseline soil moisture from the world seed, adjusted for the current season (doc 10, section 4.3).
* Adjacent parcels unlock their shared borders, connecting streams, ditches, and roads.

### 5.2 Starting Claim

The player starts with **3 parcels** (120 acres) chosen at world creation (doc 10, section 5). The land office is used for all purchases after that.

---

## 6. Hiring

### 6.1 Hired Hands

The player can hire extra labor in town (doc 13, section 11). A hired hand is a temporary settler:

* **Day rate:** pay agreed in advance. Hands expect cash, plus room and board on the farm. A hand who is not paid, fed, or housed leaves and spreads the word; future hiring is harder and costs more.
* **Season hire:** cheaper per day than day rate. A hand hired for the summer is committed through harvest.
* **Year hire:** the cheapest rate; the hand stays through winter. Useful for a family that is short-handed after a death or illness.
* Hired hands use the same settler system (doc 13): they have skills, needs, and work priorities. Their skill level varies and is shown in the hiring menu.

### 6.2 Breaking Crews

Breaking the first prairie sod is brutally hard for a single family team (doc 14, section 3.3). Breaking contractor crews exist from Day 1:
* **Local County Crew:** If an outfit is operating locally in the valley or county in spring, they can be hired directly at the land office or livery for a standard rate (\$2–\$4 per acre, *starting value*).
* **Sending for an Outside Crew:** If local contractors are fully booked or none have established nearby yet, the player can **send for an outfit from an established river town or older settlement**. This costs more (a long-distance travel & dispatch fee, e.g. +\$1–\$2/acre or a flat travel surcharge) and takes several days for them to arrive by road, but guarantees access to heavy breaking teams if the settler has the cash.
* **The Operation:** The crew arrives with heavy breaking plows and multi-yoke oxen teams, turns the designated prairie acreage over several days, and departs. They are independent contractors, not homestead settlers.
* **Cash Requirement:** Contract crews demand prompt payment in cash. A farm without sufficient currency cannot retain them.

---

## 7. Organic Regional Growth vs. Artificial Unlocks

There are **no artificial tech trees, game levels, or arbitrary milestone locks** (e.g. no *"Sell 500 bushels to unlock the Blacksmith"*). The town is an autonomous regional hub serving dozens of square miles, not a reward track for the player.

### 7.1 Day-One Market Availability
Everything that existed on the 1850s American frontier is physically available from **Day 1**, governed solely by economics and logistics:
* **Tools, Hardware & Special Orders:** Common items are in stock at the general store. If an uncommon tool, specialized casting, or specific iron share is not on the shelf, the merchant can **special-order it from a larger river port or city back east** for a freight surcharge, arriving with the next scheduled freight wagon.
* **Horses & Draft Stock:** Available at the livery stable immediately if you have the capital; specialized breeds or heavy draft teams can be ordered via livestock drovers.
* **Contractor Breaking Crews:** Available locally if one is working in the county, or can be dispatched from a neighboring county/town for an extra travel fee.
* **Building Types & Crafting:** If you have the physical timber, stone, and stamina, you can build a two-story hewn-oak barn or dig an 8-foot cellar on your first day. Nothing is "locked behind progression."

### 7.2 Emergent Town Infrastructure
As regional population and commerce expand across the county (governed deterministically by regional history in the master world seed, influenced by broad valley settlement):
* Private tradesmen and entrepreneurs move into the town on their own schedules.
* **The Grist Mill:** When an independent miller establishes a water mill on the regional river, the economic effect is simple: custom milling becomes cheaper and immediate, eliminating the need to hand-grind or ship grain to a distant port.
* **The Local Blacksmith:** When a smith sets up a local forge, iron tools no longer carry heavy long-distance freight markups, and custom tool repairs become available locally.
* **The Railroad & Telegraph:** Regional infrastructure advances based on historical timelines, expanding market access and telegraph weather dispatches.

The player's success may bring prosperity to the valley, but the world never halts its simulation to demand an arbitrary chore before unlocking a realistic tool.

### 7.2 The Grist Mill in Detail

Before the mill, the settler process is: harvest → thresh → hand-grind (slow, indoor winter work) → cornmeal or flour. The mill replaces the hand-grinding step:

* The player loads sacked grain onto the wagon and sends a settler to the mill.
* The mill returns a fraction as flour or meal (milling yield *starting value*: 70% by weight for wheat flour, 80% for cornmeal; the rest is bran and loss).
* The settler brings the flour sacks home. No new stockpile type is needed; flour is an item in the existing grain stockpile.
* Flour earns more per bushel than raw grain when sold.

---

## 8. Neighbors

Other homesteads in the valley are off-map, like the town. They settle as the years pass on a schedule planned from the seed (doc 16 open question 1). The player sees them arrive on the Valley Overview.

Neighbors:
* **Trade:** a neighbor might have a bull for breeding, or extra seed, or timber to sell. Trade is at agreed prices, cheaper than the town.
* **Help at harvest:** a neighbor with capacity may come at harvest for a share of the crop or future labor returned.
* **Family:** a grown child who settles next door becomes a neighbor (doc 13, section 11). They start small and grow their own claim.

Neighbor simulation is minimal: they are not fully simulated farms. They contribute to the regional grain supply (affecting prices), provide or request occasional trade, and appear as flags on the Valley Overview.

---

## 9. What the Player Sees

* **Town menu** (`t` from the Valley Overview): shows current stock, prices, land office parcels, hired hands available, and breaking crew availability. Can be opened at any time, but buying requires sending a settler.
* **Trip report:** when a settler returns from town, a summary appears: what was sold (and for how much), what was bought (and what was out of stock), and any news (new facility, neighbor arrived, outbreak warning).
* **Cash display:** always visible in the homestead status panel (doc 07).
* **Town growth alert:** when a new facility opens, an alert appears.

---

## 10. Cost & Storage

* **Town state** is a small record in `state.bin` (doc 12, section 4.2): current stock levels, prices for the current year, cash on hand, town growth progress, and available hired hands.
* **No simulation:** the town runs no ticks. Stock is set at the seasonal arrival, decremented by player purchases, and partially restocked at the next arrival.
* **Neighbor state** is a small list of flags in `state.bin`: which parcels are settled, which neighbors exist, and pending trade offers.

---

## 11. Tests

* **Arrival calendar:** verify that seasonal stock arrives at the right season and is not available before it.
* **Sold-out:** buy all of a good; verify the next settler trip finds it out of stock and returns with the news.
* **Bumper crop:** sell a large harvest; verify the selling price drops below the starting range.
* **Breaking crew:** contract a crew; verify the designated field transitions to broken ground on arrival day.
* **Town growth:** simulate enough settled parcels; verify each facility unlocks at the right threshold.
* **Same seed, same prices:** verify prices for a given year are identical across reloads and machines.
* **Trip duration:** verify a round trip by ox wagon takes the expected number of game days based on road distance and load.

---

## 12. Open Questions

1. **Neighbor schedule:** when and how fast do other homesteads settle? Should the player have any influence (selling supplies to a new family)?
2. **Debt:** should the town general store offer credit (buying now, paying at harvest)? This was common on the frontier and adds risk.
3. **Almanac:** sold in town; useful for moon phases and planting customs, but long-range weather calls are no better than guessing (doc 11, section 6.3). Should it have a mechanical effect or be flavor only?
4. **Town raid and unrest:** frontier towns occasionally had violence, disease outbreaks (cholera), or supply disruptions from flooded rivers. Should these appear as rare seeded events?
5. **Barter:** should the player be able to barter goods directly (seed for labor, timber for iron) rather than always going through cash?
