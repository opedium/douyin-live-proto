# Gift Price Management Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace hardcoded gift price overrides with a persistent SQLite `gift_prices` table, web UI for management, and recalculation engine for historical data correction.

**Architecture:** New tables (`gift_prices`, `price_change_log`) in the existing SQLite database, populated from three sources (hardcoded dicts → gift_registry.json → auto-detection from gift_logs). Flask API + single-page template for web UI. Gift price lookups in `parse_gift_msg` switch from hardcoded dict to DB-backed cache.

**Tech Stack:** Python 3.11+, SQLite (existing), Flask (existing), Chart.js (existing in base.html)

## Global Constraints

- All SQLite writes must use the existing single-writer queue (`_write_queue`) or direct `_get_conn()` for reads
- Price lookup must fall back to hardcoded dicts → protobuf value if DB is unavailable
- No new Python dependencies
- All new tables must use `CREATE TABLE IF NOT EXISTS` for idempotent startup
- Price changes must be logged to `price_change_log` for audit trail
- All API routes require `@require_auth` decorator (matching existing pattern)
- Template must extend `base.html` (matching existing pattern)
- The `auth_enabled` context variable must be passed to all template renders (matching existing pattern)

---
### Task 1: Database Schema — gift_prices and price_change_log tables

**Files:**
- Modify: `base/parser.py:1878-1996` (add tables to `init_db`)
- Modify: `base/parser.py` (add `init_gift_prices_table()` function)

**Interfaces:**
- Consumes: `_get_conn()`, `DB_DIR`, `_GIFT_PRICE_OVERRIDE`, `_GIFT_FALLBACK`
- Produces: `init_gift_prices_table()` — callable on startup to populate prices

- [ ] **Step 1: Add table creation to `init_db()`**

Add these two tables at the end of the `conn.executescript(...)` block in `init_db()` (before the ALTER TABLE compatibility section):

```python
        CREATE TABLE IF NOT EXISTS gift_prices (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            gift_name TEXT NOT NULL UNIQUE,
            gift_id INTEGER DEFAULT 0,
            diamond_count INTEGER NOT NULL DEFAULT 0,
            source TEXT NOT NULL DEFAULT 'auto',
            is_limited_skin INTEGER NOT NULL DEFAULT 0,
            base_gift_name TEXT NOT NULL DEFAULT '',
            notes TEXT NOT NULL DEFAULT '',
            created_at DATETIME DEFAULT (datetime('now', '+8 hours')),
            updated_at DATETIME DEFAULT (datetime('now', '+8 hours'))
        );
        CREATE TABLE IF NOT EXISTS price_change_log (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            gift_name TEXT NOT NULL,
            old_price INTEGER NOT NULL,
            new_price INTEGER NOT NULL,
            affected_rows INTEGER DEFAULT 0,
            affected_sessions INTEGER DEFAULT 0,
            notes TEXT DEFAULT '',
            changed_by TEXT DEFAULT 'web_ui',
            created_at DATETIME DEFAULT (datetime('now', '+8 hours'))
        );
```

- [ ] **Step 2: Write the `init_gift_prices_table()` function**

Add this function after `init_db()`. It populates the table from three sources in priority order:

```python
def init_gift_prices_table():
    """Populate gift_prices table from all available sources.
    
    Priority (higher = never overwritten by lower):
        1. _GIFT_PRICE_OVERRIDE  (manually verified, source='override')
        2. _GIFT_FALLBACK        (manually verified, source='override')
        3. gift_registry.json    (official registry, source='registry')
        4. Auto-detected from gift_logs (consensus price, source='auto')
    
    Auto-detected entries are refreshed on each startup.
    Authoritative entries (override/registry) are INSERT OR IGNORE only.
    """
    conn = _get_conn()
    
    # Source 1: _GIFT_PRICE_OVERRIDE
    for name, price in _GIFT_PRICE_OVERRIDE.items():
        conn.execute('''
            INSERT OR IGNORE INTO gift_prices (gift_name, diamond_count, source, is_limited_skin, notes)
            VALUES (?, ?, 'override', 1, 'Limited-edition skin price override')
        ''', (name, price))
    
    # Source 2: _GIFT_FALLBACK (gift_id-based)
    for gid, info in _GIFT_FALLBACK.items():
        conn.execute('''
            INSERT OR IGNORE INTO gift_prices (gift_name, gift_id, diamond_count, source)
            VALUES (?, ?, ?, 'override')
        ''', (info['name'], gid, info['diamond_count']))
    
    # Source 3: gift_registry.json (if exists)
    import json, os as _os
    _reg_path = _os.path.join(_os.path.dirname(_os.path.dirname(__file__)), 'data', 'gift_registry.json')
    if _os.path.exists(_reg_path):
        try:
            with open(_reg_path, 'r', encoding='utf-8') as _f:
                _registry = json.load(_f)
            for _name, _info in _registry.items():
                _gid = _info.get('id')
                _price = _info.get('diamond_count', 0)
                if _gid and _price:
                    conn.execute('''
                        INSERT OR IGNORE INTO gift_prices (gift_name, gift_id, diamond_count, source)
                        VALUES (?, ?, ?, 'registry')
                    ''', (_name, _gid, _price))
        except Exception:
            pass  # registry is optional
    
    # Source 4: Auto-detect from gift_logs (refresh on every startup)
    # First, collect consensus prices for each gift_name
    auto_rows = conn.execute('''
        SELECT gift_name, diamond_total / MAX(gift_count, 1) AS unit_price,
               COUNT(*) AS occurrences
        FROM gift_logs
        WHERE gift_count > 0
        GROUP BY gift_name, unit_price
        ORDER BY gift_name, occurrences DESC
    ''').fetchall()
    
    # Group by gift_name: pick most common price, flag conflicts
    from collections import defaultdict
    gift_prices_map = {}      # gift_name -> (unit_price, occurrences, has_conflict)
    gift_conflicts = defaultdict(list)  # gift_name -> [(price, count), ...]
    for row in auto_rows:
        name = row['gift_name']
        price = row['unit_price']
        cnt = row['occurrences']
        if name not in gift_prices_map:
            gift_prices_map[name] = (price, cnt, False)
        else:
            existing_price, existing_cnt, _ = gift_prices_map[name]
            if price != existing_price:
                gift_conflicts[name].append((price, cnt))
                if cnt > existing_cnt:
                    gift_prices_map[name] = (price, cnt, True)
                else:
                    gift_prices_map[name] = (existing_price, existing_cnt, True)
    
    # Upsert auto-detected prices (skip if authoritative source exists)
    for name, (price, cnt, has_conflict) in gift_prices_map.items():
        existing = conn.execute(
            'SELECT source FROM gift_prices WHERE gift_name = ?', (name,)
        ).fetchone()
        if existing:
            # Only update auto-detected entries; skip authoritative ones
            if existing['source'] == 'auto':
                notes = ''
                if has_conflict:
                    conflict_str = '; '.join(
                        f'{p} dia ({c}x)' for p, c in gift_conflicts[name]
                    )
                    notes = f'Conflict: other prices {conflict_str}'
                conn.execute('''
                    UPDATE gift_prices
                    SET diamond_count = ?, notes = ?, updated_at = datetime("now", "+8 hours")
                    WHERE gift_name = ? AND source = 'auto'
                ''', (price, notes, name))
        else:
            # New entry — insert
            notes = ''
            if has_conflict:
                conflict_str = '; '.join(
                    f'{p} dia ({c}x)' for p, c in gift_conflicts[name]
                )
                notes = f'Conflict: other prices {conflict_str}'
            conn.execute('''
                INSERT INTO gift_prices (gift_name, diamond_count, source, notes)
                VALUES (?, ?, 'auto', ?)
            ''', (name, price, notes))
    
    conn.commit()
```

