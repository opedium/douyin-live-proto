# Gift Price Management System

**Date**: 2026-07-01
**Status**: Draft
**Designer**: Claude Code (opedium/barra2)

## 1. Motivation

The Douyin live capture system records gift events as `(gift_name, gift_count, diamond_total)` in the `gift_logs` table. The `diamond_total` is computed at parse time as `unit_price × gift_count`, where `unit_price` comes from one of three sources, in priority order:

| Source | Current Form | Limitations |
|--------|-------------|-------------|
| `_GIFT_PRICE_OVERRIDE` | Hardcoded dict in `base/parser.py` (~11 entries) | Source-controlled, no UI, no versioning |
| `_GIFT_FALLBACK` | Hardcoded dict in `base/parser.py` (~3 entries) | Same |
| `gift.diamond_count` | Value from protobuf message | Reports base price for limited-edition skins (e.g., 至尊超跑 protobuf returns 1,200 but real price is 12,000) |
| `gift_registry.json` | File loaded at startup | **File doesn't exist** — silently swallowed by `except: pass` |

This has real consequences. Analysis of the existing database shows:

- **154 distinct gift names** in `gift_logs`, but only ~11 have hardcoded overrides
- **2 detected mispricings**: `闪烁星河` recorded at 1 diamond but should be 99; `点点星光` recorded at 1 but should be 9 — because the overrides were added after data collection
- **No `gift_id` stored** in `gift_logs` — only the display name string, so any price correction must key on `gift_name`
- **No audit trail** of what price was applied at parse time

### Design Goals

1. **Replace hardcoded dicts** with a persistent SQLite table
2. **Provide a web UI** to browse, search, edit, and audit gift prices
3. **Recalculation engine** to fix historical data after a price correction
4. **Transparent fallback chain**: DB → hardcoded dicts → protobuf value

## 2. Data Model

### New Table: `gift_prices`

```sql
CREATE TABLE IF NOT EXISTS gift_prices (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    gift_name       TEXT NOT NULL UNIQUE,         -- display name (lookup key)
    gift_id         INTEGER DEFAULT 0,            -- Douyin numeric gift ID
    diamond_count   INTEGER NOT NULL DEFAULT 0,   -- official price in diamonds
    source          TEXT NOT NULL DEFAULT 'auto',  -- 'auto'|'manual'|'override'|'registry'
    is_limited_skin INTEGER NOT NULL DEFAULT 0,   -- 1 = limited-edition gift skin
    base_gift_name  TEXT NOT NULL DEFAULT '',      -- e.g. '嘉年华' for '钻石嘉年华'
    notes           TEXT NOT NULL DEFAULT '',      -- human notes on this price
    created_at      DATETIME DEFAULT (datetime('now', '+8 hours')),
    updated_at      DATETIME DEFAULT (datetime('now', '+8 hours'))
);
CREATE INDEX IF NOT EXISTS idx_gift_prices_name ON gift_prices(gift_name);
```

### Key Design Decisions

- **`gift_name` as primary lookup key**: `gift_logs` stores `gift_name` but not `gift_id`. The table has its own auto-increment `id` for editing and cross-references, with `gift_name` UNIQUE for fast lookups.
- **`source` enum**: Tracks provenance so the UI can highlight entries that need human verification.
- **`is_limited_skin` + `base_gift_name`**: Captures the limited-edition relationship (e.g., 钻石嘉年华 is a limited skin of 嘉年华). This is metadata only — the price override is independent.

### Existing Table Modification

No schema changes to `gift_logs`. The `diamond_total` column already exists and will be updated in-place during recalculation.

## 3. Data Population Strategy

Three sources, consulted in priority order:

### Source 1: Existing Hardcoded Dicts (Priority: HIGH)
On first run, migrate all entries from `_GIFT_PRICE_OVERRIDE` and `_GIFT_FALLBACK` into the `gift_prices` table with `source = 'override'`. These are manually verified prices and should never be overwritten by auto-detection.

### Source 2: gift_registry.json (Priority: MEDIUM)
If `data/gift_registry.json` exists, load entries with `source = 'registry'`. Only insert gifts not already in the table.

### Source 3: Auto-Detection from gift_logs (Priority: LOW — FALLBACK)
For every distinct `gift_name` in `gift_logs`, compute the consensus unit price:
```sql
SELECT gift_name, diamond_total / MAX(gift_count, 1) AS unit_price, COUNT(*) AS occurrences
FROM gift_logs
GROUP BY gift_name, unit_price
HAVING occurrences > 0
```

