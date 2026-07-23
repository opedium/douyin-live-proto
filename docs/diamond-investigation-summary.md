# 抖音直播音浪采集精度：全面调查汇总

**日期**: 2026-05-24  
**目的**: 定位采集器音浪计数与真实值的误差根因，并尝试修复  
**关键结论**: 误差根因是 **OS 层 WebSocket TCP 帧丢失**，不是去重逻辑、不是未知消息类型、不是 Python GIL

---

## 1. 问题定义

**采集对象**: 抖音直播间 WebSocket 弹幕协议（Protobuf 编码的 PushFrame）  
**核心数据**: `GiftMessage` → `GiftStruct.diamond_count × count` 累加  
**已知误差**: 与手动录屏对账，单场直播钻石缺口 3-20%（取决于直播间流量密度）

## 2. 诊断链路（从怀疑到确认）

```
去重逻辑排查 → 无损
    ↓
未知消息类型审计 → 无遗漏收入源
    ↓
手动录屏逐礼物对账 → 高密度时段漏采更多
    ↓
PushFrame.seq_id 跟踪 → 帧序号跳跃 = 确凿帧丢失证据
    ↓
对比不同 rcvbuf / 线程模型 → 帧丢失从 5.2% 降到 3.3%，但无法归零
```

### 2.1 排除去重逻辑

- 去重逻辑基于 `(group_id, gift_name, repeat_count)` 三元组，99.9% 大礼物 `repeat_count=1`（单次发送），去重不影响
- 修复了一个 **counter reset bug**（repeat_count 非单调递增），但修复后精度提升仅 0.12%
- 52-64% 的消息被去重是正常现象（抖音每条礼物消息发 ~2 份）

### 2.2 排除未知消息类型遗漏

- 54 种未注册的 protobuf 消息类型全部审计分类
- 分类结果：PK/连麦 17 种、聊天/UI 15 种、礼物相关 14 种、其他 8 种
- 14 种礼物相关中：BindingGift 是 GiftMessage 的重复通知、LuckyBox 是红包支出、其余为 UI 展示/特效
- **无遗漏收入源**

### 2.3 手动录屏对账（铁证）

孙恩盛 2026-05-23 无声总榜，逐礼物人工计数：

| 礼物类型 | 实际收到 | 采集器捕获 | 捕获率 |
|---------|---------|-----------|-------|
| 穹顶之巅 | 27 | 27 | 100% |
| 飞机 | 41 | 35 | 85.4% |
| 跑车 | 20 | 19 | 95% |
| 真的爱你 | 58-62 | 49 | 79-84% |

**规律**: 同一时间段送礼密度越高，漏采越多。稀疏单人送的大礼物 100% 捕获。

### 2.4 PushFrame.seq_id 跟踪（定锤）

抖音 WebSocket 协议的 Protobuf 外层结构：

```protobuf
message PushFrame {
  uint64 seq_id = 1;   // 帧序号，严格递增
  uint64 log_id = 2;
  bytes payload = 8;   // gzip 压缩的内层消息
}
```

监控 seq_id 序列：`55 → 57`（缺 56）、`91 → 93`（缺 92）...

**帧序号跳跃 = OS socket 缓冲区接收时就已经丢失。零 PushFrame/Response/gzip 解析失败。丢帧发生在 Python 代码到达之前。**

---

## 3. 优化尝试与效果

| # | 方案 | 效果 | 结论 |
|---|------|------|------|
| 1 | rcvbuf 256KB → 1024KB + 收发线程分离 | 帧丢失 5.2% → 3.3% | ✓ 有效，37% 改善 |
| 2 | rcvbuf 1024KB → 4096KB | 帧丢失 3.3% → 5.5% | ✗ 越大越差，已回退 |
| 3 | websocket-client(同步多线程) → websockets(asyncio) | 帧丢失 3.3% → 3.2% | ✗ 无实质改善，GIL 非瓶颈 |
| 4 | C 扩展绕过 GIL | 未实施 | ✗ ROI 极低，需 C 重写 protobuf/gzip |

### 3.1 方案 1 详情：rcvbuf + 收发分离

```
原架构：单线程 websocket-client 回调
  - socket.recv() → 回调处理 → 下一个 recv()
  - GIL 竞争：socket 读和消息处理争抢同一个锁

改造后：
  - 接收线程：只做 socket.recv() → 入队，不做处理
  - 工作线程：只做出队 → 处理，不做 socket 操作
  - SO_RCVBUF: 256KB → 1024KB
```

### 3.2 方案 2 详情：4096KB rcvbuf 失败

1024KB 稳定在 3.3%，4096KB 稳定在 5.5%（实测值，非波动）。  
假设：大缓冲区引入了 TCP 延迟抖动，或者 Windows 内核在处理大缓冲区时效率下降。  
已回退到 1024KB。

### 3.3 方案 3 详情：asyncio 无效

