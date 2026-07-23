# User Behavior Analytics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build Phase 1 of User Behavior Analytics — Retention Funnel + Big Spender Recognition — as a new `/analytics` page.

**Architecture:** Incremental extension of existing Flask/Chart.js patterns. New query functions in `parser.py`, new routes in `app.py`, new template `analytics.html` with Tab switching.

**Tech Stack:** Flask, SQLite, Chart.js (CDN, already loaded in base.html), vanilla JS.

**Global Constraints**
- All analytics queries must filter by `anchor_name` when specified, using `sessions.start_time` for ordering (not raw `session_id`)
- The streamer dropdown defaults to the first available streamer
- All API routes return JSON; page route returns HTML
- No new npm/nuget/pip dependencies

---
## File Map

| File | Action | Purpose |
|------|--------|---------|
| `base/parser.py` | Modify (append after line 3489) | Add 3 query functions |
| `app.py` | Modify (insert before `main()`) | Add 1 page route + 3 API routes |
| `templates/analytics.html` | Create | Full analytics page |
| `templates/base.html` | Modify (insert after line 965) | Add nav link |

---

### Task 1: Add query functions to parser.py

**Files:**
- Modify: `base/parser.py` (append after line 3489, before EOF)

**Interfaces:**
- Produces:
  - `query_user_retention(anchor, period, tier, page, size) → dict`
  - `query_big_spenders(min_consume, trend, anchor, page, size) → dict`
  - `query_silent_whales(threshold, silent_days, anchor) → dict`

- [ ] **Step 1: Add `query_user_retention()` function**

Append after the last function (`query_audit_diagnostic` at line 3489):

```python
# ═══════════════════════════════════════════════════════════════
#  User Behavior Analytics — query functions
# ═══════════════════════════════════════════════════════════════

def query_user_retention(anchor='', period='30d', tier=0, page=1, size=20):
    """用户留存漏斗：按主播过滤，按 sessions.start_time 排序定义第N场。

    Returns:
        dict with keys: summary (dict), retention_curve (dict), details (list), total (int), page (int)
    """
    conn = _get_conn()
    offset = (page - 1) * size

    # ── period → SQL snippet ──
    days_map = {'7d': 7, '30d': 30, '90d': 90}
    time_filter = ''
    time_params = ()
    if period in days_map:
        time_filter = 'AND s.start_time >= datetime(\'now\', \'-%d days\', \'+8 hours\')' % days_map[period]

    anchor_filter = ''
    anchor_params = ()
    if anchor:
        anchor_filter = 'AND s.anchor_name = ?'
        anchor_params = (anchor,)

    # ── target sessions with sequential numbering ──
    session_sql = f'''
        SELECT s.id, ROW_NUMBER() OVER (ORDER BY s.start_time) as session_seq
        FROM sessions s
        WHERE 1=1 {anchor_filter} {time_filter}
    '''
    # ── first session per user within target sessions ──
    first_sql = f'''
        SELECT g.user_id, MIN(sub.session_seq) as fs_seq,
               SUM(g.diamond_total) as total_consume
        FROM gift_logs g
        JOIN ({session_sql}) sub ON sub.id = g.session_id
        GROUP BY g.user_id
    '''
    tier_having = f'HAVING total_consume >= {int(tier)}' if tier > 0 else ''

    # ── count first-session users (for pagination) ──
    total = conn.execute(f'''
        SELECT COUNT(*) FROM ({first_sql}) f {tier_having}
    ''', anchor_params).fetchone()[0]

    # ── retention per session offset ──
    rows = conn.execute(f'''
        SELECT
            COUNT(*) as total_users,
            SUM(CASE WHEN retained_1 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) * 100 as retain_1_rate,
            SUM(CASE WHEN retained_2 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) * 100 as retain_2_rate,
            SUM(CASE WHEN retained_3 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) * 100 as retain_3_rate,
            SUM(CASE WHEN retained_4 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) * 100 as retain_4_rate,
            SUM(CASE WHEN retained_5 THEN 1 ELSE 0 END) * 1.0 / COUNT(*) * 100 as retain_5_rate
        FROM (
            SELECT f.user_id,
                (SELECT COUNT(*) FROM gift_logs g2
                 JOIN ({session_sql}) s2 ON s2.id = g2.session_id
                 WHERE g2.user_id = f.user_id AND s2.session_seq = f.fs_seq + 1) > 0 as retained_1,
                (SELECT COUNT(*) FROM gift_logs g2
                 JOIN ({session_sql}) s2 ON s2.id = g2.session_id
                 WHERE g2.user_id = f.user_id AND s2.session_seq <= f.fs_seq + 2) > 0 as retained_2,
                (SELECT COUNT(*) FROM gift_logs g2
                 JOIN ({session_sql}) s2 ON s2.id = g2.session_id
                 WHERE g2.user_id = f.user_id AND s2.session_seq <= f.fs_seq + 3) > 0 as retained_3,
                (SELECT COUNT(*) FROM gift_logs g2
                 JOIN ({session_sql}) s2 ON s2.id = g2.session_id
                 WHERE g2.user_id = f.user_id AND s2.session_seq <= f.fs_seq + 4) > 0 as retained_4,
                (SELECT COUNT(*) FROM gift_logs g2
                 JOIN ({session_sql}) s2 ON s2.id = g2.session_id
                 WHERE g2.user_id = f.user_id AND s2.session_seq <= f.fs_seq + 5) > 0 as retained_5
            FROM ({first_sql}) f
            {tier_having}
        ) r
    ''', anchor_params).fetchone()

    curve_data = {
        'labels': ['第1场', '第2场', '第3场', '第4场', '第5场'],
        'series': [
            {'name': '全部用户', 'color': '#94a3b8',
             'data': [100.0] + [round(rows[i] or 0, 1) for i in range(1, 6)]}
        ]
    }

    summary = {}
    if rows and rows['total_users']:
        summary = {
            'total_active_users': rows['total_users'],
            'first_session_users': rows['total_users'],
            'overall_retention_rate': round(rows['retain_1_rate'] or 0, 1),
            'high_value_retention_rate': round(rows['retain_1_rate'] or 0, 1),
        }

    # ── detail rows per anchor-date group ──
    details = conn.execute(f'''
        SELECT
            date(s_min.start_time) as first_date,
            s_min.anchor_name as anchor,
            COUNT(DISTINCT f.user_id) as first_users,
            COALESCE(SUM(CASE WHEN r2.retained THEN 1 ELSE 0 END), 0) as retained_2,
            COALESCE(SUM(CASE WHEN r5.retained THEN 1 ELSE 0 END), 0) as retained_5,
            COALESCE(SUM(f.total_consume), 0) as total_consume
        FROM (
            SELECT g.user_id, SUM(g.diamond_total) as total_consume,
                   MIN(g.session_id) as fs_id
            FROM gift_logs g
            JOIN sessions s ON s.id = g.session_id
            WHERE 1=1 {anchor_filter} {time_filter}
            GROUP BY g.user_id
        ) f
        JOIN sessions s_min ON s_min.id = f.fs_id
        LEFT JOIN (
            SELECT g2.user_id, 1 as retained
            FROM gift_logs g2
            JOIN sessions s2 ON s2.id = g2.session_id
            WHERE 1=1 {anchor_filter} {time_filter}
            GROUP BY g2.user_id
            HAVING MIN(s2.id) > f.fs_id
        ) r2 ON r2.user_id = f.user_id
        WHERE 1=1 {tier_having}
        GROUP BY date(s_min.start_time), s_min.anchor_name
        ORDER BY first_date DESC
        LIMIT ? OFFSET ?
    ''', anchor_params + (size, offset)).fetchall()

    detail_list = []
    for r in details:
        d = dict(r)
        d['retained_2_rate'] = round(d['retained_2'] / d['first_users'] * 100, 1) if d['first_users'] else 0
        d['retained_5_rate'] = round(d['retained_5'] / d['first_users'] * 100, 1) if d['first_users'] else 0
        detail_list.append(d)

    return {
        'summary': summary,
        'retention_curve': curve_data,
        'details': detail_list,
        'total': total,
        'page': page,
    }
```

