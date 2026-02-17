================================================================================
LEVELDB PLUGIN - IMPLEMENTATION GUIDE
================================================================================
Last Updated: 2026-02-17
Plugin File: leveldb.xml
Plugin ID: b34c04e52c6c7bced4508230
Author: Rodarvus
Current Version: 1.0

================================================================================
OVERVIEW
================================================================================
LevelDB is a standalone MUSHclient plugin for Aardwolf MUD that records per-kill
combat data in a persistent SQLite database. It tracks mob kills, XP gains, gold,
combat duration, damage dealt, rounds fought, and deaths across the character's
entire leveling journey.

Key Features:
- Per-kill records: mob name, zone, room, level, XP, gold, damage, rounds, duration
- Per-death records: mob, zone, room, level
- Automatic database file switching on remort/tier changes
- Rich query commands: per-level, per-zone, per-mob breakdowns, top-N rankings
- SQLite persistent storage (one database per remort)
- Schema versioning for future migrations

================================================================================
COMMANDS
================================================================================

BASIC:
  ldb                     - Show status (enabled, DB file, kill/death counts)
  ldb help                - Show all commands
  ldb on                  - Enable data collection
  ldb off                 - Disable data collection

QUERIES:
  ldb level [N]           - Kill breakdown for level N (default: current level)
                            Shows: kills, XP, gold, damage, rounds, time, deaths,
                            top mobs, zones
  ldb zone [name]         - Stats for a zone (default: current zone)
                            Substring match; shows: kills, XP, gold, avg XP/kill,
                            avg duration, deaths, level range, top mobs
  ldb mob <name>          - Stats for a mob (substring match, case-insensitive)
                            Shows: kills, XP, gold, avg XP, avg duration, avg
                            rounds, zones where killed
  ldb top mobs [N]        - Top N mobs by kill count (default 10)
  ldb top zones [N]       - Top N zones by total XP (default 10)
  ldb top xp [N]          - Top N mobs by avg XP per kill (default 10)
                            Requires 3+ kills to be ranked (HAVING cnt >= 3)
  ldb deaths [N]          - Last N deaths (default 10) with timestamp, level,
                            mob, zone

DATABASE:
  ldb db                  - Database file info: path, size, record counts,
                            schema version
  ldb db list             - List all database files across all tier/remort
                            combinations (scans T0-T9, R0-R6)
  ldb db switch           - Force re-detect tier/remort from GMCP and switch DB

================================================================================
DATABASE DESIGN
================================================================================

File Naming:
  ldb_T{tier}R{remort}.db
  Examples: ldb_T0R1.db (first character), ldb_T3R7.db (tier 3, 7th class)
  Location: GetInfo(66) (MUSHclient root directory)

Schema (V1):

  schema_version:
    version INTEGER NOT NULL

  kills:
    id           INTEGER PRIMARY KEY AUTOINCREMENT
    timestamp    INTEGER NOT NULL      -- os.time() at combat end
    mob_name     TEXT NOT NULL          -- char.status.enemy display name
    zone         TEXT NOT NULL          -- room.info.zone
    room_num     INTEGER               -- room.info.num (nullable)
    room_name    TEXT                   -- room.info.name (nullable)
    level        INTEGER NOT NULL       -- player level at combat start
    xp_gained    INTEGER NOT NULL (0)   -- TNL delta; 0 = fled/interrupted
    gold_gained  INTEGER NOT NULL (0)   -- char.worth.gold delta
    damage_total INTEGER NOT NULL (0)   -- sum of [damage] from text lines
    rounds       INTEGER NOT NULL (0)   -- enemypct change count
    duration     INTEGER NOT NULL (0)   -- seconds (os.time() granularity)

  deaths:
    id           INTEGER PRIMARY KEY AUTOINCREMENT
    timestamp    INTEGER NOT NULL      -- os.time() at death
    mob_name     TEXT                   -- enemy that killed player (nullable)
    zone         TEXT NOT NULL          -- zone where death occurred
    room_num     INTEGER               -- room number (nullable)
    room_name    TEXT                   -- room name (nullable)
    level        INTEGER NOT NULL       -- player level at death

Indexes:
  idx_kills_level, idx_kills_zone, idx_kills_mob, idx_kills_timestamp
  idx_deaths_level, idx_deaths_zone

Design Decisions:
- xp_gained = 0 means fled/interrupted (not a real kill). All queries use
  WHERE xp_gained > 0 when counting "real kills".
- mob_name from char.status.enemy includes articles ("a field mouse"). This is
  acceptable because grouping by mob_name still works correctly.
- gold_gained may be slightly inaccurate on enemy-switch in multi-mob combat
  because char.worth arrives after char.status in GMCP sequence. Totals across
  all kills in a session remain correct.
- damage_total sums text trigger [damage] values. This is player damage only
  (mob-to-player damage uses possessive mob name, not "Your").
- duration has 1-second granularity (os.time()). Acceptable for statistics.
- room_num and room_name are nullable because GMCP room.info may not have
  arrived yet when combat starts (brief window on connect/reconnect).

================================================================================
COMBAT STATE MACHINE
================================================================================

