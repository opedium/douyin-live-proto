# 三项功能详细规格说明书

基于原始功能列表，深入代码库已有数据，说明每项功能具体展示什么内容。

---

## 一、用户行为分析（User Behavior Analytics）

> 对应原始列表 **1.2 用户行为分析（高价值）**

### 现有数据基础

所有分析均基于**已有数据**：
- `gift_logs` — 每条送礼记录（用户、礼物、金额、时间、场次）
- `chat_logs` — 每条弹幕记录（用户、内容、时间、场次）
- `contributions` — 每场每用户消费汇总（consume、rank）
- `sessions` — 场次信息（时间、主播）
- `users` — 用户信息（等级、粉丝团、首次/最后出现时间）
- `daily_stats` / `monthly_stats` — 天/月级预聚合

---

### 1.2.1 用户留存漏斗

#### 展示内容

**页面标题：** 用户留存分析

**筛选条件：**
- 主播选择（下拉，从 `sessions.anchor_name` 获取）
- 时间范围（近7天/30天/90天/全部）
- 消费层级（全部/1000+/3000+/10000+ 四档）

**上方指标卡（4个）：**
- 总活跃用户数：`SELECT COUNT(DISTINCT user_id) FROM gift_logs`
- 首场用户数：首次出现的用户数量
- 整体留存率（第2场）：`COUNT(回流用户) / COUNT(首场用户) × 100%`
- 高价值用户留存率：消费 ≥ 10000 的用户的第2场留存

**主要图表 — 留存曲线（折线图）：**

X 轴为"第 N 场"，Y 轴为"留存率 %"。

绘制 4 条曲线叠加对比：
- 所有用户（灰色）
- 1000+ 用户（蓝色）
- 10000+ 用户（橙色）
- 100000+ 用户（红色）

数据来源 SQL：
```sql
WITH first_session AS (
    SELECT user_id, MIN(session_id) as fs
    FROM gift_logs GROUP BY user_id
),
retention AS (
    SELECT
        f.user_id,
        f.fs,
        (SELECT COUNT(*) FROM gift_logs g2
         WHERE g2.user_id = f.user_id
         AND g2.session_id = f.fs + 1) > 0 as retained_1,
        (SELECT COUNT(*) FROM gift_logs g2
         WHERE g2.user_id = f.user_id
         AND g2.session_id <= f.fs + 3) > 0 as retained_3,
        (SELECT COUNT(*) FROM gift_logs g2
         WHERE g2.user_id = f.user_id
         AND g2.session_id <= f.fs + 5) > 0 as retained_5
    FROM first_session f
)
SELECT
    COUNT(*) as total_users,
    SUM(CASE WHEN retained_1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as retain_1_rate,
    SUM(CASE WHEN retained_3 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as retain_3_rate,
    SUM(CASE WHEN retained_5 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as retain_5_rate
FROM retention;
```

**下方详细表格：**

| 首场日期 | 主播 | 首场人数 | 回流第2场 | 回流第5场 | 总消费 | 
|---------|------|---------|----------|----------|-------|
| 2026-06-28 | 主播A | 45 | 12 (26.7%) | 5 (11.1%) | 128,500 |

---

### 1.2.2 消费速度分析

#### 展示内容

**页面标题：** 消费转化速度

**核心指标（3个）：**
- 平均转化时间：所有送礼用户从首条弹幕到首件礼物的平均秒数
- 中位转化时间：避免极端值干扰
- <1分钟转化率：首条弹幕发出后1分钟内就送礼的用户占比

**主要图表 — 转化时间分布（直方图）：**

X 轴分段：< 1min / 1-5min / 5-30min / 30-60min / 1-24h / > 24h
Y 轴：用户数量

