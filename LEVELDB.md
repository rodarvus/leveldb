================================================================================
LEVELDB PLUGIN - IMPLEMENTATION GUIDE
================================================================================
Last Updated: 2026-04-01
Plugin File: leveldb.xml
Plugin ID: b34c04e52c6c7bced4508230
Author: Rodarvus
Current Version: 7.0

================================================================================
OVERVIEW
================================================================================
LevelDB is a standalone MUSHclient plugin for Aardwolf MUD that records combat,
quest, campaign, and powerup data in a persistent SQLite database. It tracks mob
kills, XP gains, mob level estimates, damage dealt, rounds fought, combat time,
deaths, quest completions, campaign results, and powerup (pup) events across the
character's entire leveling journey.

Key Features:
- Per-kill records: mob name, zone, room, level, XP, mob level, damage, rounds,
  combat time
- Per-death records: mob, zone, room, level, tier, remort
- Quest tracking: target, area, room, timer, result, rewards (QP/gold/TP/trains/pracs)
- Campaign tracking: mob list, result, rewards, expected rewards from cp info
- Powerup tracking: pup events with trains earned, per-area productivity stats
- Tier and remort tracked as columns in every record
- Tier/remort filtering: level/this/last/pup/quest/cp commands default to current
  tier+remort, with optional filters (all, T1 R5, R4, T1)
- Remort summary: bracket breakdown (1-50, 51-100, 101-150, 151-200) with
  avg XP, damage, level gap, rounds, zones, deaths. Separate powerup section
  with trains, combat time, and per-area productivity.
- Tier summary: compare remorts within a tier, with powerup comparison table
- Single database file (leveldb.db) for all data
- Rich query commands: per-level, per-zone, per-mob breakdowns, top-N rankings,
  powerup productivity analysis
- GMCP state restored on plugin reload (no stale cache after mid-session reload)

================================================================================
COMMANDS
================================================================================

BASIC:
  ldb                     - Show status (enabled, DB path, record counts)
  ldb help                - Show all commands
  ldb on                  - Enable data collection
  ldb off                 - Disable data collection

QUERIES (COMBAT):
  ldb level [N] [filter]  - Kill breakdown for level N (default: current level)
                            Tabular per-kill listing with totals and averages
                            At level 200+: redirects to ldb pup
  ldb this [filter]       - Shortcut for current level (redirects to ldb pup at 200+)
  ldb last [filter]       - Shortcut for previous level (redirects to ldb pup at 200+)

  Filter options:
    (default)             - Current tier and remort only
    all                   - All tiers and remorts (separate sections)
    T1 R5                 - Specific tier and remort
    T1                    - All remorts within a tier (separate sections)
    R4                    - Specific remort, current tier

QUERIES (POWERUPS):
  ldb pup [filter|zone]   - Powerup productivity stats (summary + per-area table)
                            Default: current tier & remort.
                            Filter: all, T1 R5, T1, R4
                            Zone: substring match for per-mob detail
  ldb pup <id>            - Per-kill detail for a specific powerup event by ID
  ldb pup list [N]        - Last N powerup events (default 10)

  ldb remort [R] [T<n>]   - Bracket summary for one remort (1-50, 51-100,
                            101-150, 151-200). Separate powerup section below
                            with trains, combat time, and per-area table.
                            Default: current tier & remort.
  ldb tier [T]             - Compare remorts within a tier. Bracket sections
                            followed by Powerups comparison table.
                            Default: current tier. ldb tier 1 = tier 1.

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

