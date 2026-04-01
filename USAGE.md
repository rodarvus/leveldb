# LevelDB Usage Guide

LevelDB silently records every kill, death, quest, and campaign as you play. All data is stored in a local SQLite database and can be queried at any time. Data collection is tier- and remort-aware, so your stats are segmented across your character's lifecycle.

Type `ldb on` to start collecting. Everything else is automatic.

---

## Quick Reference

| Command | What it does |
|---------|-------------|
| `ldb` | Status overview |
| `ldb on` / `ldb off` | Toggle data collection |
| `ldb this` | Kills for current level |
| `ldb last` | Kills for previous level |
| `ldb level 42` | Kills for a specific level |
| `ldb pup` | Powerup productivity stats |
| `ldb pup 42` | Per-kill detail for powerup #42 |
| `ldb pup list` | Recent powerup events |
| `ldb pup icefall` | Per-mob breakdown for a zone |
| `ldb zone` | Stats for current zone |
| `ldb mob troll` | Search kills by mob name |
| `ldb top mobs` | Leaderboard by kill count |
| `ldb top zones` | Leaderboard by total XP |
| `ldb top xp` | Leaderboard by avg XP per kill |
| `ldb remort` | Bracket breakdown for current remort |
| `ldb tier` | Compare remorts within current tier |
| `ldb quest` | Quest history |
| `ldb cp` | Campaign history |
| `ldb cp show 17` | Full detail for a campaign |
| `ldb gq` | Global quest history |
| `ldb gq show 5` | Full detail for a global quest |
| `ldb deaths` | Recent deaths |
| `ldb db` | Database file info |
| `ldb help` | List all commands in-game |

---

## Filtering

Most commands default to your **current tier and remort**. You can override this:

| Filter | Meaning |
|--------|---------|
| *(none)* | Current tier+redo + remort |
| `all` | All tiers and remorts (shown as separate sections) |
| `T1 R5` | Specific tier and remort |
| `T9+3 R5` | Tier 9, redo 3, remort 5 |
| `T9+3` | Tier 9, redo 3, all remorts |
| `T1` | All remorts within tier 1 (redo 0) |
| `R4` | Remort 4, current tier+redo |

Examples: `ldb this all`, `ldb level 50 T1 R5`, `ldb quest T2`, `ldb pup T9+3`, `ldb cp all`

Redo is hidden in display when 0 (e.g., `T2 R5` not `T2+0 R5`).

---

## Commands in Detail

### `ldb` — Status

Shows whether collection is on, database path, record counts, and current combat state.

```
[LevelDB] Status
  Enabled: ON
  Database: C:\...\state\leveldb\leveldb.db
  Kills recorded: 12,787
  Deaths recorded: 1
  Quests recorded: 17
  Campaigns recorded: 21
  Pup events recorded: 42
  Current level: 69
  Current zone: knossos
```

### `ldb this` / `ldb last` / `ldb level [N]` — Kill Breakdown

Per-kill table for a level. Shows mob name, zone, XP gained, total damage dealt, combat rounds, and estimated mob level. Totals and averages at the bottom.

At level 200+, these commands redirect to `ldb pup` (use `ldb pup` for powerup stats).

```
> ldb level 68

[LevelDB] Level 68 kills (T2 R5):
  #   Mob                   Zone       XP     Dmg  Rnd  MLvl
  --  --------------------  ---------  -----  ----  ---  ----
  1   Yulom                 diatz         54  3,511   2    70
  2   a gossiping widow     diatz        108  3,740   3    72
  3   Navin                 dsr          273  2,688   5    70
  4   the Pale Rider        fields       389  3,256   6    74
  5   a wicked elf          fields       314  3,457   3    72
  6   the bounty hunter     goldrush     246  3,738   2    70
  ...
  --  --------------------  ---------  -----  ----  ---  ----
  Tot 24 kills                         2,764 78,430  82
  Avg                                    115  3,268  3.4
```

### `ldb pup [filter|zone]` — Powerup Productivity Stats