数据来源 SQL：
```sql
WITH first_chat AS (
    SELECT user_id, MIN(created_at) as first_chat_time
    FROM chat_logs GROUP BY user_id
),
first_gift AS (
    SELECT user_id, MIN(created_at) as first_gift_time
    FROM gift_logs GROUP BY user_id
),
conversion AS (
    SELECT
        fc.user_id,
        u.user_name,
        fc.first_chat_time,
        fg.first_gift_time,
        (julianday(fg.first_gift_time) - julianday(fc.first_chat_time)) * 86400 as seconds
    FROM first_chat fc
    JOIN first_gift fg ON fg.user_id = fc.user_id
    JOIN users u ON u.user_id = fc.user_id
    WHERE fg.first_gift_time > fc.first_chat_time
)
SELECT
    CASE
        WHEN seconds < 60 THEN '<1min'
        WHEN seconds < 300 THEN '1-5min'
        WHEN seconds < 1800 THEN '5-30min'
        WHEN seconds < 3600 THEN '30-60min'
        WHEN seconds < 86400 THEN '1-24h'
        ELSE '>24h'
    END as bucket,
    COUNT(*) as user_count,
    ROUND(AVG(seconds), 0) as avg_seconds
FROM conversion
GROUP BY bucket
ORDER BY MIN(seconds);
```

**下方表格 — 最快转化用户 Top 20：**

| 排名 | 用户 | 首条弹幕 | 首件礼物 | 转化时长 | 首礼名称 | 总消费 |
|------|------|---------|---------|---------|---------|-------|
| 1 | 用户A | 20:01:15 | 20:01:35 | 20秒 | 墨镜 | 35,000 |

**中层卡片 — 层级对比：**

展示大 R（10w+）vs 普通用户（1w-）的转化速度差异：
- 大 R 平均转化：3.2 分钟
- 普通用户平均转化：45.8 分钟
- 结论：大 R 决策更快，进直播间不久就开始消费

---

### 1.2.3 大 R 识别与生命周期价值

#### 展示内容

**页面标题：** 大 R 识别与趋势

**筛选：**
- 最低消费阈值（默认 10000）
- 加速/减速/全部 筛选

**主要图表 — 用户终身价值排行（条形图）：**

X 轴 = 用户名，Y 轴 = 累计钻石数
每个条形用颜色标注趋势：🟢加速 / 🟡稳定 / 🔴减速

数据来源 SQL：
```sql
-- 判断用户消费趋势：近3场 vs 前3场
WITH ranked AS (
    SELECT user_id, consume, session_id,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY session_id DESC) as rn
    FROM contributions
    WHERE consume > 0
),
trend AS (
    SELECT
        user_id,
        SUM(CASE WHEN rn <= 3 THEN consume ELSE 0 END) as recent_3,
        SUM(CASE WHEN rn BETWEEN 4 AND 6 THEN consume ELSE 0 END) as prev_3,
        SUM(consume) as total_consume
    FROM ranked
    WHERE rn <= 6
    GROUP BY user_id
    HAVING total_consume >= 10000
)
SELECT
    t.*,
    u.user_name,
    u.fans_club,
    CASE
        WHEN recent_3 > prev_3 * 1.5 THEN 'accelerating'
        WHEN recent_3 < prev_3 * 0.5 THEN 'decelerating'
        ELSE 'stable'
    END as trend
FROM trend t
JOIN users u ON u.user_id = t.user_id
ORDER BY total_consume DESC;
```

**下方详细表格：**

| 排名 | 用户 | 粉丝团 | 总消费 | 近3场 | 前3场 | 趋势 | 活跃天数 | 沉默天数 |
|------|------|--------|-------|-------|-------|------|---------|---------|
| 1 | 用户A | Lv25 | 528,000 | 185,000 | 92,000 | ↑加速 | 45 | 2 |
| 5 | 用户B | Lv18 | 210,000 | 12,000 | 85,000 | ↓减速 | 38 | 14 |

**沉默大 R 预警区块：**
- 总消费 ≥ 30000 且最近一次出现距今 ≥ 7 天的用户列表
- 红色标记，点击可跳转用户详情页
- 数据：`SELECT FROM gift_logs GROUP BY user_id HAVING MAX(created_at) < datetime('now', '-7 days', '+8 hours')`

---

### 1.2.4 用户分群

#### 展示内容

