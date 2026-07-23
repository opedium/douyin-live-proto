# 抖音直播间 WebSocket 载荷完整参考

抖音直播间所有消息类型的完整字段参考，包含字段含义与用途说明。

---
## 覆盖统计

| 状态 | 数量 |
|--------|-------|
| 消息类型总数 | 82 |
| 已详细文档 | 48 |
| 快速参考(附录) | 34 |
| 有采样载荷 | 61 |

## Wire 名称映射
| Wire 名称 | 消息类型 | 样本 |
|-----------|-------------|--------|
| `WebcastActivityEmojiGroupsMessage` | `ActivityEmojiGroupsMessage` | — |
| `WebcastAnchorLinkmicSilenceMessage` | `AnchorLinkmicSilenceMessage` | ✓ |
| `WebcastAssetEffectUtilMessage` | `AssetEffectUtilMessage` | — |
| `WebcastAssetMessage` | `AssetMessage` | — |
| `WebcastAudioBgImgMessage` | `AudioBgImgMessage` | ✓ |
| `WebcastAudioChatMessage` | `AudioChatMessage` | ✓ |
| `WebcastBackupSeiMessage` | `BackUpSeiMessage` | — |
| `WebcastBattleAuxiliaryMessage` | `BattleAuxiliaryMessage` | — |
| `WebcastBattleEffectContainerMessage` | `BattleEffectContainerMessage` | ✓ |
| `WebcastBattleEndPunishMessage` | `BattleEndPunishMessage` | ✓ |
| `WebcastBattlePowerContainerMessage` | `BattlePowerContainerMessage` | ✓ |
| `WebcastBattleRankSeasonMessage` | `BattleRankSeasonMessage` | ✓ |
| `WebcastBattleStatusMessage` | `BattleStatusMessage` | ✓ |
| `WebcastBattleStealDragonMessage` | `BattleStealDragonMessage` | ✓ |
| `WebcastBattleTeamTaskMessage` | `BattleTeamTaskMessage` | ✓ |
| `WebcastBattleUpdatePropCardTaskMessage` | `BattleUpdatePropCardTaskMessage` | ✓ |
| `WebcastBindingGiftMessage` | `BindingGiftMessage` | — |
| `WebcastChatLikeMessage` | `ChatLikeMessage` | ✓ |
| `WebcastChatMessage` | `ChatMessage` | ✓ |
| `WebcastCommonDotMessage` | `CommonDotMessage` | ✓ |
| `WebcastCommonToastMessage` | `CommonToastMessage` | ✓ |
| `WebcastControlMessage` | `ControlMessage` | — |
| `WebcastDecorationModifyMethod` | `DecorationModifyMethod` | ✓ |
| `WebcastDecorationUpdateMessage` | `DecorationUpdateMessage` | ✓ |
| `WebcastEmojiChatMessage` | `EmojiChatMessage` | ✓ |
| `WebcastExhibitionChatMessage` | `ExhibitionChatMessage` | ✓ |
| `WebcastFansclubMessage` | `FansclubMessage` | ✓ |
| `WebcastGiftEffectGameMessage` | `GiftEffectGameMessage` | ✓ |
| `WebcastGiftMessage` | `GiftMessage` | ✓ |
| `WebcastGiftPlayEventMessage` | `GiftPlayEventMessage` | ✓ |
| `WebcastGiftSortMessage` | `GiftSortMessage` | ✓ |
| `WebcastGiftUpdateMessage` | `GiftUpdateMessage` | — |
| `WebcastHighlightCommentMessage` | `HighlightCommentMessage` | — |
| `WebcastHotChatMessage` | `HotChatMessage` | — |
| `WebcastHotRoomMessage` | `HotRoomMessage` | — |
| `WebcastInRoomBannerMessage` | `InRoomBannerMessage` | ✓ |
| `WebcastInteractEffectMessage` | `InteractEffectMessage` | — |
| `WebcastItemShareMessage` | `ItemShareMessage` | ✓ |
| `WebcastLightGiftMessage` | `LightGiftMessage` | ✓ |
| `WebcastLikeMessage` | `LikeMessage` | ✓ |
| `WebcastLinkMessage` | `LinkMessage` | ✓ |
| `WebcastLinkMicMethod` | `LinkMicMethodMessage` | ✓ |
| `WebcastLinkSettingNotifyMessage` | `LinkSettingNotifyMessage` | ✓ |
| `WebcastLinkerContributeMessage` | `LinkerContributeMessage` | ✓ |
| `WebcastLinkmicArmiesMethod` | `LinkmicArmiesMessage` | ✓ |
| `WebcastLinkmicBattleFinishMethod` | `LinkmicBattleFinishMessage` | ✓ |
| `WebcastLinkmicBattleMethod` | `LinkmicBattleMessage` | — |
| `WebcastLinkmicPlayModeUpdateScoreMessage` | `LinkmicPlayModeUpdateScoreMessage` | ✓ |
| `WebcastLinkmicPlaymodeMessage` | `LinkmicPlaymodeMessage` | ✓ |
| `WebcastLinkmicProfitMessage` | `LinkmicProfitMessage` | ✓ |
| `WebcastLotteryDrawResultEventMessage` | `LotteryDrawResultEventMessage` | — |
| `WebcastLotteryEventNewMessage` | `LotteryEventNewMessage` | — |
| `WebcastLowPcuGuideMessage` | `LowPcuGuideMessage` | ✓ |
| `WebcastLuckyBoxEndMessage` | `LuckyBoxEndMessage` | — |
| `WebcastLuckyBoxMessage` | `LuckyBoxMessage` | — |
| `WebcastLuckyBoxRewardMessage` | `LuckyBoxRewardMessage` | — |
| `WebcastLuckyBoxTempStatusMessage` | `LuckyBoxTempStatusMessage` | — |
| `WebcastMemberMessage` | `MemberMessage` | ✓ |
| `WebcastNotifyEffectMessage` | `NotifyEffectMessage` | ✓ |
| `WebcastPreviewCjRpMessage` | `PreviewCjRpMessage` | ✓ |
| `WebcastPrivilegeScreenChatMessage` | `PrivilegeScreenChatMessage` | ✓ |
| `WebcastPrizeNoticeMessage` | `PrizeNoticeMessage` | ✓ |
| `WebcastProfitGameStatusMessage` | `ProfitGameStatusMessage` | ✓ |
| `WebcastProfitInteractionScoreMessage` | `ProfitInteractionScoreMessage` | ✓ |
| `WebcastRanklistAwardMessage` | `RanklistAwardMessage` | ✓ |
| `WebcastRanklistHourEntranceMessage` | `RanklistHourEntranceMessage` | ✓ |
| `WebcastResidentGuestMessage` | `ResidentGuestMessage` | ✓ |
| `WebcastRoomCommentTopicMessage` | `RoomCommentTopicMessage` | — |
| `WebcastRoomDataSyncMessage` | `RoomDataSyncMessage` | ✓ |
| `WebcastRoomIndicatorMessage` | `RoomIndicatorMessage` | ✓ |
| `WebcastRoomMessage` | `RoomMessage` | ✓ |
| `WebcastRoomNotifyMessage` | `RoomNotifyMessage` | ✓ |
| `WebcastRoomRankMessage` | `RoomRankMessage` | ✓ |
| `WebcastRoomStatsMessage` | `RoomStatsMessage` | ✓ |
| `WebcastRoomStreamAdaptationMessage` | `RoomStreamAdaptationMessage` | ✓ |
| `WebcastRoomUserSeqMessage` | `RoomUserSeqMessage` | ✓ |
| `WebcastScreenChatMessage` | `ScreenChatMessage` | ✓ |
| `WebcastSocialMessage` | `SocialMessage` | ✓ |
| `WebcastTaskCenterEntranceMessage` | `TaskCenterEntranceMessage` | ✓ |
| `WebcastToastMessage` | `ToastMessage` | ✓ |
| `WebcastTopEffectMessage` | `TopEffectMessage` | — |
| `WebcastUserPrivilegeChangeMessage` | `UserPrivilegeChangeMessage` | ✓ |