- [ ] **Step 2: Add `query_big_spenders()` function**

Append after `query_user_retention()`:

```python
def query_big_spenders(min_consume=10000, trend='all', anchor='', page=1, size=50):
    """大R识别：消费排行 + 趋势分析（近3场 vs 前3场）。

    Returns:
        dict with keys: users (list), total (int), page (int)
    """
    conn = _get_conn()
    offset = (page - 1) * size

    anchor_filter = ''
    anchor_params = ()
    if anchor:
        anchor_filter = 'AND s.anchor_name = ?'
        anchor_params = (anchor,)

    # ── per-user ranked sessions (only target anchor) ──
    ranked = conn.execute(f'''
        SELECT c.user_id, c.consume, c.session_id, s.start_time
        FROM contributions c
        JOIN sessions s ON s.id = c.session_id
        WHERE c.consume > 0 {anchor_filter}
        ORDER BY c.user_id, s.start_time DESC
    ''', anchor_params).fetchall()

    # ── aggregate in Python (SQLite doesn't support ROW_NUMBER filtering easily) ──
    from collections import defaultdict
    user_sessions = defaultdict(list)
    for r in ranked:
        user_sessions[r['user_id']].append(r['consume'])

    rows = conn.execute(f'''
        SELECT c.user_id,
               COALESCE(NULLIF(u.user_name, ''), c.user_name) AS user_name,
               COALESCE(u.fans_club, '') AS fans_club,
               COALESCE(u.grade, '') AS grade,
               u.sec_uid, u.avatar_url,
               SUM(c.consume) as total_consume,
               COUNT(DISTINCT c.session_id) as active_days
        FROM contributions c
        JOIN sessions s ON s.id = c.session_id
        LEFT JOIN users u ON u.user_id = c.user_id
        WHERE c.consume > 0 {anchor_filter}
        GROUP BY c.user_id
        HAVING SUM(c.consume) >= ?
        ORDER BY total_consume DESC
    ''', anchor_params + (min_consume,)).fetchall()

    total = len(rows)
    users_list = []
    for i, r in enumerate(rows):
        uid = r['user_id']
        sess = user_sessions.get(uid, [])
        recent_3 = sum(sess[:3])
        prev_3 = sum(sess[3:6])
        total_cons = r['total_consume']

        if len(sess) >= 6 and prev_3 > 0:
            if recent_3 > prev_3 * 1.5:
                trend_label = 'accelerating'
            elif recent_3 < prev_3 * 0.5:
                trend_label = 'decelerating'
            else:
                trend_label = 'stable'
        else:
            trend_label = 'insufficient_data'

        if trend != 'all' and trend_label != trend:
            continue

        # silence days: days since last gift in this anchor's sessions
        last_seen = conn.execute(f'''
            SELECT MAX(g.created_at) as last_time
            FROM gift_logs g
            JOIN sessions s ON s.id = g.session_id
            WHERE g.user_id = ? {anchor_filter.replace('s.', 's2.') if anchor else ''}
        ''', (uid,) + anchor_params if anchor else (uid,)).fetchone()
        silent_days = 0
        if last_seen and last_seen['last_time']:
            try:
                lt = datetime.strptime(last_seen['last_time'][:19], '%Y-%m-%d %H:%M:%S')
                silent_days = (datetime.now() - lt).days
            except (ValueError, TypeError):
                pass

        users_list.append({
            'rank': offset + i + 1,
            'user_id': uid,
            'user_name': r['user_name'] or uid,
            'fans_club': r['fans_club'],
            'grade': r['grade'],
            'total_consume': total_cons,
            'recent_3': recent_3,
            'prev_3': prev_3,
            'trend': trend_label,
            'active_days': r['active_days'],
            'silent_days': silent_days,
            'avatar_url': r['avatar_url'] or '',
            'sec_uid': r['sec_uid'] or '',
        })

    # paginate after filtering
    paginated = users_list[offset:offset + size]

    return {'users': paginated, 'total': len(users_list), 'page': page}
```

