# 三项功能详细规格说明

---

## 1. 外部数据增强 — 当前已接收但未存储的字段

### 概述

当前系统在 WebSocket 消息解析时丢弃了大量 protobuf 字段，以及抖音 API 返回的用户信息。将这些数据捕获入库后，可以解锁大量分析能力，无需额外 API 调用。

---

### 1A. 用户 `pay_grade`（消费等级明细）— 最高优先级

#### 现有状态

当前只提取 `fmt_grade(user)` → 返回等级的数字/字符串（如 "40"），`pay_grade` 中丰富的明细字段全部丢弃。

#### protobuf 可用字段

```
User.pay_grade:
  level: int64              # 当前等级（已有）
  total_diamond_count: int64 # 累计历史总消费钻石
  this_grade_min_diamond: int64 # 本级起始钻石数
  this_grade_max_diamond: int64 # 本级上限钻石数
  upgrade_need_consume: int64   # 距下一级还需多少钻
  score: int64              # 当前经验值/积分
  grade_describe: string    # 等级名称/描述
  next_name: string         # 下级名称
  now_diamond: int64        # 当前累计钻石（可能 ≠ total_diamond_count）
  pay_diamond_bak: int64   # 备份总额
```

#### 存储方案

在 `users` 表新增列：

```sql
ALTER TABLE users ADD COLUMN pay_grade_score INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN pay_grade_total_diamond INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN pay_grade_min_diamond INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN pay_grade_max_diamond INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN pay_grade_upgrade_need INTEGER DEFAULT 0;
```

或创建独立表 `user_grade_snapshots` 记录历史等级变化：

```sql
CREATE TABLE IF NOT EXISTS user_grade_snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT NOT NULL,
    level INTEGER DEFAULT 0,
    total_diamond INTEGER DEFAULT 0,
    score INTEGER DEFAULT 0,
    upgrade_need INTEGER DEFAULT 0,
    snapshot_time DATETIME DEFAULT (datetime('now', '+8 hours'))
);
```

#### 改造成本

- `base/parser.py`: 在 `upsert_user()`/`parse_gift_msg()`/`parse_chat_msg()` 中提取 `pay_grade` 字段
- 约 10 行代码新增

#### 解锁的分析能力

1. **"距下一级还差 X 钻"** — 在用户详情页展示升级进度条
2. **用户消费成长曲线** — 定期快照 `total_diamond_count` 变化，绘制时间轴
3. **升级速度分析** — 从 0 级到当前级花了多少时间/场次
4. **大 R 早期识别** — 快速升到高等级的用户，说明消费意愿强

---

### 1B. 用户 `follow_info`（粉丝/关注数）— 高优先级

#### 现有状态

`User.follow_info` 包含 `follower_count`（粉丝数）和 `following_count`（关注数），但**从未存储**。

用户详情页用抖音 API 拉取这些信息，每次打开页面都会调 API（限频 30 次/分钟）。

#### 存储方案

在 `users` 表新增列：

```sql
ALTER TABLE users ADD COLUMN follower_count INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN following_count INTEGER DEFAULT 0;
ALTER TABLE users ADD COLUMN signature TEXT DEFAULT '';
ALTER TABLE users ADD COLUMN city TEXT DEFAULT '';
```

#### 数据来源

**方案 A（推荐）：定期通过 `/api/user/avatar-lookup` 后端批量回填**
- 对没有 `follower_count` 或数据超过 24h 的用户
- 利用限频器控制 API 调用

**方案 B：WebSocket 消息中提取（实时但字段不一定每次都带）**
- 部分消息的 User 对象含 `follow_info`
- 在 `upsert_user()` 中检查并更新

#### 解锁的分析能力

1. **大 R vs KOL 区分** — 高粉丝数的用户可能是主播/KOL，低粉丝数高消费是纯真大 R
2. **用户影响力排序** — 按粉丝数排榜
3. **粉丝数变化追踪** — 快照历史，看大 R 是否在掉粉/涨粉
4. **地域分布地图** — 按 `city` 字段做用户城市分布热力图

---

### 1C. 用户 `signature`（签名）和 `city`（城市）— 中等优先级