**页面标题：** 用户分群

**分群逻辑自动计算：**

以当前所有有送礼记录的用户为分母，按规则分类：

| 群体 | SQL 判定条件 | 预期占比 |
|------|-------------|---------|
| 🐳 **大 R** | 总消费 ≥ 100000 | ~2% |
| 💪 **中坚力量** | 总消费 10000-99999 | ~15% |
| 🗣️ **频繁互动者** | 总消费 < 10000 且发言 ≥ 100 | ~20% |
| 🤫 **礼多话少** | 送礼金额 ≥ 3000 且送礼 ≥ 3次 且发言 < 20 | ~8% |
| 👀 **只看不送** | 有发言且从未送礼 或 送礼 < 1000 | ~40% |
| 🎁 **仅PK送礼** | 消费 > 0 且 只在与 PK 同期的场次有送礼记录 | ~5% |
| 🤖 **福袋党** | 只有福袋口令发言，从未送礼 | ~10% |

**主要图表（2个）：**
1. **环形图** — 各群体用户数占比
2. **收入构成堆叠柱状图** — 各群体贡献的钻石总额及百分比

**详细表格：**

| 群体 | 人数 | 占比 | 总消费 | 人均消费 | 送礼次数 |
|------|------|------|--------|---------|---------|
| 大 R | 5 | 2% | 895,000 | 179,000 | 423 |

**群体迁移路径（桑基图原型）：**

追踪用户从"只看不送" → "送过礼物" → "中坚力量" → "大 R" 的转化路径
- 显示每个节点的转化人数
- 显示转化率（如：30% 的"只看不送"用户在 5 场内送了第一件礼物）

---

### 1.2.5 送礼-聊天关联

#### 展示内容

**页面标题：** 送礼-聊天关联分析

**主要图表 — 大礼物前后聊天量对比（柱状图）：**

选取所有 ≥ 10000 钻的单笔礼物，统计其前后各 60s 的聊天数量。

X 轴 = 时间偏移（-60s ~ +60s），Y 轴 = 平均聊天数
以礼物时刻（0）为分界线，左右对比。

**中层卡片 — 关联模式总结：**

- 送礼前有聊天的概率：76%（大礼物前 60s 内发过至少一条弹幕）
- 送礼后聊天激增倍率：2.3x（大礼物后 60s 内聊天数是前 60s 的 2.3 倍）
- 最常在大礼物前出现的关键词："666"、"帅"、"牛"、"来了"

**下方表格 — 大礼物前后行为明细：**

| 时间 | 用户 | 礼物 | 金额 | 前60s聊天数 | 后60s聊天数 | 前60s聊天内容 |
|------|------|------|------|------------|------------|--------------|
| 20:15 | 用户A | 嘉年华 | 30,000 | 3 | 12 | "来了来了" "冲冲冲" "好看" |

---

## 二、单场深度分析（Session Deep-Dive Analysis）

> 对应原始列表 **1.3 单场深度分析**

### 现有数据基础

- `gift_logs WHERE session_id = ?` — 该场所有送礼记录
- `chat_logs WHERE session_id = ?` — 该场所有弹幕
- `pk_rounds WHERE session_id = ?` — PK 回合记录
- `sessions WHERE id = ?` — 场次基础信息

---

### 1.3.1 直播时间线滑动器

#### 展示内容

**页面标题：** 单场时间线 — 主播A #2026-06-28

**顶部 — 场次摘要：**
- 主播名、日期、时长（从 `sessions.start_time` / `end_time` 计算）
- 总钻石、总礼物数、总弹幕数（从 `gift_logs` / `chat_logs` COUNT）
- PK 场次数（从 `pk_rounds` COUNT）

**主要交互组件 — 时间滑块：**

一个可拖动的滑块，底部为时间轴（从开播到结束，按分钟标记）。

滑块上方有 3 个联动图表：

