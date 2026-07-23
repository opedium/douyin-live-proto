# 音浪采集精度：当前卡点与已确认结论

**日期**: 2026-05-24
**状态**: 系统代理被确认为丢帧根因，proxy=None 后 0% 丢帧，待迅猛龙全场对账结案

---

## 根因（已确认）

**系统代理（Clash/V2Ray 127.0.0.1:7890）是 WebSocket PushFrame 3.3% 丢帧的根因。**

机制：
1. websockets 库默认 `proxy=True`，自动检测系统代理
2. 推流数据经 Clash 用户态进程中转，内部缓冲区在高频推送下溢出
3. TCP 连接被拆成两段（服务器→代理→Python），代理成了瓶颈
4. 所以 rcvbuf 256KB→1024KB 只把丢帧从 5.2% 降到 3.3%——瓶颈在代理，不在 OS buffer

## 修改清单（260524）

| 文件 | 修改 | 目的 |
|------|------|------|
| `service/fetcher.py:633` | `proxy=None` | WebSocket 绕过系统代理 |
| `service/fetcher.py:117` | `self.session.trust_env = False` | HTTP 请求绕过系统代理（VPN 共存） |
| `service/fetcher.py:773-784` | hb 帧也更新 `_last_seq_id` | 修复 seq_id 统计误报 |
| `main.py:399-401` | `join()` → `join(timeout=1)` 轮询 | 修复 Ctrl+C 无响应 |

## proxy=None 验证（5 次测试，3621 帧，0% 丢帧）

| 测试 | 时长 | 帧数 | 丢帧 |
|------|------|------|------|
| 孙恩盛 凌晨 | 5min | 476 | 0% |
| 孙恩盛 A路+USB | 5min | 772+709 | 0% |
| 迪士尼 下午 | 5min | 718 | 0% |
| 迪士尼 下午 | 20min | 946 | 0% |

proxy=None 之前，任何时段最低 1-2%，从未见过 0%。

## 确定结论

| # | 结论 |
|---|------|
| 1 | **系统代理是帧丢失根因**（260524 确认） |
| 2 | GIL 不是瓶颈（asyncio vs 多线程无差异） |
| 3 | max_queue/compression 不是瓶颈 |
| 4 | 去重逻辑不丢收入 |
| 5 | 54 种 Unknown 消息审计完成，无遗漏收入源 |
| 6 | LightGiftMessage 不可用（无 user_id） |
| 7 | 移动端 API / Ranklist API / HourRank 均为死路 |
| 8 | rcvbuf 1024KB 最优 |

## 待验证

- 迅猛龙 260524 全场采集对账（误差应接近 0%）

## 双连接实验

`dual_seq_test.py` 保留备用。若对账仍有误差，可通过双网卡（USB+WLAN）双连接 seq_id 对比判定服务端/客户端。

## 关键文件

```
G:\DouyinBarrage-main\
  service/fetcher.py       — 主采集器（proxy=None, trust_env=False）
  main.py                   — 入口（Ctrl+C 修复）
  dual_seq_test.py          — 双连接实验脚本（备用）
```
