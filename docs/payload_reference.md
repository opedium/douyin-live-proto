# Douyin Live WebSocket Payload Reference

Based on analysis of 48 unique message methods from live stream `77994347272` (主播: 皮皮皮皮朱).

---

## 1. Chat & Communication

### WebcastChatMessage `x2806` 440-5382B
**Purpose:** Text chat messages from viewers in the live room.
**Key fields:**
- `common.room_id` — Room identifier
- `user.nick_name`, `user.id`, `user.display_id` — Sender info
- `content` — The chat text content (e.g. `"@︎ ︎ 柒🍬 🌙ིྀ 感谢姐姐"`)
- `public_area_common` — User label/badge info for public chat area
- `individual_chat_priority` — Priority level for chat display
- `event_time` — Timestamp when message was sent
**Usage:** Stream chat display, chat log archival, word cloud, chat analysis.

### WebcastEmojiChatMessage `x12` 3471-4228B
**Purpose:** Emoji/sticker messages.
**Key fields:**
- `user` — Sender info
- `emoji_content` — Text object with emoji image reference
- `default_content` — Fallback text like `"[表情]"`
**Usage:** Emoji reactions display.

### WebcastScreenChatMessage `x89` 4021-4889B
**Purpose:** "Screen chat" — projected/spotlighted chat messages (paid feature).
**Key fields:**
- `field 2` — User object with nick_name, display_id
- `field 4` — Chat content text (e.g. `"@TingTing🌙 感谢"`)
- Contains `chat_rtf_content` — Rich text formatted version
**Usage:** Highlighted/paid spotlight messages displayed prominently.

### WebcastAudioChatMessage `x1` 2229B
**Purpose:** Voice chat messages.
**Key fields:**
- `field 2` — User object with nick_name, avatar
- `field 3` — Text transcript (e.g. `"what's"`)
- `field 4` — Audio file URL
- `field 5` — Audio duration in seconds
**Usage:** Voice chat playback, transcription.

### WebcastPrivilegeScreenChatMessage `x1` 5745B
**Purpose:** Privilege screen chat (subscription/membership chat).
**Key fields:**
- `field 2` — User object (sender, with fans_club info)
- `field 3` — Chat content (e.g. `"你是唯一～"`)
- `field 4` — Gift metadata (containing diamond value)
- `field 5` — Privilege type (3 = subscriber)
**Usage:** Exclusive subscriber chat messages with special display.

### WebcastHotChatMessage `x8` 141-150B
**Purpose:** Hot/trending chat topic suggestions.
**Key fields:**
- `field 2` — Category title (e.g. `"大家说"`)
- `field 3` — Suggested phrase (e.g. `"宝宝"`)
- `field 200` — Tag map: `tag_content_type` = `"comment"`
**Usage:** Chat suggestion bubbles, quick-chat prompts.

### WebcastChatLikeMessage `x10` 3500-3546B
**Purpose:** Liking a specific chat message.
**Key fields:**
- `field 2` — Contains target user_id and message reference
**Usage:** Chat message engagement tracking.

---

## 2. Gifts & Transactions

### WebcastGiftMessage `x4618` 4851-11698B
**Purpose:** Gift events — when a viewer sends a gift.
**Key fields:**
- `common.describe` — Human-readable description (e.g. `"☾⋆:送给主播 1个粉丝团灯牌"`)
- `gift_id` — Gift ID (e.g. 685 = 粉丝团灯牌, 3389 = 欢乐盲盒)
- `user` — Sender (nick_name, id, display_id, gender, pay_grade, fans_club)
- `gift` — GiftStruct with:
  - `name` — Gift name (e.g. `"粉丝团灯牌"`, `"嘉年华"`)
  - `diamond_count` — Price in diamonds
  - `describe` — Description (e.g. `"送出粉丝团灯牌"`)
  - `id` — Gift type ID
  - `duration` — Animation duration ms
