================================================================================
LEVELDB PLUGIN - IMPLEMENTATION GUIDE
================================================================================
Last Updated: 2026-02-19
Plugin File: leveldb.xml
Plugin ID: b34c04e52c6c7bced4508230
Author: Rodarvus
Current Version: 2.0

================================================================================
OVERVIEW
================================================================================
LevelDB is a standalone MUSHclient plugin for Aardwolf MUD that records per-kill
combat data in a persistent SQLite database. It tracks mob kills, XP gains, gold,
damage dealt, rounds fought, and deaths across the character's
entire leveling journey.

Key Features:
- Per-kill records: mob name, zone, room, level, XP, gold, damage, rounds
- Per-death records: mob, zone, room, level
- Tier and remort tracked as columns in every record
- Single database file (leveldb.db) for all data
- Rich query commands: per-level, per-zone, per-mob breakdowns, top-N rankings

================================================================================
COMMANDS
================================================================================

BASIC:
  ldb                     - Show status (enabled, DB path, kill/death counts)
  ldb help                - Show all commands
  ldb on                  - Enable data collection
  ldb off                 - Disable data collection

QUERIES:
  ldb level [N]           - Kill breakdown for level N (default: current level)
                            Tabular per-kill listing with totals and averages
  ldb thislevel           - Shortcut for ldb level at current level
  ldb lastlevel           - Shortcut for ldb level at (current level - 1)
  ldb zone [name]         - Stats for a zone (default: current zone)
                            Substring match; shows: kills, XP, gold, avg XP/kill,
                            deaths, level range, top mobs
  ldb mob <name>          - Stats for a mob (substring match, case-insensitive)
                            Shows: kills, XP, gold, avg XP, avg rounds,
                            zones where killed
  ldb top mobs [N]        - Top N mobs by kill count (default 10)
  ldb top zones [N]       - Top N zones by total XP (default 10)
  ldb top xp [N]          - Top N mobs by avg XP per kill (default 10)
                            Requires 3+ kills to be ranked (HAVING cnt >= 3)
  ldb deaths [N]          - Last N deaths (default 10) with timestamp, level,
                            mob, zone

DATABASE:
  ldb db                  - Database file info: path, size, record counts

================================================================================
DATABASE DESIGN
================================================================================

File:
  leveldb.db
  Location: GetInfo(60) .. "state\leveldb\" (plugins state directory)

Schema:

  kills:
    id           INTEGER PRIMARY KEY AUTOINCREMENT
    timestamp    INTEGER NOT NULL      -- os.time() at combat end
    mob_name     TEXT NOT NULL          -- char.status.enemy display name
    zone         TEXT NOT NULL          -- room.info.zone
    room_num     INTEGER               -- room.info.num (nullable)
    room_name    TEXT                   -- room.info.name (nullable)
    level        INTEGER NOT NULL       -- player level at combat start
    xp_gained    INTEGER NOT NULL (0)   -- TNL delta; 0 for mobs that yield no XP
    gold_gained  INTEGER NOT NULL (0)   -- char.worth.gold delta
    damage_total INTEGER NOT NULL (0)   -- sum of [damage] from text lines
    rounds       INTEGER NOT NULL (0)   -- enemypct change count
    tier         INTEGER               -- char.base.tier (nullable)
    remort       INTEGER               -- char.base.remorts (nullable)

  deaths:
    id           INTEGER PRIMARY KEY AUTOINCREMENT
    timestamp    INTEGER NOT NULL      -- os.time() at death
    mob_name     TEXT                   -- enemy that killed player (nullable)
    zone         TEXT NOT NULL          -- zone where death occurred
    room_num     INTEGER               -- room number (nullable)
    room_name    TEXT                   -- room name (nullable)
    level        INTEGER NOT NULL       -- player level at death
    tier         INTEGER               -- char.base.tier (nullable)
    remort       INTEGER               -- char.base.remorts (nullable)

Indexes:
  idx_kills_level, idx_kills_zone, idx_kills_mob, idx_kills_timestamp
  idx_kills_tier, idx_kills_remort
  idx_deaths_level, idx_deaths_zone, idx_deaths_tier, idx_deaths_remort

Design Decisions:
- xp_gained = 0 is valid (mobs that yield no XP, fled/interrupted combats).
  All kills are included in queries regardless of xp_gained value.
- mob_name from char.status.enemy includes articles ("a field mouse"). This is
  acceptable because grouping by mob_name still works correctly.
- gold_gained may be slightly inaccurate on enemy-switch in multi-mob combat
  because char.worth arrives after char.status in GMCP sequence. Totals across
  all kills in a session remain correct.
- damage_total sums text trigger [damage] values. This is player damage only
  (mob-to-player damage uses possessive mob name, not "Your").
