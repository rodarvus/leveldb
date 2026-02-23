================================================================================
LEVELDB PLUGIN - IMPLEMENTATION GUIDE
================================================================================
Last Updated: 2026-02-23
Plugin File: leveldb.xml
Plugin ID: b34c04e52c6c7bced4508230
Author: Rodarvus
Current Version: 4.0

================================================================================
OVERVIEW
================================================================================
LevelDB is a standalone MUSHclient plugin for Aardwolf MUD that records per-kill
combat data in a persistent SQLite database. It tracks mob kills, XP gains,
mob level estimates, damage dealt, rounds fought, and deaths across the character's
entire leveling journey.

Key Features:
- Per-kill records: mob name, zone, room, level, XP, mob level, damage, rounds
- Per-death records: mob, zone, room, level
- Tier and remort tracked as columns in every record
- Powerup support: at level 200+ (Hero/Superhero), queries auto-adapt to segment
  by powerup number instead of level
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
                            At level 200+: N is a powerup number (default: current pup)
  ldb this                - Shortcut for current level (or current powerup at 200+)
  ldb last                - Shortcut for previous level (or previous powerup at 200+)
  ldb zone [name]         - Stats for a zone (default: current zone)
                            Substring match; shows: kills, XP, avg XP/kill,
                            deaths, level range, top mobs
  ldb mob <name>          - Stats for a mob (substring match, case-insensitive)
                            Shows: kills, XP, avg XP, avg rounds,
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
    xp_gained    INTEGER NOT NULL DEFAULT 0  -- TNL delta; 0 for mobs that yield no XP
    damage_total INTEGER NOT NULL DEFAULT 0  -- sum of [damage] from text lines
    rounds       INTEGER NOT NULL DEFAULT 0  -- enemypct change count
    tier         INTEGER               -- char.base.tier (nullable)
    remort       INTEGER               -- char.base.remorts (nullable)
    pup          INTEGER               -- char.base.pups (nullable; set at 200+)
    mob_level    INTEGER               -- estimated from sacrifice gold * 2 (nullable)

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
  idx_kills_tier, idx_kills_remort, idx_kills_pup, idx_kills_mob_level
  idx_deaths_level, idx_deaths_zone, idx_deaths_tier, idx_deaths_remort

Design Decisions:
- xp_gained = 0 is valid (mobs that yield no XP, fled/interrupted combats).
  All kills are included in queries regardless of xp_gained value.
- mob_name from char.status.enemy includes articles ("a field mouse"). This is
  acceptable because grouping by mob_name still works correctly.
- mob_level is estimated from sacrifice gold (gold * 2). Nullable: NULL if no
  sacrifice message was seen (e.g., autosac off, vampire autoconsume). The formula
  is approximate. Old records from v3.0 and earlier have mob_level=NULL.
- damage_total sums text trigger [damage] values. This is player damage only
  (mob-to-player damage uses possessive mob name, not "Your").
- room_num and room_name are nullable because GMCP room.info may not have
  arrived yet when combat starts (brief window on connect/reconnect).
- tier and remort are nullable because char.base may not have arrived yet.
- pup is nullable: NULL for levels 1-199, populated from char.base.pups at 200+.
  Old records from v2.0 databases have pup=NULL (added via ALTER TABLE migration).

================================================================================
COMBAT STATE MACHINE
================================================================================

States:
  IDLE    -- not in combat (in_combat = false)
  COMBAT  -- fighting a mob (in_combat = true)

Transitions:

  1. IDLE --> COMBAT:
     When: char.status.enemy becomes non-empty
     Action: Snapshot fight_start (tnl, level, zone, room, enemy, pup)
             Reset damage_total, round_count, last_enemypct, death_flag,
             sacrifice_mob_level.

  2. COMBAT --> COMBAT (round tick):
     When: char.status.enemypct changes value (same enemy, health decreasing)
     Action: Increment round_count
     Note: 2-4 identical char.status messages per round are deduplicated
           by comparing enemypct to last_enemypct

  2b. COMBAT --> COMBAT (same-name mob switch):
     When: char.status.enemy is the same name BUT enemypct increased
     Action: record_kill() for previous mob, start_fight() for new mob
     Purpose: Consecutive kills of same-name mobs (e.g. "a spawnling" x5)
              where GMCP never sends enemy="" between kills

  3. COMBAT --> IDLE (kill):
     When: char.status.enemy becomes "" AND death_flag is false
     Action: record_kill(), end_combat()

  4. COMBAT --> COMBAT (enemy switch):
     When: char.status.enemy changes to a DIFFERENT non-empty name
     Action: record_kill() for PREVIOUS enemy, then start_fight() for NEW enemy
     Purpose: Multi-mob combat where another mob auto-targets after first dies

  5. COMBAT --> IDLE (death):
     When: "You die." text trigger fires
     Action: Set death_flag = true, INSERT into deaths table
             Wait for GMCP enemy="" to clean up (death_flag prevents kill recording)

death_flag Lifecycle:
  - Set to true by ldb_on_player_death trigger (guarded against double-fire)
  - Prevents record_kill() path when subsequent GMCP enemy="" arrives
  - Cleared by start_fight() when next combat begins
  - Prevents double-recording (death + kill for same combat)

