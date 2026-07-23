# 采样载荷

抖音 WebSocket 原始载荷采样，所有个人身份信息(PII)已脱敏。
使用 `base/messages.py` 中对应的类解析。

## 文件列表

| 文件 | 原始大小 | 脱敏后 |
|------|----------|----------|
| `AnchorLinkmicSilenceMessage.bin` | 106B | 64B |
| `AudioBgImgMessage.bin` | 4870B | 34B |
| `AudioChatMessage.bin` | 3581B | 1722B |
| `BattleEffectContainerMessage.bin` | 5743B | 41B |
| `BattleEndPunishMessage.bin` | 891B | 864B |
| `BattlePowerContainerMessage.bin` | 983B | 963B |
| `BattleRankSeasonMessage.bin` | 3705B | 36B |
| `BattleStatusMessage.bin` | 136B | 85B |
| `BattleStealDragonMessage.bin` | 8483B | 100B |
| `BattleTeamTaskMessage.bin` | 969B | 46B |
| `BattleUpdatePropCardTaskMessage.bin` | 1668B | 80B |
| `ChatLikeMessage.bin` | 3515B | 928B |
| `ChatMessage.bin` | 1294B | 776B |
| `CommonDotMessage.bin` | 224B | 57B |
| `CommonToastMessage.bin` | 173B | 55B |
| `DecorationModifyMethod.bin` | 119B | 47B |
| `DecorationUpdateMessage.bin` | 65B | 36B |
| `EmojiChatMessage.bin` | 3425B | 1976B |
| `ExhibitionChatMessage.bin` | 883B | 43B |
| `FansclubMessage.bin` | 4406B | 2632B |
| `GiftEffectGameMessage.bin` | 60B | 40B |
| `GiftMessage.bin` | 8522B | 3457B |
| `GiftPlayEventMessage.bin` | 412B | 33B |
| `GiftSortMessage.bin` | 73B | 53B |
| `InRoomBannerMessage.bin` | 4386B | 46B |
| `ItemShareMessage.bin` | 3786B | 741B |
| `LightGiftMessage.bin` | 352B | 302B |
| `LikeMessage.bin` | 7402B | 2068B |
| `LinkMessage.bin` | 4996B | 4951B |
| `LinkMicMethodMessage.bin` | 425B | 268B |
| `LinkSettingNotifyMessage.bin` | 74B | 35B |
| `LinkerContributeMessage.bin` | 164B | 55B |
| `LinkmicArmiesMessage.bin` | 16342B | 8236B |
| `LinkmicBattleFinishMessage.bin` | 16572B | 16391B |
| `LinkmicPlayModeUpdateScoreMessage.bin` | 6989B | 71B |
| `LinkmicPlaymodeMessage.bin` | 64B | 35B |
| `LinkmicProfitMessage.bin` | 137B | 110B |
| `LowPcuGuideMessage.bin` | 57B | 37B |
| `MemberMessage.bin` | 6815B | 4194B |
| `NotifyEffectMessage.bin` | 2064B | 573B |
| `PreviewCjRpMessage.bin` | 223B | 93B |
| `PrivilegeScreenChatMessage.bin` | 5004B | 338B |
| `PrizeNoticeMessage.bin` | 58B | 38B |
| `ProfitGameStatusMessage.bin` | 143B | 101B |
| `ProfitInteractionScoreMessage.bin` | 152B | 115B |
| `RanklistAwardMessage.bin` | 67B | 40B |
| `RanklistHourEntranceMessage.bin` | 280B | 40B |
| `ResidentGuestMessage.bin` | 63B | 33B |
| `RoomDataSyncMessage.bin` | 199B | 89B |
| `RoomIndicatorMessage.bin` | 484B | 37B |
| `RoomMessage.bin` | 938B | 81B |
| `RoomNotifyMessage.bin` | 4405B | 3037B |
| `RoomRankMessage.bin` | 17399B | 16920B |
| `RoomStatsMessage.bin` | 95B | 62B |
| `RoomStreamAdaptationMessage.bin` | 62B | 42B |
| `RoomUserSeqMessage.bin` | 4382B | 3907B |
| `ScreenChatMessage.bin` | 4253B | 881B |
| `SocialMessage.bin` | 4140B | 1507B |
| `TaskCenterEntranceMessage.bin` | 3947B | 50B |
| `ToastMessage.bin` | 173B | 55B |
| `UserPrivilegeChangeMessage.bin` | 92B | 72B |

## 脱敏规则

- int64/uint64 > 100,000 清零（ID、时间戳、计数）
- int32/uint32 > 10,000,000 清零
- URL、长数字字符串、MS4w sec_uid 已脱敏
- bytes 字段已清空
