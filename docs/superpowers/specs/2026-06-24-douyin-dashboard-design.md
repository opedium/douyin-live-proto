# Douyin 直播间弹幕后台 — 架构设计

> 日期: 2026-06-24
> 状态: 待实现

---

## 1. 系统架构

```
┌─────────────────────────────────────────────────┐
│               main.py (采集器)                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │WebSocket  │  │ Playwright│  │ Scheduler    │   │
│  │(消息实时) │  │(名单提取) │  │ (每60s写入)  │   │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │             │               │            │
│       ▼             ▼               ▼            │
│  ┌───────────────────────────────────────────┐   │
│  │         SQLite Database (data/db.sqlite)    │   │
│  │  sessions | users | contributions         │   │
│  │  chat_logs | gift_logs | daily/monthly    │   │
│  └──────────────────┬────────────────────────┘   │
└─────────────────────┼────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────┐
│           Flask Web Server (app.py)              │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ REST API  │  │  HTML    │  │  Dashboard   │   │
│  │ /api/*    │  │ Routes   │  │  Templates   │   │
│  └────┬─────┘  └───┬──────┘  └──────┬───────┘   │
│       │            │                │            │
│       ▼            ▼                ▼            │
│  ┌───────────────────────────────────────────┐   │
│  │     Frontend (HTML + JavaScript + CSS)     │   │
│  │  积分榜单 | 弹幕查询 | 个人详情 | 神秘人   │   │
│  └───────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

### 1.1 组件职责

| 组件 | 职责 | 运行方式 |
|------|------|----------|
| `main.py` (fetcher) | WebSocket 采集 + Playwright 提取 | 持续运行 |
| `sqlite_writer.py` (新增) | 每60s将贡献缓存写入 SQLite | 被 fetcher 调用 |
| `app.py` (新增) | Flask Web 服务器，提供 API + 前端 | 独立进程 |
| `templates/` (新增) | HTML 模板 | 由 app.py 渲染 |

---

## 2. SQLite 数据库设计

### 2.1 ER 关系

```
sessions 1──N contributions
sessions 1──N chat_logs
sessions 1──N gift_logs
users   1──N contributions
users   1──N daily_stats
users   1──N monthly_stats
```

### 2.2 完整表结构

```sql
-- ==================== 直播会话 ====================
-- 每次 fetcher 启动时 INSERT，获取 session_id 供后续写入
CREATE TABLE sessions (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    room_id     TEXT NOT NULL,              -- 抖音房间ID
    anchor_name TEXT DEFAULT '',            -- 主播名
    start_time  DATETIME DEFAULT CURRENT_TIMESTAMP,
    end_time    DATETIME,
    status      TEXT DEFAULT 'live'         -- live / ended
);

-- ==================== 用户 ====================
CREATE TABLE users (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id         TEXT UNIQUE NOT NULL,    -- 抖音用户ID (真实)
    user_name       TEXT NOT NULL,           -- 显示昵称
    fans_club       TEXT DEFAULT '',
    is_anonymous    INTEGER DEFAULT 0,       -- 1=神秘人
    anonymous_label TEXT DEFAULT '',         -- "神秘人七阶" (显示名)
    first_seen      DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_seen       DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ==================== 贡献记录（核心表）====================
-- 每60s flush一次，每个session每个user一条
CREATE TABLE contributions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id      INTEGER NOT NULL REFERENCES sessions(id),
    user_id         TEXT NOT NULL,           -- 真实ID
    user_name       TEXT NOT NULL,           -- 当时昵称
    consume         INTEGER DEFAULT 0,       -- 该session最终贡献值
    rank            INTEGER DEFAULT 0,
    fans_club       TEXT DEFAULT '',
    source          TEXT DEFAULT 'websocket',
    -- 达标标记：每项代表该session中user达到了对应门槛
    qualified_1000  INTEGER DEFAULT 0,
    qualified_3000  INTEGER DEFAULT 0,
    qualified_10000 INTEGER DEFAULT 0,
    qualified_100000 INTEGER DEFAULT 0,
    recorded_at     DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(session_id, user_id)
);

