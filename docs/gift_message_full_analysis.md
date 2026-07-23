# WebcastGiftMessage — Full Field Analysis

**Source:** Live stream `77994347272` (皮皮皮皮朱)  
**Count:** 4618 messages in ~1 hour  
**Size:** 4851–11698 bytes per message

---

## Real Data Examples

```
Room 7658313389554748187
  [1] ☾⋆(891635142) sent 粉丝团灯牌 ×1 (1 diamond)       — gift_id=685
  [2] Poggio(poggioflin) sent 粉丝团灯牌 ×1 (1 diamond)   — gift_id=685
  [3] 哈哈哈哈q(861071892) sent 为你闪耀 ×1               — gift_id=860
```

---

## Field-by-Field Breakdown

### f[1] `common` — Common metadata (see Common class)
| Sub-field | Example | Meaning |
|-----------|---------|---------|
| `method` | `"WebcastGiftMessage"` | Message type identifier |
| `msg_id` | `7658350819817296931` | Unique message ID (19 digits, server-assigned) |
| `room_id` | `7658313389554748187` | Target room ID (same for all msgs in session) |
| `create_time` | `1783098754066` | Unix timestamp (milliseconds? / 1000 = seconds) |
| `is_show_msg` | `1` | Whether this message should be shown in chat |
| `describe` | `"☾⋆:送给主播 1个粉丝团灯牌"` | **Human-readable summary** — `"{user}:送{verb}{count}个{gift_name}"` |
| `display_text` | `<2138B nested>` | Display Text object for anchor/audience UI |
| `priority_score` | `31000` | Chat priority score (higher = more prominent) |
| `field_24` | `1128` | Version/fold flag (1128 = normal) |

### f[2] `gift_id` — Gift type ID
| Value | Gift |
|-------|------|
| 685 | 粉丝团灯牌 (Fan club badge) |
| 860 | 为你闪耀 (Shine for you) |
| 3389 | 欢乐盲盒 (Happy mystery box) |
| 4021 | 欢乐拼图 (Happy puzzle) |

Used for gift dedup and pricing fallback when `gift` object is null.

### f[3] `fan_ticket_count` — Fan ticket/income count
- The number of fan tickets this gift generates for the anchor
- 0 for most small gifts

### f[4] `group_count` — Group gift count
- How many copies of this gift in a group gift send (1 = single, 2+ = group/team gift)
- When > 1, multiple users are sending the same gift together

### f[5] `repeat_count` — Repeat count (cumulative within combo)
- For combo gifts: the cumulative repeat number (1, 2, 3, ..., N)
- For single gifts: always 1
- **Delta dedup:** parser uses delta = current - last to get per-message count

### f[6] `combo_count` — Combo count (same as repeat_count for most cases)
- Same as repeat_count for the first level
- Nested structure: combo_count ≈ repeat_count, both track the running total

### f[7] `user` — Sender (full User object)
| Sub-field | Example | Meaning |
|-----------|---------|---------|
| `id` | `59621943654` | Unique user ID (11-13 digits) |
| `short_id` | `891635142` | Short user ID (numeric display ID) |
| `nick_name` | `"☾⋆"` | Display nickname |
| `gender` | `2` | 1=Male, 2=Female |
| `avatar_thumb` | (Image) | Avatar thumbnail URL object |
| `badge_image_list` | (Image[]) | Active badge images (fan club, achievement) |
| `follow_info` | `{f=12}` | Following/follower counts |
| `pay_grade` | (PayGrade) | Consumption level (diamonds spent) |
| `fans_club` | (FansClub) | Fans club membership data |
| `display_id` | `"891635142"` | Public display ID (usually equals short_id) |
| `sec_uid` | `"MS4w..."` | Secure user ID (base64, used for API lookups) |
| `id_str` | `"59621943654"` | Same as id but as string |
| `nick_name_dup` | `"☾⋆"` | Duplicate of nick_name (at field 68) |
| `sec_uid_dup` | `"MS4w..."` | Duplicate of sec_uid (at field 73) |
| `badge_bundle` | `<599B>` | All badge images + metadata bundled together |