QUERIES (QUESTS/CAMPAIGNS):
  ldb quest [filter]      - Quest history table (default: current tier & remort)
                            Shows: ID, Lvl, QP, Gold, TP, Time, Mob (Area)
                            Colored suffixes: (active) green, (failed)/(timeout) red
                            Summary with completion rate, avg QP, avg time
                            Filter options: same as level/this/last commands
                            Capped at 50 rows; total count shown if more exist

  ldb cp [filter]         - Campaign history table (default: current tier & remort)
                            Shows: ID, Lvl, Mobs, QP, Gold, TP, Time
                            Colored suffixes: (active) green, (quit) red
                            Summary with completion rate, avg QP, avg time
                            Filter options: same as level/this/last commands
                            Also: ldb cps, ldb campaign, ldb campaigns

  ldb cp show <id>        - Detailed view of a specific campaign by database ID
                            Shows: timestamp, level (T/R), result (colored),
                            mob count, rewards (actual or expected), full mob list
                            Also: ldb cps show, ldb campaign show, ldb campaigns show

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
    combat_time  REAL                  -- seconds of active combat (utils.timer() delta, nullable)

  pup_events:
    id           INTEGER PRIMARY KEY AUTOINCREMENT  -- sequential, used as pup ID
    timestamp    INTEGER NOT NULL      -- os.time() when powerup occurred
    tier         INTEGER NOT NULL       -- char.base.tier
    remort       INTEGER NOT NULL       -- char.base.remorts
    pup_number   INTEGER NOT NULL       -- powerup count after this event
    zone         TEXT                   -- zone where powerup occurred (nullable)
    trains_earned INTEGER              -- trains awarded for this powerup (nullable)
    xp_per_level INTEGER               -- char.base.perlevel at time of pup (nullable)

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

  quests:
    id           INTEGER PRIMARY KEY AUTOINCREMENT
    timestamp    INTEGER NOT NULL      -- os.time() at quest start
    level        INTEGER NOT NULL       -- player level when quest started
    tier         INTEGER               -- char.base.tier (nullable)
    remort       INTEGER               -- char.base.remorts (nullable)
    mob_name     TEXT                   -- target mob name from GMCP (nullable)
    area         TEXT                   -- target area from GMCP (nullable)
    room         TEXT                   -- target room from GMCP (nullable)
    timer        INTEGER               -- time limit in minutes from GMCP (nullable)
    result       TEXT                   -- completed/failed/timeout/unknown (nullable while active)
    duration     INTEGER               -- seconds from start to completion (nullable while active)
    qp           INTEGER               -- quest points earned (nullable)
    gold         INTEGER               -- gold earned (nullable)
    tp           INTEGER               -- trivia points earned (nullable)
    trains       INTEGER               -- training sessions earned (nullable)
    pracs        INTEGER               -- practices earned (nullable)

  campaigns:
    id           INTEGER PRIMARY KEY AUTOINCREMENT
    timestamp    INTEGER NOT NULL      -- os.time() at campaign start
    level        INTEGER NOT NULL       -- player level when campaign started
    tier         INTEGER               -- char.base.tier (nullable)
    remort       INTEGER               -- char.base.remorts (nullable)
    result       TEXT                   -- completed/quit/unknown (nullable while active)
    duration     INTEGER               -- seconds from start to completion (nullable while active)
    mob_count    INTEGER               -- number of mobs assigned (nullable)
    qp           INTEGER               -- quest points earned (nullable)
    gold         INTEGER               -- gold earned (nullable)
    tp           INTEGER               -- trivia points earned (nullable)
    trains       INTEGER               -- training sessions earned (nullable)
    pracs        INTEGER               -- practices earned (nullable)
    expected_qp  INTEGER               -- QP from cp info (for reconnect matching)
    expected_gold INTEGER              -- gold from cp info (for display)

  campaign_mobs:
    id           INTEGER PRIMARY KEY AUTOINCREMENT
    campaign_id  INTEGER NOT NULL REFERENCES campaigns(id)
    mob_name     TEXT NOT NULL          -- mob name from cp info output
    area         TEXT                   -- area name from cp info output (nullable)

Indexes:
  idx_kills_level, idx_kills_zone, idx_kills_mob, idx_kills_timestamp
  idx_kills_tier, idx_kills_remort, idx_kills_pup, idx_kills_mob_level
  idx_deaths_level, idx_deaths_zone, idx_deaths_tier, idx_deaths_remort
  idx_quests_level, idx_quests_tier, idx_quests_remort
  idx_campaigns_level, idx_campaigns_tier, idx_campaigns_remort
  idx_campaign_mobs_cid
  idx_pup_events_tier_remort

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
- combat_time is measured using utils.timer() for sub-second precision. Nullable:
  NULL for historical kills recorded before v7.0 (added via ALTER TABLE migration).