- [ ] **Step 3: Add `query_silent_whales()` function**

Append after `query_big_spenders()`:

```python
def query_silent_whales(threshold=30000, silent_days=7, anchor=''):
    """沉默大R预警：总消费 >= threshold 且最近活跃超过 silent_days 的用户。

    Returns:
        dict with key: users (list)
    """
    conn = _get_conn()

    anchor_filter = ''
    anchor_params = ()
    if anchor:
        anchor_filter = 'AND s.anchor_name = ?'
        anchor_params = (anchor,)

    rows = conn.execute(f'''
        SELECT g.user_id,
               COALESCE(NULLIF(u.user_name, ''), g.user_name) AS user_name,
               u.avatar_url, u.sec_uid,
               SUM(g.diamond_total) as total_consume,
               MAX(g.created_at) as last_seen
        FROM gift_logs g
        JOIN sessions s ON s.id = g.session_id
        LEFT JOIN users u ON u.user_id = g.user_id
        WHERE 1=1 {anchor_filter}
        GROUP BY g.user_id
        HAVING SUM(g.diamond_total) >= ?
           AND MAX(g.created_at) < datetime('now', '-%d days', '+8 hours') % silent_days
        ORDER BY total_consume DESC
    ''', anchor_params + (threshold,)).fetchall()

    users_list = []
    for r in rows:
        silent = 0
        if r['last_seen']:
            try:
                lt = datetime.strptime(r['last_seen'][:19], '%Y-%m-%d %H:%M:%S')
                silent = (datetime.now() - lt).days
            except (ValueError, TypeError):
                pass
        users_list.append({
            'user_id': r['user_id'],
            'user_name': r['user_name'] or r['user_id'],
            'total_consume': r['total_consume'],
            'last_seen': r['last_seen'] or '',
            'silent_days': silent,
            'avatar_url': r['avatar_url'] or '',
            'sec_uid': r['sec_uid'] or '',
        })

    return {'users': users_list}
```

- [ ] **Step 4: Verify syntax**

```bash
python -c "import py_compile; py_compile.compile('base/parser.py', doraise=True)"
```

Expected: no output (success).

- [ ] **Step 5: Commit**

```bash
git add base/parser.py
git commit -m "feat(analytics): add query functions for user retention, big spenders, silent whales"

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

### Task 2: Add routes to app.py + nav link in base.html

**Files:**
- Modify: `app.py` (insert 1 page route + 3 API routes before `main()`)
- Modify: `templates/base.html` (insert nav link after the audit/gift-prices links)

- [ ] **Step 1: Add analytics page route + API routes to app.py**

Insert before `def main():` (line 1882), near the other route functions:

```python
# ═══════════════════════════════════════════════════════════════
#  User Behavior Analytics
# ═══════════════════════════════════════════════════════════════

@app.route('/analytics')
@require_auth
def analytics():
    return render_template('analytics.html')


@app.route('/api/analytics/retention')
@require_auth
def api_analytics_retention():
    anchor = request.args.get('anchor', '')
    period = request.args.get('period', '30d')
    tier = request.args.get('tier', 0, type=int)
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 20, type=int)
    try:
        data = query_user_retention(anchor, period, tier, page, size)
        return jsonify(data)
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/api/analytics/big-spenders')
@require_auth
def api_analytics_big_spenders():
    min_consume = request.args.get('min_consume', 10000, type=int)
    trend = request.args.get('trend', 'all')
    anchor = request.args.get('anchor', '')
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 50, type=int)
    try:
        data = query_big_spenders(min_consume, trend, anchor, page, size)
        return jsonify(data)
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/api/analytics/silent-whales')
@require_auth
def api_analytics_silent_whales():
    threshold = request.args.get('threshold', 30000, type=int)
    silent_days = request.args.get('silent_days', 7, type=int)
    anchor = request.args.get('anchor', '')
    try:
        data = query_silent_whales(threshold, silent_days, anchor)
        return jsonify(data)
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