Shows powerup summary and per-area productivity table for your current tier and remort (or filtered). This is the primary command for analyzing pupping efficiency.

- `ldb pup` — current T+R overview
- `ldb pup all` — all tiers and remorts
- `ldb pup T1 R5` — specific tier and remort
- `ldb pup list` — last 10 powerup events (shows IDs for drill-down)
- `ldb pup list 20` — last 20 powerup events
- `ldb pup icefall` — per-mob breakdown for a zone (substring match)
- `ldb pup 42` — per-kill detail for powerup event #42

**Overview output:**

```
> ldb pup

[LevelDB] Powerup Summary (T2 R5):
  Powerups: 8     Trains: 42 (5.3/pup)
  Combat time: 1m 12s (9.0s/pup)

  Area              Kills  AvgXP  AvgTime  Est/Pup  Trn/Hr
  ----------------  -----  -----  -------  -------  ------
  icefall              35    180    2.1s      11.7s    1631
  fens                  5    199    3.0s      15.1s    1264
  fortune               5    106    1.8s      17.0s    1124

  Recent powerups:
     ID  When              T/R      Pup#  Zone              Trains
  -----  ----------------  -------  -----  ----------------  ------
     38  2026-03-28 22:26  T2R5         4  icefall                5
     39  2026-03-28 22:26  T2R5         5  fens                   6
     40  2026-03-28 22:27  T2R5         6  icefall                5
     41  2026-03-28 22:27  T2R5         7  icefall                5
     42  2026-03-28 22:28  T2R5         8  icefall                7
```

Use `ldb pup 42` to see per-kill detail for a specific powerup, or `ldb pup list` to see more.

Column guide:
- **AvgXP**: Average XP earned per kill in this area
- **AvgTime**: Average active combat time per kill
- **Est/Pup**: Estimated active combat time to earn one powerup if grinding only in this area (`xp_per_level / AvgXP * AvgTime`)
- **Trn/Hr**: Estimated trains per hour of active combat in this area

**Per-kill detail (by pup event ID):**

```
> ldb pup 42

[LevelDB] Powerup #42 (T2 R5, pup 8) - icefall:
  #  Mob                       Zone      XP      Dmg  Rnd  MLvl  Time
  -  ------------------------  --------  ------  ----  ---  ----  -----
  1  a relaxed minotaur guard  icefall      118  3022    2   232  1.8s
  2  a mountain troll          icefall      145  2871    2   228  1.5s
  3  a frost giant             icefall      132  3544    3   235  2.3s
  4  a mountain troll          icefall      148  2910    2   228  1.6s
  5  a relaxed minotaur guard  icefall      122  3105    2   232  1.9s
  -  ------------------------  --------  ------  ----  ---  ----  -----
  Tot 5 kills                             665  15452   11         9.1s
  Avg                                     133   3090  2.2        1.8s
  Trains earned: 7
  XP needed: 1,000
```

**Per-mob zone detail:**

```
> ldb pup icefall

[LevelDB] Pup zone detail: icefall (T2 R5):
  Mob                       Kills  AvgXP  AvgTime  AvgRd
  ------------------------  -----  -----  -------  -----
  a relaxed minotaur guard     14    120    1.9s    2.0
  a mountain troll             12    146    1.6s    2.1
  a frost giant                 9    135    2.4s    3.0
```

**Pup event list:**

```
> ldb pup list

[LevelDB] Last 10 powerups (T2 R5):
     ID  When              T/R      Pup#  Zone              Trains
  -----  ----------------  -------  -----  ----------------  ------
     35  2026-03-28 22:24  T2R5         1  icefall                5
     36  2026-03-28 22:25  T2R5         2  fortune                6
     37  2026-03-28 22:25  T2R5         3  icefall                5
     38  2026-03-28 22:26  T2R5         4  icefall                5
     39  2026-03-28 22:26  T2R5         5  fens                   6
     40  2026-03-28 22:27  T2R5         6  icefall                5
     41  2026-03-28 22:27  T2R5         7  icefall                5
     42  2026-03-28 22:28  T2R5         8  icefall                7
```