---

## 传输层

### Push帧 — WebSocket frame 封装
```
f[1]: seq_id       uint64    Frame sequence number (monotonic)
f[2]: log_id       uint64    Trace ID for server-side debugging
f[3]: service      uint64    Service type identifier
f[4]: method       uint64    Internal method code
f[5]: headers_list repeated  Key-value headers
f[6]: payload_encoding string "gzip" or "none"
f[7]: payload_type string   "hb"=heartbeat, "ack"=acknowledge, "pb"=protobuf data
f[8]: payload      bytes    Compressed protobuf Response
```

### Response — Message 批量 容器
```
f[1]: messages_list  repeated Message  List of business messages
f[2]: cursor         string            Pagination cursor
f[3]: fetch_interval uint64            Suggested poll interval (ms)
f[4]: now            uint64            Server timestamp (ms)
f[5]: internal_ext   string            ACK payload data
f[6]: fetch_type     uint32            Poll type
f[7]: route_params   map<string,string> Routing parameters
f[8]: heartbeat_duration uint64        Heartbeat interval (ms)
f[9]: need_ack       bool              Must send ACK back
f[10]: push_server   string            Push server address
f[11]: live_cursor   string            Cursor for reconnection
f[12]: history_no_more bool            All history consumed
```

### Message — Individual message 信封
```
f[1]: method       string  Message type identifier (e.g. "WebcastChatMessage")
f[2]: payload      bytes   Protobuf-serialized business message
f[3]: msg_id       int64   Server-assigned unique message ID
f[4]: msg_type     int32   Numeric message type
f[5]: offset       int64   Stream offset
f[6]: need_wrds_store bool  Word storage flag
f[7]: wrds_version int64   Word storage version
f[8]: wrds_sub_key string  Subscription key
```

---

## 通用 / 共享类型

### Common — 头部元数据（所有业务消息）
```
f[1]:  method             string   Message type (e.g. "WebcastGiftMessage")
f[2]:  msg_id             uint64   Unique message ID
f[3]:  room_id            uint64   Live room ID (19 digits)
f[4]:  create_time        uint64   Unix timestamp (ms)
f[5]:  monitor            uint32   Monitoring flag
f[6]:  is_show_msg        bool     Whether to display in chat
f[7]:  describe           string   Human-readable description (e.g. "张三:送了1个嘉年华")
f[8]:  display_text       bytes    Text/anchor display object (large nested Text)
f[9]:  fold_type          uint64   Chat fold grouping
f[10]: anchor_fold_type   uint64   Anchor's fold preference
f[11]: priority_score     uint64   Chat display priority (e.g. 31000)
f[12]: log_id             string   Server log ID
f[13]: msg_process_filter_k string Filter key
f[14]: msg_process_filter_v string Filter value
f[15]: user               User     Associated user (used in RoomMessage)
f[17]: anchor_fold_type_v2 uint64  Newer fold type
f[18]: process_at_sei_time_ms uint64 SEI timing
f[19]: random_dispatch_ms uint64   Random dispatch delay
f[20]: is_dispatch        bool     Dispatch flag
f[21]: channel_id         uint64   Channel ID
f[22]: diff_sei2abs_second uint64  SEI difference
f[23]: anchor_fold_duration uint64 Fold animation duration
f[24]: flag_1128          uint64   Always 1128 — version/fold flag
```

### User — 用户信息 (embedded in all 业务消息)
```
f[1]:    id               uint64   Unique user ID (17-19 digits)
f[2]:    short_id         uint64   Short numeric ID
f[3]:    nick_name        string   Display nickname
f[4]:    gender           uint32   1=Male, 2=Female
f[5]:    signature        string   User bio/description
f[6]:    level            uint32   User level
f[7]:    birthday         uint64   Birthday timestamp
f[8]:    telephone        string   Phone number
f[9]:    avatar_thumb     Image    Avatar thumbnail (100x100)
f[10]:   avatar_medium    Image    Avatar medium (720x720)
f[11]:   avatar_large     Image    Avatar large (1080x1080)
f[12]:   verified         bool     Verified account status
f[13]:   experience       uint32   Experience points
f[14]:   city             string   City location
f[15]:   status           int32    Account status
f[16]:   create_time      uint64   Account creation timestamp
f[17]:   modify_time      uint64   Last profile update
f[18]:   secret           uint32   Privacy level
f[19]:   share_qrcode_uri string   Share QR code URL
f[20]:   income_share_percent uint32 Revenue share %
f[21]:   badge_image_list repeated Image Active badges
f[22]:   follow_info      FollowInfo Following/follower counts
f[23]:   pay_grade        PayGrade  Consumption level
f[24]:   fans_club        FansClub  Fans club membership
f[26]:   special_id       string   Special identifier
f[27]:   avatar_border    Image    Avatar border image
f[28]:   medal            Image    User medal
f[29]:   real_time_icons_list repeated Image  Active icons
f[32]:   field_32         string   Always empty (reserved)
f[37]:   field_37         uint32   User subtype flag
f[38]:   display_id       string   Public display ID (usually = short_id)
f[46]:   sec_uid          string   Secure user ID (base64, for API lookups)
f[54]:   field_54         uint32   Version flag (always 3)
f[55]:   field_55         uint32   Version flag
f[61]:   field_61         bytes    Badge image (duplicate of f[21])
f[64]:   field_64         bytes    Fans club badge images bundle
f[66]:   field_66         uint32   Flag (always 1)
f[68]:   field_68         string   Nickname duplicate
f[69]:   field_69         string   Empty (reserved)
f[70]:   field_70         string   Empty (reserved)
f[73]:   field_73         string   sec_uid duplicate
f[78]:   field_78         bytes    Full badge images bundle
f[1022]: fan_ticket_count uint64   Fan tickets received
f[1028]: id_str           string   User ID as string
f[1045]: age_range        uint32   Age range code
```