- [ ] **Step 2: Add nav link to base.html**

In `templates/base.html`, insert a new nav link after the "审计" (audit) link (line 964):

```html
            <a href="/audit" class="{% if request.path == '/audit' or request.path.startswith('/audit/') %}active{% endif %}">审计</a>
            <a href="/analytics" class="{% if request.path == '/analytics' %}active{% endif %}">用户分析</a>
            <a href="/gift-prices" class="{% if request.path == '/gift-prices' %}active{% endif %}">礼物价格</a>
```

- [ ] **Step 3: Verify syntax**

```bash
python -c "import py_compile; py_compile.compile('app.py', doraise=True)"
```

Expected: no output (success).

- [ ] **Step 4: Commit**

```bash
git add app.py templates/base.html
git commit -m "feat(analytics): add routes and nav link for user behavior analytics"

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

### Task 3: Create analytics.html template — Structure + Tab switching + Filter bar

**Files:**
- Create: `templates/analytics.html` (skeleton, filter bar, tab switching, streamer loading)

- [ ] **Step 1: Create the template skeleton with tab navigation and filter bar**

```html
{% extends "base.html" %}
{% block title %}用户分析{% endblock %}
{% block content %}
<style>
/* ════════════════════════════════════════════════
   Analytics Page Styles
   ════════════════════════════════════════════════ */

/* Tab navigation */
.analytics-tabs {
    display: flex;
    gap: 0;
    border-bottom: 2px solid var(--border);
    margin-bottom: 16px;
}
.analytics-tab {
    padding: 10px 20px;
    cursor: pointer;
    font-size: 14px;
    font-weight: 600;
    color: var(--text-muted);
    border-bottom: 2px solid transparent;
    margin-bottom: -2px;
    transition: all var(--transition-fast);
    user-select: none;
}
.analytics-tab:hover { color: var(--text); }
.analytics-tab.active { color: var(--primary); border-bottom-color: var(--primary); }

/* Tab content */
.tab-content { display: none; }
.tab-content.active { display: block; }

/* Chart containers */
.chart-container {
    position: relative;
    width: 100%;
    height: 320px;
    margin: 10px 0;
}

/* Silent whale cards */
.silent-whale-card {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 12px 16px;
    background: var(--danger-bg);
    border-radius: var(--radius);
    margin-bottom: 8px;
    border-left: 4px solid var(--danger);
    cursor: pointer;
    transition: transform var(--transition-fast);
}
.silent-whale-card:hover { transform: translateX(4px); }
.silent-whale-card .sw-avatar {
    width: 36px; height: 36px; border-radius: 50%;
    background: var(--danger);
    display: flex; align-items: center; justify-content: center;
    color: #fff; font-weight: 700; font-size: 14px;
    flex-shrink: 0; overflow: hidden;
}
.silent-whale-card .sw-avatar img {
    width: 100%; height: 100%; object-fit: cover;
}
.silent-whale-card .sw-info { flex: 1; min-width: 0; }
.silent-whale-card .sw-name { font-weight: 600; font-size: 13px; }
.silent-whale-card .sw-meta { font-size: 11px; color: var(--text-muted); margin-top: 2px; }
.silent-whale-card .sw-amount { font-weight: 700; font-size: 14px; color: var(--danger); }