- room_num and room_name are nullable because GMCP room.info may not have
  arrived yet when combat starts (brief window on connect/reconnect).
- tier and remort are nullable because char.base may not have arrived yet.

================================================================================
COMBAT STATE MACHINE
================================================================================

States:
  IDLE    -- not in combat (in_combat = false)
  COMBAT  -- fighting a mob (in_combat = true)

Transitions:

  1. IDLE --> COMBAT:
     When: char.status.enemy becomes non-empty
     Action: Snapshot fight_start (tnl, gold, level, zone, room, enemy)
             Reset damage_total, round_count, last_enemypct, death_flag

  2. COMBAT --> COMBAT (round tick):
     When: char.status.enemypct changes value (same enemy, health decreasing)
     Action: Increment round_count
     Note: 2-4 identical char.status messages per round are deduplicated
           by comparing enemypct to last_enemypct

  2b. COMBAT --> COMBAT (same-name mob switch):
     When: char.status.enemy is the same name BUT enemypct increased
     Action: Record kill for previous mob, start_fight() for new mob
     Purpose: Consecutive kills of same-name mobs (e.g. "a spawnling" x5)
              where GMCP never sends enemy="" between kills

  3. COMBAT --> IDLE (kill):
     When: char.status.enemy becomes "" AND death_flag is false
     Action: Calculate XP/gold, INSERT into kills table, end_combat()

  4. COMBAT --> COMBAT (enemy switch):
     When: char.status.enemy changes to a DIFFERENT non-empty name
     Action: Record kill for PREVIOUS enemy (same as transition 3),
             then start_fight() for NEW enemy (same as transition 1)
     Purpose: Multi-mob combat where another mob auto-targets after first dies

  5. COMBAT --> IDLE (death):
     When: "You die." text trigger fires
     Action: Set death_flag = true, INSERT into deaths table
             Wait for GMCP enemy="" to clean up (death_flag prevents record_kill)

death_flag Lifecycle:
  - Set to true by ldb_on_player_death trigger
  - Prevents record_kill() from firing when subsequent GMCP enemy="" arrives
  - Cleared by start_fight() when next combat begins
  - Prevents double-recording (death + kill for same combat)

================================================================================
XP CALCULATION
================================================================================

Normal kill (same level):
  xp_gained = fight_start.tnl - cached_tnl

Level-up during combat:
  xp_gained = fight_start.tnl + (cached_perlevel - cached_tnl)

Where cached_perlevel = char.base.perlevel (constant for entire remort).
Default: 1000. Updated from GMCP char.base broadcasts.

Validated against stats_tracker.xml HandleStatus() implementation.

Multi-level-up in one kill is not precisely handled (same limitation as
stats_tracker). The formula accounts for one level-up only. This scenario
is vanishingly rare in normal gameplay.

Negative xp_gained is clamped to 0 (could happen if quest XP arrives between
fight_start snapshot and combat end).

================================================================================
DAMAGE LINE PARSING
================================================================================

Trigger regex: ^\*?(?:\[\d+\] )?Your .+[.!] \[(\d+)\]\*?$

Matches player damage lines only:
  Multi-hit:  [3] Your slash DECIMATES a pirate! [4626]
  Single-hit: Your wither devastates a pirate. [1542]
  Critical:   *Your crush does UNSPEAKABLE things to a pirate. [9999]*

Pattern breakdown:
  \*?              Optional leading * (critical hit marker)
  (?:\[\d+\] )?    Optional [N] hit count (multi-hit only, single-hit omits)
  Your             Must start with "Your" (excludes mob damage to player)
  .+               Spell description and verb
  [.!]             Sentence-ending punctuation
  \[(\d+)\]        Trailing [damage] value -- the only capture group
  \*?              Optional trailing * (critical hit marker)

Why "Your" excludes mob damage: Mob damage lines use the mob's possessive name
("A pirate's claw hits you. [234]"), not "Your". This is a reliable discriminator.

The trigger uses keep_evaluating="y" for compatibility with Aardwolf_Damage_Window.

================================================================================
GMCP BROADCASTS HANDLED
================================================================================

char.base:
  - Caches perlevel for XP level-up calculation
  - Caches tier and remort for INSERT columns

char.status:
  - Primary combat state driver (enemy, enemypct, level, tnl)
  - Caches level and tnl (skips sentinel value -1)
  - Runs state machine transitions only when enabled

char.worth:
  - Caches gold value for delta calculation on combat end
  - Note: Arrives AFTER char.status in broadcast sequence

room.info:
  - Caches zone, room_num, room_name for kill/death location

