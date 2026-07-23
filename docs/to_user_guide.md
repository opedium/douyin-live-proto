# `to_user` — Tracking Gift Recipients in Group Streams

When a viewer sends a gift in a **group stream** (multi-streamer collaboration), the `WebcastGiftMessage.to_user` field identifies **which specific streamer** received the gift. In single-streamer rooms, this field is null.

---

## Data Flow

### 1. Raw Payload (WebSocket)

```
WebcastGiftMessage:
  f[1]:  common       ... header
  f[7]:  user         ← the VIEWER who sent the gift (sender)
  f[8]:  to_user      ← the STREAMER who received the gift (recipient)
         ├── id:          68093820965
         ├── nick_name:   "皮皮皮皮朱"
         └── display_id:  "yiyizhu1031"
  f[15]: gift         ... gift details
```

In a single-streamer room, `to_user` is empty (null). In a group stream, it's populated with the recipient's `User` object.

### 2. Parser Extraction (`base/parser.py`)

In `parse_gift_msg()` (`base/parser.py:686-687`):

```python
'to_user_name': (msg.to_user.nick_name or msg.to_user.display_id or '') if msg.to_user else '',
'to_user_id': get_user_id(msg.to_user) if msg.to_user else '',
```

**Fallback chain:**
1. `msg.to_user.nick_name` — the streamer's display name (e.g. "皮皮皮皮朱")
2. `msg.to_user.display_id` — short ID fallback (e.g. "yiyizhu1031")  
3. `''` — empty (will fall back to room anchor name in the UI)

### 3. Combo Buffer (`base/parser.py:586-595`)

For combo gifts (连击礼物), the same extraction happens at buffer entry creation:

```python
to_uname = (msg.to_user.nick_name or msg.to_user.display_id or '') if msg.to_user else ''
to_uid = get_user_id(msg.to_user) if msg.to_user else ''
entry = {
    ...
    'to_user_name': to_uname,
    'to_user_id': to_uid,
}
```

When the combo finalizes, `_combo_finalize()` passes these to `record_gift()`.

### 4. Database Write

`record_gift()` (`base/parser.py:2741`) writes both fields:

```python
_write_queue.put_nowait(('gift', session_id, user_id, user_name, gift_name,
                          gift_count, diamond_total, grade, fans_club, group_id,
                          to_user_name, to_user_id))
```

The writer thread inserts into `gift_logs` with the 12-element tuple:

```python
INSERT INTO gift_logs (session_id, user_id, user_name, gift_name, gift_count,
                       diamond_total, grade, fans_club, group_id,
                       to_user_name, to_user_id)
VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
```

### 5. SQLite Schema

```sql
CREATE TABLE IF NOT EXISTS gift_logs (
    ...
    to_user_name TEXT DEFAULT '',   -- recipient streamer's nickname
    to_user_id TEXT DEFAULT '',     -- recipient streamer's user ID
    ...
);
```

### 6. API Query (`app.py:1241-1257`)

```python
SELECT g.gift_name, g.gift_count, g.diamond_total, g.created_at,
       COALESCE(NULLIF(g.to_user_name, ''), s.anchor_name, '') as to_user
FROM gift_logs g
LEFT JOIN sessions s ON s.id = g.session_id
WHERE g.user_id=?
ORDER BY g.created_at DESC
```

The `COALESCE` logic:
- If `g.to_user_name` is non-empty → use it (this is the actual recipient)
- If `g.to_user_name` is empty → fall back to `s.anchor_name` (for old data or single-streamer)
- If both are empty → show blank

### 7. Web Display (`templates/user.html`)

```javascript
// API returns: {gift_name, gift_count, diamond_total, created_at, to_user}
tr.innerHTML =
    '<td>' + time + '</td>' +
    '<td>' + (g.to_user || '') + '</td>' +     // ← shows recipient name
    '<td>' + (g.gift_name || '') + '</td>' +
    '<td>' + (g.gift_count || 0) + '</td>' +
    '<td>' + (g.diamond_total || 0) + '</td>';
```

Column header: **接收主播** (Receiving Streamer)

---

## Testing

### Verify data is being stored

```bash
sqlite3 data/douyin_barrage.db "SELECT to_user_name, to_user_id, gift_name, diamond_total FROM gift_logs WHERE to_user_name != '' LIMIT 10;"
```

### Verify data in web panel

1. Open `/user?uid=VIEWER_ID`
2. Click the **送礼流水** tab
3. Check the **接收主播** column — should show the recipient streamer's name

### For single-streamer rooms

`to_user` will be empty. The `COALESCE` query falls back to the session's `anchor_name`. This means old data and single-streamer gifts will still show correctly.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Shows UID instead of name | `msg.to_user.nick_name` is empty, fallback to `display_id` | Check if `to_user` object has `nick_name` in raw payload |
| Shows old anchor name | Gift was recorded before the `to_user` migration | Old data won't have `to_user_name` — only applies to new gifts |
| Shows blank for group stream gifts | `to_user` might be null in the raw payload | Verify the raw `WebcastGiftMessage` has `f[8]` populated |