#### 现有状态

`User.signature` 和 `User.city` 在每条消息的 User 对象中都有，但完全丢弃。

即使 `signature` 可能为空/广告，`city` 字段通常由抖音填充（基于 IP 属地/注册信息）。

#### 存储方案

见 1B 的 `users` 表新增列。

#### 解锁的分析能力

1. **用户画像标签** — signature 中可能包含职业、爱好、身份标识
2. **地域分布** — 城市维度分析观众在哪里
3. **签名关键词词云** — 所有用户的 signature 做词频统计

---

### 1D. `gift_logs` 表扩展字段 — 高优先级

#### 现有状态

`gift_logs` 表只存了 `user_id, user_name, gift_name, gift_count, diamond_total, group_id`，但 `parse_gift_msg()` 的 data dict 里有很多字段被丢弃了。

#### 表结构变更

```sql
ALTER TABLE gift_logs ADD COLUMN grade TEXT DEFAULT '';      -- 送礼时财富等级
ALTER TABLE gift_logs ADD COLUMN fans_club TEXT DEFAULT '';  -- 送礼时粉丝团
ALTER TABLE gift_logs ADD COLUMN sec_uid TEXT DEFAULT '';
ALTER TABLE gift_logs ADD COLUMN avatar_url TEXT DEFAULT '';
ALTER TABLE gift_logs ADD COLUMN trace_id TEXT DEFAULT '';   -- 去重追踪
ALTER TABLE gift_logs ADD COLUMN gift_type INTEGER DEFAULT 0; -- 礼物类型(来自gift.type)
ALTER TABLE gift_logs ADD COLUMN send_time INTEGER DEFAULT 0; -- 精确发送时间戳
```

> 注意：`grade` 和 `fans_club` 的迁移代码已存在（parser.py line 1855-1858），但其他字段还未加。

#### 解锁的分析能力

1. **按礼物类型分析收入** — `gift.type` 区分面板礼物/特效礼物/粉丝团礼物
2. **送礼时间分布** — `send_time` 做精确到秒的时间分布
3. **送礼时的用户状态** — `grade` 和 `fans_club` 让分析可以回答"高等级用户送什么礼物"之类问题

---

### 1E. LikeMessage（点赞）入库 — 高优先级

#### 现有状态

点赞消息只打印日志，**不入库也不在 Web 面板展示**。

#### 存储方案

新建 `like_logs` 表：

```sql
CREATE TABLE IF NOT EXISTS like_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER REFERENCES sessions(id),
    user_id TEXT NOT NULL,
    user_name TEXT NOT NULL,
    count INTEGER DEFAULT 1,     -- 本次点赞数
    total INTEGER DEFAULT 0,     -- 该用户累计点赞数
    created_at DATETIME DEFAULT (datetime('now', '+8 hours'))
);
```

在 `parser.py` 的 `parse_like_msg()` 中增加 `record_like()` 调用。

#### 解锁的分析能力

1. **点赞总量统计** — 每场/每天/每月总点赞数
2. **用户点赞活跃度** — 频繁点赞的用户是潜在转化对象
3. **点赞时间分布** — 什么时候点赞峰值最高
4. **点赞-礼物关联** — 点赞后多久会送礼物？

---

### 1F. `RoomStatsMessage` 完整数据 — 中等优先级

#### 现有状态

当前只存了 `total`（数值），但 `RoomStatsMessage` 包含：

```python
RoomStatsMessage:
  display_short: string     # "1.2w"
  display_middle: string    # "12,345"
  display_long: string      # "12,345 人次观看"
  display_value: int64      # 纯数值（与 total 可能不同）
  display_type: int64       # 展示类型（观看/互动等）
  incremental: bool         # 是否为增量值
  total: int64              # 累计总数
```

#### 存储方案

```sql
ALTER TABLE sessions ADD COLUMN total_views INTEGER DEFAULT 0;
-- 或新建表
CREATE TABLE IF NOT EXISTS stats_snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id INTEGER REFERENCES sessions(id),
    snapshot_type TEXT NOT NULL,  -- 'views', 'likes', 'share'
    display_type INTEGER DEFAULT 0,
    value INTEGER DEFAULT 0,
    display_text TEXT DEFAULT '',
    is_incremental INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT (datetime('now', '+8 hours'))
);
```