### Image — 基于URL的图片资源，包含多分辨率链接
```
f[1]: url_list_list repeated string  CDN URLs (usually 3: p3, p11, p26)
f[2]: uri           string           Image path/identifier
f[3]: height        uint64           Image height (px)
f[4]: width         uint64           Image width (px)
f[5]: avg_color     string           Average color hex (e.g. "#A37C7C")
f[6]: image_type    uint32           Image format type
f[7]: open_web_url  string           Click-through URL
f[8]: content       ImageContent     Text overlay content
f[9]: is_animated   bool             Animated image flag
f[10]: flex_setting_list NinePatchSetting 9-patch settings
f[11]: text_setting_list NinePatchSetting Text display settings
```

### Text — Rich text with formatting (used for display messages)
```
f[1]: key              string   Text template key
f[2]: default_patter   string   Default text pattern with {0} placeholders
f[3]: default_format   TextFormat  Default formatting
f[4]: pieces_list      repeated TextPiece  Rich text segments
```

### TextPiece — Segment of rich text (user mention, gift name, heart, etc.)
```
f[1]: type            bool         0=text, 1=user/gift/heart
f[2]: format          TextFormat   Segment formatting
f[3]: string_value    string       Plain text content
f[4]: user_value      TextPieceUser User reference
f[5]: gift_value      TextPieceGift Gift name reference
f[6]: heart_value     TextPieceHeart Heart icon
f[7]: pattern_ref_value TextPiecePatternRef Template pattern
f[8]: image_value     TextPieceImage Inline image
```

### PublicAreaCommon — 用户在公共聊天区的状态徽章
```
f[1]: user_label                Image  User label/badge image
f[2]: user_consume_in_room      uint64 Total diamonds consumed in room
f[3]: user_send_gift_cnt_in_room uint64 Number of gifts sent in room
```

### PayGrade — 消费等级 / user rank
```
f[1]:  total_diamond_count int64   Lifetime diamonds spent
f[2]:  diamond_icon        Image   Diamond icon
f[3]:  name                string  Level name
f[4]:  icon                Image   Level icon
f[5]:  next_name           string  Next level name
f[6]:  level               int64   Current level number
f[7]:  next_icon           Image   Next level icon
f[8]:  next_diamond        int64   Diamonds needed for next level
f[9]:  now_diamond         int64   Diamonds at current level
f[10]: this_grade_min_diamond int64 Min diamonds for this level
f[11]: this_grade_max_diamond int64 Max diamonds for this level
f[12]: pay_diamond_bak     int64   Backup diamond count
f[13]: grade_describe      string  Level description
f[14]: grade_icon_list     repeated GradeIcon Level icons
f[23]: background          Image   Level background
f[24]: background_back     Image   Level background (back)
f[25]: score               int64   Score
f[26]: buff_info           GradeBuffInfo Buff info
```

### FansClub — 粉丝团 data
```
f[1]: data        FansClubData  Current fans club info
f[2]: prefer_data map<int32,FansClubData> Other fans clubs
```

### FansClubData
```
f[1]: club_name           string  Fans club name
f[2]: level               int32   Fans club level
f[3]: user_fans_club_status int32 0=none, 1=joined
f[4]: badge               UserBadge Fans club badge
f[5]: available_gift_ids  repeated int64  Club-specific gift IDs
f[6]: anchor_id           int64   Anchor's user ID
```

### FollowInfo
```
f[1]: following_count  uint64  Number of accounts they follow
f[2]: follower_count   uint64  Number of followers
f[3]: follow_status    uint64  Follow relationship status
f[4]: push_status      uint64  Notification status
```

---

## 1. 聊天与通讯

### WebcastChatMessage Sample: [`sampled_payload/ChatMessage.bin`](sampled_payload/ChatMessage.bin) — 观众文字弹幕
```
f[1]:  common                    Common      Room, timestamp, priority
f[2]:  user                      User        Sender's full profile
f[3]:  content                   string      The chat text (e.g. "@姐姐 感谢")
f[4]:  visible_to_sender         bool        Whether sender can see it
f[5]:  background_image          Image       Chat background
f[6]:  full_screen_text_color    string      Full-screen text color
f[9]:  public_area_common        PublicAreaCommon Rank/badge info
f[10]: gift_image                Image       Associated gift image
f[11]: agree_msg_id              uint64      Agreed/reference message ID
f[12]: priority_level            uint32      Display priority
f[13]: landscape_area_common     LandscapeAreaCommon Landscape settings
f[15]: event_time                uint64      Message timestamp (seconds)
f[21]: individual_chat_priority  uint32      Per-message priority
f[22]: rtf_content               Text        Rich text formatted content
```

### WebcastEmojiChatMessage Sample: [`sampled_payload/EmojiChatMessage.bin`](sampled_payload/EmojiChatMessage.bin) — 表情/贴纸消息
```
f[1]: common              Common       Standard header
f[2]: user                User         Sender
f[3]: emoji_id            int64        Emoji ID
f[4]: emoji_content       Text         Emoji content (image reference)
f[5]: default_content     string       Fallback text (e.g. "[表情]")
f[6]: background_image    Image        Background
```

### WebcastScreenChatMessage Sample: [`sampled_payload/ScreenChatMessage.bin`](sampled_payload/ScreenChatMessage.bin) — 投屏/付费醒目留言(屏幕展示)
```
f[1]: common              Common       Standard header
f[2]: user                User         Sender (id, nick_name, avatar, badges)
f[3]: content              string       Chat text
f[4]: event_time           uint64       Event timestamp
f[10]: public_area_common  PublicAreaCommon  User badge
f[12]: priority_score      uint64       Priority for display
f[32]: field_32            bytes        Rich text format content
```

### WebcastAudioChatMessage Sample: [`sampled_payload/AudioChatMessage.bin`](sampled_payload/AudioChatMessage.bin) — 语音聊天消息
```
f[1]: common              Common       Standard header
f[2]: user                User         Sender profile
f[3]: content             string       Voice transcript text
f[4]: audio_url           string       Voice file URL
f[5]: audio_duration      uint64       Duration in seconds
f[6]: field_6             bytes        Consume/badge info
```

### WebcastPrivilegeScreenChatMessage Sample: [`sampled_payload/PrivilegeScreenChatMessage.bin`](sampled_payload/PrivilegeScreenChatMessage.bin) — 订阅专属醒目留言
```
f[1]: common              Common       Standard header
f[2]: user                User         Sender (with fans_club info)
f[3]: content             string       Chat text
f[4]: gift_data           bytes        Gift metadata (diamond value)
f[5]: privilege_type      uint32       3=subscriber
```

### WebcastHotChatMessage — 推荐聊天话题 ("大家都在说")
```
f[1]: common              Common       Standard header
f[2]: title               string       Category ("大家说")
f[3]: content             string       Suggested phrase ("宝宝")
f[7]: event_time          uint64       Timestamp
f[200]: tags              bytes        Tags: "tag_content_type"="comment"
```

### WebcastChatLikeMessage Sample: [`sampled_payload/ChatLikeMessage.bin`](sampled_payload/ChatLikeMessage.bin) — 点赞聊天消息
```
f[1]: common              Common       Standard header
f[2]: field_2             bytes        Target message reference
```