States:
  IDLE    -- not in combat (in_combat = false)
  COMBAT  -- fighting a mob (in_combat = true)

Transitions:

  1. IDLE --> COMBAT:
     When: char.status.enemy becomes non-empty
     Action: Snapshot fight_start (time, tnl, gold, level, zone, room, enemy)
             Reset damage_total, round_count, last_enemypct, death_flag

  2. COMBAT --> COMBAT (round tick):
     When: char.status.enemypct changes value (same enemy)
     Action: Increment round_count
     Note: 2-4 identical char.status messages per round are deduplicated
           by comparing enemypct to last_enemypct

  3. COMBAT --> IDLE (kill):
     When: char.status.enemy becomes "" AND death_flag is false
     Action: Calculate XP/gold/duration, INSERT into kills table, end_combat()

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
  - Detects tier/remort changes to switch database files
  - Caches perlevel for XP level-up calculation
  - Only triggers DB switch when tier or remort actually changes

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
  - comm.quest: Quest XP not tracked in V1

================================================================================
PLUGIN LIFECYCLE
================================================================================

OnPluginInstall:
  1. Restore enabled state from saved variable
  2. Try to detect tier/remort and open database (may fail if GMCP not ready)
  3. Display load message

OnPluginBroadcast (char.base):
  - When tier or remort changes, automatically close current DB and open new one
  - Also fires on initial GMCP data arrival after connect

OnPluginSaveState:
  - Persists enabled flag (only persistent setting)

OnPluginClose:
  - Close database handle

OnPluginDisconnect:
  - End any active combat tracking (prevents stale state on reconnect)

State Persistence:
  save_state="y" with one variable: enabled (boolean)
  Database file is re-detected from GMCP char.base on every load/connect,
  so it does not need to be persisted.

================================================================================
DATABASE OPERATIONS
================================================================================

Following patterns from aard_GMCP_mapper.xml:

- sqlite3 is a pre-loaded global in MUSHclient (no require needed)
- Open: sqlite3.open(GetInfo(66) .. filename)
- WAL mode for concurrent read/write: PRAGMA journal_mode=WAL
- Write: db:exec() for INSERT statements
- Read: db:nrows() for SELECT queries
- Error checking: dbcheck() validates return codes against sqlite3.OK/ROW/DONE
- String escaping: fixsql() wraps strings in quotes, escapes embedded quotes,
  returns "NULL" for nil values (matching aard_GMCP_mapper.xml pattern)

Schema Migration:
  schema_version table tracks current version. On DB open:
  1. Check if schema_version table exists (version = 0 if not)
  2. Apply migrations sequentially: version < 1 -> migrate_to_v1()
  3. Future migrations: version < 2 -> migrate_to_v2(), etc.
  Each migration creates tables/indexes and updates the version number.

================================================================================
QUERY IMPLEMENTATION NOTES
================================================================================

All query commands:
- Check require_db() before executing (ensures DB is open)
- Use xp_gained > 0 filter for "real kills" (excludes fled/interrupted)
- Use format_number() for comma-separated large numbers
- Use format_duration() for human-readable time (12s, 3m 45s, 1h 2m 30s)

Substring matching (ldb zone, ldb mob):
- Uses SQL LIKE with % wildcards: WHERE zone LIKE '%search%'
- Case-insensitive by default in SQLite
- Aardwolf mob/zone names never contain SQL LIKE wildcards (% or _),
  so no escaping is needed

ldb top xp:
- Uses HAVING cnt >= 3 to exclude one-off kills that skew averages
- Ranks by avg XP per kill (CAST SUM AS REAL / COUNT)

ldb db list:
- Scans all possible tier/remort combinations (T0-T9, R0-R6)
- Uses io.open() to check file existence and size
- Marks active database with "<-- active"

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

3. Brief window on connect/reconnect:
   GMCP data may not be available when plugin loads. Database opens when first
   char.base broadcast arrives. Kills during this window (unlikely) are silently
   dropped. No crashes -- all recording functions check for nil db.

4. Non-kill combat ends in kills table:
   Fled/interrupted combats are recorded with xp_gained=0. This is by design --
   all queries filter with WHERE xp_gained > 0 for "real kills". The raw data
   is preserved for potential future analysis.

5. os.time() granularity:
   Duration tracking has 1-second resolution. Sub-second precision is not available
   in standard Lua. Acceptable for aggregate statistics.

6. LIKE wildcard characters in search:
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
Database files: {MUSHclient root}/ldb_T{tier}R{remort}.db
Documentation:
  - LEVELDB.md (this file) - implementation guide
  - LEVELDB_DESIGN.md - original design document with full rationale

================================================================================
FUTURE CONSIDERATIONS (NOT IN V1)
================================================================================

- Session tracking / XP-per-hour calculation
- Miniwindow / real-time display
- Item loot tracking
- Quest/campaign/GQ tracking
- Spell-per-kill tracking
- Cross-database queries (compare remorts)
- Data export (CSV/JSON)
- Level time estimation
- ldb compare command (compare stats between remorts)
- ldb history command (XP/hour graph)
- ldb leveltime command (average time per level)

================================================================================
END OF IMPLEMENTATION GUIDE
================================================================================