#### 解锁的分析能力

1. **观看人数/场次曲线** — 直播过程中观看人次变化
2. **观看峰值时间** — 什么时间段观看量最大
3. **观看与消费关联** — 观看峰值和送礼峰值是否重叠？

---

## 2. 用户行为分析（User Behavior Analytics）

### 概述

利用已有数据（多场次的 `gift_logs`、`chat_logs`、`contributions`），构建跨场次的用户行为分析。不需要额外数据采集，全部基于现有 SQL 查询。

---

### 2.1 用户留存漏斗

#### 功能描述

追踪用户从首次出现到后续场次的留存情况，按消费层级（1000+/3000+/10000+）分层展示。

#### 数据来源

`gift_logs`（按 `user_id` 分组）+ `sessions`（按 `start_time` 排序）

#### SQL 示例

```sql
-- 用户首次出现场次（首访）
SELECT user_id, MIN(session_id) as first_session
FROM gift_logs GROUP BY user_id;

-- 用户第 N 场回流（留存）
WITH first AS (
    SELECT user_id, MIN(session_id) as fs,
           MIN(s.id) as first_session_id,
           MIN(s.start_time) as first_time
    FROM gift_logs g JOIN sessions s ON s.id = g.session_id
    GROUP BY user_id
)
SELECT
    f.first_session_id,
    COUNT(DISTINCT f.user_id) as total_users,
    COUNT(DISTINCT CASE WHEN g2.session_id = f.fs + 1 THEN g2.user_id END) as retained_1,
    COUNT(DISTINCT CASE WHEN g2.session_id = f.fs + 2 THEN g2.user_id END) as retained_2,
    COUNT(DISTINCT CASE WHEN g2.session_id <= f.fs + 5 THEN g2.user_id END) as retained_5
FROM first f
LEFT JOIN gift_logs g2 ON g2.user_id = f.user_id AND g2.session_id > f.fs
GROUP BY f.first_session_id
ORDER BY f.first_session_id DESC;
```

#### 页面展示

- **留存曲线** — 折线图展示第 1/3/5/10 场回流的留存率
- **分层漏斗** — 1000+ / 3000+ / 10000+ 层级分别展示留存
- **场次标注** — X 轴标注对应主播/日期

#### API 端点

```
GET /api/analytics/retention?anchor_name=xxx&min_consume=1000
```

---

### 2.2 消费速度分析

#### 功能描述

计算用户从首次发弹幕到首次送礼的时间差，识别"快转化"和"慢转化"用户。

#### 数据来源

`chat_logs`（首次聊天时间）→ `gift_logs`（首次送礼时间），按用户分组

#### SQL 示例

```sql
WITH first_chat AS (
    SELECT user_id, MIN(created_at) as first_chat_time
    FROM chat_logs GROUP BY user_id
),
first_gift AS (
    SELECT user_id, MIN(created_at) as first_gift_time
    FROM gift_logs GROUP BY user_id
)
SELECT
    fc.user_id,
    u.user_name,
    fc.first_chat_time,
    fg.first_gift_time,
    CAST(
        julianday(fg.first_gift_time) - julianday(fc.first_chat_time)
    ) * 86400 as seconds_to_first_gift
FROM first_chat fc
JOIN first_gift fg ON fg.user_id = fc.user_id
JOIN users u ON u.user_id = fc.user_id
WHERE fg.first_gift_time > fc.first_chat_time
ORDER BY seconds_to_first_gift;
```

#### 页面展示

- **转化时间分布直方图** — <1min / 1-5min / 5-30min / 30-60min / >60min
- **快转化用户列表** — 前 20 名最快的用户及其首礼
- **层间对比** — 高消费（10w+）vs 中消费（1w+）的转化速度差异

#### 解锁洞察

- 快转化用户 → 冲动消费型，可以重点互动
- 慢转化用户 → 观望型，需要更多引导

---

### 2.3 大 R 识别与生命周期价值（LTV）

#### 功能描述