---

## 2. 礼物与交易

### WebcastGiftMessage Sample: [`sampled_payload/GiftMessage.bin`](sampled_payload/GiftMessage.bin) — 礼物事件(完整详情)
```
f[1]:  common                  Common       Header with describe text
f[2]:  gift_id                 uint64       Gift type ID (685=fan badge, 3389=box, etc.)
f[3]:  fan_ticket_count        uint64       Fan tickets awarded
f[4]:  group_count             uint32       Group gift count (1=individual, >1=group)
f[5]:  repeat_count            uint32       Combo repeat cumulative count
f[6]:  combo_count             uint32       Same as repeat_count (combo position)
f[7]:  user                    User         Sender's full profile
f[8]:  to_user                 User         Target user (only for user-to-user gifts)
f[9]:  repeat_end              uint32       1=last message in combo
f[10]: text_effect             TextEffect   On-screen text animation
f[11]: group_id                uint64       Combo group ID (dedup key)
f[12]: income_taskgifts        uint64       Income task tracking
f[13]: room_fan_ticket_count   uint64       Room's total fan tickets
f[14]: priority                GiftIMPriority  Queue position
f[15]: gift                    GiftStruct   Gift definition (name, price, icon)
f[16]: log_id                  string       Operation log ID
f[17]: send_type               uint64       1=normal, 2=group gift
f[18]: public_area_common      PublicAreaCommon User's rank/badge in room
f[19]: tray_display_text       Text         Tray notification text ("送{0}")
f[20]: banned_display_effects  uint64       Banned effect mask
f[21]: asset_effect_mix_info   bytes        Animation + sound + effect bundle
f[25]: display_for_self        bool         Show animation to sender
f[26]: interact_gift_info      string       Interactive gift JSON (empty for normal)
f[27]: diy_item_info           string       DIY gift info (甄选礼盒 compound gift JSON)
f[28]: min_asset_set_list      repeated uint64  Minimum display asset IDs
f[29]: total_count             uint64       Final total in combo
f[30]: client_gift_source      uint32       1=panel, 2=store, 3=activity
f[31]: anchor_gift_data        bytes        Anchor revenue split data
f[32]: to_user_ids_list        repeated uint64  Recipients for group gifts
f[33]: send_time               uint64       Unix timestamp (ms)
f[34]: force_display_effects   uint64       Force effects mask
f[35]: trace_id                string       Dedup trace ID (32-char hex)
f[36]: effect_display_ts       uint64       Effect display timestamp
f[39]: field_39                bytes        Effect info {3: 5} — layer/position
f[43]: field_43                uint32       Effect flag (1=show animation)
f[45]: field_45                string       Send ms as string (0=instant)
```

### GiftStruct — 礼物定义 (GiftMessage 内部)
```
f[1]:  image               Image        Gift icon in panel
f[2]:  describe            string       Description ("送出粉丝团灯牌")
f[3]:  notify              bool         Show system notification
f[4]:  duration            uint64       Animation duration (ms)
f[5]:  id                  uint64       Gift database ID (same as gift_id)
f[6]:  fansclub_info       bytes        Fans club gift data
f[7]:  for_linkmic         bool         Available during PK
f[8]:  doodle              bool         Is doodle/DIY gift
f[9]:  for_fansclub        bool         Fans club exclusive
f[10]: combo               bool         Supports combo sending
f[11]: type                uint32       Category: 1=normal, 2=stream, 4=fan badge
f[12]: diamond_count       uint32       Price in diamonds
f[13]: is_displayed_on_panel bool      Visible in gift panel
f[14]: primary_effect_id   uint64       Main animation effect ID
f[15]: gift_label_icon     Image        Label badge (e.g. "限定")
f[16]: name                string       Gift name ("嘉年华", "粉丝团灯牌")
f[17]: region              string       Region code
f[18]: manual              string       Manual/description text
f[19]: for_custom          bool         Customizable gift
f[20]: special_effects_map bytes        Effect mapping
f[21]: icon                Image        Alternative icon
f[22]: action_type         uint32       Interaction type
f[28]: display_index       uint32       Panel sort order
f[38]: fold_type           uint64       Fold/group type (1128=normal)
f[42]: gift_category_id    uint32       Category (753=fan badge)
f[44]: gift_label          string       Label ("粉丝","限定","热门")
f[48]: display_strategy    uint32       Display mode (3=full screen anim)
f[52]: scheme_url          string       Gift purchase panel URL
f[54]: gift_source         uint32       1=normal panel
f[55]: banner              bytes        Panel banner images
f[56]: tag_images          bytes        Tag icon assets
f[57]: gift_tag            bytes        Gift tag metadata
f[63]: sub_type            uint32       1=normal, 2=subscription
f[74]: anchor_gift_data    bytes        Anchor revenue bundle
```

### WebcastLightGiftMessage Sample: [`sampled_payload/LightGiftMessage.bin`](sampled_payload/LightGiftMessage.bin) — 轻礼物(粉丝票,人气票)
```
f[1]:  common              Common       Standard header
f[2]:  msg_type            uint64       Always 1
f[3]:  version             uint64       Always 1
f[5]:  user_id             uint64       Sender's user ID
f[7]:  gift_info           LightGiftInfo Gift details (ID, diamond_count, image)
f[8]:  unknown_8           bytes        Contains 1000 (timing/interval?)
f[9]:  field_9             uint32       Always 4
f[10]: count               uint64       Quantity sent
f[12]: field_12            uint64       Always 2
f[13]: field_13            bytes        Nested with gift_id reference
```

### LightGiftInfo — 轻礼物详情
```
f[1]: gift_id       uint32   Gift type ID
f[2]: image         Image    Gift icon
f[3]: diamond_count uint32   Price per unit
```

### WebcastGiftSortMessage Sample: [`sampled_payload/GiftSortMessage.bin`](sampled_payload/GiftSortMessage.bin) — 礼物面板排序/展示模式
```
f[1]: common       Common   Standard header
f[2]: sort_type    uint32   Sort mode (3=exhibition)
f[4]: scene        string   Scene label ("anchor_exhibition")
```

### WebcastItemShareMessage Sample: [`sampled_payload/ItemShareMessage.bin`](sampled_payload/ItemShareMessage.bin) — 分享成就物品
```
f[1]: common      Common      Standard header
f[2]: item_data   bytes       Share item (type: "grade_league", title: "荣耀赛季报告")
f[3]: text        bytes       Share text template ("{0:user} 分享了TA的")
f[4]: field_4     bytes       Consume/badge info
```

---

## 3. 用户活动