- pup_events.trains_earned is nullable: NULL if char.worth data was not available
  when the pup event fired (e.g., GMCP ordering edge case, stale pending_pup).
  Practices are not earned from powerups; only trains are tracked.
- pup_events.id is sequential and used as the public pup identifier in ldb pup <id>.
  Kills for a specific pup are found by querying kills between consecutive
  pup_events timestamps (same tier+remort).
- Quest result is NULL while active, set on completion/failure/timeout. "unknown"
  is set when a quest is force-closed (new quest starts while old one has no result,
  or reconnect finds no active quest).
- Campaign expected_qp/expected_gold are captured from cp info output for two
  purposes: reconnect matching (compare level + expected_qp to detect same campaign)
  and display in ldb cp show for active campaigns.

================================================================================
COMBAT STATE MACHINE
================================================================================

States:
  IDLE    -- not in combat (in_combat = false)
  COMBAT  -- fighting a mob (in_combat = true)

Transitions:

  1. IDLE --> COMBAT:
     When: char.status.enemy becomes non-empty
     Action: Snapshot fight_start (tnl, level, zone, room, enemy, pup, start_time)
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
  sacrifice_mob_level directly from module-level state, computes XP gained
  and combat_time (utils.timer() - fight_start.start_time), and INSERTs the
  kill record into the database.

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
QUEST TRACKING
================================================================================

Quests are tracked via GMCP comm.quest broadcasts. The aard_GMCP_handler plugin
processes quest-related GMCP messages and broadcasts them to all plugins.

GMCP comm.quest actions handled:
  - start:   New quest assigned. Closes any open quest (as "unknown"), creates
             new record with mob_name, area, room, timer from GMCP data.
  - comp:    Quest completed. Updates result to "completed", records duration,
             captures rewards (totqp, gold, tp, trains, pracs) from GMCP data.
  - fail:    Quest failed. Updates result to "failed", records duration.
  - timeout: Quest timed out. Updates result to "timeout", records duration.
  - status:  Login/reconnect status. If status="ready" (no active quest), closes
             any open quest record. If a quest target is present and no active
             record exists, creates a new record for the in-progress quest.

Quest Lifecycle:
  1. Quest assigned (GMCP start) -> INSERT with NULL result
  2. Quest completed/failed/timeout (GMCP comp/fail/timeout) -> UPDATE result + duration
  3. New quest before old one resolved -> old quest closed as "unknown"

Active Quest State:
  active_quest_id tracks the DB id of the current open quest. Persisted via
  save_state across plugin reloads. Set on INSERT, cleared on completion/failure.

================================================================================
CAMPAIGN TRACKING
================================================================================

Campaigns are tracked via a combination of text triggers (always-on) and silent
cp info parsing (on-demand). Unlike quests, campaigns have no GMCP broadcast for
mob lists, so the plugin sends "cp info" to the MUD to capture this data.

Campaign Lifecycle:
  1. Campaign requested -> Questmaster says "Good luck in your campaign!"
     (trg_ldb_cp_start) -> INSERT campaign record, send silent cp info
  2. cp info response parsed -> mob list captured, expected rewards recorded
  3. Mobs killed -> trg_ldb_cp_mob_killed fires (handler is a no-op, kept for
     potential future use)
  4a. Campaign completed -> "CONGRATULATIONS!" trigger fires -> UPDATE result to
      "completed", enable reward parsing triggers
  4b. Campaign quit -> "Campaign cleared." trigger fires -> UPDATE result to "quit"
  5. Reward lines parsed -> UPDATE campaign with actual QP/gold/TP/trains/pracs
  6. Closing separator -> disable reward triggers, clear active_campaign_id