按用户终身总消费排榜，标注消费趋势是加速还是减速。识别"沉默大 R"（曾经高消费但近期没来）。

#### 数据来源

`contributions`（每场消费）+ `monthly_stats`（月度统计）

#### SQL 示例

```sql
-- 用户最近 3 场 vs 之前 3 场消费对比（判断加速/减速）
WITH ranked AS (
    SELECT user_id, consume, session_id,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY session_id DESC) as rn
    FROM contributions
)
SELECT
    user_id,
    SUM(CASE WHEN rn <= 3 THEN consume ELSE 0 END) as recent_3,
    SUM(CASE WHEN rn BETWEEN 4 AND 6 THEN consume ELSE 0 END) as prev_3,
    CASE
        WHEN SUM(CASE WHEN rn <= 3 THEN consume ELSE 0 END) >
             SUM(CASE WHEN rn BETWEEN 4 AND 6 THEN consume ELSE 0 END) * 1.5 THEN '加速'
        WHEN SUM(CASE WHEN rn <= 3 THEN consume ELSE 0 END) <
             SUM(CASE WHEN rn BETWEEN 4 AND 6 THEN consume ELSE 0 END) * 0.5 THEN '减速'
        ELSE '稳定'
    END as trend
FROM ranked
GROUP BY user_id
HAVING SUM(CASE WHEN rn <= 3 THEN consume ELSE 0 END) > 10000
ORDER BY recent_3 DESC;
```

#### 页面展示

- **大 R 排行榜** — 终身总消费 + 近 3 场消费 + 趋势箭头（↑/↓/→）
- **沉默大 R 预警** — 最新消费距今 > 7 天的高价值用户
- **用户标签系统集成** — 自动建议标签如"大 R"、"加速中"、"流失风险"
- **单用户生命周期曲线** — 每场消费柱状图 + 累计消费折线图

---

### 2.4 用户分群

#### 功能描述

将用户按行为模式聚类为不同群体，自动标注。

#### 分群维度

| 群体 | 判定条件 | 特征 |
|------|---------|------|
| **大 R** | 总消费 ≥ 10w 或单场 ≥ 3w | 核心收入来源 |
| **中坚力量** | 总消费 1w-10w | 稳定贡献群体 |
| **频繁互动者** | 发言 ≥ 100 条但消费 < 1w | 氛围贡献者 |
| **礼多话少** | 送礼 ≥ 5 次且礼物金额 ≥ 3k 但发言 < 20 | 低调送礼型 |
| **只看不送** | 有发言但从未送过礼 | 潜在转化对象 |
| **仅PK送礼** | 礼物集中在有 PK 的场次 | PK 型消费者 |
| **福袋党** | 只有福袋口令发言 | 羊毛党 |

#### SQL 示例

```sql
SELECT
    g.user_id,
    SUM(g.diamond_total) as total_consume,
    COUNT(*) as gift_count,
    (SELECT COUNT(*) FROM chat_logs WHERE user_id = g.user_id) as chat_count,
    (SELECT COUNT(DISTINCT CASE WHEN content LIKE '%口令%' OR chat_by=9
        THEN id END) FROM chat_logs WHERE user_id = g.user_id) as lucky_bag_count
FROM gift_logs g
GROUP BY g.user_id;
```

#### 页面展示

- **饼图** — 各群体用户数占比
- **收入构成** — 各群体贡献的钻石占比
- **用户列表** — 每个群体可点击展开用户明细
- **群体迁移** — 用户从"只看不送"→"送过礼物"→"大 R"的转化路径

---

### 2.5 送礼-聊天关联分析

#### 功能描述

分析用户行为模式：送礼前/后是否有聊天？大礼物前后的聊天氛围有什么变化？

#### 分析维度

1. **送礼前聊天** — 送大礼前的最后 5/10/20 条消息内容
2. **送礼前后比较** — 送礼前 60s vs 送礼后 60s 的聊天数量变化
3. **群体效应** — 一个大礼物触发后，其他人是否也开始送礼

#### SQL 示例