### WebcastMemberMessage Sample: [`sampled_payload/MemberMessage.bin`](sampled_payload/MemberMessage.bin) — 用户进入直播间
```
f[1]:  common               Common       Header. f[8]=display text
f[2]:  user                 User         Viewer's full profile
f[3]:  member_count         uint64       Current room member count
f[4]:  operator             User         Who performed the action (admin ops)
f[5]:  is_set_to_admin      bool         Was set as admin
f[6]:  is_top_user          bool         Is top contributor
f[7]:  rank_score           uint64       User's rank score
f[8]:  top_user_no          uint64       Top user number
f[9]:  enter_type           uint64       Entry type (1=normal, 2=fans club)
f[10]: action               uint64       1=enter
f[11]: action_description   string       Description of action
f[12]: userId               uint64       User ID (duplicate)
f[13]: effect_config        EffectConfig Entry animation config
f[14]: pop_str              string       Display string
f[15]: enter_effect_config  EffectConfig Fans club entry effect
f[16]: background_image     Image        Background image
f[18]: anchor_display_text  Text         Anchor's display ("{0:user} 来了")
f[19]: public_area_common   PublicAreaCommon  User badge info
f[20]: user_enter_tip_type  uint64       Tip type
f[21]: anchor_enter_tip_type uint64      Anchor tip type
f[22]: field_22              bytes       Map: enter_tip_type, msg_show_type
```

### WebcastSocialMessage Sample: [`sampled_payload/SocialMessage.bin`](sampled_payload/SocialMessage.bin) — 关注/分享事件
```
f[1]: common               Common       Standard header
f[2]: user                 User         Actor (who followed/shared)
f[3]: share_type           uint64       Share destination type
f[4]: action               uint64       1=follow, 2=share
f[5]: share_target         string       Target user ID (for follows)
f[6]: follow_count         uint64       Running total followers
f[7]: public_area_common   PublicAreaCommon  User badge
```

### WebcastLikeMessage Sample: [`sampled_payload/LikeMessage.bin`](sampled_payload/LikeMessage.bin) — 点赞事件
```
f[1]: common               Common       Standard header. f[8]=display text
f[2]: count                uint64       Like count in this message (often 30)
f[3]: total                uint64       Running total likes this session
f[4]: color                uint64       Like animation color
f[5]: user                 User         User who liked
f[6]: icon                 string       Like icon URL
f[7]: double_like_detail   DoubleLikeDetail  Double-like info
f[8]: display_control_info DisplayControlInfo  Show text/icons
f[12]: field_12            bytes        On-screen display config
```

### WebcastFansclubMessage Sample: [`sampled_payload/FansclubMessage.bin`](sampled_payload/FansclubMessage.bin) — 粉丝团加入/升级
```
f[1]: common_info    Common       Header. Contains describe text
f[2]: type           uint32       1=upgrade, 2=join
f[3]: content        string       Description ("εїз蝴蝶海 升级至粉丝团 Lv8")
f[4]: user           User         Member's full profile (all sizes)
f[6]: field_6        bytes        Deeplink URL for fans club
```

---

## 4. 直播间状态

### WebcastRoomUserSeqMessage Sample: [`sampled_payload/RoomUserSeqMessage.bin`](sampled_payload/RoomUserSeqMessage.bin) — 周期性贡献排行
```
f[1]:  common               Common       Standard header
f[2]:  ranks_list           repeated RoomUserSeqMessageContributor  Top users by score
f[3]:  total                int64        Current online count
f[4]:  pop_str              string       Popularity display string
f[5]:  seats_list           repeated RoomUserSeqMessageContributor  Top users alternative
f[6]:  popularity           int64        Room popularity number
f[7]:  total_user           int64        Cumulative unique users
f[8]:  total_user_str       string       Formatted ("10万+")
f[9]:  total_str            string       Formatted ("1万+")
f[10]: online_user_for_anchor string    Anchor's view count ("1.1万")
f[11]: total_pv_for_anchor  string       Cumulative page views ("52.1万")
```

### WebcastRoomStatsMessage Sample: [`sampled_payload/RoomStatsMessage.bin`](sampled_payload/RoomStatsMessage.bin) — 周期性直播间统计
```
f[1]: common          Common       Standard header
f[2]: display_short   string       Short form ("1万")
f[3]: display_middle  string       Medium form ("1万")
f[4]: display_long    string       Long form ("1万在线观众")
f[5]: display_value   int64        Raw number (10764)
f[6]: display_version int64        Version timestamp
f[7]: incremental     bool         Is incremental update
f[8]: is_hidden       bool         Hidden from display
f[9]: total           int64        Total count (same as display_value)
f[10]: display_type   int64        Metric type (1=online count)
```

### WebcastRoomMessage Sample: [`sampled_payload/RoomMessage.bin`](sampled_payload/RoomMessage.bin) — 直播间公告/场景变化
```
f[1]:  common               Common       Standard header
f[2]:  content              string       Usually "" or " "
f[3]:  supprot_landscape    bool         Landscape support
f[4]:  roommessagetype      RoomMsgTypeEnum  Message sub-type
f[5]:  system_top_msg       bool         Is system top message
f[6]:  forced_guarantee     bool         Forced guarantee
f[7]:  field_7              bytes        Status flags {4:50, 8:2}
f[20]: biz_scene            string       Scene ("live_recommend", "public_screen_hot_word_gather")
f[30]: buried_point_map     map<string,string>  Buried point data
```
**注意：** 订阅信息（会员/星守护）嵌入在 `common.describe` 字段的文本中, 不在独立字段中，通过扫描原始字节检测关键词。

### WebcastRoomRankMessage Sample: [`sampled_payload/RoomRankMessage.bin`](sampled_payload/RoomRankMessage.bin) — 完整排行榜快照
```
f[1]: common      Common                    Standard header
f[2]: ranks_list  repeated RoomRankMessageRoomRank  Rank entries
```

### WebcastRoomRankMessageRoomRank — 单条排行条目
```
f[1]: user           User    Ranked user
f[2]: score_str      string  Score display ("12345")
f[3]: profile_hidden bool    Profile hidden
```

### WebcastRoomStreamAdaptationMessage Sample: [`sampled_payload/RoomStreamAdaptationMessage.bin`](sampled_payload/RoomStreamAdaptationMessage.bin) — 推流质量变化
```
f[1]: common          Common   Standard header
f[2]: adaptation_type int32    Adaptation mode
```

### WebcastRoomDataSyncMessage Sample: [`sampled_payload/RoomDataSyncMessage.bin`](sampled_payload/RoomDataSyncMessage.bin) — 直播间数据同步
```
f[1]: common      Common   Standard header
f[2]: room_id     uint64   Room ID
f[3]: sync_type   string   Sync type ("InteractEffectSyncData")
f[4]: trace_id    uint64   Sync trace identifier
f[5]: sync_data   bytes    Data payload (user_id + room_id mapping)
f[6]: request_id  string   Request tracking ID
```

### WebcastRoomNotifyMessage Sample: [`sampled_payload/RoomNotifyMessage.bin`](sampled_payload/RoomNotifyMessage.bin) — 直播间通知
```
f[1]: common        Common      Standard header
f[2]: notify_data   bytes       Notification content
f[3]: notify_type   uint32      Type ID
f[4]: scene         string      Scene ("live_gift_notify")
f[5]: user          User        Associated user
```

