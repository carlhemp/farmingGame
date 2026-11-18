# 11 - Weather & Climate

Weather drives almost everything else: water (doc 03), soil heat and frost (doc 02), crops and trees (doc 05), building snow loads (doc 04), and settler comfort (doc 01).

Values marked *starting value* are for tuning. Climate numbers are rough targets for 1850s eastern Kansas, Nebraska, and Iowa, to be checked against real records before tuning.

---

## 1. Scope

* **One weather state for the whole valley.** 500 acres is small enough that the same storm covers all of it.
* **Local differences come from the land, not the weather:** microclimate offsets from the coarse map (warm south slopes, frost pockets in low spots; doc 10), shade from `sunExposure`, and shelter from walls and trees.
* **Update rate:** every 600 ticks (10 game minutes, doc 06). Weather is one small global record, so it costs almost nothing.

---

## 2. Climate Targets

| Season | Day Highs | Night Lows | Rain / Snow | Typical Weather |
| :--- | :--- | :--- | :--- | :--- |
| **Spring** | $50\text{–}75^\circ\text{F}$, rising | $30\text{–}50^\circ\text{F}$ | Wettest season; soaking frontal rain, first thunderstorms | Mud, late frosts early in the season |
| **Summer** | $85\text{–}95^\circ\text{F}$, heat waves over $100^\circ\text{F}$ | $60\text{–}70^\circ\text{F}$ | Short, heavy thunderstorms | Hot, humid, dry spells |
| **Autumn** | $75^\circ\text{F}$ falling to $50^\circ\text{F}$ | $55^\circ\text{F}$ falling to $30^\circ\text{F}$ | Drier, mostly clear | First killing frost partway through |
| **Winter** | $25\text{–}40^\circ\text{F}$ | $5\text{–}20^\circ\text{F}$, cold snaps below $0^\circ\text{F}$ | Snow, sometimes blizzards | Frozen ground, strong winds |

### 2.1 The Short Calendar
Seasons are about 12 days (doc 06). The rule is:

* **Each day stays physically real.** A storm drops a realistic amount of rain at a realistic rate. Soil dries, freezes, and thaws at realistic speeds.
* **What is compressed is the pattern.** The season's temperature curve runs over 12 days instead of about 90. Dry spells, wet spells, and cold snaps last days instead of weeks. The share of rainy days per season stays realistic.

So a year has far less total rain than a real one, but each day's water balance (rain in, soaking, drying, crop use) feels right. A 4-day dry spell in a 12-day summer matters the way a 4-week one does in real life.

### 2.2 Frost Dates (*starting values*)
* **Last spring frost:** around day 3 to 5 of spring.
* **First autumn frost:** around day 6 to 9 of autumn.
* That leaves a frost-free growing season of about 25 to 30 days: late spring, summer, and early autumn.

---

## 3. The Weather State

```rust
pub struct Weather {
    pub air_temp: i16,       // tenths of a degree F
    pub humidity: u8,        // 0..100 %
    pub cloud_cover: u8,     // 0..100 %
    pub precip_rate: u16,    // 1/64" per hour
    pub precip_kind: PrecipKind, // None, Rain, Snow, Sleet, Hail
    pub wind_speed: u8,      // mph, steady
    pub wind_gust: u8,       // mph, gusts
    pub wind_dir: u8,        // 0..15, compass points
    pub regime: Regime,      // current weather pattern (section 4.2)
}
```

All values are integers, following the determinism rules in doc 06.

---

## 4. How Weather Is Made

### 4.1 Pre-Planned From the Seed
Each day's weather is a pure function of `(world seed, year, day)`, worked out at dawn. That means:
* Reloading a save never changes the weather. You cannot reroll a bad storm.
* Forecasts can look ahead at the real plan (section 6).
* Only the current day's plan needs saving, and even that can be recomputed.

### 4.2 Weather Regimes
Weather moves between a few **regimes**, each lasting 1 to 3 days (*starting value*). The chance of moving from one to another depends on the season.