**图表 1：钻石速率（柱状图）**
- X 轴 = 分钟
- Y 轴 = 该分钟内的钻石总额
- 超过均值 + 2σ 的柱子标记为红色高亮
- 数据来源：
```sql
SELECT
    strftime('%H:%M', created_at) as minute_slot,
    SUM(diamond_total) as diamonds,
    COUNT(*) as gift_count
FROM gift_logs WHERE session_id = ?
GROUP BY minute_slot
ORDER BY minute_slot;
```

**图表 2：聊天速率（折线图，叠加在图表1上）**
- X 轴与图表1对齐
- Y 轴（右轴）= 该分钟内的弹幕数
- 数据来源：
```sql
SELECT
    strftime('%H:%M', created_at) as minute_slot,
    COUNT(*) as chat_count
FROM chat_logs WHERE session_id = ?
GROUP BY minute_slot
ORDER BY minute_slot;
```

**图表 3：送礼用户分布（堆叠柱状图）**
- X 轴 = 分钟
- Y 轴 = 送礼人数
- 堆叠按用户等级分段显示

**时间轴标注（在图表下方以竖线/色块标注）：**
- 🟣 PK 时段（从 `pk_rounds` 获取 start_time/end_time）
- 🟢 大礼物时刻（单笔 ≥ 5000 钻）
- ⚪ 鼠标悬停显示该分钟详情

**鼠标悬停 Popover 显示：**
```
20:15 — 峰值时刻
钻石: 85,000 (全场最高)
礼物: 嘉年华 x2 (用户A), 跑车 x1 (用户B)
弹幕: 124 条
PK: 第2轮PK进行中
```

---

### 1.3.2 活动热力图

#### 展示内容

**页面标题：** 活动热力图

**图表 — 热力图网格：**

X 轴 = 时间（30分钟一个区块），如 20:00-20:29、20:30-20:59...
Y 轴 = 消费层级，从上到下：10w+ / 1w+ / 1k+ / 1k-

每个格子的颜色深浅 = 该层级在该时间段的礼物总钻石数。

颜色映射：浅黄（0钻） → 橙（1000钻） → 红（10000钻） → 深紫（100000钻）

鼠标悬停显示：
```
20:30-20:59 | 1w+ 层级
送礼: 12 次
总钻石: 45,000
活跃用户: 3 人
主要礼物: 浪漫马车, 墨镜x5
```

数据来源 SQL：
```sql
SELECT
    CAST(strftime('%H', created_at) AS INTEGER) * 60 +
    CAST(CAST(strftime('%M', created_at) AS INTEGER) / 30 AS INTEGER) * 30 as minute_of_day,
    CASE
        WHEN diamond_total >= 10000 THEN '10w+'
        WHEN diamond_total >= 1000 THEN '1w+'
        ELSE '1k-'
    END as tier,
    COUNT(*) as gift_count,
    SUM(diamond_total) as total_diamonds,
    COUNT(DISTINCT user_id) as user_count
FROM gift_logs WHERE session_id = ?
GROUP BY minute_of_day, tier
ORDER BY minute_of_day, tier;
```

**右侧附加信息 — 热力图总结：**
- 最活跃时段：20:30-21:00（钻石占全场的 45%）
- 高价值用户活跃时段：20:00-21:30（10w+ 层级的格子最多的时段）
- 冷场时段：21:30-21:45（连续 15 分钟无送礼）

---

### 1.3.3 关键时刻检测

#### 展示内容

**页面标题：** 关键时刻

**顶部 — 总结卡片（6个关键数字）：**

| 指标 | 数值 | 说明 |
|------|------|------|
| ⚡ 送礼风暴 | 3 次 | 钻石速率超过均值+3σ的时刻 |
| 🎁 大礼物 | 5 件 | 单笔 ≥ 3000 钻的礼物 |
| ⚔️ PK 轮次 | 3 轮 | 本场 PK 次数 |
| 🔥 连击潮 | 2 次 | 同一用户 30s 内连送 ≥ 3 次 |
| 💬 弹幕潮 | 4 次 | 聊天速率超过均值+3σ的时刻 |
| 💤 冷场 | 1 次 | 连续 ≥ 5 分钟无活动 |