Kill Recording:
  record_kill() is called synchronously BEFORE end_combat() or start_fight().
  It reads fight_start, combat accumulators (damage_total, round_count), and
  sacrifice_mob_level directly from module-level state, computes XP gained,
  and INSERTs the kill record into the database.

  The sacrifice text trigger fires on the sacrifice line, which arrives from the
  server before the GMCP char.status broadcast. So sacrifice_mob_level is already
  set by the time record_kill() runs. No timer delay is needed.

  After record_kill() returns, end_combat() or start_fight() clears the state.

================================================================================
XP CALCULATION
================================================================================

Normal kill (same level):
  xp_gained = fight_start.tnl - cached_tnl

Level-up during combat (1-199):
  xp_gained = fight_start.tnl + (cached_perlevel - cached_tnl)

Powerup during combat (200/201):
  xp_gained = fight_start.tnl + (cached_perlevel - cached_tnl)
  Detected when: fight_start.level >= 200 AND cached_tnl > fight_start.tnl
  At hero/SH, a powerup resets TNL from near-0 back to ~1000 without changing
  level. The normal formula would produce a negative value (clamped to 0).
  The powerup branch uses the same math as level-up to handle the TNL reset.

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
MOB LEVEL ESTIMATION (SACRIFICE TRIGGER)
================================================================================

Trigger regex: ^.+ gives you (\d+) gold coins? for .*\.$

Matches the sacrifice message from the player's deity (e.g., "Ayla gives you
100 gold coins for the twisted corpse of a pirate."). The deity name varies by
clan. Only the gold amount is captured.

Formula: mob_level = sacrifice_gold * 2

This is approximate. The exact Aardwolf formula is undocumented. GMCP does not
provide mob level through any field.

Timing: The sacrifice message arrives as text from the server in the same data
batch as the kill. MUSHclient processes text triggers on lines as they arrive,
before GMCP subnegotiation broadcasts. So the sacrifice trigger fires and sets
sacrifice_mob_level BEFORE the char.status broadcast triggers record_kill().

sacrifice_mob_level is reset in start_fight() and end_combat().

================================================================================
GMCP BROADCASTS HANDLED
================================================================================

char.base:
  - Caches perlevel for XP level-up calculation
  - Caches tier and remort for INSERT columns
  - Caches pups (powerup count) for powerup segmentation at 200+

char.status:
  - Primary combat state driver (enemy, enemypct, level, tnl)
  - Caches level and tnl (skips sentinel value -1)
  - Runs state machine transitions only when enabled

room.info:
  - Caches zone, room_num, room_name for kill/death location

Broadcasts NOT listened to (by design):
  - char.vitals: HP/mana/moves not tracked by LevelDB
  - char.stats: Stats not relevant to kill tracking
  - char.worth: Gold no longer tracked (removed in v4.0)
  - comm.channel: LevelDB has no interaction detection
  - comm.quest: Quest XP not tracked

================================================================================
PLUGIN LIFECYCLE
================================================================================

OnPluginInstall:
  1. Restore enabled state from saved variable
  2. Open database (leveldb.db)
  3. Display load message

OnPluginBroadcast (char.base):
  - Caches perlevel, tier, remort, and pups values from GMCP

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
  open_db() runs three steps in order:
  1. init_tables() - CREATE TABLE IF NOT EXISTS (no indexes)
  2. run_migrations() - ALTER TABLE for new columns (errors silently ignored)
  3. init_indexes() - CREATE INDEX IF NOT EXISTS (all columns now exist)

  This ordering ensures migrations add columns before indexes reference them.
  (v3.0 had a bug where idx_kills_pup was created before the pup column existed.)

Schema Migrations:
  - v2.0 -> v3.0: ALTER TABLE kills ADD COLUMN pup INTEGER
  - v3.0 -> v4.0: ALTER TABLE kills ADD COLUMN mob_level INTEGER
  Errors from "duplicate column name" are silently ignored. Old records retain
  NULL for new columns.

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

Powerup auto-adapt (ldb level, ldb this, ldb last):
- When cached_level >= 200, these commands query by powerup number instead of level
- ldb level [N]: N is a powerup number (default: current cached_pups)
- ldb this: shows kills for current powerup (WHERE level=200 AND pup=N)
- ldb last: shows kills for previous powerup (pup=N-1)
- show_level_stats(level, pup) adds AND pup=N to WHERE clause when pup is provided
- Header displays "Powerup N kills:" instead of "Level N kills:"

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

1. Multi-level-up in one kill:
   XP formula handles exactly one level-up. If a single kill grants enough XP for
   two level-ups (extraordinarily rare), the calculation will be short. Same
   limitation as stats_tracker.xml.

2. Zero-XP kills in kills table:
   Kills yielding 0 XP (low-level mobs, fled/interrupted combats) are recorded
   and included in all queries. This is by design -- LevelDB serves as a
   historical fight log, not just an XP tracker.

3. LIKE wildcard characters in search:
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
FUTURE CONSIDERATIONS (NOT IN V4)
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
