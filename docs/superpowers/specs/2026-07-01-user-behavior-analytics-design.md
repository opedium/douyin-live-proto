# 用户行为分析（User Behavior Analytics）功能设计文档

> 基于 `docs/features/feature-details-three-parts.md` 第一部分 · 第一期实现范围：
> **1.2.1 用户留存漏斗** + **1.2.3 大R识别与生命周期价值**

---

## 一、架构方案

### 决策：增量式扩展（推荐）

在现有 Flask + Jinja2 + Chart.js 架构上增量扩展，严格遵循已有模式：

| 层 | 模式 | 参考 |
|------|------|------|
| 数据查询 | `parser.py` 新增 `query_*` 函数 | `query_leaderboard()` (L2741)、`query_user_timeline()` (L3110) |
| API 路由 | `app.py` 新增 `/api/analytics/*` 返回 JSON | 现有 `/api/leaderboard`、`/api/user` 模式 |
| 页面路由 | `app.py` 新增 `/analytics` → 渲染模板 | 现有 `/leaderboard` → `leaderboard.html` |
| 前端图表 | Chart.js（已在 base.html 加载） | 现有所有页面共用 |
| 前端数据加载 | fetch + 原生 JS 渲染 | 现有 `loadData()` 模式 |

### 新增文件

```
base/parser.py     → 新增 3 个 query_* 函数
app.py             → 新增 1 个页面路由 + 3 个 API 路由
templates/
  analytics.html   → 新建，用户行为分析页面
```

### 路由表

| 方法 | 路径 | 用途 |
|------|------|------|
| GET | `/analytics` | 用户行为分析主页 |
| GET | `/api/analytics/retention` | 留存漏斗数据 |
| GET | `/api/analytics/big-spenders` | 大R识别数据 |
| GET | `/api/analytics/silent-whales` | 沉默大R预警列表 |
| GET | `/api/leaderboard/streamers` | 主播下拉列表（复用已有路由） |

---

## 二、第一期实现范围

### 2.1 用户留存漏斗（1.2.1）

#### 2.1.1 筛选条件

- **主播**：下拉，来自 `sessions.anchor_name`，**默认选中用户的主力主播**（页面加载时自动选择第一个有数据的 anchor_name）
- **时间范围**：近7天 / 近30天 / 近90天 / 全部
- **消费层级**：全部 / 1000+ / 3000+ / 10000+ / 100000+

> **重要：主播筛选策略**
> 
> 所有留存/大R分析查询必须限定在指定主播的 sessions 范围内。
> 原因是 `session_id` 是全局自增的，不同主播的场次 ID 会交错，如果不按主播过滤，
> "第N场"的计算会混入其他主播的数据，导致留存/趋势计算错误。
> 
> 当主播筛选为"全部"时，保留全局数据查看能力，但 SQL 不再使用
> `session_id` 顺序，而是按 `sessions.start_time` 时间排序来定义"第N场"。

#### 2.1.2 上方指标卡（4个）

| 指标 | SQL |
|------|-----|
| 总活跃用户数 | `SELECT COUNT(DISTINCT user_id) FROM gift_logs` |
| 首场用户数 | 首次出现（在筛选时间范围内第一次送礼）的用户数量 |
| 整体留存率（第2场） | `COUNT(回流用户) / COUNT(首场用户) × 100%` |
| 高价值用户留存率 | 消费 ≥ 10000 的用户的第2场留存 |

#### 2.1.3 留存曲线（折线图）

- X 轴：第 N 场（第1场/第2场/第3场/第4场/第5场）
- Y 轴：留存率 %
- 4 条曲线叠加对比：全部用户（灰）、1000+（蓝）、10000+（橙）、100000+（红）

SQL 实现（按主播过滤版本）：