- `combo_count` / `repeat_count` — Combo progress
- `group_count` — Group gift count
- `send_type` — 1 = normal, 2 = group gift
- `priority` — Gift IM priority (queue position)
- `trace_id` — Unique trace for dedup
- `total_count` — Running total in combo
- `send_time` — Unix timestamp
- `public_area_common` — User rank/consume badge
- `tray_display_text` — Tray notification text
- `client_gift_source` — 1 = normal panel
- `field 21` — Asset data (gift animation assets, effect IDs)
- `field 39, 43, 45` — Unknown flags/status fields
**Usage:** Gift tracking, revenue calculation, gift animations, thank-you messages, leaderboard contributions.

### WebcastLightGiftMessage `x11578` 330-397B
**Purpose:** "Light" gifts — quick/small gifts like fan tickets, popularity votes.
**Key fields:**
- `msg_type` — Always 1
- `version` — Always 1
- `user_id` — Sender's user ID
- `gift_info.gift_id` — Gift ID
- `gift_info.diamond_count` — Value in diamonds
- `count` — Quantity sent
- `unknown_8` — Contains value 1000 (some timing/interval)
- `field 9` — Unknown (value 4)
- `unknown_12` — Unknown (value 2)
- `field 13` — Nested with gift_id reference
**Usage:** High-frequency small transactions (fan tickets, popularity gifts). ~32% of all messages.

### WebcastGiftSortMessage `x387` 73B
**Purpose:** Gift panel sort order updates (e.g. anchor exhibition mode).
**Key fields:**
- `field 2` — Sort type (3 = exhibition?)
- `field 4` — Scene label (e.g. `"anchor_exhibition"`)
**Usage:** Gift panel UI updates.

### WebcastItemShareMessage `x6` 3743-7542B
**Purpose:** Sharing achievement items (grade league reports, etc.)
**Key fields:**
- `field 2` — Share item metadata (type: `"grade_league"`, title: `"荣耀赛季报告"`)
- `field 3` — Share text template (`"{0:user} 分享了TA的"`)
**Usage:** "User shared an achievement" notifications.

---

## 3. User Activity & Social

### WebcastMemberMessage `x6066` 1318-11214B
**Purpose:** User enter/join notifications — when a viewer enters the room.
**Key fields:**
- `user` — Viewer info (id, nick_name, display_id, gender, pay_grade, fans_club, sec_uid, avatar)
- `member_count` — Current room member count
- `action` — Always 1 (enter)
- `enter_effect_config` — Entry animation effect configuration (type=8 = fansclub)
  - Contains text like `"{0:user} 加入了直播间"`
  - Badge and icon URLs
  - Effect source: `"fansclub"`
- `anchor_display_text` — Text object for anchor display (`"{0:user} 来了"`)
- `public_area_common` — User label for public area
- `field 22` — Map fields: `enter_tip_type`, `msg_show_type`, `msg_content_type`
**Usage:** User count, "X entered" notifications, entry effects/animation, analytics.

### WebcastSocialMessage `x156` 1490-8963B
**Purpose:** Social actions — follows and shares.
**Key fields:**
- `user` — Actor (id, nick_name, display_id, avatar, pay_grade)
- `action` — 1 = follow, 2 = share
- `share_target` — Target user ID (for follows) or share destination
- `follow_count` — Running total of followers
- `public_area_common` — User label
**Usage:** Follow notifications, share tracking, social graph building.

### WebcastLikeMessage `x318` 1469-9443B
**Purpose:** Like events — viewers liking the stream.
**Key fields:**
- `count` — Like count in this message (often 30)
- `total` — Running total likes for this session
- `user` — User who liked (id, nick_name, display_id, pay_grade)
- `display_control_info.show_icons` — Whether to show like icons
- `field 12` — On-screen display info (like count threshold, display mode)
**Usage:** Like counter, like animation triggers.