| Regime | Seasons | What It Brings |
| :--- | :--- | :--- |
| **Fair** | All | Clear or light cloud, light wind, normal temperatures |
| **Unsettled** | Spring, Summer | Humid, cloudy, scattered showers |
| **Frontal Rain** | Spring, Autumn | Long, gentle, soaking rain; temperature drop after |
| **Thunderstorm** | Late Spring, Summer | Hot morning, short heavy afternoon storm, strong gusts, lightning, sometimes hail, rarely a tornado |
| **Heat Wave** | Summer | Very hot, dry, windy; crops and settlers suffer |
| **Dry Spell** | Summer, Autumn | Clear, no rain, high drying |
| **Cold Snap** | Autumn, Winter, early Spring | Sharp temperature drop; frost or hard freeze |
| **Snow** | Winter, edges of Spring and Autumn | Steady snowfall |
| **Blizzard** | Winter | Heavy snow, high wind, deadly cold for anyone caught outside |

### 4.3 Within a Day
* **Temperature:** follows a daily curve from the night low near dawn to the high in mid-afternoon. The curve and the season's curve come from lookup tables, not `sin` (doc 10, section 1).
* **Rain timing:** thunderstorms come in the afternoon or evening; frontal rain can last most of a day.
* **Rain rate matters as much as the amount** (doc 03, section 3.1):

  | Kind | Amount | Rate | Result |
  | :--- | :--- | :--- | :--- |
  | Light shower | $0.05\text{–}0.25\text{"}$ | Low | Soaks in almost everywhere |
  | Soaking rain | $0.25\text{–}1\text{"}$ | Low, over hours | Fills the soil; little runoff except on hardpan |
  | Thunderstorm | $0.5\text{–}3\text{"}$ | Up to $2\text{–}3\text{"}$ per hour in bursts | Runoff, erosion, flash rises in the creek |

### 4.4 Year Types
At the start of each year, the seed picks a **year type** that shifts the odds for the whole year (*starting values*):

| Year Type | Chance | Effect |
| :--- | :---: | :--- |
| Normal | 55% | Targets in section 2 |
| Wet | 15% | More rain days, more flooding, late planting |
| Dry | 15% | Fewer rain days, longer dry spells and heat waves |
| Hard Winter | 10% | Colder, more snow and blizzards, later spring |
| Early Frost | 5% | Autumn frost comes several days early |

Year types give the long game its risk: a run of good years tempts you to overextend, and a dry year or hard winter tests your stores.

---

## 5. Effects on the Simulation

### 5.1 Rain
Rain is added to the surface layer at the current rate (doc 03, section 3). Columns under a roof get none; it goes to the drip line.

### 5.2 Snow and Ice
* **Snow cover** is a depth per column in the surface layer, in fine units (doc 02, section 5). It is not made of $6\text{"}$ voxels, which would be far too coarse.
* Snow piles up during snowfall, settles over time, and melts into surface water when the air is above freezing. Rain on snow speeds up the melt.
* Snow **insulates** the soil, so frost goes less deep under good snow cover.
* **Pond ice** is a thickness per column in the surface layer, growing in cold weather on standing water and melting in spring. Thick ice can be walked on and cut.
* `matterState = 6` (Snow/Ice) is used only for cut ice blocks (for an ice house) and similar placed items.
* **Drifting** (wind piling snow against fences and walls) is a later addition.

### 5.3 Drying (Evaporation and Plant Use)
Each surface water step works out a **drying rate** from air temperature, sun (cloud cover and `sunExposure`), wind, and humidity:
* It removes water from puddles first, then from the top soil voxel's `poreWater`.
* Crops use the same rate on their daily tick to decide how much water they pull from their root voxels (doc 05).

### 5.4 Soil Heat and Frost
On the hourly step (doc 06), the top soil voxel moves toward the air temperature. How fast depends on:
* snow cover (slows it a lot), plant cover and mulch (slows it), wet soil (slows it), bare dry soil (fastest);
* the column's microclimate offset (doc 10).

Deeper voxels follow more slowly, so frost works its way down in winter and thaw works down in spring. A voxel at band 0 for long enough sets `isFrozen = 1`.

### 5.5 Frost Damage to Plants
* **Light frost** (air at or below $32^\circ\text{F}$): damages tender crops (corn, beans, squash) that are up.
* **Killing frost** (air at or below $28^\circ\text{F}$): kills tender crops outright and ends the growing season for them.
* Frost pockets in low spots get frost first, which teaches the player not to plant tender crops in the bottom of a hollow.

### 5.6 Wind
* Gusts drive tree wind damage (doc 05, section 3).
* Wind adds to drying (5.3) and to cold for settlers (5.7).
* Wind direction matters later for fire and snow drifting.

### 5.7 Settlers
* **Feels-like temperature** combines air temperature and wind. Shelter in a room (doc 04, section 6) blocks wind; a hearth adds heat.
* Cold below a limit drains health without shelter; a blizzard can kill a settler caught far from the house. Heat waves drain stamina faster.
* Rain and mud slow walking and hauling, and heavy rain stops some outdoor jobs.

### 5.8 Roofs
Snow depth on roof columns adds load. Heavy snow lowers support reach while it lasts (doc 04, section 5).

### 5.9 Lightning and Prairie Fire
* Thunderstorms bring lightning. A strike on dry prairie grass can start a fire. The chance is highest late in summer and in autumn, during dry spells, and in Dry years.
* Fire runs downwind through dry grass and can move faster than a person can walk. Rain puts it out.
* Plowed strips and burned strips (firebreaks) stop it. Settlers can set a back-fire to burn a break in front of an oncoming fire, as real settlers did.
* Burned prairie recovers next spring and is often greener for it.
* Fuel, spread rules, and damage belong to the fire doc (doc 09, docs still to write). This doc only decides when lightning strikes.

### 5.10 Hail
* About 10% of thunderstorms bring hail (*starting value*).
* Hail damages standing crops by growth stage: it shreds young leaves, and it can flatten ripe wheat and oats just before harvest, which hurts the most.
* Crops under cover and stored goods are safe.

### 5.11 Tornadoes
* How often they come is a difficulty setting (section 7). On the realistic setting they are very rare, about once in 250 years for a valley this size.
* A tornado follows a narrow path, $50\text{–}300\text{ ft}$ wide, across part of the valley.
* Along that path, wind force is high enough to knock down trees and built voxels whatever their support (doc 04), and to throw loose items.
* Settlers caught outside are in great danger. A dug cellar is the safe place, as with the storm cellars of the plains.
* The path is planned from the seed like the rest of the weather, so a reload does not move it.

---

## 6. Forecasts

There was no weather service on the frontier in the 1850s. Settlers read the sky, the wind, and animal behavior, and drew on long experience.

### 6.1 How Forecasts Work
* **Dawn assessment** (doc 01): each morning a settler gives the forecast in plain words, with the sign they read it from: *"Ring around the moon last night. Rain by tomorrow, I'd say."*
* Weather is pre-planned (section 4.1), so a forecast is the real plan with errors added. With the chance of a miss for that look-ahead, the forecast names a different regime that fits the season. Temperature gets a hashed error within the limit.
* Forecasts come from the seed too, so a reload gives the same forecast.

### 6.2 Forecast Accuracy (a difficulty setting)

| Setting | Rest of Today | Tomorrow | 2 Days Ahead | 3+ Days |
| :--- | :--- | :--- | :--- | :--- |
| **Frontier (default, realistic)** | rain call right ~90%, temp $\pm 5^\circ\text{F}$ | ~70%, $\pm 8^\circ\text{F}$ | ~55%, $\pm 12^\circ\text{F}$ | No forecast |
| **Seasoned** | ~95%, $\pm 3^\circ\text{F}$ | ~85%, $\pm 5^\circ\text{F}$ | ~75%, $\pm 8^\circ\text{F}$ | ~60% for day 3 |
| **Clear Skies** | Always right | Always right | Always right | Always right through day 3 |

* **Basis for the realistic numbers:** there are no records of how good folk forecasting was. These targets assume a careful watcher can see the rest of today coming, has real but limited skill for tomorrow, and is barely better than guessing from the season two days out. For comparison, modern forecasts are right about 90% of the time at 5 days and about 50% at 10 days (NOAA).
* This is one of the weather difficulty settings in section 7.

### 6.3 Better Forecasts Over Time (Frontier and Seasoned)
* **Telegraph:** the U.S. Army Signal Service began telegraph weather forecasts around 1870. If the town gets a telegraph office with the railroad (doc 01), a posted next-day forecast in town is right about 80% of the time (*starting value*). Someone has to go to town to read it.
* **Almanac:** sold in town. Like real almanacs, its long-range weather calls are no better than guessing. It is still useful for moon phases, sunrise times, and planting customs.

---

## 7. Weather Difficulty Settings

Each weather danger has its own setting. **Realistic** is the default and is based on real records where they exist. Easier and harder choices sit on either side of it. A preset sets all of them at once, and each can then be changed on its own at any time.

| Setting | Easier | **Realistic (default)** | Harder |
| :--- | :--- | :--- | :--- |
| **Forecast accuracy** (section 6.2) | Seasoned, Clear Skies | Frontier | none |
| **Tornadoes** (section 5.11) | None | About once in 250 years | Stormy: about once in 25 years. Tornado Alley: about once in 5 to 10 years |
| **Year types** (section 4.4) | Gentle: more Normal years, fewer Dry and Hard Winter years | The table in section 4.4 | Harsh: Dry and Hard Winter years twice as likely |
| **Frost dates** (section 2.2) | Gentle: frost-free season about 3 days longer, no surprise frosts | Section 2.2 dates, with a small chance of a surprise frost outside them | Harsh: season about 3 days shorter, surprise frosts more likely |
| **Blizzard danger** (section 5.7) | Gentle: blizzards come with a day's warning, cold hurts slower | Blizzards build over a few hours, with a short warning | Sudden: some blizzards arrive with almost no warning on a mild day, as in the 1888 "Children's Blizzard" |

**Presets:** *Gentle* (easier column), *Frontier* (realistic column), *Harsh* (harder column). The presets also set predator danger and grasshopper plague (doc 14, section 8).

### 7.1 Where the Realistic Numbers Come From
* **Tornadoes:** Kansas averages roughly 4 tornadoes per $10,000\text{ km}^2$ per year. A path a few miles long and a few hundred feet wide has to cross a valley about $1.4\text{ km}$ on a side, which gives roughly one tornado in 250 years. This is a rough estimate from state averages, not a record for any one valley. Most players on this setting will never see one.
* **Year types and frost dates:** the section 4.4 odds and section 2.2 dates are starting estimates. They should be checked against 1850s to 1890s records for Kansas, Nebraska, and Iowa, which had several severe drought years (such as 1860) in that span.
* **Blizzard warning:** most plains blizzards gave some warning, but the January 1888 blizzard came suddenly after a mild morning. The Harder setting makes that kind of storm more common.

---

## 8. Cost & Storage

* **Cost:** one small record updated every 10 game minutes, plus per-column work already counted in the surface water step (doc 06, section 2.3).
* **Storage:** nothing beyond the seed and the current day. Snow and ice depths are saved with the surface layer.

---

## 9. Tests

* **Climate statistics:** over 100 simulated years, season temperatures, rainy-day shares, storm sizes, and frost dates match the targets in section 2.
* **Same seed, same weather:** the same seed gives the same weather every time, on every machine.
* **Snow balance:** snow that falls is either still there, melted into surface water, or evaporated. Nothing is lost.
* **Year types:** each type shows its effect on average across many seeded years.

---

## 10. Open Questions

Decided: lightning-started prairie fires, hail, and rare tornadoes are in (sections 5.9 to 5.11). Forecast accuracy, tornado rate, year-type odds, frost dates, and blizzard danger are difficulty settings with a realistic default (section 7).

1. **Settler skill:** Should an older or more experienced settler forecast better than a young one?