- If all occurrences of a gift name have the **same** `diamond_total / gift_count`, that's the auto-detected price
- If there are **multiple distinct unit prices** for the same `gift_name`, flag the gift for manual review
- Gifts already in the table from sources 1-2 are **not** overwritten

### Idempotent Initialization

The population routine is safe to run on every startup — entries from authoritative sources are INSERTed once, while auto-detected prices are refreshed in case new data has appeared:

```python
def init_gift_prices_table():
    conn = _get_conn()
    # 1. Create table if not exists
    # 2. INSERT OR IGNORE from _GIFT_PRICE_OVERRIDE (source='override')
    #    — manually verified, never overwrite
    # 3. INSERT OR IGNORE from _GIFT_FALLBACK (source='override')
    #    — manually verified, never overwrite
    # 4. INSERT OR IGNORE from gift_registry.json if exists (source='registry')
    #    — from official registry, never overwrite
    # 5. UPSERT auto-detected prices from gift_logs:
    #    For each gift_name in gift_logs:
    #      - If gift_name NOT in gift_prices → INSERT with source='auto'
    #      - If gift_name EXISTS with source='auto' → UPDATE diamond_count
    #        (re-detect in case data volume changed the consensus)
    #      - If gift_name EXISTS with source='manual'|'override'|'registry'
    #        → SKIP (authoritative sources win)
```

## 4. Web UI

### 4.1 Route: `/gift-prices`
Main listing page. Shows all gifts in a searchable, filterable table.

```
┌─────────────────────────────────────────────────────────────┐
│ [Search gifts...]  [▼ All] [▼ Needs Review] [▼ Limited]     │
├──────────────┬───────┬────────┬──────────┬──────┬───────────┤
│ Gift Name    │ Price │ Source │ Skin?    │ Logs │ Actions   │
├──────────────┼───────┼────────┼──────────┼──────┼───────────┤
│ 嘉年华       │ 30000 │ auto   │ No       │  43  │ [Edit]    │
│ 钻石嘉年华   │ 36000 │ manual │ Yes→嘉华  │   5  │ [Edit]    │
│ 闪烁星河     │  1 →? │ audit  │ —        │ 330  │ [Edit] ⚠  │
│ 神秘飞机     │  1800 │ auto   │ No       │   6  │ [Edit]    │
│ ...          │       │        │          │      │           │
└──────────────┴───────┴────────┴──────────┴──────┴───────────┘
```

Columns:
- **Gift Name**: Display name, clickable to edit
- **Price**: Current diamond_count in table
- **Source**: Colored badge — green=manual, blue=override, gray=auto, red=conflict
- **Skin?**: Yes/No + base gift name if is_limited_skin
- **Logs**: Count of gift_logs rows with this gift_name (link to filtered search)
- **Actions**: [Edit] [Recalculate]

### 4.2 Route: `/gift-prices/<id>/edit`
Edit form for a single gift.

| Field | Type | Notes |
|-------|------|-------|
| Gift Name | Text (readonly) | Key field, must match gift_logs |
| Gift ID | Number | Optional Douyin numeric ID |
| Diamond Count | Number | The price to use |
| Source | Select | auto / manual / override / registry |
| Is Limited Skin | Checkbox | |
| Base Gift Name | Text (conditional) | Only shown if limited skin checked |
| Notes | Textarea | Free text |

**Actions**: [Save] [Save & Recalculate] [Cancel]

### 4.3 Route: `/gift-prices/audit`
Discrepancy and data quality dashboard.

**Section 1: Price Discrepancies**
Gifts where the same `gift_name` has different `diamond_total / gift_count` values in `gift_logs`.

**Section 2: Needs Review**
Gifts with `source = 'auto'` that should be manually verified.

**Section 3: Unrecognized Gifts**
Gift names that appear in `gift_logs` but aren't in any authoritative source (only auto-detected).

### 4.4 API Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/gift-prices` | List all gift prices (search, filter, paginate) |
| GET | `/api/gift-prices/<id>` | Get single gift price |
| POST | `/api/gift-prices/<id>` | Update a gift price |
| POST | `/api/gift-prices/bulk-recalculate` | Recalculate affected sessions after price changes |
| GET | `/api/gift-prices/audit` | Get discrepancy audit data |
| GET | `/api/gift-prices/csv` | Export all gift prices as CSV |

## 5. Recalculation Engine

When a gift's `diamond_count` is changed via the UI, the system can recalculate historical data.

### Scope of Recalculation

