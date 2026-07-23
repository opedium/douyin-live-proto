# SQLite + Flask Dashboard Implementation Plan

**Goal:** Replace CSV output with SQLite database and build a Flask web dashboard.

**Architecture:** sqlite_writer.py handles all DB operations, integrated into fetcher.py's stats_task. app.py serves Flask pages + REST API from the same DB.

**Tech Stack:** Python, SQLite (via sqlite3), Flask, HTML/JS/CSS

---

### Task 1: Create sqlite_writer.py — Database Module

**Files:**
- Create: `base/sqlite_writer.py`

- [ ] Write the complete DB module

```python
"""SQLite 数据库模块 — 替代 CSV 输出

所有采集数据写入 SQLite，供 Flask Web 端查询。
每60s由 fetcher._stats_task 调用 flush_to_sqlite()。
"""

import csv
import json
import logging
import os
import sqlite3
import threading
import time
from datetime import datetime

logger = logging.getLogger(__name__)

# 数据库路径（相对于项目根目录）
DB_DIR = os.path.join(os.path.dirname(os.path.dirname(os.path.abspath(__file__))), 'data')
DB_PATH = os.path.join(DB_DIR, 'douyin_barrage.db')
_local = threading.local()


def _get_conn():
    """获取线程本地数据库连接（自动 WAL 模式）。"""
    if not hasattr(_local, 'conn') or _local.conn is None:
        os.makedirs(DB_DIR, exist_ok=True)
        _local.conn = sqlite3.connect(DB_PATH)
        _local.conn.execute('PRAGMA journal_mode=WAL')
        _local.conn.execute('PRAGMA busy_timeout=5000')
        _local.conn.row_factory = sqlite3.Row
    return _local.conn


def init_db():
    """创建所有表（幂等）。"""
    conn = _get_conn()
    conn.executescript('''
        CREATE TABLE IF NOT EXISTS sessions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            room_id TEXT NOT NULL,
            anchor_name TEXT DEFAULT '',
            start_time DATETIME DEFAULT CURRENT_TIMESTAMP,
            end_time DATETIME,
            status TEXT DEFAULT 'live'
        );

        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT UNIQUE NOT NULL,
            user_name TEXT NOT NULL,
            fans_club TEXT DEFAULT '',
            is_anonymous INTEGER DEFAULT 0,
            anonymous_label TEXT DEFAULT '',
            first_seen DATETIME DEFAULT CURRENT_TIMESTAMP,
            last_seen DATETIME DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS contributions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            session_id INTEGER NOT NULL REFERENCES sessions(id),
            user_id TEXT NOT NULL,
            user_name TEXT NOT NULL,
            consume INTEGER DEFAULT 0,
            rank INTEGER DEFAULT 0,
            fans_club TEXT DEFAULT '',
            source TEXT DEFAULT 'websocket',
            qualified_1000 INTEGER DEFAULT 0,
            qualified_3000 INTEGER DEFAULT 0,
            qualified_10000 INTEGER DEFAULT 0,
            qualified_100000 INTEGER DEFAULT 0,
            recorded_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            UNIQUE(session_id, user_id)
        );

        CREATE TABLE IF NOT EXISTS chat_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            session_id INTEGER REFERENCES sessions(id),
            user_id TEXT NOT NULL,
            user_name TEXT NOT NULL,
            content TEXT NOT NULL,
            grade TEXT DEFAULT '',
            fans_club TEXT DEFAULT '',
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS gift_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            session_id INTEGER REFERENCES sessions(id),
            user_id TEXT NOT NULL,
            user_name TEXT NOT NULL,
            gift_name TEXT NOT NULL,
            gift_count INTEGER DEFAULT 1,
            diamond_total INTEGER DEFAULT 0,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE IF NOT EXISTS daily_stats (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT NOT NULL,
            user_name TEXT NOT NULL,
            date TEXT NOT NULL,
            total_consume INTEGER DEFAULT 0,
            sessions_1000 INTEGER DEFAULT 0,
            sessions_3000 INTEGER DEFAULT 0,
            sessions_10000 INTEGER DEFAULT 0,
            sessions_100000 INTEGER DEFAULT 0,
            gift_count INTEGER DEFAULT 0,
            chat_count INTEGER DEFAULT 0,
            UNIQUE(user_id, date)
        );

        CREATE TABLE IF NOT EXISTS monthly_stats (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT NOT NULL,
            user_name TEXT NOT NULL,
            year_month TEXT NOT NULL,
            total_consume INTEGER DEFAULT 0,
            sessions_1000 INTEGER DEFAULT 0,
            sessions_3000 INTEGER DEFAULT 0,
            sessions_10000 INTEGER DEFAULT 0,
            sessions_100000 INTEGER DEFAULT 0,
            days_active INTEGER DEFAULT 0,
            max_rank INTEGER DEFAULT 0,
            UNIQUE(user_id, year_month)
        );

        CREATE INDEX IF NOT EXISTS idx_contributions_session ON contributions(session_id);
        CREATE INDEX IF NOT EXISTS idx_contributions_user ON contributions(user_id);
        CREATE INDEX IF NOT EXISTS idx_chat_logs_user ON chat_logs(user_id);
        CREATE INDEX IF NOT EXISTS idx_monthly_stats ON monthly_stats(year_month, sessions_1000 DESC);
        CREATE INDEX IF NOT EXISTS idx_daily_stats ON daily_stats(date, sessions_1000 DESC);
    ''')
    conn.commit()
    logger.info(f"[DB] 已初始化: {DB_PATH}")


def create_session(room_id, anchor_name=''):
    """创建新直播场次，返回 session_id。"""
    conn = _get_conn()
    cur = conn.execute('INSERT INTO sessions (room_id, anchor_name) VALUES (?, ?)',
                       (room_id, anchor_name))
    conn.commit()
    sid = cur.lastrowid
    logger.info(f"[DB] 新场次 #{sid}: {anchor_name} ({room_id})")
    return sid


def end_session(session_id):
    """结束直播场次。"""
    conn = _get_conn()
    conn.execute('UPDATE sessions SET end_time = datetime("now"), status = "ended" WHERE id = ?',
                 (session_id,))
    conn.commit()


def upsert_user(user_id, user_name, fans_club='', is_anonymous=0, anonymous_label=''):
    """插入或更新用户信息。"""
    conn = _get_conn()
    conn.execute('''
        INSERT INTO users (user_id, user_name, fans_club, is_anonymous, anonymous_label, last_seen)
        VALUES (?, ?, ?, ?, ?, datetime('now'))
        ON CONFLICT(user_id) DO UPDATE SET
            user_name = excluded.user_name,
            fans_club = CASE WHEN excluded.fans_club != '' THEN excluded.fans_club ELSE fans_club END,
            last_seen = datetime('now')
    ''', (user_id, user_name, fans_club, is_anonymous, anonymous_label))
    conn.commit()


def flush_to_sqlite(session_id, contribution_users):
    """将贡献缓存写入 SQLite（含达标计数）。

    Args:
        session_id: 当前场次 ID
        contribution_users: get_contribution_users() 返回的列表
    """
    conn = _get_conn()
    now = datetime.now()
    today = now.strftime('%Y-%m-%d')
    month = now.strftime('%Y-%m')

    for u in contribution_users:
        uid = u.get('user_id', '')
        nick = u.get('nick', '')
        consume = u.get('consume', 0) or 0
        rank = u.get('rank', 0)
        fans_club = u.get('fans_club', '')
        source = 'websocket'

        # 1. 查询之前 qualified 状态（防重复计数）
        prev = conn.execute(
            'SELECT qualified_1000, qualified_3000, qualified_10000, qualified_100000 FROM contributions WHERE session_id=? AND user_id=?',
            (session_id, uid)
        ).fetchone()

        pq_1000 = prev['qualified_1000'] if prev else 0
        pq_3000 = prev['qualified_3000'] if prev else 0
        pq_10000 = prev['qualified_10000'] if prev else 0
        pq_100000 = prev['qualified_100000'] if prev else 0

        # 计算新达标状态
        q_1000 = 1 if consume >= 1000 else 0
        q_3000 = 1 if consume >= 3000 else 0
        q_10000 = 1 if consume >= 10000 else 0
        q_100000 = 1 if consume >= 100000 else 0

        # 2. UPSERT contributions
        conn.execute('''
            INSERT INTO contributions
                (session_id, user_id, user_name, consume, rank, fans_club, source,
                 qualified_1000, qualified_3000, qualified_10000, qualified_100000)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            ON CONFLICT(session_id, user_id) DO UPDATE SET
                consume = excluded.consume,
                rank = excluded.rank,
                qualified_1000 = MAX(qualified_1000, excluded.qualified_1000),
                qualified_3000 = MAX(qualified_3000, excluded.qualified_3000),
                qualified_10000 = MAX(qualified_10000, excluded.qualified_10000),
                qualified_100000 = MAX(qualified_100000, excluded.qualified_100000)
        ''', (session_id, uid, nick, consume, rank, fans_club, source,
              q_1000, q_3000, q_10000, q_100000))

        # 3. 达标次数计数（仅首次达标时递增）
        if q_1000 and not pq_1000:
            conn.execute('''
                INSERT INTO daily_stats (user_id, user_name, date, sessions_1000, total_consume)
                VALUES (?, ?, ?, 1, ?)
                ON CONFLICT(user_id, date) DO UPDATE SET
                    sessions_1000 = sessions_1000 + 1,
                    total_consume = total_consume + ?
            ''', (uid, nick, today, consume, consume))
            conn.execute('''
                INSERT INTO monthly_stats (user_id, user_name, year_month, sessions_1000, total_consume)
                VALUES (?, ?, ?, 1, ?)
                ON CONFLICT(user_id, year_month) DO UPDATE SET
                    sessions_1000 = sessions_1000 + 1,
                    total_consume = total_consume + ?
            ''', (uid, nick, month, consume, consume))

        if q_3000 and not pq_3000:
            conn.execute('''
                UPDATE daily_stats SET sessions_3000 = sessions_3000 + 1
                WHERE user_id = ? AND date = ?
            ''', (uid, today))
            conn.execute('''
                UPDATE monthly_stats SET sessions_3000 = sessions_3000 + 1
                WHERE user_id = ? AND year_month = ?
            ''', (uid, month))

        if q_10000 and not pq_10000:
            conn.execute('UPDATE daily_stats SET sessions_10000 = sessions_10000 + 1 WHERE user_id = ? AND date = ?', (uid, today))
            conn.execute('UPDATE monthly_stats SET sessions_10000 = sessions_10000 + 1 WHERE user_id = ? AND year_month = ?', (uid, month))

        if q_100000 and not pq_100000:
            conn.execute('UPDATE daily_stats SET sessions_100000 = sessions_100000 + 1 WHERE user_id = ? AND date = ?', (uid, today))
            conn.execute('UPDATE monthly_stats SET sessions_100000 = sessions_100000 + 1 WHERE user_id = ? AND year_month = ?', (uid, month))

    conn.commit()


def record_chat(session_id, user_id, user_name, content, grade='', fans_club=''):
    """记录一条弹幕。"""
    conn = _get_conn()
    conn.execute('INSERT INTO chat_logs (session_id, user_id, user_name, content, grade, fans_club) VALUES (?, ?, ?, ?, ?, ?)',
                 (session_id, user_id, user_name, content, grade, fans_club))
    conn.commit()


def record_gift(session_id, user_id, user_name, gift_name, gift_count, diamond_total):
    """记录一条礼物。"""
    conn = _get_conn()
    conn.execute('INSERT INTO gift_logs (session_id, user_id, user_name, gift_name, gift_count, diamond_total) VALUES (?, ?, ?, ?, ?, ?)',
                 (session_id, user_id, user_name, gift_name, gift_count, diamond_total))
    conn.commit()


# ── 查询接口 ──

def query_leaderboard(threshold=1000, period='session', page=1, size=100, session_id=None):
    """查询积分榜单。"""
    conn = _get_conn()
    offset = (page - 1) * size

    if period == 'session' and session_id:
        col = f'qualified_{threshold}'
        rows = conn.execute(f'''
            SELECT c.user_id, c.user_name, c.consume, c.rank, c.fans_club,
                   COALESCE(m.sessions_{threshold}, 0) AS sessions_count
            FROM contributions c
            LEFT JOIN monthly_stats m ON m.user_id = c.user_id AND m.year_month = strftime('%Y-%m', 'now')
            WHERE c.session_id = ? AND c.{col} = 1
            ORDER BY c.consume DESC
            LIMIT ? OFFSET ?
        ''', (session_id, size, offset)).fetchall()
        total = conn.execute(f'SELECT COUNT(*) FROM contributions WHERE session_id = ? AND qualified_{threshold} = 1',
                            (session_id,)).fetchone()[0]
    elif period == 'today':
        today = datetime.now().strftime('%Y-%m-%d')
        rows = conn.execute(f'''
            SELECT d.user_id, d.user_name, d.total_consume AS consume, d.sessions_{threshold} AS sessions_count
            FROM daily_stats d
            WHERE d.date = ? AND d.sessions_{threshold} > 0
            ORDER BY d.total_consume DESC
            LIMIT ? OFFSET ?
        ''', (today, size, offset)).fetchall()
        total = conn.execute('SELECT COUNT(*) FROM daily_stats WHERE date = ? AND sessions_1000 > 0',
                            (today,)).fetchone()[0]
    elif period == 'month':
        month = datetime.now().strftime('%Y-%m')
        rows = conn.execute(f'''
            SELECT m.user_id, m.user_name, m.total_consume AS consume, m.sessions_{threshold} AS sessions_count
            FROM monthly_stats m
            WHERE m.year_month = ? AND m.sessions_{threshold} > 0
            ORDER BY m.total_consume DESC
            LIMIT ? OFFSET ?
        ''', (month, size, offset)).fetchall()
        total = conn.execute('SELECT COUNT(*) FROM monthly_stats WHERE year_month = ? AND sessions_1000 > 0',
                            (month,)).fetchone()[0]
    else:  # all
        rows = conn.execute(f'''
            SELECT m.user_id, m.user_name, m.total_consume AS consume, m.sessions_{threshold} AS sessions_count
            FROM monthly_stats m
            WHERE m.sessions_{threshold} > 0
            ORDER BY m.total_consume DESC
            LIMIT ? OFFSET ?
        ''', (size, offset)).fetchall()
        total = conn.execute('SELECT COUNT(*) FROM monthly_stats WHERE sessions_1000 > 0').fetchone()[0]

    users = [dict(r) for r in rows]
    return {'users': users, 'total': total, 'page': page}


def query_user(user_id):
    """查询用户详情。"""
    conn = _get_conn()
    user = conn.execute('SELECT * FROM users WHERE user_id = ?', (user_id,)).fetchone()
    if not user:
        user = conn.execute('''
            SELECT user_id, user_name, fans_club, 0 as is_anonymous, '' as anonymous_label
            FROM contributions WHERE user_id = ? LIMIT 1
        ''', (user_id,)).fetchone()
    if not user:
        return None

    # 月度统计
    monthly = conn.execute('''
        SELECT year_month, total_consume, sessions_1000, sessions_3000,
               sessions_10000, sessions_100000, days_active
        FROM monthly_stats WHERE user_id = ? ORDER BY year_month DESC
    ''', (user_id,)).fetchall()

    # 总贡献
    total_consume = conn.execute(
        'SELECT SUM(consume) FROM contributions WHERE user_id = ?', (user_id,)
    ).fetchone()[0] or 0

    result = dict(user)
    result['total_consume'] = total_consume
    result['monthly'] = [dict(r) for r in monthly]
    return result


def query_user_timeline(user_id, type_filter='all', keyword='', page=1, size=50):
    """查询用户活动时间线（弹幕+礼物+里程碑合一）。"""
    conn = _get_conn()
    offset = (page - 1) * size
    results = []

    # 弹幕
    if type_filter in ('all', 'chat'):
        sql = 'SELECT created_at as time, "chat" as type, content, "" as amount, grade FROM chat_logs WHERE user_id = ?'
        params = [user_id]
        if keyword:
            sql += ' AND content LIKE ?'
            params.append(f'%{keyword}%')
        sql += ' ORDER BY created_at DESC LIMIT ? OFFSET ?'
        for row in conn.execute(sql, params + [size, offset]):
            results.append(dict(row))

    # 礼物
    if type_filter in ('all', 'gift'):
        if results:
            offset2 = 0
            size2 = size - len(results)
        else:
            offset2 = offset
            size2 = size
        sql = 'SELECT created_at as time, "gift" as type, gift_name || " x" || gift_count as content, diamond_total as amount FROM gift_logs WHERE user_id = ?'
        params = [user_id]
        if keyword:
            sql += ' AND (gift_name LIKE ? OR content LIKE ?)'
            params.extend([f'%{keyword}%', f'%{keyword}%'])
        sql += ' ORDER BY created_at DESC LIMIT ? OFFSET ?'
        for row in conn.execute(sql, params + [size2, offset2]):
            results.append(dict(row))

    # 按时间排序
    results.sort(key=lambda x: x.get('time', ''), reverse=True)
    results = results[:size]

    total = conn.execute('SELECT COUNT(*) FROM chat_logs WHERE user_id = ?', (user_id,)).fetchone()[0]
    return {'timeline': results, 'total': total, 'page': page}


def query_chat(user_id, keyword='', page=1, size=50):
    """查询用户弹幕记录（管理员用）。"""
    conn = _get_conn()
    offset = (page - 1) * size
    sql = 'SELECT created_at as time, user_id, user_name, content, grade FROM chat_logs WHERE 1=1'
    params = []
    if user_id:
        sql += ' AND user_id = ?'
        params.append(user_id)
    if keyword:
        sql += ' AND content LIKE ?'
        params.append(f'%{keyword}%')
    total = conn.execute(f'SELECT COUNT(*) FROM ({sql})', params).fetchone()[0]
    sql += ' ORDER BY created_at DESC LIMIT ? OFFSET ?'
    chats = [dict(r) for r in conn.execute(sql, params + [size, offset]).fetchall()]
    return {'chats': chats, 'total': total, 'page': page}


def query_anonymous(page=1, size=50, search=''):
    """查询神秘人用户。"""
    conn = _get_conn()
    offset = (page - 1) * size
    sql = '''
        SELECT u.user_id AS real_user_id, u.user_name AS anonymous_label,
               COALESCE(SUM(c.consume), 0) AS consume,
               COUNT(c.id) AS sessions_count,
               MAX(c.recorded_at) AS last_seen
        FROM users u
        LEFT JOIN contributions c ON c.user_id = u.user_id
        WHERE u.is_anonymous = 1
    '''
    params = []
    if search:
        sql += ' AND (u.user_name LIKE ? OR u.user_id LIKE ?)'
        params.extend([f'%{search}%', f'%{search}%'])
    sql += ' GROUP BY u.user_id ORDER BY consume DESC LIMIT ? OFFSET ?'
    users = [dict(r) for r in conn.execute(sql, params + [size, offset]).fetchall()]
    total_sql = 'SELECT COUNT(*) FROM users WHERE is_anonymous = 1'
    if search:
        total_sql += ' AND (user_name LIKE ? OR user_id LIKE ?)'
        total = conn.execute(total_sql, params[:2]).fetchone()[0] if search else conn.execute(total_sql).fetchone()[0]
    else:
        total = conn.execute(total_sql).fetchone()[0]
    return {'users': users, 'total': total, 'page': page}


def query_million(year_month='', page=1, size=100):
    """月百万榜。"""
    conn = _get_conn()
    if not year_month:
        year_month = datetime.now().strftime('%Y-%m')
    offset = (page - 1) * size
    rows = conn.execute('''
        SELECT user_id, user_name, total_consume, days_active
        FROM monthly_stats
        WHERE year_month = ? AND total_consume >= 1000000
        ORDER BY total_consume DESC LIMIT ? OFFSET ?
    ''', (year_month, size, offset)).fetchall()
    total = conn.execute(
        'SELECT COUNT(*) FROM monthly_stats WHERE year_month = ? AND total_consume >= 1000000',
        (year_month,)
    ).fetchone()[0]
    return {'users': [dict(r) for r in rows], 'total': total, 'page': page}


def query_sessions(limit=20):
    """查询历史场次。"""
    conn = _get_conn()
    rows = conn.execute('''
        SELECT s.*, (SELECT COUNT(*) FROM contributions WHERE session_id = s.id AND qualified_1000 = 1) as user_count
        FROM sessions s ORDER BY id DESC LIMIT ?
    ''', (limit,)).fetchall()
    return [dict(r) for r in rows]


def query_search(q, page=1, size=20):
    """搜索用户（按 user_id 精确或 user_name 模糊）。"""
    conn = _get_conn()
    offset = (page - 1) * size
    # 先精确匹配 user_id
    rows = conn.execute('''
        SELECT DISTINCT c.user_id, c.user_name,
               COALESCE(m.total_consume, 0) as total_consume,
               COALESCE(m.sessions_1000, 0) as sessions_1000,
               c.fans_club
        FROM contributions c
        LEFT JOIN monthly_stats m ON m.user_id = c.user_id AND m.year_month = strftime('%Y-%m', 'now')
        WHERE c.user_id = ?
        ORDER BY c.consume DESC
        LIMIT ? OFFSET ?
    ''', (q, size, offset)).fetchall()
    if not rows:
        # 模糊匹配 user_name
        rows = conn.execute('''
            SELECT DISTINCT c.user_id, c.user_name,
                   COALESCE(m.total_consume, 0) as total_consume,
                   COALESCE(m.sessions_1000, 0) as sessions_1000,
                   c.fans_club
            FROM contributions c
            LEFT JOIN monthly_stats m ON m.user_id = c.user_id AND m.year_month = strftime('%Y-%m', 'now')
            WHERE c.user_name LIKE ?
            ORDER BY c.consume DESC
            LIMIT ? OFFSET ?
        ''', (f'%{q}%', size, offset)).fetchall()
    return {'users': [dict(r) for r in rows], 'total': len(rows), 'page': page}
```