### `ldb zone [name]` — Zone Stats

Stats for a zone. Defaults to current zone. Supports substring search.

```
> ldb zone tol

[LevelDB] Zone: tol
  Kills: 659 | XP: 229,424
  Avg XP/kill: 348
  Level range: 161 - 180
  Top mobs:
    1. a hawk (192 kills, 69,215 XP)
    2. a finch (175 kills, 62,572 XP)
    3. a parakeet (158 kills, 46,037 XP)
    4. a wasp (49 kills, 19,967 XP)
    5. a fly (44 kills, 14,404 XP)
```

### `ldb mob <name>` — Mob Stats

Search for a mob by name (substring match, case-insensitive). Shows kill count, XP, average rounds, and zones where killed. Up to 10 matching mobs shown.

```
> ldb mob bulette

[LevelDB] Mob search: bulette
  a bulette: 491 kills, 114,672 XP
    Avg XP: 233 | Avg rounds: 8.6
    Zones: adaldar
```

### `ldb top mobs [N]` — Top Mobs by Kill Count

Ranked list of your most-killed mobs across all tiers/remorts. Default N = 10.

```
> ldb top mobs 5

[LevelDB] Top 5 mobs by kill count:
  1. some reeds (576 kills, 120,007 XP)
  2. a bulette (491 kills, 114,672 XP)
  3. a troll warrior (396 kills, 114,148 XP)
  4. a filthy harpy (242 kills, 77,030 XP)
  5. a Stone giant gladiator (239 kills, 71,273 XP)
```

### `ldb top zones [N]` — Top Zones by Total XP

```
> ldb top zones 5

[LevelDB] Top 5 zones by total XP:
  1. desolation (1,006 kills, 271,838 XP)
  2. tol (659 kills, 229,424 XP)
  3. fens (859 kills, 191,380 XP)
  4. adaldar (568 kills, 164,683 XP)
  5. duskvalley (469 kills, 118,511 XP)
```

### `ldb top xp [N]` — Top Mobs by Average XP

Requires 3+ kills to rank (avoids one-off outliers).

```
> ldb top xp 5

[LevelDB] Top 5 mobs by avg XP per kill:
  1. a crow (avg 691 XP, 12 kills)
  2. a Stone giant berserker (avg 566 XP, 3 kills)
  3. a Quickling Thief (avg 453 XP, 4 kills)
  4. a shark (avg 440 XP, 14 kills)
  5. a fish (avg 436 XP, 13 kills)
```

### `ldb remort [R] [T<n>]` — Remort Bracket Summary

Breaks a remort into level brackets (1-50, 51-100, 101-150, 151-200) with aggregate stats. If powerup data exists, a separate pup section is shown below with productivity stats and per-area breakdown. Defaults to current tier + remort.

- `ldb remort` — current remort
- `ldb remort 4` — remort 4, current tier
- `ldb remort T1 R4` — tier 1, remort 4

Column guide: **AvgGap** = average (mob level - player level).

```
> ldb remort

[LevelDB] Remort summary - T2 R5 (966 kills):
  Bracket  Kills  AvgXP  AvgDmg  AvgGap  AvgRnd  Zones  Deaths
  -------  -----  -----  ------  ------  ------  -----  ------
  1-50       608    162   1,220    +4.6     1.8     75       -
  51-100     358    140   2,870    +4.4     3.0     41       -
  101-150      -      -       -       -       -      -       -
  151-200      -      -       -       -       -      -       -
  -------  -----  -----  ------  ------  ------  -----  ------
  Total      966    154   1,831    +4.5     2.2              -

  Powerups: 8     Trains: 42 (5.3/pup)
  Combat time: 1m 12s (9.0s/pup)

  Area              Kills  AvgXP  AvgTime  Est/Pup  Trn/Hr
  ----------------  -----  -----  -------  -------  ------
  icefall              35    180    2.1s      11.7s    1631
  fens                  5    199    3.0s      15.1s    1264
```

### `ldb tier [T]` — Tier Comparison