```sql
-- 大礼物前后的聊天数量变化
WITH big_gifts AS (
    SELECT id, session_id, user_id, created_at
    FROM gift_logs
    WHERE diamond_total >= 10000  -- 大礼物：1w钻以上
)
SELECT
    bg.id as gift_log_id,
    bg.session_id,
    bg.user_id,
    (SELECT COUNT(*) FROM chat_logs
     WHERE session_id = bg.session_id
     AND created_at >= datetime(bg.created_at, '-60 seconds')
     AND created_at < bg.created_at) as chats_before,
    (SELECT COUNT(*) FROM chat_logs
     WHERE session_id = bg.session_id
     AND created_at > bg.created_at
     AND created_at <= datetime(bg.created_at, '+60 seconds')) as chats_after
FROM big_gifts bg
ORDER BY bg.id;
```

#### 页面展示

- **关联模式卡片** — "用户 X 在送礼前通常会发 Y 条弹幕"
- **大礼物热词** — 大礼物前的弹幕关键词云
- **时间窗口分析** — 从首次聊天到首次送礼的典型时长

---

## 3. 单场深度分析（Session Deep-Dive Analysis）

### 概述

将单场直播的数据按时间线精细化呈现，帮助主播/运营理解每一分钟发生了什么。

---

### 3.1 关键指标时间线

#### 功能描述

以 1 分钟为粒度，展示整场直播的核心指标变化曲线。

#### 数据来源

`gift_logs`（按分钟聚合钻石数）+ `chat_logs`（按分钟聚合消息数）

#### SQL 示例

```sql
-- 每分钟的钻石和聊天数
WITH minutes AS (
    SELECT
        strftime('%Y-%m-%d %H:%M', created_at) as minute_slot,
        SUM(COALESCE(diamond_total, 0)) as minute_diamonds,
        COUNT(*) as gift_count
    FROM gift_logs WHERE session_id = ?
    GROUP BY minute_slot
),
chat_minutes AS (
    SELECT
        strftime('%Y-%m-%d %H:%M', created_at) as minute_slot,
        COUNT(*) as chat_count
    FROM chat_logs WHERE session_id = ?
    GROUP BY minute_slot
)
SELECT
    COALESCE(m.minute_slot, c.minute_slot) as minute_slot,
    COALESCE(m.minute_diamonds, 0) as diamonds,
    COALESCE(m.gift_count, 0) as gifts,
    COALESCE(c.chat_count, 0) as chats
FROM minutes m
FULL OUTER JOIN chat_minutes c ON c.minute_slot = m.minute_slot
ORDER BY minute_slot;
```

#### API 端点

```
GET /api/sessions/<session_id>/hourly  -- 已有的，按小时聚合
—— 新增 ——
GET /api/sessions/<session_id>/minutely  -- 按分钟聚合
```

#### 展示方式

- **双轴折线图** — 钻石速率（左轴，柱状）+ 聊天速率（右轴，折线）
- **时间滑块** — 拖动选择时间段、缩放查看
- **关键事件标注** — 在时间线上标注 PK 开始/结束、大礼物时刻

---

### 3.2 活动热力图

#### 功能描述

以 30 分钟 × 用户层级为网格的热力图，直观展示"什么人在什么时候最活跃"。

#### 数据来源

`gift_logs` + `users.grade`（财富等级）

#### SQL 示例

```sql
SELECT
    CAST(strftime('%H', created_at) AS INTEGER) as hour_block,
    CAST(CAST(strftime('%M', created_at) AS INTEGER) / 30 AS INTEGER) * 30 as min_block,
    CASE
        WHEN diamond_total >= 10000 THEN '10w+'
        WHEN diamond_total >= 1000 THEN '1w+'
        WHEN diamond_total >= 100 THEN '1k+'
        ELSE '1k-'
    END as tier,
    COUNT(*) as count,
    SUM(diamond_total) as diamonds
FROM gift_logs WHERE session_id = ?
GROUP BY hour_block, min_block, tier
ORDER BY hour_block, min_block;
```

#### 展示方式

- **网格热力图**：X 轴 = 时间块，Y 轴 = 消费层级，颜色深浅 = 活动密度
- **钻级叠加**：不同层级用不同颜色叠加显示
- **可交互**：点击网格块显示该时间段的送礼明细