**主要模块 — 关键时刻时间轴（垂直时间线样式）：**

按时间顺序排列所有关键事件，每个事件是一个卡片：

```
 ⚡ 20:15 — 送礼风暴
 ├ 钻石: 85,000 (均值2,300的37倍)
 ├ 触发: 用户A 送出 嘉年华 x2
 ├ 弹幕: 124条 (前1分钟)
 └ PK: 第2轮 PK 进行中

 🎁 20:18 — 大礼物
 ├ 用户B 送出 浪漫马车 (30,000钻)
 ├ 当前榜一: 用户A (128,000钻)
 └ 距离前一个礼物: 3分钟

 ⚔️ 20:20-20:25 — PK 第3轮
 ├ 时长: 5分钟
 ├ 模式: 1v1
 ├ 我方得分: 185,000
 ├ 对手得分: 92,000
 └ 结果: 胜

 💤 21:35-21:40 — 冷场
 ├ 持续: 5分钟
 ├ 无送礼、无PK
 └ 弹幕: 仅2条
```

**检测逻辑（Python）：**

```python
def detect_key_moments(session_id):
    moments = []
    
    # 1. 钻石峰值 > 均值+3σ
    minute_data = get_minute_aggregation(session_id)  # 见1.3.1
    values = [m['diamonds'] for m in minute_data]
    if len(values) > 3:
        mean = sum(values) / len(values)
        variance = sum((v - mean)**2 for v in values) / len(values)
        std = variance ** 0.5
        threshold = mean + 3 * std
        for m in minute_data:
            if m['diamonds'] > threshold:
                # 查该分钟的大礼物
                big = conn.execute("""
                    SELECT user_name, gift_name, diamond_total
                    FROM gift_logs WHERE session_id = ?
                    AND created_at LIKE ? || '%'
                    ORDER BY diamond_total DESC LIMIT 3
                """, (session_id, m['minute'])).fetchall()
                moments.append({
                    'type': 'diamond_spike',
                    'time': m['minute'],
                    'diamonds': m['diamonds'],
                    'threshold': round(threshold),
                    'top_gifts': [dict(r) for r in big]
                })
    
    # 2. 大礼物检测
    big_gifts = conn.execute("""
        SELECT g.*, s.anchor_name FROM gift_logs g
        JOIN sessions s ON s.id = g.session_id
        WHERE g.session_id = ? AND g.diamond_total >= 3000
        ORDER BY g.created_at
    """, (session_id,)).fetchall()
    
    # 3. PK 时段（从已有 pk_rounds 表）
    pk_rounds = query_pk_rounds(session_id)
    
    # 4. 冷场检测（连续 N 分钟无送礼）
    # 5. 聊天峰值检测（同钻石峰值的 3σ 逻辑）
    # 6. 连击潮（同一用户短时间内多次送礼）
    
    return sorted(moments, key=lambda x: x['time'])
```

---

## 三、外部数据增强 — 未存储字段详情

> 对应原始列表 **3.3 外部数据增强** + 最初探索发现的"已接收但未存储的字段"

---

### 3.3.1 用户 protobuf 字段补全

#### 当前已存储 vs 可补充字段对照表

| User 字段 | 当前状态 | 可补充字段 | 补充价值 |
|-----------|---------|-----------|---------|
| `nick_name` | ✅ 已存 | — | — |
| `user_id` | ✅ 已存 | — | — |
| `sec_uid` | ✅ 已存 | — | — |
| `avatar_thumb` | ✅ 已存 | `avatar_medium`, `avatar_large` | 高清头像展示 |
| `display_id` | ⚠️ 部分使用 | — | 抖音号搜索 |
| `gender` | ✅ 已存 | — | — |
| `grade` (level) | ✅ 已存 | `pay_grade.total_diamond_count` | 累计消费钻石 |
| | | `pay_grade.this_grade_min_diamond` | 本级起始值 |
| | | `pay_grade.this_grade_max_diamond` | 本级上限 |
| | | `pay_grade.upgrade_need_consume` | 升下级还需多少 |
| | | `pay_grade.score` | 经验值 |
| | | `pay_grade.grade_describe` | 等级名称/描述 |
| `fans_club` | ✅ 已存 | — | — |
| `follow_info` | ❌ 未存 | `follower_count` | 粉丝数 |
| | | `following_count` | 关注数 |
| | | `follow_status` | 关注关系 |
| `signature` | ❌ 未存 | — | 个人简介 |
| `city` | ❌ 未存 | — | 城市 |
| `birthday` | ❌ 未存 | — | 生日 |
| `verified` | ❌ 未存 | — | 认证状态 |
| `age_range` | ❌ 未存 | — | 年龄段 |
| `badge_image_list` | ❌ 未存 | — | 徽章图片 |