Compares all remorts within a tier, one row per remort for each bracket. A Powerups comparison section at the end shows pupping efficiency across remorts.

```
> ldb tier 2

[LevelDB] Tier summary - T2 (5 remorts, 10,934 kills):

  -- 1-50 --
  Remort   Kills  AvgXP  AvgDmg  AvgGap  AvgRnd  Zones  Deaths
  -------  -----  -----  ------  ------  ------  -----  ------
  R1         181     84     820    +3.8     1.4     22       -
  R2         385    104     980    +4.1     1.5     38       -
  R3         611    120   1,050    +4.3     1.6     52       -
  R4         626    150   1,180    +4.5     1.7     61       -
  R5         608    162   1,220    +4.6     1.8     75       -

  -- 51-100 --
  Remort   Kills  AvgXP  AvgDmg  AvgGap  AvgRnd  Zones  Deaths
  -------  -----  -----  ------  ------  ------  -----  ------
  R1           -      -       -       -       -      -       -
  R2         290     98   1,520    +3.9     2.4     28       -
  ...

  -- Powerups --
  Remort  Pups  Trains  Trn/Pup  AvgTime  Trn/Hr
  ------  ----  ------  -------  -------  ------
  R2         3      15      5.0    12.3s    1463
  R4         8      42      5.3     9.0s    2120
  R5        22     121      5.5     8.1s    2444
```

### `ldb quest [filter]` — Quest History

Tabular quest log with QP, gold, TP, time, target mob and area. Failed/timed-out quests are flagged. Active quest shown in green. Supports tier/remort filtering.

```
> ldb quest

[LevelDB] Quests - T2 R5:
  #   Lvl   QP   Gold  TP   Time  Mob (Area)
  --  ---  ----  -----  ---  -----  --------------------------------
  1    48    18  2,825    0    25s  a dealer (Onyx Bazaar)
  2    50    46  3,214    0    32s  records worker (Descent to Hell)
  3    50    32  2,932    0    22s  the soul of Scricha (The Silver+
  4    51    22  2,878    0   2m   an orchid (Raganatittu)
  5    52    24  3,166    0    23s  a huge crocodile (The Yurgach Do+
  6    54    26  3,070    0    23s  a posse member (Gold Rush)
  --  ---  ----  -----  ---  -----  --------------------------------
  6 completed (of 6 total)
  Avg QP: 28 | Avg time: 35s | Total QP: 168 | Total gold: 18,085
```

### `ldb cp [filter]` — Campaign History

Same structure as quests but for campaigns. Quit campaigns are flagged.

```
> ldb cp

[LevelDB] Campaigns - T2 R5:
  #   Lvl  Mobs   QP   Gold   TP   Time
  --  ---  ----  ----  ------  ---  -----
  14   60    15    28  38,210    -    12m
  15   61    13    25  41,500    -    18m
  16   63    11    31  52,334    -    22m
  17   64    15    33  42,121    -    10m
  18   65    11    29  64,828    -    26m
  19   66    13    30  49,914    -    20m
  20   67    15    26  42,557    -     6m
  21   68    15     -       -    -      -  (active)
  --  ---  ----  ----  ------  ---  -----
  7 completed (of 8 total)
  Avg QP: 28 | Avg time: 16m | Total QP: 202 | Total gold: 331,464
```

### `ldb cp show <id>` — Campaign Detail

Full detail for a single campaign by its database ID (the `#` column from `ldb cp`). Shows timestamp, level, result, rewards, and the complete mob list with areas.

```
> ldb cp show 17

[LevelDB] Campaign #17:
  Started:  2025-07-24 23:49
  Level:    64 (T2 R5)
  Result:   completed (10m)
  Mobs:     15
  Rewards:  33 QP, 42,121 gold

  Mob list (15):
     1. Marie Antoinette (Diamond Soul Revelation)
     2. a Nulan'Boar tribesman (Plains of Nulan'Boar)
     ...
```

### `ldb gq [filter]` — Global Quest History

Tabular GQ log with QP, gold, TP, time, and result status. Won GQs are highlighted in cyan, expired/cancelled/quit in red. Supports tier/remort filtering.