---

### 3.3 关键时刻自动检测

#### 功能描述

自动检测直播中的数据异常点，标记"发生了什么"。

#### 检测规则

| 规则 | 条件 | 标记 |
|------|------|------|
| **钻石峰值** | 某分钟钻石数 ≥ 均值 + 3σ | ⚡ 送礼风暴 |
| **聊天峰值** | 某分钟聊天数 ≥ 均值 + 3σ | 💬 弹幕风暴 |
| **大礼物** | 单笔礼物 ≥ 3000 钻 | 🎁 大礼物时刻 |
| **连击潮** | 同一用户 30s 内连续送礼 | 🔥 连击潮 |
| **PK 时刻** | PK 开始/结束时间段 | ⚔️ PK 时段 |
| **新人涌入** | 在线人数突然激增 | 📈 流量涌入 |
| **静默期** | 连续 N 分钟无礼物/聊天 | 💤 冷场检测 |

#### SQL/代码逻辑

```python
def detect_key_moments(session_id):
    # 1. 获取每分钟聚合数据
    minute_data = get_minute_aggregation(session_id)
    
    # 2. 计算均值和标准差
    diamond_values = [m['diamonds'] for m in minute_data]
    mean = statistics.mean(diamond_values)
    stdev = statistics.stdev(diamond_values) if len(diamond_values) > 1 else 0
    threshold = mean + 3 * stdev
    
    # 3. 标注峰值点
    for m in minute_data:
        if m['diamonds'] > threshold:
            m['events'].append('送礼风暴')
        
    # 4. 大礼物检测（直接查gift_logs）
    big_gifts = find_big_gifts(session_id, min_diamond=3000)
    
    # 5. PK时段检测（查pk_rounds表）
    pk_rounds = query_pk_rounds(session_id)
    
    return annotated_timeline
```

#### API 端点

```
GET /api/sessions/<session_id>/key-moments
```

返回值格式：

```json
{
  "moments": [
    {"time": "20:15", "type": "diamond_spike", "label": "送礼风暴",
     "diamonds": 85000, "top_gift": "嘉年华", "top_user": "用户A"},
    {"time": "20:18", "type": "big_gift", "label": "🎁 大礼物",
     "gift_name": "浪漫马车", "user_name": "用户B", "diamond": 30000},
    {"time": "20:20-20:25", "type": "pk", "label": "⚔️ PK",
     "duration": 300, "score": 120000, "result": "win"}
  ],
  "summary": {
    "total_moments": 8,
    "diamond_peak": "20:15 (85000钻)",
    "chat_peak": "20:16 (320条)",
    "cold_start": "20:35-20:40 (5分钟冷场)",
    "big_gift_count": 3
  }
}
```

---

### 3.4 用户行为时间轴（甘特图）

#### 功能描述

以场次内的时间为轴，展示每个用户的活动轨迹——什么时候进来、什么时候送礼、什么时候发言。

#### 数据来源

`gift_logs` + `chat_logs`，按用户和 session 分组

#### SQL

```sql
-- 获取该场次所有活跃用户的活动
SELECT user_id, user_name, created_at, 'chat' as activity_type, content as detail
FROM chat_logs WHERE session_id = ?
UNION ALL
SELECT user_id, user_name, created_at, 'gift' as activity_type,
       gift_name || ' x' || gift_count || ' (' || diamond_total || '钻)' as detail
FROM gift_logs WHERE session_id = ?
ORDER BY user_id, created_at;
```

#### 展示方式

- **甘特图**：Y 轴 = 用户（按总消费排序），X 轴 = 时间
- 每个用户一行，礼物 = 蓝色方块（大小=金额），聊天 = 灰色短线
- PK 时段背景色高亮
- 鼠标悬浮显示详情

---

### 3.5 单场 PK 分析

#### 功能描述

整合现有的 `pk_rounds` 表数据，展示每场 PK 的详细分析和影响力。

#### 数据来源

`pk_rounds` 表（已有）+ `gift_logs`（PK 时段的礼物）

#### 分析维度