#### 方案示意图

每条消息到达时：

```
User 对象 → get_user_id() → 已存
         → get_user_name() → 已存
         → get_user_sec_uid() → 已存
         → get_user_avatar_url() → 已存
         → fmt_grade() → 只取 level，丢弃 pay_grade 其余9个字段 ❌
         → user.follow_info → 完全丢弃 ❌
         → user.signature → 完全丢弃 ❌
         → user.city → 完全丢弃 ❌
```

改造后 `upsert_user()` 扩展为：

```python
def upsert_user(user_id, user_name, grade='', fans_club='',
                sec_uid='', avatar_url='', badges='',
                # 新增参数（来自 User 对象）
                pay_grade_score=None,
                pay_grade_total_diamond=None,
                pay_grade_upgrade_need=None,
                follower_count=None,
                following_count=None,
                signature='',
                city='',
                avatar_medium='',
                avatar_large=''):
    # 更新 SQL 新增对应列…
```

---

### 3.3.2 礼物价格数据库（已实现部分 + 可扩展）

#### 当前实现

`gift_prices` 表已存在（见 `parser.py` 2050-2059）：
```sql
CREATE TABLE gift_prices (
    gift_name TEXT NOT NULL UNIQUE,
    gift_id INTEGER DEFAULT 0,
    diamond_count INTEGER NOT NULL DEFAULT 0,
    source TEXT DEFAULT 'auto',      -- 'override' | 'registry' | 'auto' | 'manual'
    is_limited_skin INTEGER DEFAULT 0,
    created_at ..., updated_at ...
);
```

已实现的功能：
- ✅ 自动从 `gift_registry.json` 导入礼物价格
- ✅ 手动覆盖（`_GIFT_PRICE_OVERRIDE` 字典）
- ✅ Web 面板管理（`/gift-prices` 页面）
- ✅ 价格变更时触发重新计算（`recalculate_gift_price()`）
- ✅ 变更日志（`price_change_log` 表）

#### 当前 `gift_logs` 表的问题

数据来源：
```
protobuf GiftStruct:
  name: string         → 礼物名称
  diamond_count: int32 → 单价（钻石）
  type: uint32         → 类型（面板礼物/特效礼物等）
  id: uint64           → 礼物 ID
  combo: bool          → 是否可连击
  for_fansclub: bool   → 是否粉丝团礼物
  describe: string     → 描述
  region: string       → 区域/系列
```

当前 `gift_logs` 表只存了：
```sql
gift_name TEXT, gift_count INTEGER, diamond_total INTEGER, group_id TEXT
```

**缺失字段及其价值：**

| 字段 | 价值 |
|------|------|
| `gift_id`（礼物 ID） | 即使礼物改名也能匹配到同一礼物 |
| `gift_type`（礼物类型，来自 GiftStruct.type） | 按类型分析收入（面板礼物 vs 特效礼物） |
| `for_fansclub` | 区分"粉丝团专属礼物"和普通礼物 |
| `send_time`（精确时间戳） | 毫秒级时间分析 |

#### 礼物类型分类体系

抖音礼物 type 字段（部分已知）：
- 0 = 普通面板礼物
- 1 = 特效礼物（全屏特效）
- 2 = 连击礼物
- 3 = 粉丝团礼物
- 4 = 互动礼物
- 5 = 限定/活动礼物

---