```
Changed gift_price.diamond_count for "闪烁星河": 1 → 99

┌─ Affected data ───────────────────────────────────────────────┐
│ gift_logs:   UPDATE diamond_total = 99 * gift_count           │
│             WHERE gift_name = '闪烁星河'                        │
│                                                               │
│ contributions:  UPDATE consume = (re-aggregate from gift_logs)│
│                WHERE session_id IN (affected session IDs)      │
│                                                               │
│ daily_stats:   UPDATE total_consume = (re-aggregate)          │
│               WHERE user_id IN (affected) AND date = ...       │
│                                                               │
│ monthly_stats: UPDATE total_consume = (re-aggregate)          │
│               WHERE user_id IN (affected) AND year_month = ... │
└───────────────────────────────────────────────────────────────┘
```

### Recalculation Flow

1. User edits gift price and clicks "Save & Recalculate"
2. System computes old vs new diamond_count difference
3. Preview: "This will affect 330 gift_logs rows across 12 sessions. Total diamond increase: +32,340"
4. User confirms
5. Transaction wraps all updates (executed via the existing single-writer queue for safe concurrency):

   ```sql
   -- Step A: Update gift_logs.diamond_total
   UPDATE gift_logs
   SET diamond_total = <new_price> * gift_count
   WHERE gift_name = '<changed_gift>'
     AND diamond_total != <new_price> * gift_count;

   -- Step B: Recompute contributions.consume for affected sessions
   -- (sum of all diamond_total per user per session)
   UPDATE contributions
   SET consume = (
       SELECT COALESCE(SUM(g.diamond_total), 0)
       FROM gift_logs g
       WHERE g.session_id = contributions.session_id
         AND g.user_id = contributions.user_id
   )
   WHERE session_id IN (
       SELECT DISTINCT session_id FROM gift_logs WHERE gift_name = '<changed_gift>'
   );

   -- Step C: Recompute daily_stats.total_consume for affected users/dates
   UPDATE daily_stats
   SET total_consume = (
       SELECT COALESCE(SUM(g.diamond_total), 0)
       FROM gift_logs g
       WHERE g.user_id = daily_stats.user_id
         AND date(g.created_at) = daily_stats.date
   )
   WHERE user_id IN (
       SELECT DISTINCT user_id FROM gift_logs WHERE gift_name = '<changed_gift>'
   );

   -- Step D: Recompute monthly_stats.total_consume (same pattern with year_month)
   UPDATE monthly_stats
   SET total_consume = (
       SELECT COALESCE(SUM(g.diamond_total), 0)
       FROM gift_logs g
       WHERE g.user_id = monthly_stats.user_id
         AND strftime('%Y-%m', g.created_at) = monthly_stats.year_month
   )
   WHERE user_id IN (
       SELECT DISTINCT user_id FROM gift_logs WHERE gift_name = '<changed_gift>'
   );
   ```

6. Log the change to a `price_change_log` table for audit

### New Table: `price_change_log`

```sql
CREATE TABLE IF NOT EXISTS price_change_log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    gift_name       TEXT NOT NULL,
    old_price       INTEGER NOT NULL,
    new_price       INTEGER NOT NULL,
    affected_rows   INTEGER DEFAULT 0,
    affected_sessions INTEGER DEFAULT 0,
    notes           TEXT DEFAULT '',
    changed_by      TEXT DEFAULT 'web_ui',
    created_at      DATETIME DEFAULT (datetime('now', '+8 hours'))
);
```

## 6. Parser Integration

### Replacement of `_GIFT_PRICE_OVERRIDE`

```python
# In-memory cache (lazy-loaded from DB)
_gift_price_cache = None   # Dict[str, int]

def _load_gift_price_cache():
    """Load all gift prices from DB into a dict, then overlay hardcoded overrides."""
    global _gift_price_cache
    cache = {}
    conn = _get_conn()
    for row in conn.execute('SELECT gift_name, diamond_count FROM gift_prices'):
        cache[row['gift_name']] = row['diamond_count']
    # Hardcoded overrides still win (safety net for gifts not yet in DB)
    cache.update(_GIFT_PRICE_OVERRIDE)
    cache.update({fb['name']: fb['diamond_count'] for fb in _GIFT_FALLBACK.values()})
    _gift_price_cache = cache

def _lookup_gift_price(gift_name, protobuf_price):
    """Look up a gift's price. Falls back to protobuf value if not in any source."""
    if _gift_price_cache is None:
        _load_gift_price_cache()
    return _gift_price_cache.get(gift_name, protobuf_price)
```

The original `_GIFT_PRICE_OVERRIDE` and `_GIFT_FALLBACK` dictionaries are **preserved** as a code-level fallback, but the primary lookup moves to the DB. This means:
- New deployments: DB is empty → hardcoded dicts are used → first-run population fills DB
- Existing deployments with data: DB is populated from dicts + auto-detection → dicts still used as fallback
- Emergency fix: edit the dict, restart, it takes precedence over DB