/* Trend badge */
.trend-badge {
    display: inline-block; padding: 2px 8px; border-radius: 4px;
    font-size: 11px; font-weight: 700;
}
.trend-badge.accelerating { background: var(--success-bg); color: #065f46; }
.trend-badge.stable { background: #fef3c7; color: #92400e; }
.trend-badge.decelerating { background: var(--danger-bg); color: #be123c; }
.trend-badge.insufficient_data { background: #f1f5f9; color: #64748b; }

/* Metric cards */
.stats-row.analytics-stats {
    grid-template-columns: repeat(auto-fit, minmax(170px, 1fr));
}
</style>

<div class="card">
    <!-- Tab Navigation -->
    <div class="analytics-tabs">
        <div class="analytics-tab active" data-tab="retention" onclick="switchTab('retention')">📊 留存漏斗</div>
        <div class="analytics-tab" data-tab="spenders" onclick="switchTab('spenders')">👑 大R识别</div>
    </div>

    <!-- Global Filter Bar -->
    <div class="filter-bar" style="margin-bottom:16px;">
        <span class="threshold-label">主播</span>
        <select id="anchor-select" onchange="onFilterChange()" style="max-width:150px;">
            <option value="">全部主播</option>
        </select>

        <span class="filter-divider"></span>
        <span class="threshold-label">时间范围</span>
        <select id="period-select" onchange="onFilterChange()">
            <option value="7d">近7天</option>
            <option value="30d" selected>近30天</option>
            <option value="90d">近90天</option>
            <option value="all">全部</option>
        </select>

        <span class="filter-divider"></span>
        <span class="threshold-label">消费层级</span>
        <select id="tier-select" onchange="onFilterChange()">
            <option value="0">全部</option>
            <option value="1000">1000+</option>
            <option value="3000">3000+</option>
            <option value="10000">10000+</option>
            <option value="100000">100000+</option>
        </select>

        <button onclick="loadCurrentTab()">&#x27F3; 刷新</button>
    </div>

    <!-- ════════ Tab: Retention Funnel ════════ -->
    <div id="tab-retention" class="tab-content active">
        <div class="loading" id="retention-loading">
            <div class="spinner"></div>
            <div class="msg">加载留存数据...</div>
        </div>
        <div id="retention-content" style="display:none;">
            <div class="stats-row analytics-stats" id="retention-stats"></div>
            <div class="card" style="margin-top:16px;">
                <h2>📈 留存曲线</h2>
                <div class="chart-container">
                    <canvas id="retention-chart"></canvas>
                </div>
            </div>
            <div class="card" style="margin-top:16px;">
                <h2>📋 详细数据</h2>
                <div class="table-wrap" id="retention-table-wrap">
                    <div class="empty-state"><div class="icon">📊</div>加载中...</div>
                </div>
                <div class="pagination" id="retention-pagination"></div>
            </div>
        </div>
        <div class="empty-state" id="retention-empty" style="display:none;">
            <div class="icon">📊</div>
            <div>暂无足够数据</div>
            <div class="hint">至少需要 2 场直播才能计算留存率</div>
        </div>
    </div>

    <!-- ════════ Tab: Big Spenders ════════ -->
    <div id="tab-spenders" class="tab-content">
        <div class="loading" id="spenders-loading">
            <div class="spinner"></div>
            <div class="msg">加载大R数据...</div>
        </div>
        <div id="spenders-content" style="display:none;">
            <div class="card">
                <h2>🏆 用户终身价值排行</h2>
                <div class="chart-container" style="height:420px;">
                    <canvas id="spender-chart"></canvas>
                </div>
            </div>
            <div class="card" style="margin-top:16px;">
                <h2>📋 大R明细</h2>
                <div class="filter-bar">
                    <span class="threshold-label">最低消费</span>
                    <input type="number" id="spender-min-consume" value="10000" style="width:100px;"
                           onchange="loadSpenders()">
                    <span class="filter-divider"></span>
                    <span class="threshold-label">趋势</span>
                    <select id="spender-trend" onchange="loadSpenders()">
                        <option value="all">全部</option>
                        <option value="accelerating">🚀 加速中</option>
                        <option value="stable">➡️ 稳定</option>
                        <option value="decelerating">🔻 减速中</option>
                    </select>
                    <button onclick="loadSpenders()">&#x27F3; 刷新</button>
                </div>
                <div class="table-wrap" id="spender-table-wrap">
                    <div class="empty-state"><div class="icon">👑</div>加载中...</div>
                </div>
                <div class="pagination" id="spender-pagination"></div>
            </div>
            <div class="card" style="margin-top:16px;" id="silent-whales-card">
                <h2>🔕 沉默大R预警</h2>
                <div id="silent-whales-list">
                    <div class="loading" style="padding:20px;">
                        <div class="spinner-ring"></div>
                    </div>
                </div>
            </div>
        </div>
        <div class="empty-state" id="spenders-empty" style="display:none;">
            <div class="icon">👑</div>
            <div>暂无大R数据</div>
            <div class="hint">调整最低消费阈值或确认有送礼记录</div>
        </div>
    </div>
</div>
```

- [ ] **Step 2: Add JS for tab switching, filter bar, streamer loading, and retention tab logic**

Append before `{% endblock %}`:

```html
<script>
/* ════════════════════════════════════════════════════
   Analytics Page JavaScript
   ════════════════════════════════════════════════════ */
var currentTab = 'retention';

/* ── Tab switching ── */
function switchTab(tab) {
    currentTab = tab;
    document.querySelectorAll('.analytics-tab').forEach(function(el) {
        el.classList.toggle('active', el.dataset.tab === tab);
    });
    document.querySelectorAll('.tab-content').forEach(function(el) {
        el.classList.toggle('active', el.id === 'tab-' + tab);
    });
    loadCurrentTab();
}

/* ── Filter helpers ── */
function getFilters() {
    return {
        anchor: document.getElementById('anchor-select').value,
        period: document.getElementById('period-select').value,
        tier: parseInt(document.getElementById('tier-select').value) || 0,
    };
}

function onFilterChange() {
    loadCurrentTab();
}

function loadCurrentTab() {
    if (currentTab === 'retention') loadRetention();
    else loadSpenders();
}

/* ── Streamer dropdown ── */
function loadStreamers() {
    fetch('/api/leaderboard/streamers')
        .then(function(r) { return r.json(); })
        .then(function(data) {
            var sel = document.getElementById('anchor-select');
            var streamers = data.streamers || data || [];
            streamers.forEach(function(s) {
                var opt = document.createElement('option');
                opt.value = s.anchor_name || s;
                opt.textContent = s.anchor_name || s;
                sel.appendChild(opt);
            });
            // auto-select first streamer if available
            if (streamers.length > 0) {
                sel.value = streamers[0].anchor_name || streamers[0];
            }
            loadRetention();
        })
        .catch(function() {
            loadRetention();
        });
}

/* ════════════════════════════════════════════════════
   RETENTION FUNNEL
   ════════════════════════════════════════════════════ */
var retentionChart = null;

function loadRetention() {
    var f = getFilters();
    var url = '/api/analytics/retention?anchor=' + encodeURIComponent(f.anchor)
        + '&period=' + f.period + '&tier=' + f.tier + '&page=1&size=20';

    document.getElementById('retention-loading').style.display = 'flex';
    document.getElementById('retention-content').style.display = 'none';
    document.getElementById('retention-empty').style.display = 'none';

    fetch(url)
        .then(function(r) { return r.json(); })
        .then(function(data) {
            document.getElementById('retention-loading').style.display = 'none';
            if (data.error || !data.summary || !data.summary.total_active_users) {
                document.getElementById('retention-empty').style.display = 'block';
                if (data.error) {
                    document.querySelector('#retention-empty .hint').textContent = data.error;
                }
                return;
            }
            document.getElementById('retention-content').style.display = 'block';
            renderRetentionStats(data.summary);
            renderRetentionChart(data.retention_curve);
            renderRetentionTable(data);
        })
        .catch(function(err) {
            document.getElementById('retention-loading').style.display = 'none';
            document.getElementById('retention-empty').style.display = 'block';
            document.querySelector('#retention-empty .hint').textContent = '加载失败: ' + err.message;
        });
}

function renderRetentionStats(summary) {
    var html = '';
    var cards = [
        {label: '总活跃用户', num: summary.total_active_users.toLocaleString(), color: '#6366f1'},
        {label: '首场用户数', num: summary.first_session_users.toLocaleString(), color: '#06b6d4'},
        {label: '整体留存率 (第2场)', num: summary.overall_retention_rate + '%', color: '#10b981'},
        {label: '高价值用户留存', num: summary.high_value_retention_rate + '%', color: '#f59e0b'},
    ];
    cards.forEach(function(c) {
        html += '<div class="stat-card" style="border-left-color:' + c.color + ';">'
            + '<div class="num">' + c.num + '</div>'
            + '<div class="label">' + c.label + '</div></div>';
    });
    document.getElementById('retention-stats').innerHTML = html;
}

function renderRetentionChart(curve) {
    if (retentionChart) retentionChart.destroy();
    if (!curve || !curve.series || curve.series.length === 0) return;

    var datasets = curve.series.map(function(s) {
        return {
            label: s.name,
            data: s.data,
            borderColor: s.color,
            backgroundColor: s.color + '20',
            borderWidth: 2,
            pointRadius: 4,
            pointHoverRadius: 6,
            tension: 0.2,
            fill: false,
        };
    });

    var ctx = document.getElementById('retention-chart').getContext('2d');
    retentionChart = new Chart(ctx, {
        type: 'line',
        data: { labels: curve.labels, datasets: datasets },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            interaction: { intersect: false, mode: 'index' },
            scales: {
                y: {
                    beginAtZero: true,
                    max: 100,
                    ticks: { callback: function(v) { return v + '%'; } },
                    grid: { color: 'var(--border-light)' },
                },
                x: { grid: { display: false } },
            },
            plugins: {
                legend: {
                    position: 'top',
                    labels: { usePointStyle: true, padding: 16, font: { size: 12 } },
                },
                tooltip: {
                    callbacks: {
                        label: function(ctx) {
                            return ctx.dataset.label + ': ' + ctx.parsed.y.toFixed(1) + '%';
                        }
                    }
                }
            }
        }
    });
}

function renderRetentionTable(data) {
    var html = '';
    if (!data.details || data.details.length === 0) {
        html = '<div class="empty-state"><div class="icon">📋</div>暂无详细数据</div>';
        document.getElementById('retention-table-wrap').innerHTML = html;
        document.getElementById('retention-pagination').innerHTML = '';
        return;
    }
    html = '<table><thead><tr>'
        + '<th>首场日期</th><th>主播</th><th>首场人数</th>'
        + '<th>回流第2场</th><th>回流第5场</th><th>总消费</th>'
        + '</tr></thead><tbody>';
    data.details.forEach(function(r) {
        html += '<tr>'
            + '<td>' + (r.first_date || '-') + '</td>'
            + '<td>' + (r.anchor || '-') + '</td>'
            + '<td>' + r.first_users + '</td>'
            + '<td>' + r.retained_2 + ' (' + (r.retained_2_rate || 0).toFixed(1) + '%)</td>'
            + '<td>' + r.retained_5 + ' (' + (r.retained_5_rate || 0).toFixed(1) + '%)</td>'
            + '<td>' + (r.total_consume || 0).toLocaleString() + '</td>'
            + '</tr>';
    });
    html += '</tbody></table>';
    document.getElementById('retention-table-wrap').innerHTML = html;

    // pagination
    var total = data.total || 0;
    var page = data.page || 1;
    var size = 20;
    var totalPages = Math.ceil(total / size);
    if (totalPages <= 1) {
        document.getElementById('retention-pagination').innerHTML = '';
        return;
    }
    var phtml = '<span>共 ' + total + ' 条</span>';
    for (var i = 1; i <= totalPages && i <= 10; i++) {
        phtml += '<a href="#" class="' + (i === page ? 'active' : '') + '" onclick="return loadRetentionPage(' + i + ')">' + i + '</a>';
    }
    document.getElementById('retention-pagination').innerHTML = phtml;
}

function loadRetentionPage(p) {
    var f = getFilters();
    var url = '/api/analytics/retention?anchor=' + encodeURIComponent(f.anchor)
        + '&period=' + f.period + '&tier=' + f.tier + '&page=' + p + '&size=20';
    fetch(url)
        .then(function(r) { return r.json(); })
        .then(function(data) {
            if (data.retention_curve) renderRetentionChart(data.retention_curve);
            renderRetentionTable(data);
        })
        .catch(function(err) { showToast('加载失败: ' + err.message, 'error'); });
    return false;
}
</script>
```

- [ ] **Step 3: Add JS for big spender tab (chart + table + silent whales)**

Append inside the `<script>` block, after the retention functions:

```javascript
/* ════════════════════════════════════════════════════
   BIG SPENDERS
   ════════════════════════════════════════════════════ */
var spenderChart = null;

function loadSpenders() {
    var f = getFilters();
    var minConsume = document.getElementById('spender-min-consume').value || 10000;
    var trend = document.getElementById('spender-trend').value;
    var url = '/api/analytics/big-spenders?min_consume=' + minConsume
        + '&trend=' + trend
        + '&anchor=' + encodeURIComponent(f.anchor)
        + '&page=1&size=50';

    document.getElementById('spenders-loading').style.display = 'flex';
    document.getElementById('spenders-content').style.display = 'none';
    document.getElementById('spenders-empty').style.display = 'none';

    fetch(url)
        .then(function(r) { return r.json(); })
        .then(function(data) {
            document.getElementById('spenders-loading').style.display = 'none';
            if (data.error || !data.users || data.users.length === 0) {
                document.getElementById('spenders-empty').style.display = 'block';
                if (data.error) {
                    document.querySelector('#spenders-empty .hint').textContent = data.error;
                }
                return;
            }
            document.getElementById('spenders-content').style.display = 'block';
            renderSpenderChart(data.users);
            renderSpenderTable(data);
            loadSilentWhales();
        })
        .catch(function(err) {
            document.getElementById('spenders-loading').style.display = 'none';
            document.getElementById('spenders-empty').style.display = 'block';
            document.querySelector('#spenders-empty .hint').textContent = '加载失败: ' + err.message;
        });
}

function renderSpenderChart(users) {
    if (spenderChart) spenderChart.destroy();

    var labels = users.slice(0, 20).map(function(u) { return u.user_name; }).reverse();
    var values = users.slice(0, 20).map(function(u) { return u.total_consume; }).reverse();
    var colors = users.slice(0, 20).map(function(u) {
        if (u.trend === 'accelerating') return '#10b981';
        if (u.trend === 'decelerating') return '#ef4444';
        if (u.trend === 'stable') return '#f59e0b';
        return '#94a3b8';
    }).reverse();

    var ctx = document.getElementById('spender-chart').getContext('2d');
    spenderChart = new Chart(ctx, {
        type: 'bar',
        data: {
            labels: labels,
            datasets: [{
                label: '累计钻石',
                data: values,
                backgroundColor: colors,
                borderColor: colors.map(function(c) { return c + '80'; }),
                borderWidth: 1,
                borderRadius: 4,
            }]
        },
        options: {
            indexAxis: 'y',
            responsive: true,
            maintainAspectRatio: false,
            scales: {
                x: {
                    ticks: { callback: function(v) { return v.toLocaleString(); } },
                    grid: { color: 'var(--border-light)' },
                },
                y: { grid: { display: false } },
            },
            plugins: {
                legend: { display: false },
                tooltip: {
                    callbacks: {
                        label: function(ctx) {
                            var u = users[users.length - 1 - ctx.dataIndex] || users[0];
                            var trendMap = {
                                accelerating: '🚀 加速中',
                                stable: '➡️ 稳定',
                                decelerating: '🔻 减速中',
                                insufficient_data: '⚠️ 数据不足'
                            };
                            return ctx.parsed.x.toLocaleString() + ' 钻 | '
                                + (trendMap[u.trend] || u.trend);
                        }
                    }
                }
            }
        }
    });
}

function renderSpenderTable(data) {
    var users = data.users || [];
    var html = '';
    if (users.length === 0) {
        html = '<div class="empty-state"><div class="icon">👑</div>暂无匹配的大R用户</div>';
        document.getElementById('spender-table-wrap').innerHTML = html;
        document.getElementById('spender-pagination').innerHTML = '';
        return;
    }

    var trendLabels = {
        accelerating: '<span class="trend-badge accelerating">🚀 加速</span>',
        stable: '<span class="trend-badge stable">➡️ 稳定</span>',
        decelerating: '<span class="trend-badge decelerating">🔻 减速</span>',
        insufficient_data: '<span class="trend-badge insufficient_data">⚠️ 数据不足</span>',
    };

    html = '<table><thead><tr>'
        + '<th>排名</th><th>用户</th><th>粉丝团</th><th>总消费</th>'
        + '<th>近3场</th><th>前3场</th><th>趋势</th><th>活跃天数</th><th>沉默天数</th>'
        + '</tr></thead><tbody>';
    users.forEach(function(u) {
        html += '<tr>'
            + '<td><span class="rank-' + (u.rank <= 3 ? u.rank : '') + '">' + u.rank + '</span></td>'
            + '<td><a href="/user?uid=' + encodeURIComponent(u.user_id) + '">'
            + (u.user_name || u.user_id) + '</a></td>'
            + '<td>' + (u.fans_club || '-') + '</td>'
            + '<td><strong>' + (u.total_consume || 0).toLocaleString() + '</strong></td>'
            + '<td>' + (u.recent_3 || 0).toLocaleString() + '</td>'
            + '<td>' + (u.prev_3 || 0).toLocaleString() + '</td>'
            + '<td>' + (trendLabels[u.trend] || '-') + '</td>'
            + '<td>' + (u.active_days || 0) + '</td>'
            + '<td>' + (u.silent_days > 0 ? '<span style="color:var(--danger);font-weight:600;">' + u.silent_days + '天</span>' : '-') + '</td>'
            + '</tr>';
    });
    html += '</tbody></table>';
    document.getElementById('spender-table-wrap').innerHTML = html;

    // pagination
    var total = data.total || 0;
    var page = data.page || 1;
    var totalPages = Math.ceil(total / 50);
    if (totalPages <= 1) {
        document.getElementById('spender-pagination').innerHTML = '';
        return;
    }
    var phtml = '<span>共 ' + total + ' 条</span>';
    for (var i = 1; i <= totalPages && i <= 10; i++) {
        phtml += '<a href="#" class="' + (i === page ? 'active' : '') + '" onclick="return loadSpenderPage(' + i + ')">' + i + '</a>';
    }
    document.getElementById('spender-pagination').innerHTML = phtml;
}

function loadSpenderPage(p) {
    var f = getFilters();
    var minConsume = document.getElementById('spender-min-consume').value || 10000;
    var trend = document.getElementById('spender-trend').value;
    var url = '/api/analytics/big-spenders?min_consume=' + minConsume
        + '&trend=' + trend
        + '&anchor=' + encodeURIComponent(f.anchor)
        + '&page=' + p + '&size=50';
    fetch(url)
        .then(function(r) { return r.json(); })
        .then(function(data) {
            if (data.users) renderSpenderTable(data);
        })
        .catch(function(err) { showToast('加载失败: ' + err.message, 'error'); });
    return false;
}

/* ════════════════════════════════════════════════════
   SILENT WHALES
   ════════════════════════════════════════════════════ */
function loadSilentWhales() {
    var f = getFilters();
    var url = '/api/analytics/silent-whales?threshold=30000&silent_days=7'
        + '&anchor=' + encodeURIComponent(f.anchor);

    fetch(url)
        .then(function(r) { return r.json(); })
        .then(function(data) {
            var list = document.getElementById('silent-whales-list');
            if (!data.users || data.users.length === 0) {
                document.getElementById('silent-whales-card').style.display = 'none';
                return;
            }
            document.getElementById('silent-whales-card').style.display = 'block';
            var html = '';
            data.users.forEach(function(u) {
                var avatarContent = u.avatar_url
                    ? '<img src="' + u.avatar_url + '" onerror="this.parentElement.textContent=\'' + (u.user_name || '?').charAt(0) + '\'">'
                    : (u.user_name || '?').charAt(0);
                html += '<a href="/user?uid=' + encodeURIComponent(u.user_id) + '" class="silent-whale-card" style="text-decoration:none;color:inherit;">'
                    + '<div class="sw-avatar">' + avatarContent + '</div>'
                    + '<div class="sw-info">'
                    + '<div class="sw-name">' + (u.user_name || u.user_id) + '</div>'
                    + '<div class="sw-meta">最后出现: ' + (u.last_seen ? u.last_seen.slice(11, 19) + ' (' + u.silent_days + '天前)' : '未知') + '</div>'
                    + '</div>'
                    + '<div class="sw-amount">' + (u.total_consume || 0).toLocaleString() + '</div>'
                    + '</a>';
            });
            list.innerHTML = html;
        })
        .catch(function() {
            document.getElementById('silent-whales-card').style.display = 'none';
        });
}

/* ════════════════════════════════════════════════════
   INIT
   ════════════════════════════════════════════════════ */
loadStreamers();
</script>
{% endblock %}
```

- [ ] **Step 4: Verify template by starting the Flask app**

```bash
cd /path/to/barra2 && python app.py --port 8080
```

Then open `http://localhost:8080/analytics` in a browser.
Expected: Page loads with tab navigation, filter bar, streamer dropdown auto-selected.

- [ ] **Step 5: Commit**

```bash
git add templates/analytics.html
git commit -m "feat(analytics): add analytics page template with retention and big spender tabs"

Co-Authored-By: Claude <noreply@anthropic.com>
```

---

### Task 4: Verification and edge case testing

- [ ] **Step 1: Test retention funnel**

1. Load `/analytics` — verify streamer dropdown auto-selects first streamer
2. Verify 4 metric cards render with correct numbers
3. Verify retention line chart renders with 4 curves
4. Switch time range to "近7天" — verify data reloads
5. Switch consumer tier — verify curve updates
6. Switch to "全部主播" — verify global data loads

- [ ] **Step 2: Test big spender tab**

1. Click "大R识别" tab — verify horizontal bar chart renders
2. Verify table shows rank, user, fans club, total consume, trend badges
3. Click a user name — verify it navigates to `/user?uid=...`
4. Change trend filter to "加速中" — verify list filters
5. Verify silent whale section appears (if any whales exist)

- [ ] **Step 3: Test empty/edge states**

1. If no data for a period, verify empty state message appears
2. If no silent whales, verify the section is hidden
3. If only 1 session exists, verify retention shows appropriate message

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "feat(analytics): complete phase 1 - retention funnel and big spender recognition"

Co-Authored-By: Claude <noreply@anthropic.com>
```
