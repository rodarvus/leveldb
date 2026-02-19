# LevelDB

A leveling database plugin for [Aardwolf MUD](http://www.aardwolf.com/), built for [MUSHclient](http://www.gammon.com.au/mushclient/).

## What it does

LevelDB silently records every combat encounter into a persistent SQLite database. Each kill captures the mob name, zone, room, player level, XP gained, gold earned, total damage dealt, and rounds fought. Deaths are tracked separately with the killing mob, zone, and level. Tier and remort are stored with every record.

The plugin is fully passive — it observes GMCP broadcasts and combat text without ever sending commands to the MUD. It works alongside manual play, automation plugins like S&D and SpellUp, or any other setup.

## Features

- **Per-kill tracking** — mob name, zone, room, level, XP, gold, damage, rounds
- **Death tracking** — mob, zone, room, level
- **Tier/remort columns** — every record includes the current tier and remort
- **Query commands** — per-level breakdowns, per-zone stats, per-mob stats, top-N rankings, death history

## Commands

| Command | Description |
|---------|-------------|
| `ldb` | Show status |
| `ldb help` | List all commands |
| `ldb on` / `ldb off` | Enable/disable data collection |
| `ldb level [N]` | Per-kill table for a level (default: current) |
| `ldb zone [name]` | Stats for a zone (default: current; substring match) |
| `ldb mob <name>` | Stats for a mob (substring match) |
| `ldb top mobs [N]` | Top N mobs by kill count |
| `ldb top zones [N]` | Top N zones by total XP |
| `ldb top xp [N]` | Top N mobs by average XP per kill |
| `ldb deaths [N]` | Last N deaths |
| `ldb db` | Database file info |

## Installation

1. Copy `leveldb.xml` into your MUSHclient `worlds/plugins/` directory.
2. In MUSHclient, load the plugin via **File > Plugins > Add**.
3. Type `ldb on` to start collecting data.

Requires the **aard_GMCP_handler** plugin (included with the standard Aardwolf MUSHclient package).

## How it works

LevelDB listens to GMCP broadcasts (`char.status`, `char.base`, `char.worth`, `room.info`) to detect combat start/end, track XP and gold deltas, and identify zones and rooms. A text trigger captures damage values from combat output lines. Another trigger detects player death.

XP per kill is calculated from the TNL (to-next-level) delta between combat start and end, with level-up detection. Gold is calculated from `char.worth.gold` delta. Rounds are counted by tracking `enemypct` changes.

Data is stored in a single SQLite database (`leveldb.db`) in the MUSHclient root directory.

See [LEVELDB.md](LEVELDB.md) for full implementation details.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