### WebcastInRoomBannerMessage Sample: [`sampled_payload/InRoomBannerMessage.bin`](sampled_payload/InRoomBannerMessage.bin) — 直播间内推广横幅
```
f[1]: common       Common   Standard header
f[2]: banner_data  bytes    Banner content (images, text, action URL)
f[3]: banner_type  uint32   Type (2=promotion)
```

### WebcastRanklistHourEntranceMessage Sample: [`sampled_payload/RanklistHourEntranceMessage.bin`](sampled_payload/RanklistHourEntranceMessage.bin) — 小时榜排行入口
```
f[1]: common      Common   Standard header
f[2]: tab_headers bytes    Rank tab headers + detailed entries
    └── repeated f[1]: RankTab — display name + score threshold + category
    └── repeated f[2]: RankSubTab — simplified tab
    └── f[3]: Category Detail — rank name + ID + top user entries (SGROUP)
    └── f[4]: Category Detail — second category's entries
```
**标签字段：** f[1]=name ("百强第N名"), f[2]=threshold (50, 3), f[3]=category ("小时榜")
**详情字段：** f[1]=rank name ("唱歌榜"), f[5]=category ID (51), f[6]=SGROUP user ranking data

### WebcastRoomCommentTopicMessage — 精选评论话题
```
f[1]: common      Common     Standard header
f[2]: topic_data  bytes      Topic image + color
f[3]: count       uint32     Number of comments
f[4]: comments    bytes      Featured comments with user info + text
```

---

## 5. PK / 对战事件

### WebcastLinkMicBattleMethod — PK对战开始
```
f[1]: common      Common      Standard header
f[2]: battle_data bytes       Full battle config (participants, 300s duration, mode)
f[6]: anchor_info bytes       Anchor participant info
f[8]: auto_start  uint32      1=auto start
f[10]: field_10   uint32      1=visible
f[13]: panel      string      "playmode_panel" — UI theme
f[15]: trace_json string      JSON trace ID
```

### WebcastLinkMicBattleFinishMethod Sample: [`sampled_payload/LinkmicBattleFinishMessage.bin`](sampled_payload/LinkmicBattleFinishMessage.bin) — PK对战结束
```
f[1]:  common         Common      Standard header
f[2]:  battle_summary bytes       Battle config + settings (mode, duration, skin)
f[3]:  field_3        bytes       Winning side's team members with scores
f[4]:  field_4        repeated    FINAL TEAM SCORES (one per team):
          f[1]: team_score    uint64   Total score (e.g. 163465)
          f[2]: captain_id    uint64   队长用户ID
          f[8]: captain_id_str string  队长ID(字符串)
          f[16]: rank          uint32  最终排名 (1=winner)
          f[18]: common_score  uint64  公共值 (150141)
          f[19]: score_value   uint64  分数 value
f[12]: winners       repeated    Winning team captains' info (name, avatar)
f[15]: effects       bytes       Battle end animation URLs
f[18]: config_json   string      Full battle config JSON
f[31]: field_31      bytes       Winner info (captain ID, mode, team_id)
```
**分数s are per-team, not per-viewer.** 每支队伍一个汇总分数 in f[4].

### WebcastLinkMicArmiesMethod Sample: [`sampled_payload/LinkmicArmiesMessage.bin`](sampled_payload/LinkmicArmiesMessage.bin) — PK军团/贡献者列表
```
f[1]: common     Common      Standard header
f[2]: side_a     repeated    Side A teams (8 teams, 3 members each)
f[3]: side_b     repeated    Side B teams (same structure)
f[5]: slogan     string      Battle theme ("天使 成为榜单NO.1")
```
**每个队伍条目：**
```
f[1]: captain_id   uint64     Team captain ID
f[2]: members      repeated   Team members:
    f[1]: user_id    uint64   Contributor user ID
    f[2]: score      uint64   Contribution score (30000=high, 5=low)
    f[3]: nickname   string   Contributor nickname
    f[4]: avatar     Image    Avatar URLs
f[3]: captain_id_str string   Captain ID as string
```

### WebcastLinkmicPlayModeUpdateScoreMessage Sample: [`sampled_payload/LinkmicPlayModeUpdateScoreMessage.bin`](sampled_payload/LinkmicPlayModeUpdateScoreMessage.bin) — PK参与者资料更新
```
f[1]:  common      Common       Standard header
f[2]:  round       uint32       Battle round (always 1)
f[3]:  app_id      uint64       App ID (6383=Douyin Web)
f[4]:  room_id     uint64       Room ID
f[5]:  anchor_a    uint64       Anchor A's user ID
f[6]:  anchor_b    uint64       Anchor B's user ID (opponent)
f[7]:  field_7     uint64       Anchor A ID (duplicate)
f[8]:  mode        uint32       PK mode (3=1v1, 101=multi, 501=random)
f[9]:  team_mode   uint32       Team mode
f[10]: session_id  string       Link session ID ("linker_A_B")
f[11]: timestamp   uint64       Battle timestamp
f[15]: opponent    User         Opponent's full profile (name, avatar, level)
f[16]: anchor      User         Your full profile (name, level Lv, followers)
```

### WebcastLinkerContributeMessage Sample: [`sampled_payload/LinkerContributeMessage.bin`](sampled_payload/LinkerContributeMessage.bin) — PK期间观众贡献
```
f[1]: common           Common        Standard header
f[2]: viewer_id        uint64        Contributor's user ID
f[3]: contribution     uint64        Points contributed (e.g. 212,225,679)
f[4]: targets          repeated uint64  Target participant(s) receiving points
f[5]: timestamp        uint64        Contribution timestamp
f[6]: level            uint32        Contribution level/tier (7)
f[7]: cumulative_total string        Side's total formatted ("2.1亿")
f[8]: display_str      string        Formatted contribution ("100万+")
```
构建完整Top-N排行： accumulate f[3] per viewer f[2] across all messages during a PK.

### WebcastLinkMessage Sample: [`sampled_payload/LinkMessage.bin`](sampled_payload/LinkMessage.bin) — 连麦频道管理
```
f[1]: common       Common       Standard header
f[2]: link_type    uint32       Link type (11=mic, 202=channel)
f[3]: channel_id   uint64       Channel/user ID
f[4]: link_action  uint32       Action type (7=update)
f[5]: field_5      uint64       Participant ID
f[8]: field_8      uint64       Secondary channel
f[9]: channel_data bytes        Full channel member data
f[24]: field_24    uint64       Channel flags
```

### WebcastLinkMicMethod Sample: [`sampled_payload/LinkMicMethodMessage.bin`](sampled_payload/LinkMicMethodMessage.bin) / LinkMicMethod — 连麦生命周期事件
```
f[1]: common      Common     Standard header
f[2]: method_type uint32     202=link mic action
f[8]: user_id     uint64     Target user ID
f[17]: params     bytes      Link parameters (channel, user IDs, room IDs)
f[24]: flag       uint64     Status flags
f[104]: other_id  uint64     Another user ID
```