### Audit Logging at Parse Time

When `_lookup_gift_price` returns a price different from the protobuf's `gift.diamond_count`, log it:
```python
if unit_price != msg.gift.diamond_count:
    logger.debug(f"[礼物-定价覆盖] {gift.name}: protobuf={msg.gift.diamond_count} → DB={unit_price}")
```

## 7. Implementation Plan

### Phase 1: Backend (SQLite + Parser)
| Step | Files | Effort |
|------|-------|--------|
| 1.1 Add `init_gift_prices_table()` to `base/parser.py` | `base/parser.py` | Small |
| 1.2 Create `_load_gift_price_cache()` + `_lookup_gift_price()` | `base/parser.py` | Small |
| 1.3 Modify `parse_gift_msg` to use DB lookup | `base/parser.py` | Small |
| 1.4 Create `price_change_log` table | `base/parser.py` | Small |
| 1.5 Add recalculation functions | `base/parser.py` | Medium |

### Phase 2: Web UI
| Step | Files | Effort |
|------|-------|--------|
| 2.1 API: GET `/api/gift-prices` with search/filter/pagination | `app.py` | Medium |
| 2.2 API: POST `/api/gift-prices/<id>` | `app.py` | Small |
| 2.3 API: POST `/api/gift-prices/bulk-recalculate` | `app.py` | Medium |
| 2.4 API: GET `/api/gift-prices/audit` | `app.py` | Medium |
| 2.5 Template: Main gift-prices listing page | `templates/gift_prices.html` | Medium |
| 2.6 Template: Edit form (modal or page) | `templates/gift_prices_edit.html` | Small |
| 2.7 Template: Audit dashboard | `templates/gift_prices_audit.html` | Medium |

### Phase 3: Testing & Migration
| Step | Effort |
|------|--------|
| 3.1 Run on existing database, verify all prices populated correctly | Small |
| 3.2 Verify the 闪烁星河/点点星光 discrepancies are detected | Small |
| 3.3 Test recalculation with a price change | Small |
| 3.4 Remove/reduce hardcoded dict dependencies (future) | Small |

## 8. Future Extensions (Out of Scope)

- **Auto-discovery of limited skins**: Periodically scrape Douyin gift catalog to detect new skins
- **Bulk import from CSV**: Upload a CSV of gift_id → price mappings
- **Price change notifications**: SSE push when a gift price is updated
- **Session-level price lock**: Record which price was in effect at the time of each session
- **AI-assisted price suggestion**: Use LLM to suggest prices based on gift names and known Douyin pricing patterns

## 9. Appendix: Current System Analysis

### Existing Hardcoded Overrides

```python
_GIFT_PRICE_OVERRIDE = {
    '至尊超跑': 12000,    # verified ✅ in data (12000/unit)
    '烈焰跑车': 6000,     # verified ✅
    '无界超跑': 36000,    # not yet in collected data
    '青绿典藏版嘉年华': 35000,
    '钻石嘉年华': 36000,  # verified ✅
    '520嘉年华': 33000,
    '御风飞机': 9000,
    '凌霄战机': 18000,
    '星际战舰': 36000,
    '闪烁星河': 99,       # ❌ data shows unit_price=1 (recorded before override existed)
    '点点星光': 9,        # ❌ data shows unit_price=1
}

_GIFT_FALLBACK = {
    685:  {'name': '粉丝灯牌', 'diamond_count': 1, 'combo': False},
    3389: {'name': '欢乐盲盒', 'diamond_count': 10, 'combo': False},
    4021: {'name': '欢乐拼图', 'diamond_count': 10, 'combo': False},
}
```

### gift_logs Schema (for reference)

```sql
CREATE TABLE gift_logs (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id    INTEGER REFERENCES sessions(id),
    user_id       TEXT NOT NULL,
    user_name     TEXT NOT NULL,
    gift_name     TEXT NOT NULL,
    gift_count    INTEGER DEFAULT 1,
    diamond_total INTEGER DEFAULT 0,
    group_id      TEXT DEFAULT '',
    grade         TEXT DEFAULT '',
    fans_club     TEXT DEFAULT '',
    created_at    DATETIME DEFAULT (datetime('now', '+8 hours'))
);
```

### Data Volume (at time of writing)

| Metric | Value |
|--------|-------|
| Distinct gift names | 154 |
| Total gift_logs rows | 28,019 |
| Gifts without any override coverage | 132 (mostly low-value static gifts, but includes unknowns) |
| Price discrepancies found | 2 (闪烁星河, 点点星光) |