### f[8] `to_user` — Target user (when sending gift to another viewer)
- Present only in user→user gift scenarios (not user→anchor)
- Contains the recipient's User object

### f[9] `repeat_end` — Combo end marker
- 0 = combo still in progress
- 1 = this is the last message of the combo, animation should play

### f[10] `text_effect` — Screen text effect
- Controls the on-screen text animation that appears with the gift
- Contains: text (Text object), start/duration timings, position (x, y), font size, colors
- Not present for small gifts like fan badges

### f[11] `group_id` — Group gift session ID
- Used to group multiple messages in a combo together
- Same group_id across all messages in one combo sequence
- **Key field for gift dedup:** `(group_id, gift_id, user_id)` → last_repeat_count

### f[12] `income_taskgifts` — Income task gifts
- Internal tracking for daily income tasks/challenges
- Usually 0 for normal gifts

### f[13] `room_fan_ticket_count` — Running total fan tickets in room
- Cumulative fan tickets received this session (for display/achievement tracking)

### f[14] `priority` — Gift IM Priority (queue position)
| Sub-field | Example | Meaning |
|-----------|---------|---------|
| `queue_sizes_list` | `[queue_data]` | Per-queue size for backpressure |
| `self_queue_priority` | `1` | This message's queue position |
| `priority` | `3` | Priority level |

Controls the gift message's position in the chat feed. VIP gifts get higher priority.

### f[15] `gift` — GiftStruct (see below)

---

## GiftStruct (field 15) Detail

The actual gift definition — what was sent.

| Field | Example | Meaning |
|-------|---------|---------|
| `image` | (Image) | Gift icon image in the panel |
| `describe` | `"送出粉丝团灯牌"` | Description text shown in UI |
| `notify` | false | Whether to show a system notification |
| `duration` | `3200` | Animation duration in milliseconds |
| `id` | `685` | Gift database ID (same as top-level gift_id) |
| `for_linkmic` | `1` | Can be sent during PK/link mic |
| `doodle` | false | Is a doodle/DIY gift |
| `for_fansclub` | `1` | Is a fans club gift (requires membership) |
| `combo` | false | Supports combo sending |
| `type` | `4` | Gift type: 1=normal, 2=stream, 4=fan badge |
| `diamond_count` | `1` | **Price in diamonds** (1 diamond = 0.1 CNY) |
| `is_displayed_on_panel` | true | Shown in gift selection panel |
| `primary_effect_id` | `1768` | Primary animation effect ID |
| `gift_label_icon` | (Image) | Label badge icon (e.g. "热门" badge) |
| `name` | `"粉丝团灯牌"` | **Gift name** |
| `scheme_url` | `"https://..."` | Gift purchase panel URL (lynx page) |
| `icon` | (Image) | Gift icon alternative |
| `action_type` | `22` | Action type for gift interaction |
| `display_index` | `1` | Panel sort position |
| `fold_type` | `1128` | Display fold strategy |
| `gift_category_id` | `753` | Category: 753=fan badges, others=premium |
| `gift_label` | `"粉丝"` | UI label: "粉丝"(fan), "限定"(limited), "热门"(hot) |
| `display_strategy` | `3` | Display mode: 3=full screen animation |
| `gift_source` | `1` | Source: 1=gift panel, 2=store, 3=activity |
| `sub_type` | `1` | Sub-type: 1=normal, 2=subscription, 3=event |

---

### f[17] `send_type` — How the gift was sent
| Value | Meaning |
|-------|---------|
| 1 | Direct send (from panel) |
| 2 | Group gift (sent as part of group) |

### f[18] `public_area_common` — Public area display badge
| Sub-field | Example | Meaning |
|-----------|---------|---------|
| `user_label` | (Image) | User label shown in public chat area |
| `user_consume_in_room` | `30` | Total diamonds consumed in current room |
| `user_send_gift_cnt_in_room` | `3` | Number of gifts sent in current room |

### f[19] `tray_display_text` — Tray notification text
- Text object for the "tray" notification (top of screen)
- Template: `"送{0}"` — displays as "送 粉丝团灯牌"
- Pieces resolve to the actual gift name