### WebcastBattleTeamTaskMessage Sample: [`sampled_payload/BattleTeamTaskMessage.bin`](sampled_payload/BattleTeamTaskMessage.bin) — 团队挑战进度
```
f[1]: common     Common     Standard header
f[2]: field_2    bytes      Task data
f[3]: field_3    bytes      Task details
```

### WebcastBattleEffectContainerMessage Sample: [`sampled_payload/BattleEffectContainerMessage.bin`](sampled_payload/BattleEffectContainerMessage.bin) — 对战效果数据
```
f[1]: common     Common     Standard header
f[2]: opponent_1 uint64     Opponent 1 ID
f[3]: opponent_2 uint64     Opponent 2 ID
f[4]: effect_data bytes     Effect animations/audio bundle
```

### WebcastBattlePowerContainerMessage Sample: [`sampled_payload/BattlePowerContainerMessage.bin`](sampled_payload/BattlePowerContainerMessage.bin) — 对战能量/投票容器
```
f[1]: common     Common     Standard header
f[2]: opponent_1 uint64     Opponent 1 ID
f[3]: opponent_2 uint64     Opponent 2 ID
f[5]: field_5    bytes      Power data
```

### WebcastNotifyEffectMessage Sample: [`sampled_payload/NotifyEffectMessage.bin`](sampled_payload/NotifyEffectMessage.bin) — 升级/成就通知
```
f[1]: common       Common      Standard header
f[4]: display_data bytes       Effect display info
f[5]: level_range  bytes       Level range (2000-5001)
f[6]: effect_data  bytes       Full effect data
f[10]: event_info  bytes       Event key ("privilege_grade_level_up_new_high_level"), volume 50000
f[20]: field_20    bytes       Map: is_uplevel=1
```

### WebcastAssetEffectUtilMessage — 星守护/特权入口效果
```
f[1]: common         Common      Standard header
f[5]: effect_text    bytes       Text template "{0:user}"
f[8]: effect_source  string      JSON: {"effect_source":"star_guard"}
f[9]: target_user_id string      Target user ID
```

---

## 6. 系统 / 基础设施

### WebcastProfitInteractionScoreMessage Sample: [`sampled_payload/ProfitInteractionScoreMessage.bin`](sampled_payload/ProfitInteractionScoreMessage.bin) — 游戏化积分更新
```
f[1]: common        Common     Standard header
f[2]: field_2       bytes      Reward data (includes anchor ID)
f[3]: field_3       uint32     Flag
f[4]: target_user   uint64     Target user ID
f[5]: timestamp     uint64     Event timestamp
```

### WebcastProfitGameStatusMessage Sample: [`sampled_payload/ProfitGameStatusMessage.bin`](sampled_payload/ProfitGameStatusMessage.bin) — 小游戏状态
```
f[1]: common         Common     Standard header
f[2]: status         uint32     Game status (2=active)
f[10]: title_json    string     JSON title: "{"title":"游戏连屏中"}"
f[18]: game_data     bytes      Map: link_game_name
f[19]: game_room     uint64     Game room ID
```

### WebcastBattleTeamTaskMessage Sample: [`sampled_payload/BattleTeamTaskMessage.bin`](sampled_payload/BattleTeamTaskMessage.bin) — 团队挑战进度
```
f[1]: common     Common     Standard header
f[2]: field_2    bytes      Task data
f[3]: field_3    bytes      Task details
```

### WebcastBattleRankSeasonMessage Sample: [`sampled_payload/BattleRankSeasonMessage.bin`](sampled_payload/BattleRankSeasonMessage.bin) — 赛季排行信息
```
f[1]: common       Common      Standard header
f[2]: season_data  bytes       Season config + rankings
```

### WebcastDecorationModifyMethod Sample: [`sampled_payload/DecorationModifyMethod.bin`](sampled_payload/DecorationModifyMethod.bin) — 直播间装修更新
```
f[1]: common   Common   Standard header
f[2]: json     string   JSON: {"deco_list":[],"reserved_deco_list":[],"version":1670}
```

### WebcastDecorationUpdateMessage Sample: [`sampled_payload/DecorationUpdateMessage.bin`](sampled_payload/DecorationUpdateMessage.bin) — 装修状态同步
```
f[1]: common   Common   Standard header
f[2]: data     string   Always "" (ping-like keepalive)
```

### WebcastActivityEmojiGroupsMessage — 活动表情包
```
f[1]: common       Common      Standard header
f[2]: emoji_data   bytes       Emoji group entries with timestamps
```

### WebcastCommonDotMessage Sample: [`sampled_payload/CommonDotMessage.bin`](sampled_payload/CommonDotMessage.bin) — 界面红点/按钮通知
```
f[1]: common  Common   Standard header
f[2]: dot_key bytes    Dot key ("room_toolbar_anchor_more")
```

### WebcastLowPcuGuideMessage Sample: [`sampled_payload/LowPcuGuideMessage.bin`](sampled_payload/LowPcuGuideMessage.bin) — 低人气引导
```
f[1]: common    Common     Standard header
f[2]: guide_data bytes     Guide parameters
```

### WebcastResidentGuestMessage Sample: [`sampled_payload/ResidentGuestMessage.bin`](sampled_payload/ResidentGuestMessage.bin) — 虚拟观众(常驻用户)
```
f[1]: common        Common   Standard header
f[2]: resident_count uint32  Number of virtual users (2)
```

### WebcastLinkSettingNotifyMessage Sample: [`sampled_payload/LinkSettingNotifyMessage.bin`](sampled_payload/LinkSettingNotifyMessage.bin) — 连麦设置通知
```
f[1]: common     Common     Standard header
f[2]: settings   bytes      Settings change data (type=7)
```

---

## 7. 控制消息

### WebcastControlMessage — 推流状态(开播/结束)
```
f[1]: common   Common    Standard header
f[2]: status   int32     1=live started, 2=paused, 3=ended
```

### WebcastRoomStreamAdaptationMessage Sample: [`sampled_payload/RoomStreamAdaptationMessage.bin`](sampled_payload/RoomStreamAdaptationMessage.bin) — 推流质量
```
f[1]: common          Common   Standard header
f[2]: adaptation_type int32    Adaptation mode
f[3]: height_ratio    float    Height ratio
f[4]: body_center_ratio float  Body center ratio
```






## 快速参考 — 未详细文档的消息


以下为尚未详细文档的消息类型的自动生成字段列表。


### WebcastAnchorLinkmicSilenceMessage — AnchorLinkmicSilenceMessage Sample: [`sampled_payload/AnchorLinkmicSilenceMessage.bin`](sampled_payload/AnchorLinkmicSilenceMessage.bin)


```
f[1]: Common
f[2]: uint64
f[3]: string
f[4]: string
```


### WebcastAssetMessage — AssetMessage


```
f[1]: Common
```


### WebcastAudioBGImgMessage — AudioBgImgMessage Sample: [`sampled_payload/AudioBgImgMessage.bin`](sampled_payload/AudioBgImgMessage.bin)