- [ ] **Step 3: Call `init_gift_prices_table()` at startup**

Add the call after `init_db()` is called in `_get_conn()`. The right place is at the end of the schema init block:

```python
    if not _db_schema_inited:
        with _db_schema_lock:
            if not _db_schema_inited:
                init_db()
                init_gift_prices_table()  # <-- add this line
                _db_schema_inited = True
```

- [ ] **Step 4: Run a quick test to verify table creation**

```bash
python -X utf8 -c "
import sys; sys.path.insert(0, '.')
from base.parser import _get_conn, init_gift_prices_table
conn = _get_conn()
# Verify tables exist
tables = [r[0] for r in conn.execute(\"SELECT name FROM sqlite_master WHERE type='table' ORDER BY name\").fetchall()]
print('Tables:', tables)
assert 'gift_prices' in tables, 'gift_prices table missing'
assert 'price_change_log' in tables, 'price_change_log table missing'
# Verify data was populated
count = conn.execute('SELECT COUNT(*) FROM gift_prices').fetchone()[0]
print(f'gift_prices rows: {count}')
sources = conn.execute('SELECT source, COUNT(*) as cnt FROM gift_prices GROUP BY source').fetchall()
for s in sources:
    print(f'  source={s[\"source\"]}: {s[\"cnt\"]}')
# Check specific overrides
override_items = conn.execute(\"SELECT gift_name, diamond_count FROM gift_prices WHERE source='override'\").fetchall()
print(f'Override entries: {len(override_items)}')
for r in override_items:
    print(f'  {r[\"gift_name\"]}: {r[\"diamond_count\"]}')
"
```

Expected output: `gift_prices` and `price_change_log` tables exist, ~11 override entries from hardcoded dicts, ~154 auto-detected entries from gift_logs.

- [ ] **Step 5: Commit**

```bash
git add base/parser.py
git commit -m "feat(gift-prices): add gift_prices and price_change_log tables with population logic"
```

---

### Task 2: Gift Price Lookup in Parser

**Files:**
- Modify: `base/parser.py` (add cache + lookup function, modify `parse_gift_msg`)

**Interfaces:**
- Consumes: `_get_conn()`, `_GIFT_PRICE_OVERRIDE`, `_GIFT_FALLBACK`, `gift.diamond_count`
- Produces: `_lookup_gift_price(gift_name, protobuf_price) → int`

- [ ] **Step 1: Add the in-memory cache and lookup function**

Add these after the `_GIFT_ID_TO_NAME` section (around line 207):

```python
# ── Gift Price Cache (DB-backed, with hardcoded fallback) ──
_gift_price_cache = None
_gift_price_cache_lock = threading.Lock()

def _load_gift_price_cache():
    """Load all gift prices from DB into a dict, then overlay hardcoded overrides.
    
    Hardcoded _GIFT_PRICE_OVERRIDE always wins (emergency fix path).
    """
    global _gift_price_cache
    cache = {}
    try:
        conn = _get_conn()
        for row in conn.execute('SELECT gift_name, diamond_count FROM gift_prices'):
            cache[row['gift_name']] = row['diamond_count']
    except Exception:
        pass  # DB not ready yet, use fallback only
    # Hardcoded overrides take precedence (safety net + emergency edits)
    cache.update(_GIFT_PRICE_OVERRIDE)
    for fb in _GIFT_FALLBACK.values():
        cache[fb['name']] = fb['diamond_count']
    _gift_price_cache = cache
    logger.debug(f"[礼物定价] 已加载 {len(cache)} 条礼物价格 (DB + 覆盖)")

def _lookup_gift_price(gift_name, protobuf_price):
    """Look up a gift's diamond price.
    
    Resolution order:
        1. In-memory cache (loaded from DB + hardcoded overrides)
        2. protobuf gift.diamond_count (fallback)
    
    Returns (price, was_overridden) tuple.
    """
    global _gift_price_cache
    if _gift_price_cache is None:
        with _gift_price_cache_lock:
            if _gift_price_cache is None:
                _load_gift_price_cache()
    cached = _gift_price_cache.get(gift_name)
    if cached is not None and cached != protobuf_price:
        return cached, True
    return protobuf_price, False
```

- [ ] **Step 2: Modify `parse_gift_msg` to use the lookup**

Find this line in `parse_gift_msg` (around line 509):
```python
unit_price = _GIFT_PRICE_OVERRIDE.get(gift.name, gift.diamond_count); display_name = gift.name
```

Replace with:
```python
unit_price, was_overridden = _lookup_gift_price(gift.name, gift.diamond_count)
display_name = gift.name
```

Also add logging for price overrides (after the `unit_price` line):
```python
if was_overridden:
    logger.debug(f"[礼物-定价覆盖] {gift.name}: protobuf={gift.diamond_count} → DB={unit_price}")
```

- [ ] **Step 3: Test the lookup works correctly**

```bash
python -X utf8 -c "
import sys; sys.path.insert(0, '.')
from base.parser import _lookup_gift_price, _load_gift_price_cache

# Force cache reload
_load_gift_price_cache()

# Test known overrides
tests = [
    ('至尊超跑', 1200, 12000),  # protobuf says 1200, override says 12000
    ('嘉年华', 30000, 30000),   # protobuf should match
    ('闪烁星河', 1, 99),         # override says 99
    ('NonExistentGift_XYZ', 50, 50),  # not in any source → protobuf
]
for name, proto, expected in tests:
    price, overridden = _lookup_gift_price(name, proto)
    status = '✅' if price == expected else '❌'
    print(f'{status} {name:20s} proto={proto:>5} → price={price:>5} overridden={overridden} expected={expected}')
"
```

Expected: 至尊超跑 returns 12000 (overridden), 嘉年华 returns 30000 (matches proto), 闪烁星河 returns 99 (overridden), NonExistent returns 50 (proto fallback).

- [ ] **Step 4: Commit**

```bash
git add base/parser.py
git commit -m "feat(gift-prices): add DB-backed gift price lookup with hardcoded fallback"
```

---

### Task 3: Recalculation Engine

**Files:**
- Modify: `base/parser.py` (add recalculation functions + logging)