Broadcasts NOT listened to (by design):
  - char.vitals: HP/mana/moves not tracked by LevelDB
  - char.stats: Stats not relevant to kill tracking
  - comm.channel: LevelDB has no interaction detection
  - comm.quest: Quest XP not tracked in V2

================================================================================
PLUGIN LIFECYCLE
================================================================================

OnPluginInstall:
  1. Restore enabled state from saved variable
  2. Open database (leveldb.db)
  3. Display load message

OnPluginBroadcast (char.base):
  - Caches perlevel, tier, and remort values from GMCP

OnPluginSaveState:
  - Persists enabled flag (only persistent setting)

OnPluginClose:
  - Close database handle

OnPluginDisconnect:
  - End any active combat tracking (prevents stale state on reconnect)

State Persistence:
  save_state="y" with one variable: enabled (boolean)

================================================================================
DATABASE OPERATIONS
================================================================================

Following patterns from aard_GMCP_mapper.xml:

- sqlite3 is a pre-loaded global in MUSHclient (no require needed)
- Open: sqlite3.open(GetInfo(60) .. "state\\leveldb\\leveldb.db")
- Default journal mode (DELETE) -- no WAL sidecar files
- Write: db:exec() for INSERT statements
- Read: db:nrows() for SELECT queries
- Error checking: dbcheck() validates return codes against sqlite3.OK/ROW/DONE
- String escaping: fixsql() wraps strings in quotes, escapes embedded quotes,
  returns "NULL" for nil values (matching aard_GMCP_mapper.xml pattern)

Schema Initialization:
  init_schema() creates tables and indexes with IF NOT EXISTS on database open.

================================================================================
QUERY IMPLEMENTATION NOTES
================================================================================

All query commands:
- Check require_db() before executing (ensures DB is open)
- Include all kills regardless of xp_gained (0-XP kills are valid)
- Use format_number() for comma-separated large numbers

Substring matching (ldb zone, ldb mob):
- Uses SQL LIKE with % wildcards: WHERE zone LIKE '%search%'
- Case-insensitive by default in SQLite
- Aardwolf mob/zone names never contain SQL LIKE wildcards (% or _),
  so no escaping is needed

ldb top xp:
- Uses HAVING cnt >= 3 to exclude one-off kills that skew averages
- Ranks by avg XP per kill (CAST SUM AS REAL / COUNT)

================================================================================
PLUGIN COEXISTENCE
================================================================================

LevelDB is fully passive -- it only observes GMCP broadcasts and text output.
It never sends commands to the MUD. No conflict with:

- stats_tracker: Both track XP via TNL delta; independent databases
- Aardwolf_Damage_Window: Both match damage lines; keep_evaluating="y"
- aard_soundpack: Both match "You die."; keep_evaluating="y"
- NPC_Info, WingleGold_Spellup, or any other plugin

================================================================================
KNOWN LIMITATIONS
================================================================================

1. Gold attribution in multi-mob combat:
   char.worth arrives AFTER char.status. On enemy switch, gold from the just-killed
   mob may not yet be reflected. Gold gets attributed to the next mob instead.
   Totals across all kills remain correct. Accepted as approximate.

2. Multi-level-up in one kill:
   XP formula handles exactly one level-up. If a single kill grants enough XP for
   two level-ups (extraordinarily rare), the calculation will be short. Same
   limitation as stats_tracker.xml.

3. Zero-XP kills in kills table:
   Kills yielding 0 XP (low-level mobs, fled/interrupted combats) are recorded
   and included in all queries. This is by design -- LevelDB serves as a
   historical fight log, not just an XP tracker.

4. LIKE wildcard characters in search:
   User input to ldb zone/mob is used directly in SQL LIKE patterns without
   escaping % or _ characters. This is acceptable because Aardwolf mob names and
   zone names never contain these characters.

================================================================================
PLUGIN DEPENDENCIES
================================================================================

Required:
- gmcphelper (Lua module)
- sqlite3 (MUSHclient built-in global)
- aard_GMCP_handler plugin (ID: 3e7dedbe37e44942dd46d264)

================================================================================
FILE LOCATIONS
================================================================================

Plugin: leveldb/leveldb.xml
Database file: {plugins}/state/leveldb/leveldb.db
Documentation:
  - LEVELDB.md (this file) - implementation guide
  - README.md - user documentation

================================================================================
FUTURE CONSIDERATIONS (NOT IN V2)
================================================================================

- Session tracking / XP-per-hour calculation
- Miniwindow / real-time display
- Item loot tracking
- Quest/campaign/GQ tracking
- Spell-per-kill tracking
- Data export (CSV/JSON)
- Level time estimation
- ldb history command (XP/hour graph)
- ldb leveltime command (average time per level)

================================================================================
END OF IMPLEMENTATION GUIDE
================================================================================
