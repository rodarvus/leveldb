# Global Quest (GQ) Tracking — Implementation Plan

Status: READY — all trigger patterns identified (from stats_tracker.xml and Crowley's Search_and_Destroy.xml)

## Overview

Add global quest tracking to leveldb, modeled after the existing campaign system.
GQ tracking uses text triggers (always-on) and silent `gq check` / `gq info` parsing
(on-demand) to record global quest participation, results, and rewards.

## Sources

Trigger patterns validated against:
- `stats_tracker.xml` — GQ join/win/finish/mob-kill triggers and reward parsing
- `Search-and-Destroy/Search_and_Destroy.xml` (Crowley) — comprehensive GQ lifecycle
  triggers including extended mode, expiration, quit, cancellation, gq info/check parsing

## Database

### New table: `gquests`

```sql
CREATE TABLE IF NOT EXISTS gquests (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp     INTEGER NOT NULL,      -- os.time() when joined
    gq_number     INTEGER,               -- global quest number (e.g., 7286)
    level         INTEGER NOT NULL,       -- player level when joined
    tier          INTEGER,               -- char.base.tier (nullable)
    remort        INTEGER,               -- char.base.remorts (nullable)
    redo          INTEGER,               -- char.base.redos (nullable)
    result        TEXT,                   -- NULL (active), "won", "completed", "expired", "cancelled", "quit", "unknown"
    duration      INTEGER,               -- seconds from join to finish (nullable)
    mob_count     INTEGER,               -- number of mob lines assigned (nullable)
    qp            INTEGER,               -- total QP earned (base + per-mob kills)
    gold          INTEGER,
    tp            INTEGER,
    trains        INTEGER,
    pracs         INTEGER,
    expected_qp   INTEGER,               -- base QP from gq info (reconnect matching + display)
    expected_gold INTEGER                -- gold from gq info
);
```

### New table: `gquest_mobs`

```sql
CREATE TABLE IF NOT EXISTS gquest_mobs (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    gquest_id   INTEGER NOT NULL REFERENCES gquests(id),
    mob_name    TEXT NOT NULL,
    area        TEXT,                    -- area short name from gq check
    mob_count   INTEGER DEFAULT 1        -- "2 * billows of smoke" -> mob_count=2
);
```

### Indexes

```sql
CREATE INDEX IF NOT EXISTS idx_gquests_level ON gquests(level);
CREATE INDEX IF NOT EXISTS idx_gquests_tier ON gquests(tier);
CREATE INDEX IF NOT EXISTS idx_gquests_remort ON gquests(remort);
CREATE INDEX IF NOT EXISTS idx_gquests_redo ON gquests(redo);
CREATE INDEX IF NOT EXISTS idx_gquest_mobs_gqid ON gquest_mobs(gquest_id);
```

### Result values

| Value | Meaning |
|-------|---------|
| NULL | Active (in progress) |
| `"won"` | First to complete |
| `"completed"` | Finished but not first (during extended mode) |
| `"expired"` | GQ ended (time ran out) while you were still on it |
| `"cancelled"` | GQ cancelled due to lack of activity |
| `"quit"` | Player quit the gquest |
| `"unknown"` | Stale record closed on reconciliation |

### Design decisions

- `qp` accumulates ALL quest points: base reward + per-mob "N quest points awarded" lines.
  This gives total QP earned from the gquest in one field.
- `gq_number` is the global quest number (e.g., 7286). Used for display and reconnect
  matching.
- `mob_count` on gquest_mobs handles the "2 * mob_name" syntax from gq check.
  Each unique mob+area is one row; mob_count says how many to kill.
- `mob_count` on gquests is the number of mob *lines* (distinct targets), not total
  kills needed. This matches how campaigns track mob_count.
- "expired" vs "cancelled" are separate results because the MUD sends distinct messages:
  "is now over" vs "has been cancelled due to lack of activity".
- No migration needed for existing tables. Two new tables + indexes only.

## State variables

```lua
local active_gquest_id = nil       -- DB id of open gquest (nil if none)
local gqinfo_mob_buffer = {}       -- temp buffer for mob list during gq check parse
local gqinfo_qp = nil              -- base QP from gq info
local gqinfo_gold = nil            -- gold from gq info
local gqinfo_gq_number = nil       -- gq number from gq info (for reconnect matching)
```

## GQ lifecycle

```
1. GQ ANNOUNCED (not tracked — player hasn't joined yet)
   "Global Quest: Global quest # 7286 has been declared for levels 9 to 20..."

2. PLAYER JOINS
   "You have now joined Global Quest # 7286."
   └─ ldb_on_gq_join()
      └─ Close any open gquest as "unknown"
      └─ INSERT gquests record (timestamp, gq_number, level, tier, remort, redo)
      └─ Set active_gquest_id
      └─ send_silent_gq_info() → parse rewards
      └─ send_silent_gq_check() → parse mob list

3. MOB KILLS (0 or more)
   "Congratulations, that was one of the GLOBAL QUEST mobs!"
   "3 quest points awarded."
   └─ Accumulate per-mob QP into gquests.qp

4a. WON (first to complete)
    "You were the first to complete this quest!"
    "Reward of 16 quest points added."
    "Reward of 1000 gold coins added."
    └─ result = "won", parse reward lines

4b. COMPLETED (finished during extended mode, not first)
    "You have finished this global quest."
    └─ result = "completed", parse reward lines

4c. EXPIRED (GQ ended before you finished)
    "Global Quest: Global quest # 7286 is now over."
    or "Global Quest: Global quest # 7286 (extended) is now over."
    └─ result = "expired"

4d. CANCELLED (lack of activity)
    "Global Quest: Global quest # 7286 has been cancelled due to lack of activity."
    └─ result = "cancelled"

4e. QUIT (player leaves voluntarily)
    "You are no longer part of Global Quest # 7286 and will be unable to rejoin."
    └─ result = "quit"

EXTENDED MODE (informational, between 4a happening to someone else and 4b/4c):
   "Global Quest: Global Quest # 7286 has been won by SomePlayer - 3rd win."
   "Global Quest: Global Quest # 7286 will go into extended time for 5 more minutes."
   └─ Not directly tracked as a result. Player is still active until 4b/4c/4e.
```

## Triggers — always-on (don't gag)

All use `keep_evaluating="y"`, `omit_from_output="n"`, `sequence="100"`.

### trg_ldb_gq_join — Player joins a GQ

```
Match: ^You have now joined Global Quest # (\d+)\. See 'help gquest' for available commands\.$
```

Source: Crowley line 9319, stats_tracker line 74

Handler: `ldb_on_gq_join()`
- Close any previously open gquest as "unknown"
- INSERT new gquest record (timestamp, gq_number, level, tier, remort, redo)
- Set `active_gquest_id` from `db:last_insert_rowid()`
- Call `send_silent_gq_info()` to fetch expected rewards
- Chain `send_silent_gq_check()` after gq info completes, to fetch mob list

### trg_ldb_gq_mob_killed — GQ mob killed

```
Match: ^Congratulations, that was one of the GLOBAL QUEST mobs!$
```

Source: Crowley line 9358, stats_tracker line 40

Handler: `ldb_on_gq_mob_killed()`
- Enable trg_ldb_gq_mob_qp for 1 second (same pattern as stats_tracker)
- No-op otherwise (placeholder for future per-kill tracking)

### trg_ldb_gq_mob_qp — Per-mob QP reward (disabled by default)

```
Match: ^(\d+) quest points? awarded\.$
```

Source: stats_tracker line 43

Handler: `ldb_on_gq_mob_qp()`
- UPDATE gquests SET qp = COALESCE(qp, 0) + N WHERE id = active_gquest_id
- Gated: enabled for 1 second after trg_ldb_gq_mob_killed fires, then auto-disabled
  via DoAfterSpecial(1, ...). This prevents non-GQ QP lines from being captured.

### trg_ldb_gq_won — First to complete

```
Match: ^You were the first to complete this quest!$
```

Source: Crowley line 9328-area, stats_tracker line 46

Handler: `ldb_on_gq_won()`
- UPDATE gquests SET result = "won", duration = os.time() - timestamp
- Enable reward parsing triggers (trg_ldb_gq_reward) for 1 second

### trg_ldb_gq_finished — Completed but not first

```
Match: ^You have finished this global quest\.$
```

Source: Crowley line 9328, stats_tracker line 75

Handler: `ldb_on_gq_finished()`
- UPDATE gquests SET result = "completed", duration = os.time() - timestamp
- Enable reward parsing triggers for 1 second
- Clear active_gquest_id

### trg_ldb_gq_ended — GQ ended (expired/over)

```
Match: ^Global Quest: Global quest # (\d+)(?: \(extended\))? is now over\.$
```

Source: Crowley line 9343

Handler: `ldb_on_gq_ended()`
- Only act if captured gq_number matches active gquest's gq_number
- If active_gquest_id exists and result is still NULL:
  UPDATE gquests SET result = "expired", duration = os.time() - timestamp
- Clear active_gquest_id

### trg_ldb_gq_cancelled — GQ cancelled

```
Match: ^Global Quest: Global quest # (\d+) has been cancelled due to lack of activity\.$
```

Source: Crowley line 9348

Handler: `ldb_on_gq_cancelled()`
- Only act if captured gq_number matches active gquest's gq_number
- UPDATE gquests SET result = "cancelled", duration = os.time() - timestamp
- Clear active_gquest_id

### trg_ldb_gq_quit — Player quits GQ

```
Match: ^You are no longer part of Global Quest # (\d+) and will be unable to rejoin\.$
```

Source: Crowley line 9353

Handler: `ldb_on_gq_quit()`
- UPDATE gquests SET result = "quit", duration = os.time() - timestamp
- Clear active_gquest_id

### trg_ldb_gq_not_in — Not in a global quest (multiple patterns)

```
Match: ^You are not in a global quest\.$
Match: ^There are no global quests running\.$
```

Source: Crowley lines 9362, 9366

Handler: `ldb_on_gq_not_in()`
- If active_gquest_id exists: close as "unknown"
- Clear active_gquest_id
- Also used during reconnect reconciliation (disable gq info/check parsing)

## Triggers — reward parsing (disabled by default)

### trg_ldb_gq_reward — Reward lines after win/completion

```
Match: ^\s*Reward of ([\d,]+) (.+) added\.$
```

Source: stats_tracker line 49, same format as campaigns

Handler: `ldb_on_gq_reward()`
- Strip commas from amount, map type to DB field:
  "quest point" → qp, "gold coin" → gold, "trivia point" → tp,
  "training session" → trains, "practice" → pracs
- UPDATE gquests SET field = COALESCE(field, 0) + amount WHERE id = active_gquest_id
- Enabled for 1 second after won/finished trigger fires (DoAfterSpecial pattern
  from stats_tracker — simpler than campaign's separator-based end detection)

## Triggers — silent `gq info` parsing (disabled by default, gag output)

Sent via `SendNoEcho("gq info")`. Sequence 89/90/91 pattern (same as cp info).
All parsing triggers have `omit_from_output="y"`.

### Data capture (sequence 89)

| Trigger | Match | Handler |
|---------|-------|---------|
| `trg_ldb_gqinfo_none` | `^You are not in a global quest\.$` | `ldb_on_gqinfo_none()` |
| `trg_ldb_gqinfo_norun` | `^There are no global quests running\.$` | `ldb_on_gqinfo_none()` |
| `trg_ldb_gqinfo_qn` | `^Quest Name\.+:\s*\[\s*Global quest # (\d+)\s*\]` | `ldb_on_gqinfo_qn()` |
| `trg_ldb_gqinfo_qp` | `^Quest points\.+:\s*\[\s*(\d+)\s*\]` | `ldb_on_gqinfo_qp()` |
| `trg_ldb_gqinfo_gold` | `^Gold Coins\.+:\s*\[\s*([\d,]+)\s*\]` | `ldb_on_gqinfo_gold()` |
| `trg_ldb_gqinfo_status` | `^Quest Status\.+:\s*\[\s*(\w+)\s*\]` | `ldb_on_gqinfo_status()` |

### Gag (sequence 90)

| Trigger | Match | Handler |
|---------|-------|---------|
| `trg_ldb_gqinfo_gag` | `^(?:-+\[|Quest (?:Name\|Type\|Status\|Duration)\|Level range\|Time to Start\|Kill at least)` | no-op |

### End detection (sequence 91)

| Trigger | Match | Handler |
|---------|-------|---------|
| `trg_ldb_gqinfo_end` | `^-{20,}$` (closing separator — last line of gq info output) | `ldb_on_gqinfo_end()` |

Note: gq info output has a closing `---...---` separator (visible in the user's example),
so end detection works the same as cp info.

### gq info status field

The `Quest Status` line from gq info can be:
- `Preparing` — gquest hasn't started yet
- `Running` — gquest is active
- `Extended` — someone won, 5-minute extension active
- `Finished` — gquest is over

If status is `Finished`, skip mob list parsing (gquest already over). This handles the
edge case where gq info is sent during reconnect but the gquest ended while disconnected.

## Triggers — silent `gq check` parsing (disabled by default, gag output)

Sent via `SendNoEcho("gq check")` after gq info completes.

### Data capture (sequence 89, omit_from_output="y")

| Trigger | Match | Handler |
|---------|-------|---------|
| `trg_ldb_gqcheck_mob` | `^\[\*?\s*\d+\]\s+(?:(\d+) \* )?(.+?)\s+-\s+(\w+)\s*$` | `ldb_on_gqcheck_mob()` |
| `trg_ldb_gqcheck_none` | `^You are not in a global quest\.$` | `ldb_on_gqcheck_none()` |

### Gag (sequence 90, omit_from_output="y")

| Trigger | Match | Handler |
|---------|-------|---------|
| `trg_ldb_gqcheck_gag` | `^Index\s+Target\s+Area\s+Notes` and `^-{20,}$` (header + separator) | no-op |

### End detection (sequence 91, omit_from_output="y")

gq check output ends after the last mob line with no closing separator (unlike gq info).
Two options:

**Option A: Timer-based end** (recommended)
- After enabling gq check parsing, set DoAfterSpecial(1, "gqcheck_timeout()", 12)
- `gqcheck_timeout()` calls `ldb_on_gqcheck_end()` and disables parsing triggers
- Each mob line resets the timer (cancel + re-create)
- Simple and reliable; 1 second is plenty for all lines to arrive

**Option B: Negative lookahead** (Crowley's approach)
- Crowley uses `^(?!You still have to kill ...)$` to detect the first non-mob line
- More complex and fragile; timer approach is simpler for our needs

### mob line parsing

From the user's example:
```
[* 1] Jille                 - beer
[* 4] 2 * billows of smoke  - fireswamp
```

The `[* N]` prefix has a `*` when the mob is still alive. The mob line regex:
`^\[\*?\s*\d+\]\s+(?:(\d+) \* )?(.+?)\s+-\s+(\w+)\s*$`

Captures:
- Group 1 (optional): kill count ("2" from "2 * billows of smoke")
- Group 2: mob name ("Jille" or "billows of smoke")
- Group 3: area short name ("beer" or "fireswamp")

If group 1 is nil, mob_count defaults to 1.

## Silent command flow

### On join (`ldb_on_gq_join`)

1. INSERT gquest record, set active_gquest_id
2. `enable_gqinfo_parsing()` — enable gq info triggers
3. `SendNoEcho("gq info")`
4. When gq info end detected → `ldb_on_gqinfo_end()`
   - UPDATE gquest with expected_qp, expected_gold, gq_number
   - `enable_gqcheck_parsing()` — enable gq check triggers
   - `SendNoEcho("gq check")`
5. When gq check end detected (via timer) → `ldb_on_gqcheck_end()`
   - `flush_gqcheck_mob_buffer()` — DELETE old mobs, INSERT new, UPDATE mob_count

Sequential sending (gq info then gq check) avoids output interleaving.

### On plugin load (reconnect)

```lua
if enabled then
    DoAfterSpecial(2, "delayed_gq_info()", 12)
end
```

`delayed_gq_info()`:
- Send silent `gq info` to detect in-progress gquest
- If gq info shows active gquest (status = Running or Extended):
  - If saved `active_gquest_id` exists and gq_number matches → keep record, refresh
  - If saved `active_gquest_id` exists but doesn't match → close old as "unknown", create new
  - If no saved gquest → create new record
  - Chain `gq check` to refresh mob list
- If gq info shows Finished status → close any saved record as "unknown"
- If "not in a gquest" / "no global quests running" → close any saved record as "unknown"

## State persistence

Add to `OnPluginSaveState`:
```lua
SetVariable("active_gquest_id", active_gquest_id and tostring(active_gquest_id) or "")
```

Restore in `OnPluginInstall`:
```lua
local saved_gquest = GetVariable("active_gquest_id")
if saved_gquest and saved_gquest ~= "" then
    active_gquest_id = tonumber(saved_gquest)
end
```

## Commands

### `ldb gq [filter]` — GQ history table

Alias: `^ldb gq(?:uest)?s?(?: (.+))?$`

Same structure as `ldb cp`:
- Tabular format: #, Lvl, Mobs, QP, Gold, TP, Time
- Color-coded: active (green), won (cyan), completed (white), expired/cancelled/quit (red)
- Summary: win count, completion rate, avg QP, avg time, total QP, total gold
- Tier/remort/redo filtering (default: current T+R)
- Capped at 50 rows

### `ldb gq show <id>` — GQ detail view

Alias: `^ldb gq(?:uest)?s? show (\d+)$` (sequence 99, before list command)

Same structure as `ldb cp show`:
- Timestamp, level (T/R), gq number
- Result (colored) with duration
- Mob count
- Actual rewards (if won/completed) or expected rewards (if active)
- Full mob list with areas and kill counts (mob_count column)

## Plugin coexistence

- **stats_tracker.xml**: Both track GQ wins/completions. stats_tracker uses aggregate
  counters; leveldb records per-event detail. Independent, no conflict.
- **Search_and_Destroy.xml** (Crowley): Both listen for GQ triggers. All leveldb triggers
  use `keep_evaluating="y"` so S&D triggers still fire. LevelDB's silent `gq info`/
  `gq check` could interfere with S&D's parsing if both send commands simultaneously.
  In practice this is unlikely: LevelDB sends on join and reconnect only, while S&D
  sends on user action. If conflicts arise, could add detection of S&D presence and
  skip silent parsing.
- **GQ_List.xml**: Only parses `gq list` output (the public GQ listing). No conflict
  with `gq info` or `gq check` parsing.

## Version

This will be v9.0. Schema changes:
- New tables: gquests, gquest_mobs (no migrations needed for existing tables)
- New indexes: 5 new indexes
- `ldb` status will show gquest count
- `ldb db` will show gquest count
- `ldb help` will list gq commands

## Resolved questions

1. **Per-mob QP in "completed" (not won) scenario**: Yes — you earn QP per mob kill
   (including the +3 QP bonus) regardless of whether you win. GQs are a useful source
   of quest points even without winning.

2. **Reward lines after "completed" (not won)**: Unknown — needs testing. The
   `trg_ldb_gq_finished` handler enables reward parsing just in case. If no rewards
   come, the 1-second timeout harmlessly disables the triggers.
   **TODO (test in-game):** Verify reward line behavior when finishing a GQ during
   extended mode (not first). Update handler if needed.

3. **Color scheme for results**: Confirmed. won=cyan, completed=white, active=green,
   expired/cancelled/quit=red.