```sql
-- 先取该主播的场次列表（按时间排序）
WITH target_sessions AS (
    SELECT id, ROW_NUMBER() OVER (ORDER BY start_time) as session_seq
    FROM sessions
    WHERE (? = '' OR anchor_name = ?)  -- ？即 anchor 参数
    AND (? = 'all' OR start_time >= datetime('now', '-' || ? || ' days', '+8 hours'))
),
-- 每个用户在该主播场次内的首次出现
first_session AS (
    SELECT g.user_id, MIN(s.session_seq) as fs_seq
    FROM gift_logs g
    JOIN target_sessions s ON s.id = g.session_id
    GROUP BY g.user_id
),
-- 计算留存
retention AS (
    SELECT
        f.user_id,
        (SELECT COUNT(*) FROM gift_logs g2
         JOIN target_sessions s2 ON s2.id = g2.session_id
         WHERE g2.user_id = f.user_id
         AND s2.session_seq = f.fs_seq + 1) > 0 as retained_1,
        (SELECT COUNT(*) FROM gift_logs g2
         JOIN target_sessions s2 ON s2.id = g2.session_id
         WHERE g2.user_id = f.user_id
         AND s2.session_seq <= f.fs_seq + 3) > 0 as retained_3,
        (SELECT COUNT(*) FROM gift_logs g2
         JOIN target_sessions s2 ON s2.id = g2.session_id
         WHERE g2.user_id = f.user_id
         AND s2.session_seq <= f.fs_seq + 5) > 0 as retained_5
    FROM first_session f
)
SELECT
    COUNT(*) as total_users,
    SUM(CASE WHEN retained_1 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as retain_1_rate,
    SUM(CASE WHEN retained_3 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as retain_3_rate,
    SUM(CASE WHEN retained_5 THEN 1 ELSE 0 END) * 100.0 / COUNT(*) as retain_5_rate
FROM retention;
```

> 核心变化：不再直接使用 `gift_logs.session_id` 的数字大小判断场次顺序，
> 而是通过 `sessions.start_time` 排序生成 `session_seq` 序号，
> 确保"第1场/第2场/..."是按时间正确排列的。

当 anchor 为空（查看全部）时，所有场次统一排序。

#### 2.1.4 详细表格

| 首场日期 | 主播 | 首场人数 | 回流第2场 | 回流第5场 | 总消费 |
|---------|------|---------|----------|----------|-------|

支持分页。

#### 2.1.5 参数语义

**`period`** — 过滤纳入计算的 `sessions` 的时间范围：
- `7d`：`sessions.start_time >= now - 7 days`
- `30d`：`sessions.start_time >= now - 30 days`
- `90d`：`sessions.start_time >= now - 90 days`
- `all`：不限制

**`anchor`** — 过滤纳入计算的 `sessions` 的主播：
- 非空值：只取 `sessions.anchor_name = ?` 的场次，场次排序按 `start_time` 重新编号
- 空值（"全部"）：取所有场次，统一按 `start_time` 排序

只有在这个时间窗口 + 主播范围内的 sessions 才会被用于计算 `first_session` 和 `retention`。

#### 2.1.6 API 结构

**`GET /api/analytics/retention?anchor=&period=30d&tier=0&page=1&size=20`**

```json
{
  "summary": {
    "total_active_users": 1234,
    "first_session_users": 456,
    "overall_retention_rate": 26.7,
    "high_value_retention_rate": 45.2
  },
  "retention_curve": {
    "labels": ["第1场", "第2场", "第3场", "第4场", "第5场"],
    "series": [
      {"name": "全部用户", "color": "#94a3b8", "data": [100, 26.7, 18.5, 12.3, 8.1]},
      {"name": "1000+",     "color": "#6366f1", "data": [100, 35.2, 22.1, 15.6, 10.4]},
      {"name": "10000+",    "color": "#f59e0b", "data": [100, 52.3, 38.7, 28.4, 21.5]},
      {"name": "100000+",   "color": "#ef4444", "data": [100, 68.5, 54.2, 42.1, 33.6]}
    ]
  },
  "details": [
    {
      "first_date": "2026-06-28",
      "anchor": "主播A",
      "first_users": 45,
      "retained_2": 12,
      "retained_2_rate": 26.7,
      "retained_5": 5,
      "retained_5_rate": 11.1,
      "total_consume": 128500
    }
  ],
  "page": 1,
  "total": 20
}
```