---

### Task 2: Integrate sqlite_writer into fetcher.py

**Files:**
- Modify: `service/fetcher.py`

- [ ] Add DB init and session creation to `start()` or `_connect_loop`

In `start()` after room info is fetched:
```python
# 初始化 SQLite
from base.sqlite_writer import init_db, create_session, end_session
init_db()
self._session_id = create_session(self.live_id, self.anchor_name)
```

In `stop()`:
```python
# 结束场次
if hasattr(self, '_session_id') and self._session_id:
    end_session(self._session_id)
```

In `_stats_task`, replace `flush_contribution_csv` with `flush_to_sqlite`:
```python
from base.sqlite_writer import flush_to_sqlite
# Instead of flush_contribution_csv:
flush_to_sqlite(self._session_id, get_contribution_users())
```

---

### Task 3: Create Flask App (app.py)

**Files:**
- Create: `app.py`

```python
#!/usr/bin/python
# coding:utf-8
"""Flask Web 服务器 — 弹幕后台管理面板

用法:
  python app.py [--port=8080] [--host=0.0.0.0]
  或与 main.py 同时运行
"""

import argparse
import json
import os
import sys
from datetime import datetime

from flask import Flask, jsonify, render_template, request

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from base.sqlite_writer import (
    query_leaderboard, query_user, query_user_timeline,
    query_chat, query_anonymous, query_million,
    query_sessions, query_search, DB_PATH, _get_conn
)

app = Flask(__name__)


# ── 页面路由 ──

@app.route('/')
def index():
    conn = _get_conn()
    # 当前场次
    session = conn.execute(
        "SELECT * FROM sessions WHERE status='live' ORDER BY id DESC LIMIT 1"
    ).fetchone()
    # 统计数据
    total_users = conn.execute('SELECT COUNT(DISTINCT user_id) FROM contributions').fetchone()[0]
    total_gifts = conn.execute('SELECT COUNT(*) FROM gift_logs').fetchone()[0]
    total_chats = conn.execute('SELECT COUNT(*) FROM chat_logs').fetchone()[0]
    # 最近场次
    recent = conn.execute(
        'SELECT * FROM sessions ORDER BY id DESC LIMIT 10'
    ).fetchall()
    return render_template('index.html',
        session=dict(session) if session else None,
        total_users=total_users,
        total_gifts=total_gifts,
        total_chats=total_chats,
        sessions=[dict(r) for r in recent],
        db_path=DB_PATH,
    )


@app.route('/leaderboard')
def leaderboard():
    return render_template('leaderboard.html')


@app.route('/user')
def user_detail():
    uid = request.args.get('uid', '')
    return render_template('user.html', uid=uid)


@app.route('/chat')
def chat():
    return render_template('chat.html')


@app.route('/anonymous')
def anonymous():
    return render_template('anonymous.html')


@app.route('/million')
def million():
    return render_template('million.html')


@app.route('/sessions')
def sessions():
    return render_template('sessions.html')


# ── API 路由 ──

@app.route('/api/leaderboard')
def api_leaderboard():
    threshold = request.args.get('threshold', 1000, type=int)
    period = request.args.get('period', 'session')
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 100, type=int)
    # 取当前 session_id
    session_id = request.args.get('session_id', None, type=int)
    if not session_id:
        conn = _get_conn()
        s = conn.execute("SELECT id FROM sessions WHERE status='live' ORDER BY id DESC LIMIT 1").fetchone()
        if s:
            session_id = s[0]
    return jsonify(query_leaderboard(threshold, period, page, size, session_id))


@app.route('/api/user')
def api_user():
    uid = request.args.get('uid', '')
    if not uid:
        return jsonify({'error': 'uid required'}), 400
    data = query_user(uid)
    if not data:
        return jsonify({'error': 'user not found'}), 404
    return jsonify(data)


@app.route('/api/user/<user_id>/timeline')
def api_user_timeline(user_id):
    type_filter = request.args.get('type', 'all')
    keyword = request.args.get('keyword', '')
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 50, type=int)
    return jsonify(query_user_timeline(user_id, type_filter, keyword, page, size))


@app.route('/api/user/<user_id>/gifts')
def api_user_gifts(user_id):
    conn = _get_conn()
    rows = conn.execute(
        'SELECT gift_name, gift_count, diamond_total, created_at FROM gift_logs WHERE user_id = ? ORDER BY created_at DESC LIMIT 100',
        (user_id,)
    ).fetchall()
    return jsonify([dict(r) for r in rows])


@app.route('/api/chat')
def api_chat():
    user_id = request.args.get('user_id', '')
    keyword = request.args.get('keyword', '')
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 50, type=int)
    return jsonify(query_chat(user_id, keyword, page, size))


@app.route('/api/anonymous')
def api_anonymous():
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 50, type=int)
    search = request.args.get('search', '')
    return jsonify(query_anonymous(page, size, search))


@app.route('/api/million')
def api_million():
    ym = request.args.get('year_month', '')
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 100, type=int)
    return jsonify(query_million(ym, page, size))


@app.route('/api/sessions')
def api_sessions():
    return jsonify(query_sessions())


@app.route('/api/search')
def api_search():
    q = request.args.get('q', '')
    page = request.args.get('page', 1, type=int)
    size = request.args.get('size', 20, type=int)
    if not q:
        return jsonify({'users': [], 'total': 0, 'page': 1})
    return jsonify(query_search(q, page, size))


@app.route('/api/stats')
def api_stats():
    conn = _get_conn()
    return jsonify({
        'total_users': conn.execute('SELECT COUNT(DISTINCT user_id) FROM contributions').fetchone()[0],
        'total_gifts': conn.execute('SELECT COUNT(*) FROM gift_logs').fetchone()[0],
        'total_chats': conn.execute('SELECT COUNT(*) FROM chat_logs').fetchone()[0],
        'total_sessions': conn.execute('SELECT COUNT(*) FROM sessions').fetchone()[0],
        'active_session': conn.execute("SELECT COUNT(*) FROM sessions WHERE status='live'").fetchone()[0] > 0,
    })


# ── 启动 ──

def main():
    parser = argparse.ArgumentParser(description='弹幕后台管理面板')
    parser.add_argument('--host', default='0.0.0.0')
    parser.add_argument('--port', default=8080, type=int)
    parser.add_argument('--debug', action='store_true')
    args = parser.parse_args()
    print(f'[Flask] 启动 http://{args.host}:{args.port}')
    app.run(host=args.host, port=args.port, debug=args.debug)


if __name__ == '__main__':
    main()
```

---

### Task 4: Create HTML Templates

**Files:**
- Create: `templates/base.html`
- Create: `templates/index.html`
- Create: `templates/leaderboard.html`
- Create: `templates/user.html`
- Create: `templates/chat.html`
- Create: `templates/anonymous.html`
- Create: `templates/million.html`
- Create: `templates/sessions.html`

All templates extend `base.html`. The base template includes:
- Navigation bar with search
- Bootstrap 5 CDN (or minimal custom CSS)
- Common JS for API calls

--- 

### Task 5: Integrate into main.py startup

**Files:**
- Modify: `main.py`

At startup, after room info is obtained:
- Call `init_db()`
- Call `create_session()`

On stop:
- Call `end_session()`