-- ==================== 弹幕记录 ====================
CREATE TABLE chat_logs (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id  INTEGER REFERENCES sessions(id),
    user_id     TEXT NOT NULL,
    user_name   TEXT NOT NULL,
    content     TEXT NOT NULL,               -- 弹幕内容
    grade       TEXT DEFAULT '',             -- 用户等级
    fans_club   TEXT DEFAULT '',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ==================== 礼物记录 ====================
CREATE TABLE gift_logs (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id    INTEGER REFERENCES sessions(id),
    user_id       TEXT NOT NULL,
    user_name     TEXT NOT NULL,
    gift_name     TEXT NOT NULL,
    gift_count    INTEGER DEFAULT 1,
    diamond_total INTEGER DEFAULT 0,
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ==================== 每日统计 ====================
-- sessions_1000 等计数在 flush 时递增（不是覆盖），
-- 由 flush_to_sqlite 控制只新增不翻倍
CREATE TABLE daily_stats (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id          TEXT NOT NULL,
    user_name        TEXT NOT NULL,
    date             TEXT NOT NULL,           -- '2026-06-24'
    total_consume    INTEGER DEFAULT 0,
    sessions_1000    INTEGER DEFAULT 0,       -- 当天达千分榜的session数
    sessions_3000    INTEGER DEFAULT 0,
    sessions_10000   INTEGER DEFAULT 0,
    sessions_100000  INTEGER DEFAULT 0,
    gift_count       INTEGER DEFAULT 0,
    chat_count       INTEGER DEFAULT 0,
    UNIQUE(user_id, date)
);

-- ==================== 月度汇总 ====================
CREATE TABLE monthly_stats (
    id               INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id          TEXT NOT NULL,
    user_name        TEXT NOT NULL,
    year_month       TEXT NOT NULL,           -- '2026-06'
    total_consume    INTEGER DEFAULT 0,
    sessions_1000    INTEGER DEFAULT 0,       -- 本月达千分榜次数
    sessions_3000    INTEGER DEFAULT 0,
    sessions_10000   INTEGER DEFAULT 0,
    sessions_100000  INTEGER DEFAULT 0,
    days_active      INTEGER DEFAULT 0,       -- 活跃天数
    max_rank         INTEGER DEFAULT 0,
    UNIQUE(user_id, year_month)
);

-- ==================== 索引 ====================
CREATE INDEX idx_contributions_session ON contributions(session_id);
CREATE INDEX idx_contributions_user ON contributions(user_id);
CREATE INDEX idx_contributions_qualified ON contributions(qualified_1000);
CREATE INDEX idx_chat_logs_user ON chat_logs(user_id);
CREATE INDEX idx_chat_logs_content ON chat_logs(content);
CREATE INDEX idx_monthly_stats ON monthly_stats(year_month, sessions_1000 DESC);
CREATE INDEX idx_daily_stats ON daily_stats(date, sessions_1000 DESC);

-- ==================== 初始化 ====================
PRAGMA journal_mode=WAL;              -- 防读写锁
PRAGMA busy_timeout=5000;             -- 等待5秒再报错
```

### 2.3 匿名用户映射

神秘人由抖音分配匿名显示名（如"神秘人七阶"、"神秘人523150"），但 protobuf 中包含真实 user_id。

```python
# 用户进入时的检测逻辑 (parser.py)
# 检测条件：昵称以"神秘人"开头，且不含其他CJK字符
import re
nick = user.nick_name
if nick.startswith('神秘人') and not re.search(r'[一-鿿]', nick[3:]):
    is_anonymous = 1
    anonymous_label = nick            # "神秘人七阶"
    real_user_id = user.id_str        # "92866481804"
    # 后续所有该用户的记录都用真实ID写入
```

在数据库中的表现：
```
users 表:
  user_id         = "92866481804"   ← 真实ID
  user_name       = "神秘人七阶"    ← 匿名显示名
  is_anonymous    = 1
  anonymous_label = "神秘人七阶"

contributions 表:
  user_id         = "92866481804"   ← 真实ID
  user_name       = "神秘人七阶"
```

前端 `/anonymous` 页面查询时：
```sql
SELECT user_id AS real_user_id, user_name AS anonymous_label,
       consume, ...
FROM contributions
WHERE user_id IN (SELECT user_id FROM users WHERE is_anonymous = 1)
ORDER BY consume DESC;
```

---

## 3. Frontend 页面设计

### 3.1 技术选型
- **后端**: Python Flask (Jinja2 模板)
- **前端**: HTML + JavaScript + 简单 CSS (Bootstrap 或自定义)
- **前后端通信**: RESTful API (`/api/*`) + 页面加载时 fetch

### 3.2 页面导航

所有页面顶部有统一导航栏，包含全局搜索框：

```
[🏠 仪表盘] [🏆 积分榜] [💎 月百万] [🕵️ 神秘人] [📋 场次管理]
  ┌──────────────────────────────────────────────┐
  │  🔍 搜索用户 (输入ID或昵称)                    │
  │  ════════════════════════════════════════════  │
  │  一只熊二 [93409816]  ⭐ 本月18次上榜          │
  │  神秘人七阶 [92866481]  🎭 匿名用户            │
  │  11real [51368364]                            │
  └──────────────────────────────────────────────┘
```

### 3.3 页面列表

| 路由 | 页面 | 功能 |
|------|------|------|
| `/` | 仪表盘 | 当前状态、速览排行、快捷操作 |
| `/leaderboard` | 积分榜单 | 门槛可选 × 时间范围 × 排序翻页 |
| `/user?uid=xxx` | 个人详情 | 用户活动时间线（弹幕+礼物+贡献合一）|
| `/million` | 月百万榜 | 月贡献百万用户 |
| `/anonymous` | 神秘人查询 | 匿名ID → 真实ID 映射 |
| `/sessions` | 场次管理 | 历史直播列表，启停采集 |

### 3.4 用户操作流程

```
用户打开 Dashboard
       │
       ├── 查看当前直播状态、速览排行
       │
       ├── 进入 积分榜单
       │     ├── 切换门槛 (1000/3000/10000/自定义)
       │     ├── 切换时间 (当前直播/今日/本月/全部)
       │     ├── 点击用户 → 进入个人详情
       │     └── 搜索框找用户 → 进入个人详情
       │
       ├── 进入 个人详情
       │     ├── 查看用户基本信息、贡献统计
       │     └── 浏览活动时间线 (弹幕+礼物+达标)
       │         └── 筛选类型 / 搜索关键词
       │
       ├── 进入 月百万榜
       ├── 进入 神秘人查询
       └── 进入 场次管理
```

### 3.5 积分榜单 (`/leaderboard`) 交互细节

```
┌──────────────────────────────────────────────────────────────┐
│  🏆 积分榜单                                                   │
├──────────────────────────────────────────────────────────────┤
│  ⏱ [当前直播] [今日] [本月] [全部]                              │
│  📊 门槛: ●1000  ○3000  ○10000  ○100000  ○自定义 [____]      │
├────┬────────────┬───────────┬────────┬──────────┬────────────┤
│排名│ 昵称       │ 用户ID    │ 贡献值  │ 本月上榜次│ 粉丝团     │
├────┼────────────┼───────────┼────────┼──────────┼────────────┤
│ #1 │ 一只熊二   │ 93409816  │ 3,600  │   18     │ 香奈儿     │
│ #2 │ 神秘人七阶 │ 92866481  │ 2,800  │   12     │ 小葵花     │
│... │            │           │        │          │            │
├────┴────────────┴───────────┴────────┴──────────┴────────────┤
│  [< 1 2 3 4 5 ... >]  共 426 位                              │
├──────────────────────────────────────────────────────────────┤
│  🔍 [搜索用户]                                                 │
└──────────────────────────────────────────────────────────────┘
```

**关键交互:**
- 门槛切换 → 前端不刷新页面，fetch `/api/leaderboard?threshold=N`
- "本月上榜次" = `monthly_stats.sessions_1000`（该用户在当月不同场次中达标的次数）
- 搜索用户 → 顶部搜索框输入 user_id 或 user_name，支持模糊匹配，点击结果跳转个人详情
- **自动刷新**: 当前直播页面每30秒自动 fetch 最新数据（`setInterval`），无需手动刷新

### 3.6 个人详情 (`/user?uid=xxx`) — 用户活动时间线

管理员通过**全局搜索框**找到用户 → 点击进入个人详情页。

弹幕、礼物、贡献里程碑全部整合在**一条时间线**上，按时间倒序排列。
无需在多个页面/标签间切换。

```
┌──────────────────────────────────────────────────────────────┐
│  一只熊二  [93409816]  粉丝团: 香奈儿     ⭐ 本月18次上分榜   │
├──────────────────────────────────────────────────────────────┤
│  总贡献: 45,600   本场: 3,600  历史上榜: 42 次               │
├──────────────────────────────────────────────────────────────┤
│  🔍 筛选: [全部] [仅弹幕] [仅礼物] [仅达标]                   │
│     日期范围: [____] ~ [____]                                │
├──────┬──────────┬────────────────────────────┬──────────────┤
│ 时间  │ 类型     │ 内容                       │ 金额/贡献    │
├──────┼──────────┼────────────────────────────┼──────────────┤
│20:38 │ 💬 弹幕  │ 这主播真菜 😡              │              │
│20:37 │ 🎁 礼物  │ PK宝箱 x5                  │ 3,000钻石    │
│20:36 │ 💬 弹幕  │ 大家点点赞                  │              │
│20:35 │ 🏆 达标  │ 达到 1000 贡献              │ 1,000        │
│20:30 │ 🎁 礼物  │ 小心心 x10                 │ 10钻石       │
│20:29 │ 💬 弹幕  │ 主播好漂亮                  │              │
├──────┴──────────┴────────────────────────────┴──────────────┤
│  [< 1 2 3 4 5 ... >]  共 189 条活动                        │
└──────────────────────────────────────────────────────────────┘
```

### 3.7 神秘人查询 (`/anonymous`)

```
┌──────────────────────────────────────────────────────────────────┐
│  🕵️ 神秘人查询                                                    │
├──────────────────────────────────────────────────────────────────┤
│  神秘人由抖音自动分配匿名名称，但真实 user_id 可通过数据查到。       │
├────┬──────────────────┬──────────────┬──────────┬────────┬───────┤
│ #  │ 匿名显示名       │ 真实用户ID    │ 贡献值    │上榜次数 │最后出现│
├────┼──────────────────┼──────────────┼──────────┼────────┼───────┤
│  1 │ 神秘人七阶       │ 92866481804  │ 2,800    │  12    │ 06-24 │
│  2 │ 神秘人523150     │ 86352410157  │ 1,200    │   3    │ 06-23 │
│  3 │ 神秘人171297     │ 44172839561  │   800    │   1    │ 06-22 │
├────┴──────────────────┴──────────────┴──────────┴────────┴───────┤
│  🔍 搜索: [________]                                              │
│  💡 提示: 匿名名称由抖音分配，后台仍可追踪真实 user_id。             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. REST API 设计

```python
# ── 积分榜单 ──
GET /api/leaderboard
    ?threshold=1000          # 1000/3000/10000/100000
    &period=month            # session/today/month/all
    &page=1
    &size=100
    → {
        users: [{ rank, user_id, user_name, consume, sessions_count, fans_club }],
        total: 426,
        page: 1
      }

# ── 个人详情 ──
GET /api/user
    ?uid=93409816
    &name=一只熊二
    → {
        user: { user_id, user_name, fans_club, total_consume, is_anonymous, anonymous_label },
        sessions: [{ date, consume, qualified_1000, qualified_3000, ... }],
        monthly: { year_month, sessions_1000, total_consume }
      }

# ── 用户活动时间线（整合弹幕+礼物+贡献里程碑）──
GET /api/user/{user_id}/timeline
    ?type=all                 # all / chat / gift / milestone
    &keyword=xxx              # 按内容关键词筛选
    &date_from=2026-06-01
    &date_to=2026-06-24
    &page=1
    &size=50
    → {
        timeline: [{
          time, type,          # chat / gift / milestone
          content,             # 弹幕内容 / 礼物名 / "达到1000贡献"
          amount,              # 钻石数 / 贡献值 (gift/milestone)
        }],
        total: 189
      }

# ── 月百万榜 ──
GET /api/million
    ?year_month=2026-06
    &page=1
    → { users: [{ rank, user_id, user_name, total_consume, days_active }], total }

# ── 神秘人查询 ──
GET /api/anonymous
    ?page=1
    &search=xxx              # 按匿名名或真实ID搜索
    → { users: [{ anonymous_label, real_user_id, consume, sessions_count, last_seen }], total }

# ── 用户搜索 ──
GET /api/search
    ?q=一只熊二               # 按 user_name 模糊匹配
    &q=93409816               # 按 user_id 精确匹配
    &page=1
    &size=20
    → { users: [{ user_id, user_name, total_consume, sessions_1000, fans_club }], total }
    # 逻辑：先按 user_id 精确查，再按 user_name LIKE %q% 模糊查

# ── 场次管理 ──
GET  /api/sessions           → 历史场次列表
POST /api/session/{id}/end   → 结束一场采集
```

---

## 5. 从 CSV 迁移到 SQLite

### 5.1 写入流程 (fetcher.py _stats_task 改造)

```python
# 原来: flush_contribution_csv(csv_path)
# 改为:
flush_to_sqlite(db_path, session_id)
```

### 5.2 flush_to_sqlite() 逻辑 — 含防重复计数

```
1. 获取 contribution_cache 中的所有用户
2. 对每个用户:
   a. 查询 contributions 表当前 qualified_1000 值 (prev_qualified)
   b. UPSERT contributions:
      INSERT INTO contributions
        (session_id, user_id, user_name, consume, rank, fans_club, source,
         qualified_1000, qualified_3000, ...)
      ON CONFLICT(session_id, user_id) DO UPDATE SET
        consume = excluded.consume,
        rank = excluded.rank,
        qualified_1000 = CASE WHEN excluded.consume >= 1000 THEN 1 ELSE qualified_1000 END,
        qualified_3000 = ...
   c. 防重复：只在 qualified_1000 从未→首次变为1时才增加计数
      IF qualified_1000 == 1 AND prev_qualified == 0 THEN
         UPSERT daily_stats:
           INSERT INTO daily_stats (user_id, user_name, date, sessions_1000, total_consume)
           VALUES (?, ?, ?, 1, ?)
           ON CONFLICT(user_id, date) DO UPDATE SET
             sessions_1000 = sessions_1000 + 1,
             total_consume = total_consume + ?;
         UPSERT monthly_stats (同上逻辑)
      END IF
3. 保留 contributions 的 UNIQUE 约束，不再需要 _flushed_users
```

**关键变化**: 使用 `ON CONFLICT DO UPDATE` 而非 `INSERT OR REPLACE`，
确保 `sessions_1000` 是累加而非覆盖。检查 `prev_qualified` 确保每个 session
每个用户每个门槛只计一次。

### 5.3 去重策略

SQLite 的 `UNIQUE(session_id, user_id)` 约束替代了原来的 `_flushed_users` 集合。每60s flush 时使用 `ON CONFLICT DO UPDATE`，自动覆盖同一 session 内同一用户的旧记录。

### 5.4 Playwright 数据写入 SQLite

Playwright 提取的昵称（仅昵称、无 ID）同样写入 `contributions` 表：
- `user_id = ''`（空，暂未知）
- `user_name = Playwright提取的昵称`
- `source = 'playwright'`
- `consume = 0`

当后续 WebSocket 消息中同一用户出现（有 user_id），下一次 flush 时会：
1. 通过 `user_name` 匹配到已有记录
2. 用真实 `user_id` 更新该行
3. `source` 更新为 `'playwright+ws'`

### 5.5 数据保留策略

| 表 | 保留时间 | 清理方式 |
|------|----------|----------|
| `chat_logs` | 永久 | 弹幕记录不清理 |
| `gift_logs` | 永久 | 礼物记录不清理 |
| `contributions` | 永久 | 贡献数据不清理 |
| `daily_stats` | 永久 | 聚合数据，不清理 |
| `monthly_stats` | 永久 | 聚合数据，不清理 |
| `sessions` | 永久 | 直播场次不清理 |

---

## 6. 分步实施计划

### Phase 1: 数据库迁移
1. 创建 `sqlite_writer.py` — 包含 `init_db()`, `flush_to_sqlite()`, 所有 upsert 逻辑
2. 修改 `fetcher.py._stats_task` — 调用 `flush_to_sqlite` 替代 `flush_contribution_csv`
3. 保留 CSV 输出不变（双写），验证 SQLite 数据正确

### Phase 2: 匿名用户检测
1. 修改 `parser.py` — 聊天/进场/礼物解析时检测 "神秘人" 昵称
2. 写入 `users.is_anonymous` + `users.anonymous_label`

### Phase 3: Flask 后端
1. 创建 `app.py` — Flask 应用，SQLite 只读连接
2. 实现全部 `/api/*` 端点
3. 创建 `templates/` — HTML 模板 + JS/CSS

### Phase 4: 前端页面
1. 积分榜单页面 (含门槛切换、时间范围、搜索)
2. 弹幕查询页面 (管理员)
3. 个人详情页面
4. 神秘人查询页面
5. 月百万榜页面
6. 仪表盘 + 场次管理

### Phase 5: 集成测试
1. 采集环境运行 → SQLite 写入 → Flask 读取验证
2. Playwright 提取 → SQLite 合并验证
3. 前端页面交互测试