---

### 2.2 大R识别与生命周期价值（1.2.3）

#### 2.2.1 筛选条件

- **最低消费阈值**：默认 10000
- **趋势筛选**：全部 / 加速中 / 减速中
- **主播筛选**：下拉，**默认选中主力主播**，可选全部

> 趋势分析基于 `contributions` 表 JOIN `sessions` 表，按 `anchor` 过滤。
> 只取该主播场次内的消费记录，跨场次的"近3场 vs 前3场"对比在该主播的
> 场次时间线内进行（按 `sessions.start_time` 排序）。

#### 2.2.2 主要图表 — 用户终身价值排行（水平条形图）

- X 轴 = 累计钻石数
- Y 轴 = 用户名
- 条形颜色标注趋势：🟢 加速 / 🟡 稳定 / 🔴 减速

趋势判定逻辑：

```python
if recent_3 > prev_3 * 1.5:  'accelerating'
elif recent_3 < prev_3 * 0.5: 'decelerating'
else:                         'stable'
```

#### 2.2.3 详细表格

| 排名 | 用户 | 粉丝团 | 总消费 | 近3场 | 前3场 | 趋势 | 活跃天数 | 沉默天数 |
|------|------|--------|-------|-------|-------|------|---------|---------|

#### 2.2.4 沉默大R预警区块

- 在该主播场次内总消费 ≥ 30000 且最近一次出现距今 ≥ 7 天的用户
- 红色标记的卡片列表，点击可跳转用户详情页 `/user?uid=...`
- 当主播筛选为"全部"时，统计全局消费

#### 2.2.5 API 结构

**`GET /api/analytics/big-spenders?min_consume=10000&trend=all&anchor=&page=1&size=50`**

```json
{
  "users": [
    {
      "rank": 1,
      "user_id": "12345",
      "user_name": "用户A",
      "fans_club": "Lv25",
      "total_consume": 528000,
      "recent_3": 185000,
      "prev_3": 92000,
      "trend": "accelerating",
      "active_days": 45,
      "silent_days": 2,
      "avatar_url": "...",
      "sec_uid": "..."
    }
  ],
  "total": 15,
  "page": 1
}
```

**`GET /api/analytics/silent-whales?threshold=30000&silent_days=7`**

```json
{
  "users": [
    {
      "user_id": "67890",
      "user_name": "用户B",
      "total_consume": 85000,
      "last_seen": "2026-06-20 21:30:00",
      "silent_days": 11,
      "avatar_url": "..."
    }
  ]
}
```

---

## 三、页面模板设计（`analytics.html`）

### 布局结构

