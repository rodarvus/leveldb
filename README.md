# LevelDB

A leveling database plugin for [Aardwolf MUD](http://www.aardwolf.com/), built for [MUSHclient](http://www.gammon.com.au/mushclient/).

LevelDB records combat encounters, quest completions, and campaign results into a persistent SQLite database. It tracks every kill, death, quest, and campaign across your character's entire leveling journey, with rich query commands to analyze your history.

## Features

**Combat tracking**
- Per-kill records: mob name, zone, room, player level, XP gained, damage dealt, rounds fought, estimated mob level (from sacrifice gold)
- Per-death records: mob, zone, room, level, tier, remort
- XP calculation with level-up and powerup detection
- Tier and remort stored with every record

**Quest tracking**
- Automatic tracking via GMCP comm.quest broadcasts
- Records target mob, area, room, timer, result (completed/failed/timeout)
- Captures rewards: QP, gold, TP, trains, pracs
- Reconnect-safe: restores active quest state on plugin reload

**Campaign tracking**
- Campaign lifecycle tracked via text triggers (start, complete, quit)
- Mob list captured from silent `cp info` parsing (avoids WinkleGold conflicts)
- Captures rewards: QP, gold, TP, trains, pracs
- Reconnect reconciliation: detects in-progress campaigns on plugin reload
- Detailed campaign view with full mob list

**Query and analysis**
- Per-level kill breakdowns with totals and averages
- Tier/remort filtering on all commands (default: current T+R; supports `all`, `T1 R5`, `T1`, `R4`)
- Remort summary: bracket breakdown (1-50, 51-100, 101-150, 151-200, Pups)
- Tier summary: compare remorts side-by-side within a tier
- Powerup support: at level 200+, queries auto-adapt to segment by powerup number
- Per-zone and per-mob stats with substring search
- Top-N rankings by kill count, total XP, or average XP
- Quest and campaign history tables with colored status, summaries, and averages

## Commands

| Command | Description |
|---------|-------------|
| `ldb` | Show status (enabled, DB path, record counts) |
| `ldb help` | List all commands |
| `ldb on` / `ldb off` | Enable/disable data collection |
| `ldb level [N] [filter]` | Per-kill table for a level (default: current). At 200+: N is a powerup number |
| `ldb this [filter]` | Per-kill table for current level/powerup |
| `ldb last [filter]` | Per-kill table for previous level/powerup |
| `ldb remort [R] [T<n>]` | Bracket summary for a remort (default: current T+R) |
| `ldb tier [T]` | Compare remorts within a tier (default: current) |
| `ldb zone [name]` | Stats for a zone (default: current; substring match) |
| `ldb mob <name>` | Stats for a mob (substring match) |
| `ldb top mobs [N]` | Top N mobs by kill count (default 10) |
| `ldb top zones [N]` | Top N zones by total XP (default 10) |
| `ldb top xp [N]` | Top N mobs by average XP per kill (default 10) |
| `ldb deaths [N]` | Last N deaths (default 10) |
| `ldb quest [filter]` | Quest history table (default: current T+R) |
| `ldb cp [filter]` | Campaign history table (default: current T+R) |
| `ldb cp show <id>` | Detailed view of a campaign by database ID |
| `ldb db` | Database file info (path, size, record counts) |

**Filter options** (for level/this/last/quest/cp): default = current tier and remort. `all` = all tiers and remorts. `T1 R5` = specific tier and remort. `T1` = all remorts within a tier. `R4` = specific remort, current tier.

See [USAGE.md](USAGE.md) for detailed descriptions and example output for every command.

## Installation

1. Copy `leveldb.xml` into your MUSHclient `worlds/plugins/` directory.
2. In MUSHclient, load the plugin via **File > Plugins > Add**.
3. Type `ldb on` to start collecting data.

Requires the **aard_GMCP_handler** plugin (included with the standard Aardwolf MUSHclient package). Data is stored in a single SQLite database (`leveldb.db`) in `worlds/plugins/state/leveldb/`.

See [LEVELDB.md](LEVELDB.md) for full implementation details.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