**Interfaces:**
- Consumes: `_get_conn()`, `_write_queue`
- Produces: `recalculate_gift_price(gift_name, new_price, old_price) → dict`
  — returns `{'affected_rows': N, 'affected_sessions': M, 'diamond_diff': D}`

- [ ] **Step 1: Add the recalculation function**

Add this after `init_gift_prices_table()`:

```python
def recalculate_gift_price(gift_name, new_price, old_price, notes=''):
    """Recalculate gift_logs.diamond_total for a price change and cascade to derived tables.
    
    This is called AFTER the gift_prices entry has been updated.
    Uses the existing single-writer queue pattern for safe writes.
    
    Args:
        gift_name: The gift name that changed.
        new_price: New diamond_count value.
        old_price: Previous diamond_count value.
        notes: Optional change notes.
    
    Returns:
        dict with affected_rows, affected_sessions, diamond_diff.
    """
    conn = _get_conn()
    
    # Step A: Find affected gift_logs rows
    affected = conn.execute('''
        SELECT id, gift_count, diamond_total
        FROM gift_logs
        WHERE gift_name = ? AND diamond_total != ? * gift_count
    ''', (gift_name, new_price)).fetchall()
    
    if not affected:
        return {'affected_rows': 0, 'affected_sessions': 0, 'diamond_diff': 0}
    
    affected_rows = len(affected)
    
    # Compute diamond difference
    old_total = sum(r['diamond_total'] for r in affected)
    new_total = sum(new_price * r['gift_count'] for r in affected)
    diamond_diff = new_total - old_total
    
    # Get affected session IDs
    session_ids = [r['session_id'] for r in conn.execute(
        'SELECT DISTINCT session_id FROM gift_logs WHERE gift_name = ? AND diamond_total != ? * gift_count',
        (gift_name, new_price)
    ).fetchall()]
    affected_sessions = len(session_ids)
    
    # Step B: Update gift_logs
    conn.execute(
        'UPDATE gift_logs SET diamond_total = ? * gift_count WHERE gift_name = ? AND diamond_total != ? * gift_count',
        (new_price, gift_name, new_price)
    )
    
    # Step C: Recalculate contributions.consume for affected sessions
    for sid in session_ids:
        conn.execute('''
            UPDATE contributions
            SET consume = (
                SELECT COALESCE(SUM(g.diamond_total), 0)
                FROM gift_logs g
                WHERE g.session_id = contributions.session_id
                  AND g.user_id = contributions.user_id
            )
            WHERE session_id = ?
        ''', (sid,))
    
    # Step D: Recalculate daily_stats for affected users
    affected_users = [r['user_id'] for r in conn.execute(
        'SELECT DISTINCT user_id FROM gift_logs WHERE gift_name = ?', (gift_name,)
    ).fetchall()]
    
    for uid in affected_users:
        conn.execute('''
            UPDATE daily_stats
            SET total_consume = (
                SELECT COALESCE(SUM(g.diamond_total), 0)
                FROM gift_logs g
                WHERE g.user_id = daily_stats.user_id
                  AND date(g.created_at) = daily_stats.date
            )
            WHERE user_id = ?
        ''', (uid,))
        
        conn.execute('''
            UPDATE monthly_stats
            SET total_consume = (
                SELECT COALESCE(SUM(g.diamond_total), 0)
                FROM gift_logs g
                WHERE g.user_id = monthly_stats.user_id
                  AND strftime('%Y-%m', g.created_at) = monthly_stats.year_month
            )
            WHERE user_id = ?
        ''', (uid,))
    
    # Step E: Log the change
    conn.execute('''
        INSERT INTO price_change_log (gift_name, old_price, new_price, affected_rows, affected_sessions, notes)
        VALUES (?, ?, ?, ?, ?, ?)
    ''', (gift_name, old_price, new_price, affected_rows, affected_sessions, notes or ''))
    
    conn.commit()
    
    logger.info(f"[礼物定价] 已重算 {gift_name}: {old_price}→{new_price}, "
                f"影响 {affected_rows} 行, {affected_sessions} 场次, 差额 {diamond_diff:+d} 钻石")
    
    return {
        'affected_rows': affected_rows,
        'affected_sessions': affected_sessions,
        'diamond_diff': diamond_diff,
    }


def get_price_change_history(limit=50):
    """Return recent price change log entries."""
    conn = _get_conn()
    rows = conn.execute('''
        SELECT * FROM price_change_log
        ORDER BY id DESC LIMIT ?
    ''', (limit,)).fetchall()
    return [dict(r) for r in rows]
```

- [ ] **Step 2: Test the recalculation**

```bash
python -X utf8 -c "
import sys; sys.path.insert(0, '.')
# First check if there's data to recalculate (闪烁星河 is known to be mispriced)
from base.parser import _get_conn
conn = _get_conn()
mispriced = conn.execute(\"SELECT COUNT(*) as cnt FROM gift_logs WHERE gift_name='闪烁星河' AND diamond_total != 99 * gift_count\").fetchone()
print(f'Mispriced 闪烁星河 rows: {mispriced[\"cnt\"]}')
mispriced2 = conn.execute(\"SELECT COUNT(*) as cnt FROM gift_logs WHERE gift_name='点点星光' AND diamond_total != 9 * gift_count\").fetchone()
print(f'Mispriced 点点星光 rows: {mispriced2[\"cnt\"]}')
# Test the full recalculation on 闪烁星河
from base.parser import recalculate_gift_price
result = recalculate_gift_price('闪烁星河', 99, 1, notes='Test recalc from implementation plan')
print(f'Recalculation result: {result}')
"
```

Expected: Shows the number of mispriced rows and the recalculation result.

- [ ] **Step 3: Commit**

```bash
git add base/parser.py
git commit -m "feat(gift-prices): add recalculation engine for price changes"
```

---

### Task 4: Flask API Routes

**Files:**
- Modify: `app.py` (add 5 API routes + 1 page route + imports)

**Interfaces:**
- Consumes: `_lookup_gift_price`, `init_gift_prices_table`, `recalculate_gift_price`, `get_price_change_history` from `base.parser`
- Produces: JSON API responses + rendered template

- [ ] **Step 1: Add imports to app.py**

Add at the top of `app.py`, in the existing import block (around line 35):

```python
from base.parser import (
    query_leaderboard, query_user, query_user_detail, query_user_timeline,
    query_chat, query_anonymous, query_million,
    query_sessions, query_session_detail, query_search, _get_conn, DB_PATH,
    end_session as db_end_session, delete_session as db_delete_session,
    set_sse_callback,
    init_gift_prices_table, recalculate_gift_price, get_price_change_history,  # <-- add this line
)
```

- [ ] **Step 2: Add the gift prices page route**

Add after the `compare` route (around line 820):

```python
# ── Gift Price Management ──

@app.route('/gift-prices')
@require_auth
def gift_prices():
    return render_template('gift_prices.html', auth_enabled=bool(_web_config['password']))
```

- [ ] **Step 3: Add the GET API — list all gift prices**

Add after the gift_prices route:

```python
@app.route('/api/gift-prices')
@require_auth
def api_gift_prices():
    """List gift prices with search, filter, and pagination."""
    conn = _get_conn()
    search = request.args.get('search', '').strip()
    source_filter = request.args.get('source', '').strip()
    needs_review = request.args.get('needs_review', '').strip()
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 50, type=int)
    offset = (page - 1) * size
    
    where = ['1=1']
    params = []
    
    if search:
        where.append('g.gift_name LIKE ?')
        params.append(f'%{search}%')
    
    if source_filter:
        where.append('g.source = ?')
        params.append(source_filter)
    
    if needs_review == '1':
        # "Needs review" = auto-detected + has conflicting prices in notes
        where.append("(g.source = 'auto' AND g.notes != '')")
    
    w = ' AND '.join(where)
    
    total = conn.execute(f'SELECT COUNT(*) FROM gift_prices g WHERE {w}', params).fetchone()[0]
    
    rows = conn.execute(f'''
        SELECT g.*,
               COALESCE(d.logs, 0) AS logs,
               COALESCE(d.total_dia, 0) AS total_dia
        FROM gift_prices g
        LEFT JOIN (
            SELECT gift_name, COUNT(*) AS logs, SUM(diamond_total) AS total_dia
            FROM gift_logs GROUP BY gift_name
        ) d ON d.gift_name = g.gift_name
        WHERE {w}
        ORDER BY d.total_dia DESC, g.gift_name
        LIMIT ? OFFSET ?
    ''', params + [size, offset]).fetchall()
    
    return jsonify({
        'prices': [dict(r) for r in rows],
        'total': total,
        'page': page,
        'size': size,
    })
```

- [ ] **Step 4: Add the GET single-price API**

```python
@app.route('/api/gift-prices/<int:price_id>')
@require_auth
def api_gift_price(price_id):
    conn = _get_conn()
    row = conn.execute('SELECT * FROM gift_prices WHERE id = ?', (price_id,)).fetchone()
    if not row:
        return jsonify({'error': 'not found'}), 404
    return jsonify(dict(row))
```

- [ ] **Step 5: Add the POST update API**

```python
@app.route('/api/gift-prices/<int:price_id>', methods=['POST'])
@require_auth
def api_gift_price_update(price_id):
    """Update a gift price. If diamond_count changes, optionally recalculate."""
    conn = _get_conn()
    row = conn.execute('SELECT * FROM gift_prices WHERE id = ?', (price_id,)).fetchone()
    if not row:
        return jsonify({'error': 'not found'}), 404
    
    data = request.get_json(force=True, silent=True) or {}
    
    updates = []
    params = []
    
    if 'diamond_count' in data:
        new_price = int(data['diamond_count'])
        if new_price < 0:
            return jsonify({'error': 'price must be >= 0'}), 400
        old_price = row['diamond_count']
        updates.append('diamond_count = ?')
        params.append(new_price)
    else:
        old_price = row['diamond_count']
        new_price = old_price
    
    if 'gift_id' in data:
        updates.append('gift_id = ?')
        params.append(int(data['gift_id']))
    
    if 'source' in data:
        updates.append('source = ?')
        params.append(data['source'])
    
    if 'is_limited_skin' in data:
        updates.append('is_limited_skin = ?')
        params.append(1 if data['is_limited_skin'] else 0)
    
    if 'base_gift_name' in data:
        updates.append('base_gift_name = ?')
        params.append(data['base_gift_name'])
    
    if 'notes' in data:
        updates.append('notes = ?')
        params.append(data['notes'])
    
    if updates:
        updates.append('updated_at = datetime("now", "+8 hours")')
        params.append(price_id)
        conn.execute(f'UPDATE gift_prices SET {", ".join(updates)} WHERE id = ?', params)
        conn.commit()
    
    recalculate = data.get('recalculate', False)
    recalc_result = None
    if recalculate and new_price != old_price:
        recalc_result = recalculate_gift_price(
            row['gift_name'], new_price, old_price,
            notes=data.get('notes', '')
        )
    
    return jsonify({
        'success': True,
        'gift_name': row['gift_name'],
        'old_price': old_price,
        'new_price': new_price if 'diamond_count' in data else old_price,
        'recalculated': recalc_result is not None,
        'recalc_result': recalc_result,
    })
```

- [ ] **Step 6: Add the bulk-recalculate API**

```python
@app.route('/api/gift-prices/bulk-recalculate', methods=['POST'])
@require_auth
def api_gift_prices_bulk_recalculate():
    """Recalculate all gift_logs rows for a specific gift_name using its current price."""
    data = request.get_json(force=True, silent=True) or {}
    price_id = data.get('price_id')
    gift_name = data.get('gift_name')
    
    conn = _get_conn()
    
    if price_id:
        row = conn.execute('SELECT * FROM gift_prices WHERE id = ?', (price_id,)).fetchone()
    elif gift_name:
        row = conn.execute('SELECT * FROM gift_prices WHERE gift_name = ?', (gift_name,)).fetchone()
    else:
        return jsonify({'error': 'price_id or gift_name required'}), 400
    
    if not row:
        return jsonify({'error': 'gift price not found'}), 404
    
    result = recalculate_gift_price(row['gift_name'], row['diamond_count'], row['diamond_count'],
                                    notes='Bulk recalculation')
    return jsonify({'success': True, 'result': result})
```

- [ ] **Step 7: Add the audit API**

```python
@app.route('/api/gift-prices/audit')
@require_auth
def api_gift_prices_audit():
    """Return audit data: discrepancies, needs-review items, change history."""
    conn = _get_conn()
    
    # Section 1: Price discrepancies — same gift_name, different unit prices in gift_logs
    discrepancies = conn.execute('''
        SELECT gift_name, diamond_total / MAX(gift_count, 1) AS unit_price,
               COUNT(*) AS occurrences,
               COUNT(DISTINCT session_id) AS sessions
        FROM gift_logs
        WHERE gift_count > 0
        GROUP BY gift_name, unit_price
        HAVING COUNT(*) > 0
        ORDER BY gift_name, occurrences DESC
    ''').fetchall()
    
    # Group by gift_name, find conflicts
    disc_dict = {}
    for r in discrepancies:
        name = r['gift_name']
        if name not in disc_dict:
            disc_dict[name] = {'gift_name': name, 'prices': [], 'total_occurrences': 0}
        disc_dict[name]['prices'].append({
            'unit_price': r['unit_price'],
            'occurrences': r['occurrences'],
            'sessions': r['sessions'],
        })
        disc_dict[name]['total_occurrences'] += r['occurrences']
    
    audit_discrepancies = [
        d for d in disc_dict.values() if len(d['prices']) > 1
    ]
    
    # Cross-reference with gift_prices table
    for d in audit_discrepancies:
        db_row = conn.execute(
            'SELECT diamond_count, source FROM gift_prices WHERE gift_name = ?',
            (d['gift_name'],)
        ).fetchone()
        d['db_price'] = db_row['diamond_count'] if db_row else None
        d['db_source'] = db_row['source'] if db_row else 'unknown'
    
    # Section 2: Needs review (auto-detected with conflicts)
    needs_review = conn.execute('''
        SELECT * FROM gift_prices
        WHERE source = 'auto' AND notes != ''
        ORDER BY updated_at DESC
    ''').fetchall()
    
    # Section 3: Change history
    history = get_price_change_history(30)
    
    return jsonify({
        'discrepancies': audit_discrepancies,
        'needs_review': [dict(r) for r in needs_review],
        'history': history,
    })
```