### 3.3.3 审计页面扩展

> 对应原始列表 **4.2 数据完整性面板**

#### 现有审计页面

当前页面（`/audit`）展示：
- 总礼物数、总钻石、总弹幕、场次数、送礼用户数
- 去重健康状态（原始消息数、通过数、拒绝数、拒绝率）
- 时间间隙检测（最近10场的 >5s 间隔）
- 损坏用户名检测

#### 扩展功能 A：数据空洞检测

**展示内容：**

页面增加"数据空洞检测"板块，每个场次一个卡片：

```
场次 #42 | 主播A | 2026-06-28
├ 直播时长: 2h 15min (20:00-22:15)
├ 礼物时间跨度: 75min (20:00-21:15)
├ 数据覆盖率: 55.6% ← 重点指标
├ 检测到空洞: 4 处
│
├ 最长空洞: 12min (21:03-21:15)
├ 第2空洞: 5min (20:45-20:50)
├ 第3空洞: 3min (21:30-21:33)
└ 第4空洞: 2min (20:30-20:32)
│
└ 推测原因:
   └ 21:03-21:15 — WebSocket 断线（高概率，持续12min无任何消息）
```

**空洞可视化（时间轴条状图）：**

```
20:00 ████████░░░░░░████░░░░░███████████████░░░ 21:15
       ↑开播     ↑空洞1   ↑空洞2      ↑空洞3   ↑最后礼物
```

绿色区块 = 有数据，空白 = 无数据。

数据来源 SQL：
```sql
SELECT created_at FROM gift_logs WHERE session_id = ? ORDER BY created_at;
```
然后在 Python 中计算前后两条记录的时间差 > 5s 的记为空洞。

冷场（无活动但 WebSocket 连接正常）vs 空洞（完全无数据）的区别：
- 空洞：连续 ≥ 60s 无任何类型消息（含聊天、礼物、统计等）
- 冷场：有聊天但无礼物，或聊天也很少

#### 扩展功能 B：重复标记

**展示内容：**

页面增加"可疑重复记录"板块，检测被去重逻辑遗漏的潜在重复：

```
检测到 3 条可疑重复记录：

[可能重复 #1] 用户A 在 20:01:15 和 20:01:16 各送了一个 墨镜
├ 时间差: 1秒
├ 礼物相同、金额相同、用户相同
└ 非连击礼物（combo=0）→ 可能重复

[可能重复 #2] 用户B 在 20:05:00 和 20:05:00.5 各送了一个 小心心
├ 时间差: 0.5秒
├ trace_id 不同但内容完全相同
└ 可能为 websocket 重放
```

检测逻辑：
```sql
-- 同一 session 内，同一用户、同一礼物、同一金额，在 3s 内出现两次
SELECT g1.*, g2.created_at as dup_time
FROM gift_logs g1
JOIN gift_logs g2 ON
    g1.session_id = g2.session_id
    AND g1.user_id = g2.user_id
    AND g1.gift_name = g2.gift_name
    AND g1.diamond_total = g2.diamond_total
    AND g1.id < g2.id
    AND strftime('%s', g2.created_at) - strftime('%s', g1.created_at) <= 3
    AND g1.gift_count = g2.gift_count
ORDER BY g1.created_at;
```

#### 扩展功能 C：异常场次检测

**展示内容：**

页面增加"异常场次检测"板块，将当前场次与历史基线对比：

```
异常基线基于近 20 场直播（排除自身）。

┌──────────────┬──────────┬──────────┬──────────┬──────────┐
│    指标      │ 本场#50  │ 历史均值 │ 标准差σ  │ 偏差程度 │
├──────────────┼──────────┼──────────┼──────────┼──────────┤
│ 总钻石       │ 985,000  │ 125,000  │ 85,000   │ +10.1σ ⚠️│ ← 异常高
│ 送礼用户数   │ 32       │ 18       │ 8        │ +1.8σ    │
│ 人均消费     │ 30,781   │ 6,944    │ 5,200    │ +4.6σ ⚠️│ ← 异常高
│ 总弹幕数     │ 1,850    │ 980      │ 420      │ +2.1σ    │
│ 场次时长(min)│ 135      │ 110      │ 35       │ +0.7σ    │
│ 钻石/分钟    │ 7,296    │ 1,136    │ 850      │ +7.2σ ⚠️│ ← 异常高
└──────────────┴──────────┴──────────┴──────────┴──────────┘

异常标记规则：
- |偏差| > 3σ → 红色 ⚠️（显著异常）
- |偏差| > 2σ → 黄色 ⚡（值得关注）
```