Silent cp info Mechanism:
  The plugin sends SendNoEcho("cp info") to retrieve campaign details without
  echoing the command. Seven triggers (disabled by default) are temporarily enabled
  to parse the response:

  Trigger Name             Match Pattern                              Seq  Purpose
  trg_ldb_cpinfo_none      ^You are not currently on a campaign\.$   89   No campaign
  trg_ldb_cpinfo_level     ^Level Taken\.+:\s*\[\s*(\d+)\s*\]       89   Capture level
  trg_ldb_cpinfo_qp        ^Quest Points\.+:\s*\[\s*(\d+)\s*\]      89   Capture QP
  trg_ldb_cpinfo_gold      ^Gold Coins\.+:\s*\[\s*([\d,]+)\s*\]     89   Capture gold
  trg_ldb_cpinfo_mob       ^Find and kill \d+ \* (.+) \((.+)\)$     89   Capture mob
  trg_ldb_cpinfo_gag       ^(?:-+\[|Complete By|Time Left|...)       90   Gag headers
  trg_ldb_cpinfo_end       ^-{20,}$                                  91   End of output

  All parsing triggers have omit_from_output="y" to suppress the cp info output.
  Sequence ordering ensures: data capture (89) < gag (90) < end detection (91).
  All triggers are disabled after parsing completes (success or "no campaign").

  cp info is sent in two scenarios:
  1. Campaign start: immediately after the "Good luck" trigger fires
  2. Plugin load: via DoAfterSpecial(2, "delayed_cp_info()", 12) to detect
     in-progress campaigns on reconnect/reload

Campaign Reconnect Reconciliation:
  On plugin load, a delayed cp info check runs after 2 seconds. The response
  is reconciled against the saved active_campaign_id:

  - No saved campaign + cp info shows campaign -> create new record
  - Saved campaign + cp info matches (level + expected_qp) -> keep record,
    refresh mob list
  - Saved campaign + cp info doesn't match -> close old as "unknown",
    create new record
  - cp info says "not on a campaign" -> close any saved record as "unknown"

Campaign Reward Parsing:
  After "CONGRATULATIONS!" fires, two reward triggers are enabled:
  - trg_ldb_cp_reward: "  Reward of N,NNN type added." -> UPDATE campaign
    field (qp/gold/tp/trains/pracs) using COALESCE for additive rewards
  - trg_ldb_cp_reward_end: closing "---" separator -> disable reward triggers,
    clear active_campaign_id

Why cp info Instead of cp check:
  WinkleGold_Mapper_Extender has triggers that parse cp check output. When
  LevelDB sends a silent "cp check", WinkleGold sees the response and errors
  because its serialization logic doesn't expect it during that context.
  "cp info" produces similar output but WinkleGold has no triggers for it,
  avoiding the conflict entirely. Additionally, cp info provides richer data
  (level taken, expected QP, expected gold) that cp check does not.

================================================================================
POWERUP (PUP) TRACKING
================================================================================

Pup Detection:
  Primary: text trigger matches "Congratulations, X. You have increased your
  powerups to N." This fires reliably for every pup. The handler records a
  pup_events row immediately with trains_earned = 0, and saves the row ID.

  Fallback: char.base GMCP broadcast with increased pups value. Records the
  pup event with NULL trains if the text trigger hasn't already handled it.

Train Tracking:
  Two text triggers capture trains awarded after each powerup:
  - "You gain N trains." -> adds N to the most recent pup_events record
  - "Lucky! You gain an extra N training sessions!" -> adds N to the same record

  Both use UPDATE ... SET trains_earned = COALESCE(trains_earned, 0) + N
  on the pup_events row saved by the powerup trigger. This is simple and
  reliable — no GMCP timing dependencies.

  Note: Practices are not earned from powerups; only trains are tracked.

Per-Area Productivity (derived, not stored):
  From kills table (level >= 200):
  - Per area: avg XP/kill, avg combat_time/kill
  - Est time per pup = (xp_per_level / avg_xp_per_kill) * avg_combat_time_per_kill
  - Est trains per hour = (avg_trains_per_pup / est_time_per_pup) * 3600

================================================================================
GMCP BROADCASTS HANDLED
================================================================================

char.base:
  - Caches perlevel for XP level-up calculation
  - Caches tier and remort for INSERT columns
  - Caches pups (powerup count) for pup event detection
  - Detects pup events when pups value increases

char.status:
  - Primary combat state driver (enemy, enemypct, level, tnl)
  - Caches level and tnl (skips sentinel value -1)
  - Runs state machine transitions only when enabled

room.info:
  - Caches zone, room_num, room_name for kill/death location

comm.quest:
  - Quest lifecycle tracking (start, comp, fail, timeout, status)
  - Dispatches to handle_quest() for database operations

Broadcasts NOT listened to (by design):
  - char.vitals: HP/mana/moves not tracked by LevelDB
  - char.stats: Stats not relevant to kill tracking
  - comm.channel: LevelDB has no chat interaction detection

================================================================================
PLUGIN LIFECYCLE
================================================================================

OnPluginInstall:
  1. Restore enabled state from saved variable
  2. Open database (leveldb.db)
  3. Read current GMCP state (char.base, char.status, room.info) via gmcp()
     to populate cached values. Handles mid-session plugin reload.
  4. Set prev_pups = cached_pups to prevent false pup detection on reload
  5. Restore active_quest_id and active_campaign_id from saved state
  6. Check quest status via GMCP comm.quest (reconcile quest state on reload)
  7. If enabled, schedule delayed_cp_info() via DoAfterSpecial(2s) to detect
     and reconcile in-progress campaigns on reconnect/reload
  8. Display load message

OnPluginSaveState:
  - Persists enabled flag, active_quest_id, active_campaign_id

OnPluginClose:
  - Close database handle

OnPluginDisconnect:
  - End any active combat tracking (prevents stale state on reconnect)

State Persistence:
  save_state="y" with three variables: enabled (boolean),
  active_quest_id (number or ""), active_campaign_id (number or "")

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
  - v5.0 -> v6.0: ALTER TABLE campaigns ADD COLUMN expected_qp INTEGER
                   ALTER TABLE campaigns ADD COLUMN expected_gold INTEGER
  - v6.0 -> v7.0: ALTER TABLE kills ADD COLUMN combat_time REAL
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

Powerup commands (ldb pup):
- ldb level, ldb this, ldb last redirect to ldb pup at level 200+
- ldb pup queries pup_events and kills (level >= 200) for productivity stats
- Per-area productivity is derived: Est/Pup = (xp_per_level / avg_xp) * avg_time
- ldb pup <id> queries kills between consecutive pup_events timestamps
- ldb pup <zone> groups kills by mob_name for per-mob breakdown

Tier/remort filtering (ldb level, ldb this, ldb last, ldb quest, ldb cp):
- Default (no filter): queries with AND tier=cached_tier AND remort=cached_remorts
- "all": queries for DISTINCT (tier, remort) combos at that level (excluding NULLs),
  then renders each combo as a separate section with its own table and totals
- "T1 R5": queries with AND tier=1 AND remort=5
- "T1": queries for DISTINCT (tier, remort) combos WHERE tier=1 (multi-section)
- "R4": queries with AND tier=cached_tier AND remort=4
- parse_filter() parses the filter string, returns nil on invalid input
- show_level_stats() orchestrates; show_level_stats_section() renders one section
- Records with NULL tier or remort are silently excluded from all filtered queries

Quest/campaign display:
- Tabular output with database ID (#), level, rewards, duration, and target info
- Database IDs (not sequential) allow cross-tier/remort lookups (ldb cp show <id>)
- Colored status suffixes via multi-argument ColourNote: green (active), red (failed/
  timeout/quit)
- Summary line shows completion breakdown and total count
- Averages line shows avg QP, avg time, total QP, total gold (completed only)
- Capped at 50 rows; total count displayed if more exist
- format_duration(): <120s shows seconds, else shows minutes

ldb top xp:
- Uses HAVING cnt >= 3 to exclude one-off kills that skew averages
- Ranks by avg XP per kill (CAST SUM AS REAL / COUNT)

================================================================================
PLUGIN COEXISTENCE
================================================================================

LevelDB is mostly passive -- it primarily observes GMCP broadcasts and text output.
The one exception is campaign tracking, where it sends "cp info" to the MUD via
SendNoEcho() (command not echoed to the user). This occurs at most twice per
campaign: once at campaign start, and once on plugin load if a campaign is active.

Compatibility notes:
- stats_tracker: Both track XP via TNL delta; independent databases
- Aardwolf_Damage_Window: Both match damage lines; keep_evaluating="y"
- aard_soundpack: Both match "You die."; keep_evaluating="y"
- WinkleGold_Mapper_Extender: Uses cp info (not cp check) to avoid triggering
  WinkleGold's cp check parsing triggers, which would error on unexpected input.
  cp info output is not parsed by any known community plugin.
- NPC_Info, WingleGold_Spellup, or any other plugin: No known conflicts

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

4. Campaign mob list is static:
   The mob list is captured once via cp info when the campaign starts. It is not
   updated as mobs are killed. The mob_killed trigger is a no-op placeholder.

5. Campaign reconnect timing:
   The delayed_cp_info() fires 2 seconds after plugin load. If GMCP data has not
   arrived by then, the campaign record may use incomplete cached values (level,
   tier, remort). This window is small in practice.

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
VERSION HISTORY
================================================================================

v7.0 (2026-04-01):
  - Powerup module redesign: productivity-focused tracking
  - New table: pup_events (sequential ID, trains_earned, zone, xp_per_level)
  - New column: kills.combat_time (active combat duration per kill)
  - New GMCP handler: char.worth (caches trains for pup train delta)
  - Pup detection via char.base.pups increment with pending_pup mechanism
  - New command: ldb pup [filter|zone|id] - powerup productivity stats
  - Removed Pups bracket from remort/tier bracket tables
  - ldb remort/tier now show separate powerup sections
  - ldb level/this/last at 200+ redirect to ldb pup

v6.0 (2026-03-26):
  - Quest tracking via GMCP comm.quest broadcasts (start/comp/fail/timeout/status)
  - Campaign tracking via text triggers + silent cp info parsing
  - New tables: quests, campaigns, campaign_mobs
  - New commands: ldb quest [filter], ldb cp [filter], ldb cp show <id>
  - Campaign mob list captured from cp info output (avoids WinkleGold conflicts)
  - Campaign reconnect reconciliation (level + expected_qp matching)
  - Campaign reward parsing (QP, gold, TP, trains, pracs)
  - Tier/remort filtering for quest and campaign commands
  - Active quest/campaign IDs persisted across plugin reloads
  - Status command (ldb) now shows quest and campaign counts

v5.0 (2026-03-26):
  - Tier/remort filtering for level/this/last commands (default: current T+R)
  - New command: ldb remort - bracket summary (1-50, 51-100, 101-150, 151-200,
    Pups) with avg XP, damage, level gap, rounds, zones, deaths, pup count
  - New command: ldb tier - compare remorts within a tier by bracket
  - Pups bracket: filters out low-level mob kills (area goals), shows pup count
  - Deaths output now shows mob name, tier, and remort
  - GMCP state restored on plugin reload (fixes stale cache issue)
  - Added format_abbreviated() for compact large numbers (1.2k, 3.4M)
  - Defensive COALESCE in aggregate queries
  - Refactored bracket queries to eliminate code duplication

v4.0 (2026-02-23):
  - Added mob_level estimation via sacrifice gold
  - MLvl column in level reports

v3.0:
  - Added powerup (pup) tracking for level 200+

v2.0:
  - Added tier and remort columns to kills and deaths tables

v1.0:
  - Initial release: kill/death tracking, per-level/zone/mob queries

================================================================================
FUTURE CONSIDERATIONS
================================================================================

- Session tracking / XP-per-hour calculation
- Miniwindow / real-time display
- Item loot tracking
- GQ (Global Quest) tracking
- Spell-per-kill tracking
- Data export (CSV/JSON)
- Level time estimation
- ldb history command (XP/hour graph)
- ldb leveltime command (average time per level)
- Cross-remort comparison tables (same level across remorts)

================================================================================
END OF IMPLEMENTATION GUIDE
================================================================================