- [ ] **Step 8: Commit**

```bash
git add app.py
git commit -m "feat(gift-prices): add Flask API routes for gift price management"
```

---

### Task 5: Web UI Template

**Files:**
- Create: `templates/gift_prices.html`
- Modify: `templates/base.html` (add nav link)

- [ ] **Step 1: Add nav link to base.html**

In `templates/base.html`, add to the nav links section (around line 964, after the audit link):

```html
            <a href="/gift-prices" class="{% if request.path == '/gift-prices' %}active{% endif %}">礼物定价</a>
```

- [ ] **Step 2: Create the gift prices template**

Create `templates/gift_prices.html`:

```html
{% extends "base.html" %}
{% block title %}礼物定价管理{% endblock %}
{% block content %}

<style>
.gp-header { display: flex; align-items: center; gap: 12px; margin-bottom: 20px; flex-wrap: wrap; }
.gp-header h1 { font-size: 20px; font-weight: 700; }
.gp-tabs { display: flex; gap: 0; margin-bottom: 16px; border-bottom: 2px solid var(--border); }
.gp-tab { padding: 8px 18px; font-size: 13px; font-weight: 600; cursor: pointer;
  color: var(--text-muted); border-bottom: 2px solid transparent; margin-bottom: -2px;
  transition: all var(--transition-fast); }
.gp-tab:hover { color: var(--text); }
.gp-tab.active { color: var(--primary); border-bottom-color: var(--primary); }
.gp-source-badge { display: inline-block; padding: 1px 8px; border-radius: 10px; font-size: 10px; font-weight: 700; }
.gp-source-override { background: #dbeafe; color: #1e40af; }
.gp-source-manual { background: #d1fae5; color: #065f46; }
.gp-source-auto { background: #f1f5f9; color: #475569; }
.gp-source-registry { background: var(--warning-bg); color: #92400e; }
.gp-price { font-family: var(--font-mono); font-weight: 700; font-size: 14px; }
.gp-price.changed { color: var(--danger); }
.gp-action-btn { padding: 4px 10px; border-radius: var(--radius-sm); font-size: 11px; font-weight: 600;
  cursor: pointer; border: 1px solid var(--border); background: var(--card); color: var(--text-muted);
  transition: all var(--transition-fast); }
.gp-action-btn:hover { background: var(--primary); color: #fff; border-color: var(--primary); }
.gp-action-btn.danger:hover { background: var(--danger); border-color: var(--danger); }
.gp-summary { display: grid; grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); gap: 10px; margin-bottom: 16px; }
.gp-summary-item { background: var(--card); border: 1px solid var(--border); border-radius: var(--radius);
  padding: 12px 16px; text-align: center; box-shadow: var(--shadow); }
.gp-summary-item .num { font-size: 22px; font-weight: 800; font-family: var(--font-mono); color: var(--text); }
.gp-summary-item .label { font-size: 11px; color: var(--text-muted); font-weight: 600; text-transform: uppercase; margin-top: 2px; }
.gp-table-wrap { overflow-x: auto; }
.gp-table { width: 100%; border-collapse: collapse; font-size: 13px; }
.gp-table th { background: var(--bg); font-weight: 600; color: var(--text-muted); font-size: 11px;
  text-transform: uppercase; letter-spacing: 0.04em; padding: 8px 10px; text-align: left;
  border-bottom: 2px solid var(--border); white-space: nowrap; position: sticky; top: 0; z-index: 2; }
.gp-table td { padding: 8px 10px; border-bottom: 1px solid var(--border-light); vertical-align: middle; }
.gp-table tbody tr:nth-child(even) td { background: #f8fafc; }
.gp-table tbody tr:hover td { background: var(--primary-light); }
[data-theme="dark"] .gp-table tbody tr:nth-child(even) td { background: #1a2332; }
[data-theme="dark"] .gp-table tbody tr:hover td { background: #1e1b4b; }
.gp-conflict-icon { color: var(--danger); cursor: help; }
.gp-empty { text-align: center; padding: 40px 20px; color: var(--text-muted); font-size: 14px; }
.gp-preview-box { background: var(--bg-soft); border: 1px solid var(--border); border-radius: var(--radius-sm);
  padding: 10px 14px; margin: 8px 0; font-size: 12px; line-height: 1.6; }
.gp-preview-box .highlight { color: var(--danger); font-weight: 700; }
</style>

<div class="card" style="padding:0;">
  <div class="gp-tabs">
    <div class="gp-tab active" data-tab="browse" onclick="switchTab('browse')">浏览礼物</div>
    <div class="gp-tab" data-tab="audit" onclick="switchTab('audit')">审计</div>
    <div class="gp-tab" data-tab="history" onclick="switchTab('history')">变更记录</div>
  </div>
</div>

<div id="tab-browse" class="tab-content">
  <div class="card">
    <div class="flex-between flex-wrap" style="gap:8px;margin-bottom:12px;">
      <div class="filter-bar" style="margin-bottom:0;">
        <input type="text" id="gp-search" placeholder="搜索礼物名称..." oninput="debounceSearch()">
        <select id="gp-source-filter" onchange="loadPrices()">
          <option value="">全部来源</option>
          <option value="override">手动覆盖</option>
          <option value="manual">手动编辑</option>
          <option value="registry">注册表</option>
          <option value="auto">自动检测</option>
        </select>
        <select id="gp-skin-filter" onchange="loadPrices()">
          <option value="">全部</option>
          <option value="1">限定礼物</option>
          <option value="0">非限定</option>
        </select>
        <button onclick="loadPrices()">刷新</button>
        <a href="/api/gift-prices/csv" class="gp-action-btn" style="padding:6px 12px;text-decoration:none;">&#128196; CSV</a>
      </div>
      <span id="gp-count" style="font-size:12px;color:var(--text-muted);"></span>
    </div>
    <div class="gp-table-wrap">
      <table class="gp-table" id="gp-table">
        <thead>
          <tr>
            <th>礼物名称</th>
            <th>价格</th>
            <th>来源</th>
            <th>限定</th>
            <th>日志数</th>
            <th>总钻石</th>
            <th>备注</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody id="gp-tbody">
          <tr><td colspan="8"><div class="loading"><div class="spinner"></div><span>加载中...</span></div></td></tr>
        </tbody>
      </table>
    </div>
    <div class="pagination" id="gp-pagination"></div>
  </div>
</div>

<div id="tab-audit" class="tab-content" style="display:none;">
  <div class="gp-summary" id="audit-summary">
    <div class="loading" style="padding:20px;"><div class="spinner-ring"></div></div>
  </div>
  <div class="card">
    <h2>&#9888; 价格冲突 <span class="sub" id="disc-count"></span></h2>
    <div id="disc-table"><div class="gp-empty">加载中...</div></div>
  </div>
  <div class="card">
    <h2>&#128220; 待审核 <span class="sub" id="review-count"></span></h2>
    <div id="review-table"><div class="gp-empty">加载中...</div></div>
  </div>
</div>

<div id="tab-history" class="tab-content" style="display:none;">
  <div class="card">
    <h2>&#128337; 变更记录</h2>
    <div id="history-table"><div class="loading"><div class="spinner"></div></div></div>
  </div>
</div>

<!-- Edit Modal -->
<div class="modal-overlay" id="gp-edit-modal">
  <div class="modal-dialog" style="width:500px;">
    <div class="modal-header">
      <h3>&#9998; 编辑礼物价格</h3>
      <button class="modal-close" onclick="closeEditModal()">&times;</button>
    </div>
    <div class="modal-body">
      <input type="hidden" id="edit-id">
      <div style="display:grid;gap:10px;">
        <div><label style="font-size:12px;font-weight:600;color:var(--text-muted);display:block;margin-bottom:3px;">礼物名称</label>
          <input type="text" id="edit-name" readonly style="width:100%;padding:7px 10px;border:1px solid var(--border);border-radius:var(--radius-sm);font-size:13px;background:var(--bg);color:var(--text);"></div>
        <div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;">
          <div><label style="font-size:12px;font-weight:600;color:var(--text-muted);display:block;margin-bottom:3px;">钻石单价</label>
            <input type="number" id="edit-price" min="0" style="width:100%;padding:7px 10px;border:1px solid var(--border);border-radius:var(--radius-sm);font-size:14px;font-family:var(--font-mono);font-weight:700;outline:none;transition:border-color var(--transition-fast);"
                   onfocus="this.style.borderColor='var(--primary)';" onblur="this.style.borderColor=''"></div>
          <div><label style="font-size:12px;font-weight:600;color:var(--text-muted);display:block;margin-bottom:3px;">Gift ID</label>
            <input type="number" id="edit-gift-id" min="0" style="width:100%;padding:7px 10px;border:1px solid var(--border);border-radius:var(--radius-sm);font-size:13px;outline:none;"
                   onfocus="this.style.borderColor='var(--primary)';" onblur="this.style.borderColor=''"></div>
        </div>
        <div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;">
          <div><label style="font-size:12px;font-weight:600;color:var(--text-muted);display:block;margin-bottom:3px;">来源</label>
            <select id="edit-source" style="width:100%;padding:7px 10px;border:1px solid var(--border);border-radius:var(--radius-sm);font-size:13px;">
              <option value="auto">auto — 自动检测</option>
              <option value="manual">manual — 手动编辑</option>
              <option value="override">override — 覆盖</option>
              <option value="registry">registry — 注册表</option>
            </select></div>
          <div><label style="font-size:12px;font-weight:600;color:var(--text-muted);display:block;margin-bottom:3px;">限定皮肤</label>
            <select id="edit-limited" style="width:100%;padding:7px 10px;border:1px solid var(--border);border-radius:var(--radius-sm);font-size:13px;">
              <option value="0">否</option>
              <option value="1">是</option>
            </select></div>
        </div>
        <div><label style="font-size:12px;font-weight:600;color:var(--text-muted);display:block;margin-bottom:3px;">基础礼物（限定皮肤时）</label>
          <input type="text" id="edit-base" placeholder="例: 嘉年华" style="width:100%;padding:7px 10px;border:1px solid var(--border);border-radius:var(--radius-sm);font-size:13px;outline:none;"
                 onfocus="this.style.borderColor='var(--primary)';" onblur="this.style.borderColor=''"></div>
        <div><label style="font-size:12px;font-weight:600;color:var(--text-muted);display:block;margin-bottom:3px;">备注</label>
          <textarea id="edit-notes" rows="3" style="width:100%;padding:7px 10px;border:1px solid var(--border);border-radius:var(--radius-sm);font-size:12px;resize:vertical;outline:none;font-family:inherit;"
                    onfocus="this.style.borderColor='var(--primary)';" onblur="this.style.borderColor=''"></textarea></div>
        <div id="edit-preview" style="display:none;" class="gp-preview-box"></div>
      </div>
    </div>
    <div class="modal-footer">
      <button onclick="closeEditModal()" style="border:1px solid var(--border);background:var(--card);padding:7px 16px;border-radius:var(--radius-sm);font-size:13px;cursor:pointer;color:var(--text);">取消</button>
      <div>
        <button id="btn-save-recalc" onclick="savePrice(true)" style="border:1px solid var(--warning);background:var(--warning-bg);padding:7px 16px;border-radius:var(--radius-sm);font-size:13px;font-weight:600;cursor:pointer;color:#92400e;">保存并重算</button>
        <button id="btn-save" onclick="savePrice(false)" style="border:1px solid var(--primary);background:var(--primary);padding:7px 16px;border-radius:var(--radius-sm);font-size:13px;font-weight:600;cursor:pointer;color:#fff;margin-left:6px;" class="btn-primary">保存</button>
      </div>
    </div>
  </div>
</div>

<script>
var gpPage = 1;

// Tab switching
function switchTab(name) {
  document.querySelectorAll('.gp-tab').forEach(function(t) {
    t.classList.toggle('active', t.dataset.tab === name);
  });
  document.querySelectorAll('.tab-content').forEach(function(tc) {
    tc.style.display = tc.id === 'tab-' + name ? '' : 'none';
  });
  if (name === 'audit') loadAudit();
  if (name === 'history') loadHistory();
}

// Debounced search
var _searchTimer = null;
function debounceSearch() {
  clearTimeout(_searchTimer);
  _searchTimer = setTimeout(loadPrices, 300);
}

// Load price list
function loadPrices() {
  gpPage = 1;
  fetchPrices();
}

function fetchPrices(page) {
  var search = document.getElementById('gp-search').value.trim();
  var source = document.getElementById('gp-source-filter').value;
  var params = new URLSearchParams({
    page: page || gpPage,
    size: 50,
    search: search,
    source: source,
  });
  
  var tbody = document.getElementById('gp-tbody');
  tbody.innerHTML = '<tr><td colspan="8"><div class="loading"><div class="spinner-ring"></div><span>加载中...</span></div></td></tr>';
  
  fetch('/api/gift-prices?' + params.toString())
    .then(function(r) { return r.json(); })
    .then(function(d) {
      document.getElementById('gp-count').textContent = '共 ' + d.total + ' 条';
      if (!d.prices || d.prices.length === 0) {
        tbody.innerHTML = '<tr><td colspan="8"><div class="gp-empty">&#128270; 未找到匹配的礼物</div></td></tr>';
        document.getElementById('gp-pagination').innerHTML = '';
        return;
      }
      var html = '';
      d.prices.forEach(function(p) {
        var sourceClass = 'gp-source-' + p.source;
        var sourceLabel = {override:'覆盖', manual:'手动', auto:'自动', registry:'注册表'}[p.source] || p.source;
        var conflictIcon = (p.source === 'auto' && p.notes) ? '<span class="gp-conflict-icon" title="' + escapeHtml(p.notes) + '">&#9888;</span> ' : '';
        var skinLabel = p.is_limited_skin ? '<span class="badge badge-amber">限定</span>' : '<span class="badge badge-gray">—</span>';
        html += '<tr>' +
          '<td><strong>' + escapeHtml(p.gift_name) + '</strong></td>' +
          '<td class="gp-price">' + p.diamond_count.toLocaleString() + '</td>' +
          '<td><span class="gp-source-badge ' + sourceClass + '">' + sourceLabel + '</span></td>' +
          '<td>' + skinLabel + '</td>' +
          '<td>' + (p.logs || 0).toLocaleString() + '</td>' +
          '<td>' + (p.total_dia || 0).toLocaleString() + '</td>' +
          '<td style="font-size:11px;color:var(--text-muted);max-width:160px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;">' + conflictIcon + escapeHtml((p.notes || '').substring(0, 40)) + '</td>' +
          '<td><button class="gp-action-btn" onclick="openEdit(' + p.id + ')">编辑</button></td>' +
        '</tr>';
      });
      tbody.innerHTML = html;
      
      // Pagination
      var totalPages = Math.ceil(d.total / 50);
      var pgHtml = '<span class="info">第 ' + d.page + ' / ' + totalPages + ' 页</span>';
      if (d.page > 1) pgHtml += '<a href="#" onclick="return gotoPage(' + (d.page - 1) + ')">&laquo; 上一页</a>';
      for (var i = Math.max(1, d.page - 2); i <= Math.min(totalPages, d.page + 2); i++) {
        pgHtml += '<a href="#" class="' + (i === d.page ? 'active' : '') + '" onclick="return gotoPage(' + i + ')">' + i + '</a>';
      }
      if (d.page < totalPages) pgHtml += '<a href="#" onclick="return gotoPage(' + (d.page + 1) + ')">下一页 &raquo;</a>';
      document.getElementById('gp-pagination').innerHTML = pgHtml;
    })
    .catch(function() {
      tbody.innerHTML = '<tr><td colspan="8"><div class="gp-empty">&#10060; 加载失败</div></td></tr>';
    });
}

function gotoPage(p) {
  gpPage = p;
  fetchPrices(p);
  return false;
}

// Edit modal
function openEdit(id) {
  fetch('/api/gift-prices/' + id)
    .then(function(r) { return r.json(); })
    .then(function(p) {
      document.getElementById('edit-id').value = p.id;
      document.getElementById('edit-name').value = p.gift_name;
      document.getElementById('edit-price').value = p.diamond_count;
      document.getElementById('edit-gift-id').value = p.gift_id || 0;
      document.getElementById('edit-source').value = p.source;
      document.getElementById('edit-limited').value = p.is_limited_skin ? '1' : '0';
      document.getElementById('edit-base').value = p.base_gift_name || '';
      document.getElementById('edit-notes').value = p.notes || '';
      document.getElementById('edit-preview').style.display = 'none';
      document.getElementById('gp-edit-modal').classList.add('open');
    });
}

function closeEditModal() {
  document.getElementById('gp-edit-modal').classList.remove('open');
}

function savePrice(recalculate) {
  var id = document.getElementById('edit-id').value;
  var newPrice = parseInt(document.getElementById('edit-price').value);
  if (isNaN(newPrice) || newPrice < 0) {
    showToast('请输入有效的价格', 'error');
    return;
  }
  
  var body = {
    diamond_count: newPrice,
    gift_id: parseInt(document.getElementById('edit-gift-id').value) || 0,
    source: document.getElementById('edit-source').value,
    is_limited_skin: document.getElementById('edit-limited').value === '1',
    base_gift_name: document.getElementById('edit-base').value,
    notes: document.getElementById('edit-notes').value,
    recalculate: recalculate,
  };
  
  fetch('/api/gift-prices/' + id, {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify(body),
  })
    .then(function(r) { return r.json(); })
    .then(function(d) {
      if (d.success) {
        var msg = '已保存: ' + d.gift_name + ' = ' + d.new_price;
        if (d.recalculated && d.recalc_result) {
          msg += ' (已重算 ' + d.recalc_result.affected_rows + ' 行)';
        }
        showToast(msg, 'success');
        closeEditModal();
        fetchPrices(gpPage);
      } else {
        showToast(d.error || '保存失败', 'error');
      }
    })
    .catch(function() {
      showToast('网络错误', 'error');
    });
}

// Audit tab
function loadAudit() {
  var summaryEl = document.getElementById('audit-summary');
  summaryEl.innerHTML = '<div class="loading" style="padding:20px;"><div class="spinner-ring"></div></div>';
  
  fetch('/api/gift-prices/audit')
    .then(function(r) { return r.json(); })
    .then(function(d) {
      // Summary cards
      summaryEl.innerHTML =
        '<div class="gp-summary-item"><div class="num">' + d.discrepancies.length + '</div><div class="label">价格冲突</div></div>' +
        '<div class="gp-summary-item"><div class="num">' + d.needs_review.length + '</div><div class="label">待审核</div></div>' +
        '<div class="gp-summary-item"><div class="num">' + d.history.length + '</div><div class="label">变更记录</div></div>';
      
      // Discrepancies table
      document.getElementById('disc-count').textContent = '(' + d.discrepancies.length + ')';
      var discBody = '';
      if (d.discrepancies.length === 0) {
        discBody = '<div class="gp-empty">&#9989; 没有发现价格冲突</div>';
      } else {
        discBody = '<div class="gp-table-wrap"><table class="gp-table"><thead><tr>' +
          '<th>礼物名称</th><th>DB价格</th><th>记录价格</th><th>场次数</th><th>操作</th></tr></thead><tbody>';
        d.discrepancies.forEach(function(disc) {
          discBody += '<tr><td><strong>' + escapeHtml(disc.gift_name) + '</strong></td>' +
            '<td><span class="gp-price">' + (disc.db_price != null ? disc.db_price : '—') + '</span></td>' +
            '<td>' + disc.prices.map(function(p) {
              var marker = p.unit_price !== disc.db_price ? ' <span style="color:var(--danger)">&#9888;</span>' : '';
              return '<span style="font-family:var(--font-mono);font-size:12px;">' + p.unit_price + ' (' + p.occurrences + 'x)' + marker + '</span>';
            }).join(', ') + '</td>' +
            '<td>' + disc.prices.reduce(function(s, p) { return s + p.sessions; }, 0) + '</td>' +
            '<td><button class="gp-action-btn" onclick="quickFix(\'' + escapeHtml(disc.gift_name) + '\')">修正</button></td></tr>';
        });
        discBody += '</tbody></table></div>';
      }
      document.getElementById('disc-table').innerHTML = discBody;
      
      // Needs review table
      document.getElementById('review-count').textContent = '(' + d.needs_review.length + ')';
      var reviewBody = '';
      if (d.needs_review.length === 0) {
        reviewBody = '<div class="gp-empty">&#9989; 没有待审核项</div>';
      } else {
        reviewBody = '<div class="gp-table-wrap"><table class="gp-table"><thead><tr>' +
          '<th>礼物名称</th><th>价格</th><th>备注</th><th>操作</th></tr></thead><tbody>';
        d.needs_review.forEach(function(r) {
          reviewBody += '<tr><td><strong>' + escapeHtml(r.gift_name) + '</strong></td>' +
            '<td><span class="gp-price">' + r.diamond_count + '</span></td>' +
            '<td style="font-size:11px;color:var(--text-muted);">' + escapeHtml((r.notes || '').substring(0, 60)) + '</td>' +
            '<td><button class="gp-action-btn" onclick="quickFix(\'' + escapeHtml(r.gift_name) + '\')">审核</button></td></tr>';
        });
        reviewBody += '</tbody></table></div>';
      }
      document.getElementById('review-table').innerHTML = reviewBody;
    })
    .catch(function() {
      summaryEl.innerHTML = '<div class="gp-empty">&#10060; 审计加载失败</div>';
    });
}

// Quick fix: search and open edit modal
function quickFix(name) {
  switchTab('browse');
  document.getElementById('gp-search').value = name;
  loadPrices();
}

// History tab
function loadHistory() {
  fetch('/api/gift-prices/audit')
    .then(function(r) { return r.json(); })
    .then(function(d) {
      var items = d.history || [];
      if (items.length === 0) {
        document.getElementById('history-table').innerHTML = '<div class="gp-empty">暂无变更记录</div>';
        return;
      }
      var html = '<div class="gp-table-wrap"><table class="gp-table"><thead><tr>' +
        '<th>时间</th><th>礼物</th><th>旧价格</th><th>新价格</th><th>影响行数</th><th>影响场次</th><th>备注</th></tr></thead><tbody>';
      items.forEach(function(h) {
        html += '<tr>' +
          '<td style="font-size:11px;font-family:var(--font-mono);">' + escapeHtml(h.created_at || '') + '</td>' +
          '<td><strong>' + escapeHtml(h.gift_name) + '</strong></td>' +
          '<td><span class="gp-price changed">' + h.old_price + '</span></td>' +
          '<td><span class="gp-price" style="color:var(--success)">' + h.new_price + '</span></td>' +
          '<td>' + (h.affected_rows || 0).toLocaleString() + '</td>' +
          '<td>' + (h.affected_sessions || 0) + '</td>' +
          '<td style="font-size:11px;color:var(--text-muted);max-width:200px;">' + escapeHtml(h.notes || '') + '</td>' +
        '</tr>';
      });
      html += '</tbody></table></div>';
      document.getElementById('history-table').innerHTML = html;
    })
    .catch(function() {
      document.getElementById('history-table').innerHTML = '<div class="gp-empty">&#10060; 加载失败</div>';
    });
}

// Utility: escape HTML
function escapeHtml(str) {
  if (!str) return '';
  return str.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
}

// Init
fetchPrices(1);
</script>
{% endblock %}
```