数据来源 SQL：
```sql
-- 统计某场指标
SELECT
    SUM(g.diamond_total) as total_diamonds,
    COUNT(DISTINCT g.user_id) as gift_users,
    COUNT(*) as gift_count,
    (SELECT COUNT(*) FROM chat_logs WHERE session_id = ?) as chat_count
FROM gift_logs g WHERE g.session_id = ?;

-- 历史均值与标准差（排除本场）
SELECT
    AVG(total_diamonds) as avg_diamonds,
    STDEV(total_diamonds) as std_diamonds,
    AVG(gift_users) as avg_users
FROM (
    SELECT g.session_id,
           SUM(g.diamond_total) as total_diamonds,
           COUNT(DISTINCT g.user_id) as gift_users
    FROM gift_logs g
    WHERE g.session_id != ?
    GROUP BY g.session_id
    ORDER BY g.session_id DESC
    LIMIT 20
);
```

---

## 附录：所有数据字段与代码位置对照

| 数据 | 消息类型 / API | protobuf 字段 / API 字段 | 代码位置 | 当前状态 |
|------|--------------|------------------------|---------|---------|
| 用户等级 | User | `pay_grade.level` | `utils.py:fmt_grade()` | ✅ 已存 |
| 累计总钻石 | User | `pay_grade.total_diamond_count` | `messages.py:225` | ❌ 未存 |
| 升级所需 | User | `pay_grade.this_grade_min/max_diamond` | `messages.py:234-235` | ❌ 未存 |
| 距下级还需 | User | `pay_grade.upgrade_need_consume` | `messages.py:245` | ❌ 未存 |
| 经验值 | User | `pay_grade.score` | `messages.py:249` | ❌ 未存 |
| 粉丝数 | User | `follow_info.follower_count` | `messages.py:202` | ❌ 未存 |
| 关注数 | User | `follow_info.following_count` | `messages.py:201` | ❌ 未存 |
| 签名 | User | `signature` | `messages.py:288` | ❌ 未存 |
| 城市 | User | `city` | `messages.py:297` | ❌ 未存 |
| 礼物 ID | GiftStruct | `id` | `messages.py:478` | ⚠️ 部分 |
| 礼物类型 | GiftStruct | `type` | `messages.py:483` | ❌ 未存 |
| 粉丝团礼物标记 | GiftStruct | `for_fansclub` | `messages.py:481` | ❌ 未存 |
| 礼物描述 | GiftStruct | `describe` | `messages.py:475` | ❌ 未存 |
| 送礼目标用户 | GiftMessage | `to_user` | `messages.py:541` | ❌ 未存 |
| 发送方式 | GiftMessage | `send_type` | `messages.py:550` | ❌ 未存 |
| 房间粉丝票 | GiftMessage | `room_fan_ticket_count` | `messages.py:546` | ❌ 未存 |
| 点赞 | LikeMessage | `count / total` | `messages.py:595-596` | ❌ 不入库 |
| 观看人数 | RoomStatsMessage | `display_long / display_value / display_type` | `messages.py:462-469` | ⚠️ 只存total |
| 贡献者精确分数 | RoomUserSeqMessageContributor | `exactly_score` | `messages.py:438` | ❌ 未存 |
| 用户本场消费 | PublicAreaCommon | `user_consume_in_room` | `messages.py:351` | ❌ 未存 |
| 用户本场送礼数 | PublicAreaCommon | `user_send_gift_cnt_in_room` | `messages.py:352` | ❌ 未存 |