### WebcastFansclubMessage `x92` 4194-13071B
**Purpose:** Fans club membership events (join/upgrade).
**Key fields:**
- `type` — 1 = upgrade, 2 = join
- `content` — Description (e.g. `"εїз蝴蝶海 刚刚升级至【皮皮皮皮朱】粉丝团 Lv8"`)
- `user` — Member (full user profile with all avatar sizes, signature, fans_club data)
- `field 6` — Contains deeplink URL for fans club
**Usage:** Fans club tracking, loyalty metrics, "welcome to fans club" effects.

---

## 4. Room Status & Statistics

### WebcastRoomMessage `x32` 822-3908B
**Purpose:** Room announcements, scene changes, subscription detection.
**Key fields:**
- `content` — Usually empty (`" "`)
- `field_7` — Status flags `{4: 50, 8: 2}`
- `biz_scene` — Scene identifier (e.g. `"live_recommend"`, `"public_screen_hot_word_gather"`)
- Subscription data is embedded as text in the `common` field (see parser.py `_extract_subscribe`)
**Usage:** Room lifecycle events, subscription detection (会员/星守护).

### WebcastRoomStatsMessage `x13` 98B
**Purpose:** Periodic room statistics.
**Key fields:**
- `display_short` — Short form (e.g. `"1万"`)
- `display_middle` — Medium form (e.g. `"1万"`)
- `display_long` — Long form (e.g. `"1万在线观众"`)
- `display_value` — Raw number (e.g. 10764)
- `total` — Total count (same as display_value)
- `display_type` — Metric type (1 = online count)
**Usage:** Real-time audience count display.

### WebcastRoomUserSeqMessage `x352` 4506-4785B
**Purpose:** Periodic contribution rankings and room stats.
**Key fields:**
- `ranks_list` — Top contributor list (each with user + rank + score)
- `total` — Current online count
- `total_user` — Cumulative unique users
- `total_user_str` — Formatted (e.g. `"10万+"`)
- `total_str` — Formatted total (e.g. `"1万+"`)
- `online_user_for_anchor` — Anchor's online count view
- `total_pv_for_anchor` — Total page views
**Usage:** Leaderboard display, room stats panel, contribution rankings.

### WebcastRoomRankMessage `x2` 18301B
**Purpose:** Full leaderboard ranking snapshot.
**Key fields:**
- `ranks_list` — Rank entries (user + score_str)
- `field 3` — Extended ranking entries with sub-ranks and scores
**Usage:** Detailed leaderboard for web panel.

### WebcastRoomStreamAdaptationMessage `x3` 62B
**Purpose:** Stream quality/bitrate adaptation.
**Key fields:**
- `adaptation_type` — Adaptation mode
**Usage:** Adaptive bitrate streaming.

### WebcastRoomDataSyncMessage `x6` 139-364B
**Purpose:** Room data synchronization.
**Key fields:**
- `field 3` — Sync type (e.g. `"InteractEffectSyncData"`)
- `field 5` — Contains user_id + room_id mapping
- `field 6` — Trace ID
**Usage:** Internal state synchronization.

### WebcastRoomNotifyMessage `x2` 1144-3299B
**Purpose:** Room notification messages.
**Key fields:**
- `field 4` — Notify type (e.g. `"live_gift_notify"`)
- `field 5` — User object (for gift notifications)
**Usage:** Gift notification popups.

### WebcastRoomCommentTopicMessage `x1` 1632B
**Purpose:** Featured comment topics.
**Key fields:**
- `field 2` — Topic image URL and color
- `field 4` — Featured comments with user info and text
**Usage:** "大家都在说" feature comments.

### WebcastInRoomBannerMessage `x3572` 588-4379B
**Purpose:** In-room banner/promotion messages.
**Key fields:**
- `field 2` — Banner content data (user, text, images)
- `field 3` — Banner type (2 = promotion?)
**Usage:** In-room promotional banners.