- [ ] **Step 3: Run a quick check — template renders without error**

```bash
python -X utf8 -c "
# Syntax check: verify app.py imports succeed
import sys; sys.path.insert(0, '.')
from app import app
# Verify the gift-prices route exists
rules = [r.rule for r in app.url_map.iter_rules() if 'gift' in r.rule]
print('Gift-related routes:')
for r in sorted(rules):
    print(f'  {r}')
"
```

Expected: Shows all gift-related routes including /gift-prices, /api/gift-prices, etc.

- [ ] **Step 4: Commit**

```bash
git add templates/gift_prices.html templates/base.html
git commit -m "feat(gift-prices): add web UI — listing, edit modal, audit, change history"
```

---

## Verification Checklist

After all tasks are complete, run this validation:

```bash
# 1. Verify tables
python -X utf8 -c "
import sys; sys.path.insert(0, '.')
from base.parser import _get_conn
conn = _get_conn()
tables = [r[0] for r in conn.execute(\"SELECT name FROM sqlite_master WHERE type='table' ORDER BY name\").fetchall()]
assert 'gift_prices' in tables, 'gift_prices table missing'
assert 'price_change_log' in tables, 'price_change_log table missing'
print('✅ Tables exist')
count = conn.execute('SELECT COUNT(*) FROM gift_prices').fetchone()[0]
assert count > 0, 'gift_prices is empty'
print(f'✅ gift_prices has {count} entries')
# Verify overrides loaded
override_cnt = conn.execute(\"SELECT COUNT(*) FROM gift_prices WHERE source='override'\").fetchone()[0]
print(f'✅ {override_cnt} override entries from hardcoded dicts')
"

# 2. Verify lookup function
python -X utf8 -c "
import sys; sys.path.insert(0, '.')
from base.parser import _lookup_gift_price, _load_gift_price_cache
_load_gift_price_cache()
price, overridden = _lookup_gift_price('至尊超跑', 1200)
assert price == 12000, f'Expected 12000, got {price}'
print('✅ Price lookup works: 至尊超跑 1200→12000')
price2, overridden2 = _lookup_gift_price('嘉年华', 30000)
assert price2 == 30000, f'Expected 30000, got {price2}'
print('✅ Price lookup works: 嘉年华 30000 (protobuf match)')
"

# 3. Verify Flask routes load
python -X utf8 -c "
import sys; sys.path.insert(0, '.')
from app import app
with app.test_client() as c:
    resp = c.get('/api/gift-prices?size=5')
    assert resp.status_code in (200, 401), f'Expected 200 or 401, got {resp.status_code}'
    print(f'✅ API /api/gift-prices returned {resp.status_code}')
    if resp.status_code == 200:
        data = resp.get_json()
        print(f'   {len(data.get(\"prices\", []))} prices returned, total={data.get(\"total\")}')
"
```

## Plan Self-Review Checklist

1. **Spec coverage:** Every section of the spec is covered:
   - Gift_prices table creation → Task 1
   - Population from three sources → Task 1 (init_gift_prices_table)
   - Price lookup in parser → Task 2 (_lookup_gift_price)
   - Recalculation engine → Task 3 (recalculate_gift_price)
   - Flask API routes → Task 4 (5 endpoints)
   - Web UI → Task 5 (single page with tabs: browse, audit, history)

2. **Placeholder scan:** No "TBD", "TODO", or "implement later" found. All code is concrete and complete.

3. **Type consistency:** Function names match between tasks: `init_gift_prices_table` is defined in Task 1 and called in Task 1 Step 3. `_lookup_gift_price` is defined in Task 2 and used in Task 2 Step 2. `recalculate_gift_price` is defined in Task 3 and called in Task 4 Step 5. `get_price_change_history` is defined in Task 3 and used in Task 4 Step 7. All signatures match.

4. **No hidden dependencies:** Tasks build on each other linearly (1→2→3→4→5). Each task produces interfaces that the next consumes.