```
┌─────────────────────────────────────────────────┐
│  导航 Tab:  [留存漏斗]  [大R识别]                │
├─────────────────────────────────────────────────┤
│  筛选条: 主播 ▼  时间范围 ▼  消费层级 ▼  (刷新) │
├─────────────────────────────────────────────────┤
│  ┌─ 留存漏斗 Tab (默认) ──────────────────────┐ │
│  │  指标卡 x4                                   │ │
│  │  ┌── 留存曲线折线图 ──────────────────────┐ │ │
│  │  │  Chart.js line chart                     │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  │  ┌── 详细表格 ────────────────────────────┐ │ │
│  │  │  分页表格                                │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────┘ │
│  ┌─ 大R识别 Tab ──────────────────────────────┐ │
│  │  ┌── LTV水平条形图 ───────────────────────┐ │ │
│  │  │  Chart.js horizontal bar                │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  │  ┌── 详细表格 ────────────────────────────┐ │ │
│  │  │  分页表格                                │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  │  ┌── 沉默大R预警 ─────────────────────────┐ │ │
│  │  │  红色卡片列表，点击跳转用户详情         │ │ │
│  │  └─────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### 默认主播选中逻辑

页面加载时：
1. 调用 `/api/leaderboard/streamers` 获取所有主播列表
2. 自动选中**列表中的第一个主播**（通常是用户的主力/唯一主播）
3. 如果没有任何主播数据，显示空状态
4. 用户可随时切换主播下拉框，切换后重新加载当前 Tab 数据

> 这是为了适配"主要分析单一主播"的使用场景，减少每次手动选择的操作。

### Tab 切换

前端 JS 控制 `display: none/block`，每个 Tab 独立通过 `fetch()` 加载对应的 API。
筛选条件变化时重新加载当前 Tab 数据。

### 使用的 CSS 类

全部复用 `base.html` 中已有的类：
- `.card` / `.stat-card` / `.num` / `.label` — 指标卡
- `.table-wrap` / `table` / `thead th` / `td` — 表格
- `.filter-bar` / `.btn-group` / `.threshold-label` — 筛选条
- `.loading` / `.spinner` / `.empty-state` — 状态展示
- `.badge` / `.badge-green` / `.badge-red` / `.badge-amber` — 趋势标签

### 新增 CSS（约 30 行，放在 analytics.html 的 `<style>` 块内）

```css
/* Tab 导航样式 */
.analytics-tabs { display: flex; gap: 0; border-bottom: 2px solid var(--border); margin-bottom: 16px; }
.analytics-tab { padding: 10px 20px; cursor: pointer; font-size: 14px; font-weight: 600;
  color: var(--text-muted); border-bottom: 2px solid transparent; margin-bottom: -2px;
  transition: all var(--transition-fast); }
.analytics-tab:hover { color: var(--text); }
.analytics-tab.active { color: var(--primary); border-bottom-color: var(--primary); }

/* 沉默预警卡片 */
.silent-whale-card { display: flex; align-items: center; gap: 12px;
  padding: 12px 16px; background: var(--danger-bg); border-radius: var(--radius);
  margin-bottom: 8px; border-left: 4px solid var(--danger); cursor: pointer;
  transition: transform var(--transition-fast); }
.silent-whale-card:hover { transform: translateX(4px); }
```

---

## 四、数据流

```
用户操作 (切Tab/改筛选)
  → JS fetch(`/api/analytics/${tab}?${params}`)
  → Flask route (app.py) → 调用 parser.py 的 query_* 函数
  → SQLite 查询
  → 返回 JSON
  → JS 解析 JSON → 更新 Chart.js + 表格 HTML
```

### 错误处理链

- **API 层**：Flask route try/except → 返回 `{"error": "message"}` + 500
- **前端 fetch**：`.catch()` → toast 通知 + 图表区域显示 "数据加载失败"
- **空数据**：API 正常返回空数组 → 前端显示 empty-state
- **参数无效**：服务端验证参数 → 400 JSON

---

## 五、后续迭代计划（不在本期范围内）

| 子功能 | 预期工作量 | 前置条件 |
|--------|-----------|---------|
| 1.2.2 消费速度分析 | 中型（chat_logs + gift_logs 关联查询） | 无 |
| 1.2.4 用户分群 | 大型（多维度 SQL + 桑基图复杂） | 留存漏斗上线后 |
| 1.2.5 送礼-聊天关联 | 中型（时间窗口关联分析） | 消费速度上线后 |

---

## 六、实现顺序（build sequence）

1. **`parser.py` 新增 4 个 `query_*` 函数**
   - `query_user_retention(anchor, period, tier, page, size)`
   - `query_big_spenders(min_consume, trend, anchor, page, size)`
   - `query_silent_whales(threshold, silent_days)`

2. **`app.py` 新增路由**
   - 页面路由 `/analytics` → `analytics.html`
   - 3 个 API 路由

3. **创建 `templates/analytics.html`**
   - 页面骨架 + Tab 切换
   - 留存漏斗 Tab（图表 + 指标卡 + 表格）
   - 大R识别 Tab（图表 + 表格 + 沉默预警）
   - 筛选联动逻辑

4. **导航栏添加链接**
   - `base.html` 中新增 "用户分析" 导航项

5. **验证**
   - 手动测试各 Tab 数据加载
   - 测试筛选联动
   - 测试空数据/错误状态