### WebcastRanklistHourEntranceMessage `x877` 2671-2689B
**Purpose:** Hourly rank list entrance updates.
**Key fields:**
- `field 2` — Contains hour rank data (nested user info, scores)
**Usage:** Hourly ranking system updates.

---

## 5. PK / Battle Events

### LinkMicMethod / WebcastLinkMicMethod `x6110 + x558`
**Purpose:** Link mic connection lifecycle (create, join, leave, etc.)
**Key fields:**
- `field 2` — Method type (202 = ?)
- `field 8` — Target user ID
- `field 17` — Link params (channel ID, user IDs, room IDs)
- `field 104` — Another user ID
**Usage:** Managing mic/link connections between streamers.

### WebcastLinkMicBattleMethod `x2` 19329B
**Purpose:** PK battle start.
**Key fields:**
- `field 2` — Battle config (duration=300s, participants, team mode)
- `field 6` — Anchor info
- `field 10` — Auto-start flag
- `field 15` — Trace ID JSON
- `field 13` — Panel theme (`"playmode_panel"`)
**Usage:** PK match initialization.

### WebcastLinkMicBattleFinishMethod `x2` 15766B
**Purpose:** PK battle end/result.
**Key fields:**
- `field 2` — Battle summary (participants, duration, mode)
- `field 3` — Winner info (user_id)
- `field 4` — Score breakdown (winner, loser scores, win margin)
- `field 9` — Result text (e.g. `"由于对方主动结束PK，我方本场PK被判定为胜利"`)
- `field 12` — Reward/prize info
**Usage:** PK result display, winner announcement.

### WebcastLinkMicArmiesMethod `x10` 6560-17226B
**Purpose:** PK armies participant info.
**Key fields:**
- `field 2` — Army participant data
- `field 5` — Battle text/slogan (e.g. `"其实 成为榜单NO.1"`)
**Usage:** PK team member display.

### WebcastLinkmicPlayModeUpdateScoreMessage `x22` 6614-9147B
**Purpose:** PK score update.
**Key fields:**
- `field 5-11` — Participant 1 (user_id, nick_name, signature, avatar)
- `field 15-16` — Participant 2 (user_id, nick_name, signature, avatar, city)
- Contains full user profiles for both participants
**Usage:** Real-time PK score display.

### WebcastLinkerContributeMessage `x155` 166-170B
**Purpose:** Linker (viewer) contribution update to PK scores.
**Key fields:**
- `field 4` — Target linker ID
- `field 5` — Score timestamp
- `field 6` — Contribution level
- `field 7` — Contribution display string (e.g. `"2.1亿"`)
- `field 8` — Formatted total (e.g. `"100万+"`)
**Usage:** PK contribution tracking.

### WebcastBattleEffectContainerMessage `x59` 1963-45018B
**Purpose:** Battle effect data container.
**Key fields:**
- `field 2-4` — Battle instance IDs
**Usage:** Battle animation/effect system.

### WebcastBattleTeamTaskMessage `x726` 1271-2500B
**Purpose:** Battle team task progress.
**Key fields:**
- `field 2-3` — Task data
**Usage:** Team challenge progress.

### WebcastBattlePowerContainerMessage `x11` 961-1051B
**Purpose:** Battle power/vote container.
**Key fields:**
- `field 2` — Opponent 1 ID
- `field 3` — Opponent 2 ID
**Usage:** Power value management during PK.

### WebcastNotifyEffectMessage `x3` 1702-2026B
**Purpose:** Level-up and achievement notify effects.
**Key fields:**
- `field 4` — Effect display info
- `field 5` — Level range (e.g. 2000-5001)
- `field 10` — Event key (e.g. `"privilege_grade_level_up_new_high_level"`), volume 50000
- `field 20` — Map fields (`is_uplevel=1`)
**Usage:** Level-up animations, achievement popups.