- `ldb gq` — current T+R
- `ldb gq all` — all tiers and remorts
- `ldb gq T1 R5` — specific tier and remort

```
> ldb gq

[LevelDB] Global Quests - T1 R1:
  #   Lvl  Mobs  QP    Gold   TP   Time
  --  ---  ----  ----  -----  ---  -----
  1    12     8    40  1,000    -    12m  (won)
  2    15     6     9      -    -     8m  (expired)
  3    18    10    55  1,500    -    15m  (won)
  --  ---  ----  ----  -----  ---  -----
  2 won, 1 expired (of 3 total)
  Avg QP: 47 | Avg time: 13m | Total QP: 104 | Total gold: 2,500
```

Per-mob QP (from "N quest points awarded" lines) is accumulated into the QP total even for GQs that weren't won or completed.

### `ldb gq show <id>` — Global Quest Detail

Full detail for a single global quest by its database ID. Shows GQ number, timestamp, level, result, rewards, and the complete mob list with kill counts.

```
> ldb gq show 1

[LevelDB] Global Quest #1 (GQ 7286):
  Started:  2026-04-01 16:05
  Level:    12 (T1 R1)
  Result:   won (12m)
  Mobs:     8
  Rewards:  40 QP, 1,000 gold

  Mob list (8):
    1. Jille (beer)
    2. a Morukian officer (callhero)
    3. a fighting troll (fantasy)
    4. 2 * billows of smoke (fireswamp)
    5. 2 * a suffocating witch (gallows)
    6. the watch ghost (gauntlet)
    7. a half-griffon judge (sirens)
    8. a stool (wizards)
```

For multi-target mobs (e.g., "2 * billows of smoke"), the kill count is shown.

### `ldb deaths [N]` — Death Log

Last N deaths (default 10). Shows timestamp, level, mob, zone, and tier/remort.

```
> ldb deaths

[LevelDB] Last 10 deaths:
  2026-02-23 08:34 - Lvl 11  - unknown                   (sanctity) [T1R7]
```

### `ldb db` — Database Info

Shows the database file path, size, and record counts.

```
> ldb db

[LevelDB] Database info:
  Path: C:\...\state\leveldb\leveldb.db
  Size: 2.3 MB
  Kills: 12,787
  Deaths: 1
  Quests: 17
  Campaigns: 21
```

---

## How Data Is Collected

- **Kills**: Tracked via GMCP enemy state changes. XP is calculated from TNL difference before and after each kill. Mob level is estimated from sacrifice gold (gold x 2). Damage and rounds are counted from combat text. Active combat time is measured per kill using `utils.timer()`.
- **Deaths**: Captured from the "You have died" trigger.
- **Powerups**: Detected via the "Congratulations" text trigger on each powerup. Trains earned are captured from subsequent "You gain N trains" text lines.
- **Quests**: Tracked automatically via GMCP `comm.quest` messages (start, complete, fail, timeout).
- **Campaigns**: Tracked via text triggers for campaign start/complete/quit. Mob lists are captured by silently running `cp info`.
- **Global Quests**: Tracked via text triggers for join/won/completed/expired/cancelled/quit. Mob lists are captured by silently running `gq check`, and expected rewards from `gq info`. Per-mob QP is accumulated from "N quest points awarded" lines after each mob kill.

All records include your current level, tier, and remort at the time of the event.

---

## Tips

- Use `ldb pup` to compare pupping efficiency across different areas.
- Use `ldb pup <zone>` to see which mobs in an area give the best XP.
- Use `ldb this` during a grind session to see XP progress at your current level.
- Use `ldb top xp 20` to find which mobs give the best XP on average.
- Use `ldb remort` at the end of a remort to review your bracket performance and pupping stats.
- Use `ldb tier` to compare how your current remort stacks up against previous ones.
- Use `ldb cp show` after a campaign to review the mob list and rewards.
- The `zone` and `mob` commands use **substring matching** — `ldb zone fen` matches "fens", `ldb mob troll` matches "a troll warrior".