```
原架构：websocket-client（同步回调 + 5 线程）
  - 回调在独立线程中执行
  - 理论瓶颈：多个线程争抢 GIL

新架构：websockets（asyncio 事件循环 + 3 协程 Task）
  - 所有处理在单线程事件循环中
  - 理论优势：零 GIL 竞争

实测：帧丢失率 3.3%（原）vs 3.2%（新）— 无统计显著性差异
结论：GIL 从未是瓶颈，帧在到达 Python 之前就已经丢失了
```

---

## 4. 已排除的死路（避免重复踩坑）

| 方向 | 排除原因 |
|------|---------|
| LightGiftMessage | 无 user_id、无连击字段，不可靠，且无独立钻石数据 |
| 移动端 API (X-Gorgon) | 需要 APK 逆向获取签名算法，ROI 极低 |
| Ranklist API (a_bogus) | 签名成功但 API 返回空数据 |
| total_count 替代 delta | 12.7% 行中 total_count < repeat_count，用它反而少算 |
| ProfitInteractionScoreMessage | 连屏互动游戏玩家分数（8-24），非音浪收入 |
| RanklistHourEntranceMessage | 仅 UI 文本（"领先第二名57.7万"），无结构化数字 |
| BindingGiftMessage | 与 GiftMessage 是同一事件的双份通知，不是独立收入 |
| RoomDataSyncMessage | 房间通用状态同步，无结构化钻石数据 |
| 54 种 Unknown 消息 | 全部审计，无遗漏收入源 |
| FanTicket / income_taskgifts | 520 迅猛龙 6 场数据中始终为 0 |
| 采集器重连间隔 | <10s，对 6 小时直播影响可忽略 |
| 4096KB rcvbuf | 反而更差（5.5% vs 3.3%） |
| C 扩展绕过 GIL | 需 C 重写 protobuf/gzip，且 GIL 已证实非瓶颈 |
| asyncio 替代多线程 | 帧丢失无改善 |

---

## 5. 当前状态

### 已确认的稳定基线

- **帧丢失率**: ~3.3%（1024KB rcvbuf，无论线程模型还是 asyncio，无论流量高低）
- **对应的音浪缺口**: ~3-5%（真实值）
- **丢失规律**: 固定比例丢失（不是固定数量），意味着高流量时绝对丢失更多

### 关键观察

1. **新连接爬升期**: 重连后帧丢失从 0% 爬到 ~3.3%，需要一段时间稳定
2. **长期连接更稳定**: 运行 1 小时以上的连接，即使在 PK 高流量下也稳定在 3.3%
3. **重连撞上流量高峰 = 最差情况**: 丢失率直接飙到 5%+
4. **帧丢失与消息速率无关**: PK 时消息量涨 7 倍，丢失率纹丝不动（3.30% → 3.32%）

---

## 6. 剩余可选方案

### 方案 A：双 WebSocket 连接合并去重

**原理**: 两个独立 TCP 连接的帧丢失是独立随机事件。  
合并后理论丢失率 = 3.3% × 3.3% ≈ 0.1%

**风险**:
- 同 IP 两个号同时维护 WebSocket 长连接，触发抖音风控
- 录屏小号被占用，失去独立对账能力
- 需要解决合并去重逻辑（两个连接的同一件礼物如何识别）

**未实施原因**: 风控风险不可逆，一旦两个号都被标记，单连接采集也保不住

### 方案 B：接受 3-5% 误差

采集器的定位是趋势追踪和礼物分布分析，不是财务结算。  
3.3% 帧丢失是系统性的、稳定的（不是随机漂移），不同场次可比。

### 方案 C：住宅代理 + 账号池（商业方案）

类似小红书上盈利的数据服务商。需要：
- 几百个抖音账号做轮换（账号是消耗品）
- 住宅代理 IP 池（$3-10/GB）
- Cookie 自动刷新流水线
- 成本与 1-2 个主播的采集规模不匹配

---

## 7. 技术栈详情

| 层 | 实现 |
|----|------|
| 协议 | 抖音 WebSocket Protobuf（wss://webcast3-ws-web-lf.douyin.com） |
| 外层帧 | `PushFrame { seq_id, log_id, payload }` — gzip 压缩 |
| 内层消息 | `Response { messages[] }` — 每条 message 是一种业务类型 |
| 礼物消息 | `GiftMessage { gift: GiftStruct { diamond_count, ... }, count, group_id, repeat_count }` |
| WebSocket 库 | Python `websockets` (asyncio) |
| 运行环境 | Windows 11, Python 3.14, 家用宽带单 IP |

### 音浪 = 钻石 = 抖币

`diamond_count` 是礼物的抖币单价。1 钻石 = 1 音浪。  
`diamond_count × count` 即为该次送礼的音浪收入。  
所有开源抖音采集项目均使用此公式。

---

## 8. 关键源码位置

```
G:\DouyinBarrage-main\
  service/fetcher.py     — WebSocket 连接 + 消息分发（asyncio 版本）
  base/parser.py          — 消息解析 + 礼物去重
  base/messages.py        — 全部 Protobuf 消息定义
  base/output.py          — CSV 输出 + 日志
  config.yaml             — 配置（rcvbuf=1024KB 等）
```