### WebcastBattleRankSeasonMessage `x1` 3715B
**Purpose:** Battle season ranking info.
**Key fields:**
- `field 2` — Season data
- `field 3` — Season description text
**Usage:** Season ranking display.

---

## 6. System / Infrastructure

### WebcastDecorationModifyMethod `x10` 119B
**Purpose:** Room decoration changes.
**Key fields:**
- `field 2` — JSON with deco_list, reserved_deco_list, version
**Usage:** Room decoration UI updates.

### WebcastDecorationUpdateMessage `x10` 65B
**Purpose:** Decoration state update.
**Key fields:**
- `field 2` — Empty string (ping-like)
**Usage:** Decoration sync.

### WebcastProfitInteractionScoreMessage `x849` 176-213B
**Purpose:** Profit/interaction score updates (gamification).
**Key fields:**
- `field 2` — Contains reward anchor ID and interaction data
- `field 4-5` — User IDs and timestamps
**Usage:** Interaction/reward scoring.

### WebcastProfitGameStatusMessage `x2` 143B
**Purpose:** Profit game status updates.
**Key fields:**
- `field 10` — JSON `{"title":"游戏连屏中"}`
- `field 18` — Map: `link_game_name`
- `field 19` — Game room ID
**Usage:** Mini-game status in stream.

### WebcastActivityEmojiGroupsMessage `x2` 1483B
**Purpose:** Activity emoji group updates.
**Key fields:**
- `field 2` — Emoji group data with timestamps
**Usage:** Event/activity emoji packs.

### WebcastCommonDotMessage `x2` 462B
**Purpose:** Common dot/button notification.
**Key fields:**
- `field 2` — Dot key (e.g. `"room_toolbar_anchor_more"`)
**Usage:** UI dot badges, "new feature" indicators.

### WebcastLowPcuGuideMessage `x2` 57B
**Purpose:** Low concurrent user guide.
**Key fields:**
- `field 2` — Guide data (type params)
**Usage:** Streamer guidance when viewership is low.

### WebcastResidentGuestMessage `x4` 63B
**Purpose:** Resident guest (always-present virtual audience) info.
**Key fields:**
- `field 2` — Resident count (2)
**Usage:** Virtual audience simulation.

### WebcastLinkMessage `x3` 14691-17910B
**Purpose:** Link mic channel management.
**Key fields:**
- `field 2-4` — Channel/session management
- `field 9` — Channel member data
**Usage:** Multi-user stream link management.

### WebcastLinkSettingNotifyMessage `x1` 74B
**Purpose:** Link settings notification.
**Key fields:**
- `field 2` — Settings change (type=7)
**Usage:** Link configuration sync.

### WebcastAssetEffectUtilMessage `x2` 3639-3805B
**Purpose:** Asset effect utility (star guardian effects).
**Key fields:**
- `field 5` — Effect text template `"{0:user}"`
- `field 8` — Effect source JSON: `"{"effect_source":"star_guard"}"`
- `field 9` — Target user ID
**Usage:** Star guardian entry effects, special privilege animations.

---

## Data Volume Summary

| Rank | Method | Messages | % of Total |
|------|--------|----------|-----------|
| 1 | WebcastLightGiftMessage | 11,578 | 28.0% |
| 2 | LinkMicMethod | 6,110 | 14.8% |
| 3 | WebcastMemberMessage | 6,066 | 14.7% |
| 4 | WebcastGiftMessage | 4,618 | 11.2% |
| 5 | WebcastInRoomBannerMessage | 3,572 | 8.6% |
| 6 | WebcastChatMessage | 2,806 | 6.8% |
| 7 | WebcastRanklistHourEntranceMessage | 877 | 2.1% |
| 8 | WebcastProfitInteractionScoreMessage | 849 | 2.1% |
| 9 | WebcastBattleTeamTaskMessage | 726 | 1.8% |
| 10 | WebcastLinkMicMethod | 558 | 1.3% |
| 11-48 | Others | ~3,000 | ~8.6% |