```
f[1]: Common
f[2]: sfixed32
f[3]: int32
f[4]: int32
```


### WebcastBackupSeiMessage — BackUpSeiMessage


```
f[1]: Common
```


### WebcastBattleAuxiliaryMessage — BattleAuxiliaryMessage


```
f[1]: Common
f[2]: uint64
```


### WebcastBattleEndPunishMessage — BattleEndPunishMessage Sample: [`sampled_payload/BattleEndPunishMessage.bin`](sampled_payload/BattleEndPunishMessage.bin)


```
f[1]: Common
f[2]: PunishmentDetail
```


### WebcastBattleStatusMessage — BattleStatusMessage Sample: [`sampled_payload/BattleStatusMessage.bin`](sampled_payload/BattleStatusMessage.bin)


```
f[1]: Common
f[2]: string
f[3]: string
f[4]: int32
f[6]: int64
f[7]: int64
f[8]: string
f[9]: string
```


### WebcastBattleStealDragonMessage — BattleStealDragonMessage Sample: [`sampled_payload/BattleStealDragonMessage.bin`](sampled_payload/BattleStealDragonMessage.bin)


```
f[1]: Common
f[2]: BattleStealDragonDetail
```


### WebcastBattleUpdatePropCardTaskMessage — BattleUpdatePropCardTaskMessage Sample: [`sampled_payload/BattleUpdatePropCardTaskMessage.bin`](sampled_payload/BattleUpdatePropCardTaskMessage.bin)


```
f[1]: Common
f[2]: string
f[3]: string
f[4]: string
```


### WebcastBindingGiftMessage — BindingGiftMessage


```
f[1]: Common
f[2]: sfixed32
```


### WebcastCommonToastMessage — CommonToastMessage Sample: [`sampled_payload/CommonToastMessage.bin`](sampled_payload/CommonToastMessage.bin)


```
f[1]: Common
f[3]: int32
f[4]: int32
f[6]: string
f[7]: string
f[8]: int32
```


### WebcastExhibitionChatMessage — ExhibitionChatMessage Sample: [`sampled_payload/ExhibitionChatMessage.bin`](sampled_payload/ExhibitionChatMessage.bin)


```
f[1]: Common
f[2]: sfixed32
f[3]: int32
f[4]: int32
f[5]: int32
f[6]: int32
f[7]: string
```


### WebcastGiftEffectGameMessage — GiftEffectGameMessage Sample: [`sampled_payload/GiftEffectGameMessage.bin`](sampled_payload/GiftEffectGameMessage.bin)


```
f[1]: Common
f[2]: int32
f[8]: GiftEffectGameState
```


### WebcastGiftPlayEventMessage — GiftPlayEventMessage Sample: [`sampled_payload/GiftPlayEventMessage.bin`](sampled_payload/GiftPlayEventMessage.bin)


```
f[1]: Common
f[2]: int32
f[3]: sfixed32
```


### WebcastGiftUpdateMessage — GiftUpdateMessage


```
f[1]: Common
```


### WebcastHighlightComment — HighlightCommentMessage


```
f[1]: Common
f[2]: uint64
f[3]: uint64
f[4]: uint64
f[5]: int32
f[6]: uint64
f[7]: string
f[8]: HighlightCommentParticipant
f[9]: int32
f[10]: uint64
f[11]: uint64
f[12]: int32
f[14]: uint64
f[15]: string
```


### WebcastHotRoomMessage — HotRoomMessage


```
f[1]: Common
```


### WebcastInteractEffectMessage — InteractEffectMessage


```
f[1]: Common
```


### WebcastLinkmicPlaymodeMessage — LinkmicPlaymodeMessage Sample: [`sampled_payload/LinkmicPlaymodeMessage.bin`](sampled_payload/LinkmicPlaymodeMessage.bin)


```
f[1]: Common
f[5]: string
```


### WebcastLinkmicProfitMessage — LinkmicProfitMessage Sample: [`sampled_payload/LinkmicProfitMessage.bin`](sampled_payload/LinkmicProfitMessage.bin)


```
f[1]: Common
```


### WebcastLotteryDrawResultEventMessage — LotteryDrawResultEventMessage


```
f[1]: Common
```


### WebcastLotteryEventNewMessage — LotteryEventNewMessage


```
f[1]: Common
```


### WebcastLuckyBoxEndMessage — LuckyBoxEndMessage


```
f[1]: Common
```


### WebcastLuckyBoxMessage — LuckyBoxMessage


```
f[1]: Common
```


### WebcastLuckyBoxRewardMessage — LuckyBoxRewardMessage


```
f[1]: Common
```


### WebcastLuckyBoxTempStatusMessage — LuckyBoxTempStatusMessage


```
f[1]: Common
```


### WebcastPreviewCjRpMessage — PreviewCjRpMessage Sample: [`sampled_payload/PreviewCjRpMessage.bin`](sampled_payload/PreviewCjRpMessage.bin)


```
f[1]: PreviewCjRpRedPacket
f[2]: Common
```


### WebcastPrizeNoticeMessage — PrizeNoticeMessage Sample: [`sampled_payload/PrizeNoticeMessage.bin`](sampled_payload/PrizeNoticeMessage.bin)


```
f[1]: Common
f[2]: int32
f[6]: int32
f[8]: int32
```


### WebcastRankListAwardMessage — RanklistAwardMessage Sample: [`sampled_payload/RanklistAwardMessage.bin`](sampled_payload/RanklistAwardMessage.bin)


```
f[1]: Common
f[2]: int32
f[3]: int32
f[4]: int32
f[9]: int32
```


### WebcastRoomIndicatorMessage — RoomIndicatorMessage Sample: [`sampled_payload/RoomIndicatorMessage.bin`](sampled_payload/RoomIndicatorMessage.bin)


```
f[1]: Common
f[2]: int32
f[3]: int32
f[4]: sfixed32
```


### WebcastTaskCenterEntranceMessage — TaskCenterEntranceMessage Sample: [`sampled_payload/TaskCenterEntranceMessage.bin`](sampled_payload/TaskCenterEntranceMessage.bin)


```
f[1]: Common
f[4]: string
```


### WebcastToastMessage — ToastMessage Sample: [`sampled_payload/ToastMessage.bin`](sampled_payload/ToastMessage.bin)


```
f[1]: Common
```


### WebcastTopEffectMessage — TopEffectMessage


```
f[1]: Common
```


### WebcastUserPrivilegeChangeMessage — UserPrivilegeChangeMessage Sample: [`sampled_payload/UserPrivilegeChangeMessage.bin`](sampled_payload/UserPrivilegeChangeMessage.bin)


```
f[1]: Common
```


## 数据流总结

```
礼物统计：    max(combo_count) × group_count，当 repeat_end=1 时结算
PK 贡献：    累加 LinkerContributeMessage → 按观众汇总
用户标识：    id (17-19位) + sec_uid (base64) + display_id
房间标识：    room_id (19位, 每场直播不变)
去重：    trace_id (单条消息) + (group_id, gift_id, user_id) 增量法
```