### f[20] `banned_display_effects` — Banned effects mask
- Bitmask of banned effects (0 = none banned)
- Used to filter out certain animations based on room settings

### f[21] `asset_effect_mix_info` — Full asset/effect bundle (812B)
**Contains:** The complete animation package:
- `f[2]` — Effect images (252B)
- `f[5]` — Version (1)
- `f[6]` — Asset type flags (9B)
- `f[9]` — Animation config (204B)
- `f[11]` — Loop flag (1)
- `f[12]` — Display duration ms (2000)
- `f[15]` — Effect sound/particle data (51B)
- `f[16]` — Display position (11B)
- `f[17]` — Gift ID reference (685)
- `f[20]` — End animation/icon (250B)
- `f[22]` — Asset group ID (99)
- `f[23]` — Max display count (9999)

### f[25] `display_for_self` — Show effect to sender
- true = sender sees the animation too
- false = only other viewers see it

### f[26] `interact_gift_info` — Interactive gift info (JSON)
- For interactive gifts (like guessing games), the JSON payload
- Empty `""` for normal gifts

### f[27] `diy_item_info` — DIY gift info (JSON)
- For custom/DIY gifts (e.g. 甄选礼盒)
- JSON array of `[{"values": {"gift_id": "int", "count": int}}]`
- The `parser.py` uses this to sum sub-gift prices

### f[28] `min_asset_set_list` — Minimum asset set IDs
- List of asset IDs that form the minimum display set
- For gifts with multiple asset layers, the IDs that must always show

### f[29] `total_count` — Total count in combo
- For the final message of a combo: the total number sent
- Equal to the last repeat_count value

### f[30] `client_gift_source` — Client source
| Value | Meaning |
|-------|---------|
| 1 | Normal gift panel |
| 2 | Gift store page |
| 3 | Activity/event page |
| 30+ | Third-party/live-room embedded |

### f[31] `anchor_gift_data` — Anchor revenue data (when present)
- Contains anchor-specific gift data: revenue share, user info, amounts
- Only present for gifts with anchor revenue split

### f[32] `to_user_ids_list` — Target user IDs (for group gifts)
- List of user IDs when the gift is sent to multiple recipients
- Empty for single-recipient gifts

### f[33] `send_time` — Send timestamp
- Unix milliseconds timestamp of when the gift was sent

### f[34] `force_display_effects` — Force effects flag
- Bitmask forcing certain effects to display (0 = no force)

### f[35] `trace_id` — Dedup trace ID (32-char hex)
- Unique per gift, used for:
  1. **Shadow dedup** in parser (500ms window)
  2. **Bulk dedup** for `repeat_count > 1` gifts

### f[36] `effect_display_ts` — Effect display timestamp
- When the effect was actually displayed/played (may differ from send_time)

### f[39] `effect_info` — Effect display info (2B)
- Nested `{3: 5}` — layer/position info for the effect display

### f[43] `effect_flag` — Effect flag
- 1 = normal animation
- 0 = suppressed (no animation, badge-only)

### f[45] `send_time_ms_str` — Send time as string
- `"0"` for instant sends
- Contains actual millisecond timestamp for scheduled/queued gifts

---

## Usage Summary

| Purpose | Key Fields |
|---------|-----------|
| **Revenue tracking** | `gift.diamond_count * combo_count` |
| **Who sent it** | `user.id`, `user.nick_name`, `user.display_id` |
| **What gift** | `gift_id`, `gift.id`, `gift.name` |
| **How many** | `combo_count`, `total_count`, `group_count` |
| **Dedup** | `trace_id`, `group_id`, `(group_id, gift_id, user_id)` |
| **Animation** | `asset_effect_mix_info`, `text_effect`, `effect_display_ts` |
| **Leaderboard** | `diamond_count`, `gift_category_id` |
| **User analysis** | `user.pay_grade.level`, `user.fans_club.level` |
| **Combo detection** | `combo_count > repeat_count`, `repeat_end == 1` |
| **Group gifts** | `group_count > 1`, `send_type == 2` |