| 维度 | 说明 |
|------|------|
| **PK 时间线** | 每轮 PK 的开始/结束/时长 |
| **PK 收入影响** | PK 时段的钻石速率 vs 非 PK 时段 |
| **PK 贡献者** | 每轮 PK 中送礼最多的用户 |
| **PK 胜率** | 所有 PK 的胜/负/平统计 |
| **对手分析** | 经常匹配到的对手及其胜负关系 |

#### SQL 示例

```sql
-- PK 时段 vs 非 PK 时段的钻石收入对比
WITH pk_periods AS (
    SELECT start_time, end_time, mode
    FROM pk_rounds WHERE session_id = ?
)
SELECT
    'PK时段' as period,
    COUNT(*) as gift_count,
    SUM(g.diamond_total) as total_diamonds,
    CAST(SUM(g.diamond_total) AS REAL) /
        (SELECT COALESCE(SUM(julianday(end_time) - julianday(start_time)) * 86400, 1)
         FROM pk_periods) as diamonds_per_second
FROM gift_logs g
WHERE g.session_id = ?
AND EXISTS (
    SELECT 1 FROM pk_periods p
    WHERE g.created_at >= p.start_time
    AND g.created_at <= p.end_time
)
UNION ALL
SELECT
    '非PK时段' as period,
    COUNT(*),
    SUM(g.diamond_total),
    CAST(SUM(g.diamond_total) AS REAL) /
        ((SELECT COALESCE(SUM(julianday(end_time) - julianday(start_time)) * 86400, 1)
          FROM pk_periods))
FROM gift_logs g
WHERE g.session_id = ?
AND NOT EXISTS (
    SELECT 1 FROM pk_periods p
    WHERE g.created_at >= p.start_time
    AND g.created_at <= p.end_time
);
```

#### Web 页面展示

- **PK 胜率卡片** — 总场次、胜率、连胜/连败
- **PK 收入对比** — PK 时段 vs 非 PK 时段的钻石速率对比柱状图
- **单轮 PK 展开** — 点击展开每轮 PK 的贡献榜
- **关键对手** — 高频对手列表

---

### 3.6 单场变更汇总（Session Changes Summary）

#### 功能描述

将现有 `upgrade_logs` 表的数据展示在单场详情页——这场直播中谁升级了财富等级、谁加入了粉丝团。

#### 数据来源

`upgrade_logs` 表（已有，类型含 `grade_upgrade` 和 `fansclub_join/upgrade`）

#### SQL

```sql
SELECT u.*, ua.nickname
FROM upgrade_logs u
LEFT JOIN users ua ON ua.user_id = u.user_id
WHERE u.session_id = ?
ORDER BY u.created_at DESC;
```

#### 展示方式

- **升级滚动条** — 直播过程中的升级事件实时滚动（SSE）
- **升级总结卡片** — 本场共 X 人升级财富等级，Y 人加入粉丝团

---

## 总结：实现路线图

### 第一阶段（基础设施 — 2-3天）

1. ✅ `gift_logs` 扩展字段（`grade`, `fans_club`, `send_time`, `gift_type`）
2. ✅ `users` 表新增 `follower_count`, `following_count`, `signature`, `city`
3. ✅ 新增 `like_logs` 表 + 入库
4. ✅ 新增 `stats_snapshots` 表 + 入库

### 第二阶段（分析与 API — 3-5天）

5. 新增 `/api/analytics/*` 路由：
   - `retention` — 留存分析
   - `spending-velocity` — 消费速度
   - `segmentation` — 用户分群
   - `ltv` — 大 R 生命周期
6. 新增 `/api/sessions/<id>/minutely` — 分钟级聚合
7. 新增 `/api/sessions/<id>/key-moments` — 关键时刻检测

### 第三阶段（前端可视化 — 3-5天）

8. 用户行为分析页面（多 tab）
9. 单场深度分析页面（时间线、热力图、关键时刻列表）
10. 用户详情页增强（升级进度条、粉丝数、城市、签名）

### 总计预估

- 后端：~500 行 Python
- 前端：~800 行 HTML/JS
- 新增文件：~5 个（3 个 HTML 模板 + 1 个分析模块 + DB 迁移）
