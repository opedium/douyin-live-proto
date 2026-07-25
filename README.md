# Douyin Live Proto — 抖音直播间 Protobuf 定义参考
  
> 抖音 Web 端 WebSocket 协议逆向工程 — 完整 Protobuf 消息定义与采样数据。

![License](https://img.shields.io/badge/License-MIT-lightgrey.svg)

## 文件说明

| 文件 | 说明 |
|------|------|
| [`douyin_live.proto`](douyin_live.proto) | 完整 Protobuf 定义（1589 行，85+ 业务消息类型） |
| [`docs/complete_payload_reference.md`](docs/complete_payload_reference.md) | 完整字段参考手册 — 每个消息类型的字段含义与用途 |
| [`sampled_payload/`](sampled_payload/) | 61 个脱敏采样载荷（.bin），对应每种消息类型的真实数据 |


## 消息类型

覆盖传输层、通用层和业务层，共计 **82 种消息类型**，包括：

| 分类 | 消息 |
|------|------|
| 聊天 | ChatMessage, EmojiChatMessage, ScreenChatMessage, AudioChatMessage 等 |
| 礼物 | GiftMessage, LightGiftMessage, GiftSortMessage 等 |
| 互动 | LikeMessage, MemberMessage, SocialMessage, FansclubMessage 等 |
| 直播间状态 | RoomMessage, RoomRankMessage, RoomStatsMessage, RoomUserSeqMessage 等 |
| PK/对战 | LinkmicBattleMessage, LinkmicArmiesMessage, LinkerContributeMessage 等 |
| 系统 | ControlMessage, ToastMessage, NotifyEffectMessage, CommonDotMessage 等 |

完整映射见 [`docs/complete_payload_reference.md`](docs/complete_payload_reference.md) 的 Wire 名称映射表。

## 使用方法

直接使用 `.proto` 文件配合任意 Protobuf 工具链：

```bash
protoc --decode_raw < sampled_payload/ChatMessage.bin
```

## 数据来源

所有消息定义来自抖音 Web 端 WebSocket 协议（`wss://webcast.douyin.com`）的逆向分析。采样载荷经过完整的 PII 脱敏处理（ID 清零、URL 替换、字符串脱敏），可安全分发。

## 协议栈

```
PushFrame (WebSocket 帧)
  └─ payload → gzip 解压 → Response
       └─ messages_list → Message
            ├─ method: "WebcastGiftMessage"
            └─ payload → GiftMessage { ... }
```

## License

MIT
