# LevelDB Usage Guide

LevelDB silently records every kill, death, quest, and campaign as you play. All data is stored in a local SQLite database and can be queried at any time. Data collection is tier- and remort-aware, so your stats are segmented across your character's lifecycle.

Type `ldb on` to start collecting. Everything else is automatic.

---

## Quick Reference

| Command | What it does |
|---------|-------------|
| `ldb` | Status overview |
| `ldb on` / `ldb off` | Toggle data collection |
| `ldb this` | Kills for current level or powerup |
| `ldb last` | Kills for previous level or powerup |
| `ldb level 42` | Kills for a specific level |
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
| `ldb deaths` | Recent deaths |
| `ldb db` | Database file info |
| `ldb help` | List all commands in-game |

---

## Filtering

Most commands default to your **current tier and remort**. You can override this:

| Filter | Meaning |
|--------|---------|
| *(none)* | Current tier + remort |
| `all` | All tiers and remorts (shown as separate sections) |
| `T1 R5` | Specific tier and remort |
| `T1` | All remorts within tier 1 |
| `R4` | Remort 4, current tier |

Examples: `ldb this all`, `ldb level 50 T1 R5`, `ldb quest T2`, `ldb cp all`

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
  Current level: 69
  Current zone: knossos
```

### `ldb this` / `ldb last` / `ldb level [N]` — Kill Breakdown

Per-kill table for a level. Shows mob name, zone, XP gained, total damage dealt, combat rounds, and estimated mob level. Totals and averages at the bottom.

At level 200+, these commands work with **powerup numbers** instead of levels (e.g., `ldb level 3` = powerup 3).

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

Breaks a remort into level brackets (1-50, 51-100, 101-150, 151-200, Pups) with aggregate stats. Defaults to current tier + remort.

- `ldb remort` — current remort
- `ldb remort 4` — remort 4, current tier
- `ldb remort T1 R4` — tier 1, remort 4

Column guide: **AvgGap** = average (mob level - player level), **Pups** = powerup count in that bracket.

```
> ldb remort

[LevelDB] Remort summary - T2 R5 (966 kills):
  Bracket  Kills  AvgXP  AvgDmg  AvgGap  AvgRnd  Zones  Deaths  Pups
  -------  -----  -----  ------  ------  ------  -----  ------  ----
  1-50       608    162   1,220    +4.6     1.8     75       -
  51-100     358    140   2,870    +4.4     3.0     41       -
  101-150      -      -       -       -       -      -       -
  151-200      -      -       -       -       -      -       -
  Pups         -      -       -       -       -      -       -
  -------  -----  -----  ------  ------  ------  -----  ------  ----
  Total      966    154   1,831    +4.5     2.2              -
```

### `ldb tier [T]` — Tier Comparison

Compares all remorts within a tier, one row per remort for each bracket. Useful for spotting trends across remorts.

```
> ldb tier 2

[LevelDB] Tier summary - T2 (5 remorts, 10,934 kills):

  -- 1-50 --
  Remort   Kills  AvgXP  AvgDmg  AvgGap  AvgRnd  Zones  Deaths  Pups
  -------  -----  -----  ------  ------  ------  -----  ------  ----
  R1         181     84     820    +3.8     1.4     22       -
  R2         385    104     980    +4.1     1.5     38       -
  R3         611    120   1,050    +4.3     1.6     52       -
  R4         626    150   1,180    +4.5     1.7     61       -
  R5         608    162   1,220    +4.6     1.8     75       -

  -- 51-100 --
  Remort   Kills  AvgXP  AvgDmg  AvgGap  AvgRnd  Zones  Deaths  Pups
  -------  -----  -----  ------  ------  ------  -----  ------  ----
  R1           -      -       -       -       -      -       -
  R2         290     98   1,520    +3.9     2.4     28       -
  ...
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
     3. a fox pup (Plains of Nulan'Boar)
     4. a Templar Patrol (Rosewood Castle)
     5. a red ball (Rosewood Castle)
     6. a drow emissary (The Goblin Fortress)
     7. a grizzled goblin dressed in skins (The Goblin Fortress)
     8. Fiznatch, a ratling pirate (The Grand City of Aylor)
     9. a sinister vandal (The Three Pillars of Diatz)
    10. Dumazzi (The Three Pillars of Diatz)
    11. an agathinon aasimon (The Upper Planes)
    12. Robirc the head surgeon (Tilule Rehabilitation Clinic)
    13. a bugbear bandit (Tilule Rehabilitation Clinic)
    14. a Chaprenulan lab assistant (Tilule Rehabilitation Clinic)
    15. Earshda the receptionist (Tilule Rehabilitation Clinic)
```

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

- **Kills**: Tracked via GMCP enemy state changes. XP is calculated from TNL difference before and after each kill. Mob level is estimated from sacrifice gold (gold x 2). Damage and rounds are counted from combat text.
- **Deaths**: Captured from the "You have died" trigger.
- **Quests**: Tracked automatically via GMCP `comm.quest` messages (start, complete, fail, timeout).
- **Campaigns**: Tracked via text triggers for campaign start/complete/quit. Mob lists are captured by silently running `cp info`.

All records include your current level, tier, and remort at the time of the event.

---

## Tips

- Use `ldb this` during a grind session to see XP progress at your current level.
- Use `ldb top xp 20` to find which mobs give the best XP on average.
- Use `ldb remort` at the end of a remort to review your bracket performance.
- Use `ldb tier` to compare how your current remort stacks up against previous ones.
- Use `ldb cp show` after a campaign to review the mob list and rewards.
- The `zone` and `mob` commands use **substring matching** — `ldb zone fen` matches "fens", `ldb mob troll` matches "a troll warrior".
