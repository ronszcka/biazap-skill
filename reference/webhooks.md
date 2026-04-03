# Webhooks Reference

## Webhook CRUD

### Create Webhook

```
POST /v1/webhooks
Auth: Owner, Admin
```

```json
{
  "url": "https://example.com/webhook",
  "instance_id": "84c2e480-...",
  "events": "message.received,message.sent",
  "secret": "whsec_my_shared_key"
}
```

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `url` | string | yes | Endpoint URL (HTTPS by default) |
| `instance_id` | string | no | Instance to filter. Empty = global (all instances) |
| `events` | string | yes | Comma-separated event types. `*` = all |
| `secret` | string | no | Custom secret. Auto-generated if omitted |

Response 201 includes the `secret` field. This is the ONLY response that exposes the secret.

```json
{
  "id": 1,
  "url": "https://example.com/webhook",
  "secret": "a1b2c3d4e5f6...64_hex_chars",
  "events": "message.received,message.sent",
  "active": true
}
```

### List Webhooks

```
GET /v1/webhooks
Auth: Owner, Admin
```

Response 200: Array of webhook objects (without `secret`).

### Get Webhook

```
GET /v1/webhooks/{webhookId}
Auth: Owner, Admin
```

Response 200: Webhook object (without `secret`).

### Update Webhook

```
PATCH /v1/webhooks/{webhookId}
Auth: Owner, Admin
```

All fields optional:

```json
{
  "url": "https://new-server.com/webhook",
  "events": "*",
  "active": false,
  "secret": "whsec_rotated_manual"
}
```

Response 200: Updated webhook object (without `secret`).

### Delete Webhook

```
DELETE /v1/webhooks/{webhookId}
Auth: Owner, Admin
```

Response 204: No body.

### Delivery Logs

```
GET /v1/webhooks/{webhookId}/logs
Auth: Owner, Admin
Query: page (default 1), limit (default 50, max 100)
```

Retention: success logs deleted after 7 days, failed logs after 14 days. After 48h, payloads are redacted.

### Retry Failed Delivery

```
POST /v1/webhooks/{webhookId}/logs/{logId}/retry
Auth: Owner, Admin
```

Only accepts logs with `success=false` and `payload_raw` still preserved. Returns 409 if already delivered or redacted.

---

## HMAC-SHA256 Signature Verification

Every webhook delivery includes two headers:

```
X-BiaZap-Timestamp: 1711035600
X-BiaZap-Signature: sha256=<hmac_hex>
```

The signature covers `timestamp + "." + raw_body` using the webhook's `secret` as the HMAC key.

Validation rules:
- Reject timestamps with skew > 5 minutes
- Use constant-time comparison
- Deduplicate by `event_id` (delivery is at-least-once)

**Python:**

```python
import hmac, hashlib, time

def verify(body, secret, timestamp, signature):
    if abs(int(time.time()) - int(timestamp)) > 300:
        return False
    signed = f"{timestamp}.".encode() + body
    expected = "sha256=" + hmac.new(
        secret.encode(), signed, hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**Node.js:**

```javascript
const crypto = require('crypto');

function verify(body, secret, timestamp, signature) {
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > 300) return false;
  const signed = Buffer.concat([Buffer.from(`${timestamp}.`), Buffer.from(body)]);
  const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(signed).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(signature));
}
```

---

## Event Envelope

All events share this envelope:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "message.received",
  "instance_id": "84c2e480-...",
  "timestamp": "2026-03-07T15:30:45.123Z",
  "data": { ... }
}
```

---

## All 37 Event Types

### Messages (10 events)

#### message.received

Inbound message. Persisted in `messages` table. Filters out `status@broadcast` and own echoes.

```json
{
  "phone": "554192464230",
  "from": "554192464230@s.whatsapp.net",
  "chat": "554192464230@s.whatsapp.net",
  "sender_lid": "216251804708983:34@lid",
  "chat_lid": "216251804708983@lid",
  "message_id": "3EB0ABC123DEF456",
  "type": "audio",
  "content": "",
  "media_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "mime_type": "audio/ogg; codecs=opus",
  "file_size": 34567,
  "duration": 15,
  "quoted_msg_id": "3EB0AAA111BBB222",
  "reaction_target_id": "",
  "ad_context": null,
  "is_forwarded": false,
  "timestamp": 1741360245,
  "is_group": false,
  "push_name": "Joao Silva"
}
```

Fields: `phone` (plain number), `from`/`chat` (JIDs, resolved from LID when possible), `sender_lid`/`chat_lid` (original LIDs if used internally), `message_id`, `type` (text/image/video/audio/document/sticker/location/contact/reaction/poll/live_location/list/group_invite/view_once), `content`, `media_id`/`mime_type`/`file_name`/`file_size`/`duration` (media fields), `quoted_msg_id` (reply threading), `reaction_target_id` (reaction target), `ad_context` (Instagram/Facebook ad origin), `is_forwarded`, `timestamp`, `is_group`, `push_name`.

`ad_context` fields: `source_type`, `source_url`, `source_app`, `source_id`, `title`, `body`, `thumbnail_url`, `media_type`, `media_url`, `ctwa_clid`, `ref`.

#### message.sent

Outbound message sent successfully. Persisted in `messages` table.

```json
{
  "phone": "554192464230",
  "to": "554192464230@s.whatsapp.net",
  "chat": "554192464230@s.whatsapp.net",
  "chat_lid": "42812401807390@lid",
  "message_id": "3EB0DEF456ABC789",
  "type": "image",
  "content": "Product photo",
  "source": "external",
  "media_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "mime_type": "image/jpeg",
  "file_name": "product.jpg",
  "file_size": 123456,
  "quoted_msg_id": "3EB0AAA111BBB222",
  "reaction_target_id": "",
  "timestamp": 1741360300,
  "is_group": false
}
```

Fields: `phone`, `to`/`chat` (resolved JIDs), `chat_lid`, `message_id`, `type`, `content`, `source` (`"api"` from BiaZap queue or `"external"` from WhatsApp Web/phone/other API), `media_id`/`mime_type`/`file_name`/`file_size` (media), `quoted_msg_id`, `reaction_target_id`, `timestamp`, `is_group`.

#### message.delivered

Recipient received the message (grey double check). Updates `delivered_at` in `messages`.

```json
{
  "phone": "554192464230",
  "message_ids": ["3EB0DEF456ABC789", "3EB0DEF456ABC790"],
  "chat": "554192464230@s.whatsapp.net",
  "sender": "554192464230@s.whatsapp.net",
  "chat_lid": "42812401807390@lid",
  "sender_lid": "42812401807390:77@lid",
  "status": "delivered",
  "timestamp": 1741360310,
  "is_group": false
}
```

Fields: `phone` (resolved via 5-layer fallback), `message_ids` (can be multiple), `chat`/`sender` (resolved JIDs), `chat_lid`/`sender_lid`, `status` (always `"delivered"`), `timestamp`, `is_group`.

#### message.read

Recipient read the message (blue double check). Updates `read_at` in `messages`. Same payload shape as `message.delivered` with `status: "read"`.

```json
{
  "phone": "554192464230",
  "message_ids": ["3EB0DEF456ABC789"],
  "chat": "554192464230@s.whatsapp.net",
  "sender": "554192464230@s.whatsapp.net",
  "chat_lid": "42812401807390@lid",
  "sender_lid": "42812401807390:77@lid",
  "status": "read",
  "timestamp": 1741360350,
  "is_group": false
}
```

#### message.played

Recipient opened view-once media. Updates `played_at` in `messages`. Same shape as delivered/read with `status: "played"`.

```json
{
  "phone": "554192464230",
  "message_ids": ["3EB0DEF456ABC789"],
  "chat": "554192464230@s.whatsapp.net",
  "sender": "554192464230@s.whatsapp.net",
  "status": "played",
  "timestamp": 1741360400,
  "is_group": false
}
```

#### message.deleted

Message deleted for everyone. Updates `status=deleted` and `deleted_at` in `messages`.

```json
{
  "phone": "554192464230",
  "message_id": "3EB0ABC123DEF456",
  "chat": "554192464230@s.whatsapp.net",
  "sender": "554192464230@s.whatsapp.net",
  "chat_lid": "216251804708983@lid",
  "sender_lid": "216251804708983:34@lid",
  "is_from_me": false,
  "timestamp": 1741360400
}
```

Fields: `phone`, `message_id`, `chat`/`sender` (resolved JIDs), `chat_lid`/`sender_lid`, `is_from_me`, `timestamp`.

#### message.edited

Message content edited. Updates `content` in `messages`.

```json
{
  "phone": "554192464230",
  "message_id": "3EB0ABC123DEF456",
  "chat": "554192464230@s.whatsapp.net",
  "sender": "554192464230@s.whatsapp.net",
  "chat_lid": "216251804708983@lid",
  "sender_lid": "216251804708983:34@lid",
  "new_content": "Edited message text",
  "is_from_me": false,
  "is_group": false,
  "push_name": "Joao",
  "timestamp": 1741360450
}
```

Fields: `phone`, `message_id`, `chat`/`sender`, `chat_lid`/`sender_lid`, `new_content`, `is_from_me`, `is_group`, `push_name`, `timestamp`.

#### message.undecryptable

Message received but could not be decrypted. Persisted with status `undecryptable`.

```json
{
  "phone": "554192464230",
  "from": "554192464230@s.whatsapp.net",
  "chat": "554192464230@s.whatsapp.net",
  "message_id": "3EB0ABC123DEF456",
  "is_unavailable": true,
  "unavailable_type": "account_removed",
  "decrypt_fail_mode": "",
  "timestamp": 1741360800
}
```

Fields: `phone`, `from`, `chat`, `message_id`, `is_unavailable`, `unavailable_type` (e.g. `"account_removed"`), `decrypt_fail_mode` (e.g. `"no_session"`), `timestamp`.

#### message.starred

Message starred/unstarred. Not persisted.

```json
{
  "chat": "554192464230@s.whatsapp.net",
  "message_id": "3EB0ABC123DEF456",
  "starred": true,
  "is_from_me": false,
  "timestamp": 1741360850
}
```

Fields: `chat`, `message_id`, `starred` (true=starred, false=unstarred), `is_from_me`, `timestamp`.

#### message.deleted_for_me

Message deleted locally (delete for me only). Not persisted.

```json
{
  "chat": "554192464230@s.whatsapp.net",
  "message_id": "3EB0ABC123DEF456",
  "is_from_me": false,
  "timestamp": 1741360900
}
```

Fields: `chat`, `message_id`, `is_from_me`, `timestamp`.

---

### Calls (5 events)

#### call.received

Incoming call. Persisted in `call_logs`.

```json
{
  "phone": "554192464230",
  "from": "554192464230@s.whatsapp.net",
  "call_id": "CALL-ABC123",
  "timestamp": 1741360500
}
```

#### call.accepted

Call accepted by remote party. Persisted in `call_logs`.

```json
{
  "phone": "554192464230",
  "from": "554192464230@s.whatsapp.net",
  "call_id": "CALL-ABC123",
  "timestamp": 1741360950
}
```

#### call.rejected

Call rejected by remote party. Persisted in `call_logs`.

```json
{
  "phone": "554192464230",
  "from": "554192464230@s.whatsapp.net",
  "call_id": "CALL-ABC123",
  "timestamp": 1741361000
}
```

#### call.ended

Call ended. Persisted in `call_logs`.

```json
{
  "phone": "554192464230",
  "from": "554192464230@s.whatsapp.net",
  "call_id": "CALL-ABC123",
  "reason": "normal",
  "duration": 45,
  "timestamp": 1741361050
}
```

Fields: `phone`, `from`, `call_id`, `reason` (`"normal"`, `"timeout"`, `"busy"`, `"unavailable"`), `duration` (seconds, 0 if unanswered), `timestamp`.

#### call.group_offer

Group call received. Persisted in `call_logs`.

```json
{
  "call_id": "CALL-GRP456",
  "media": "audio",
  "type": "group",
  "timestamp": 1741361100
}
```

Fields: `call_id`, `media` (`"audio"` or `"video"`), `type` (always `"group"`), `timestamp`.

---

### Connection (2 events)

#### connection.update

WhatsApp connection status changed. Persisted in `instances` table.

```json
{
  "status": "connected",
  "reason": "",
  "ban_expiry": null
}
```

Status values: `connected`, `disconnected`, `banned`, `logged_out`, `replaced`, `connect_failure`, `client_outdated`, `stream_error`, `keepalive_timeout`, `keepalive_restored`.

Fields: `status`, `reason` (ban/disconnect reason), `ban_expiry` (unix timestamp, only for `"banned"`).

#### instance.banned

Instance received a WhatsApp ban. Persisted in instance `ban_expiry` and `ban_reason`.

```json
{
  "instance_id": "84c2e480-...",
  "permanent": false,
  "reason": "spam",
  "ban_expiry": 1741446000
}
```

Fields: `instance_id`, `permanent` (true=permanent), `reason`, `ban_expiry` (null if permanent).

---

### Presence (1 event)

#### presence.update

Contact online/offline or typing status. Transient, not persisted in MySQL. Optional Redis snapshot available.

```json
{
  "phone": "554192464230",
  "from": "554192464230@s.whatsapp.net",
  "available": true,
  "last_seen": 1741360000,
  "state": "composing",
  "is_chat": true,
  "chat": "554192464230@s.whatsapp.net",
  "is_group": false
}
```

Fields: `phone` (empty for groups), `from`, `available`, `last_seen` (unix, 0 if hidden; only for online/offline), `state` (`"composing"`, `"paused"`, `"recording"`; only when `is_chat=true`), `is_chat` (true=typing, false=online/offline), `chat` (only for typing), `is_group`.

Note: Requires `POST /v1/instances/{id}/contacts/{contactId}/presence/subscribe` for online/offline presence.

---

### Groups (2 events)

#### group.update

Any group change. Persisted in `group_events`.

```json
{
  "group_jid": "120363025555555555@g.us",
  "action": "participant_join",
  "sender": "554192464230@s.whatsapp.net",
  "timestamp": 1741360600,
  "value": "",
  "members": ["554199999999@s.whatsapp.net", "554188888888@s.whatsapp.net"]
}
```

Actions: `name_change`, `topic_change`, `participant_join`, `participant_leave`, `participant_promote`, `participant_demote`, `locked`, `unlocked`, `announce`, `unannounce`, `picture_change`, `picture_remove`, `ephemeral_change`, `membership_approval_on`, `membership_approval_off`, `group_deleted`, `group_linked`, `group_unlinked`, `invite_link_reset`, `suspended`, `unsuspended`.

Fields: `group_jid`, `action`, `sender`, `timestamp`, `value` (new name/topic/picture_id/link/community JID/duration), `members` (affected JIDs).

#### group.joined

Instance was added to a group. Persisted in `group_events`.

```json
{
  "group_jid": "120363025555555555@g.us",
  "group_name": "Sales 2026",
  "group_topic": "Sales team channel",
  "reason": "invite",
  "sender": "554192464230@s.whatsapp.net",
  "timestamp": 1741361150
}
```

Fields: `group_jid`, `group_name`, `group_topic`, `reason` (`"invite"`, `"link"`, `"admin_add"`), `sender`, `timestamp`.

---

### Contacts (3 events)

#### contact.update

Contact profile change (picture, name, etc.). Persisted in `contact_events`.

```json
{
  "phone": "554192464230",
  "jid": "554192464230@s.whatsapp.net",
  "action": "picture_change",
  "value": "pic_id_123",
  "old_value": "",
  "timestamp": 1741360700
}
```

Actions: `picture_change`, `picture_remove`, `about_change`, `push_name_change`, `business_name_change`.

Fields: `phone`, `jid`, `action`, `value` (new value), `old_value` (previous, only for name changes), `timestamp`.

#### contact.identity_changed

Contact re-registered WhatsApp on a new device (encryption key changed). Persisted in `contact_events`.

```json
{
  "phone": "554192464230",
  "jid": "554192464230@s.whatsapp.net",
  "implicit": false,
  "timestamp": 1741361250
}
```

Fields: `phone`, `jid`, `implicit` (true=detected implicitly, false=server notification), `timestamp`.

#### contact.sync

Contact synced from device address book. Persisted in `contact_events`.

```json
{
  "phone": "554192464230",
  "jid": "554192464230@s.whatsapp.net",
  "first_name": "Joao",
  "full_name": "Joao Silva",
  "timestamp": 1741361300
}
```

Fields: `phone`, `jid`, `first_name`, `full_name`, `timestamp`.

---

### Blocklist (1 event)

#### blocklist.update

Blocked contacts list changed. Not persisted.

```json
{
  "action": "modify",
  "changes": [
    { "phone": "554192464230", "jid": "554192464230@s.whatsapp.net", "action": "block" },
    { "phone": "554188322497", "jid": "554188322497@s.whatsapp.net", "action": "unblock" }
  ],
  "timestamp": 1741361200
}
```

Fields: `action` (`"set"` = full list replaced, `"modify"` = incremental), `changes[]` with `phone`, `jid`, `action` (`"block"` or `"unblock"`), `timestamp`.

---

### Chat State (6 events)

#### chat.pin

Chat pinned/unpinned. Not persisted.

```json
{
  "phone": "554192464230",
  "chat": "554192464230@s.whatsapp.net",
  "pinned": true,
  "timestamp": 1741361350
}
```

#### chat.archive

Chat archived/unarchived. Not persisted.

```json
{
  "phone": "554192464230",
  "chat": "554192464230@s.whatsapp.net",
  "archived": true,
  "timestamp": 1741361400
}
```

#### chat.mute

Chat muted/unmuted. Not persisted.

```json
{
  "phone": "554192464230",
  "chat": "554192464230@s.whatsapp.net",
  "muted": true,
  "mute_end": 1741447800,
  "timestamp": 1741361450
}
```

Fields: `phone`, `chat`, `muted`, `mute_end` (unix timestamp when mute expires, 0=indefinite), `timestamp`.

#### chat.read_state

Chat marked as read/unread. Not persisted.

```json
{
  "phone": "554192464230",
  "chat": "554192464230@s.whatsapp.net",
  "marked_as_read": true,
  "timestamp": 1741361500
}
```

#### chat.clear

Chat history cleared. Not persisted.

```json
{
  "phone": "554192464230",
  "chat": "554192464230@s.whatsapp.net",
  "delete_media": false,
  "timestamp": 1741361550
}
```

#### chat.delete

Chat deleted entirely. Not persisted.

```json
{
  "phone": "554192464230",
  "chat": "554192464230@s.whatsapp.net",
  "delete_media": true,
  "timestamp": 1741361600
}
```

---

### Labels (3 events)

#### label.edit

Label created, edited, or removed. Not persisted.

```json
{
  "label_id": "5",
  "name": "Urgent",
  "color": 1,
  "deleted": false,
  "timestamp": 1741361850
}
```

Fields: `label_id`, `name`, `color` (0-19 in WhatsApp Business), `deleted`, `timestamp`.

#### label.chat_association

Chat labeled/unlabeled. Not persisted.

```json
{
  "label_id": "5",
  "chat": "554192464230@s.whatsapp.net",
  "phone": "554192464230",
  "associated": true,
  "timestamp": 1741361900
}
```

Fields: `label_id`, `chat`, `phone` (empty for groups), `associated`, `timestamp`.

#### label.message_association

Message labeled/unlabeled. Not persisted.

```json
{
  "label_id": "5",
  "chat": "554192464230@s.whatsapp.net",
  "message_id": "3EB0ABC123DEF456",
  "associated": true,
  "timestamp": 1741361950
}
```

Fields: `label_id`, `chat`, `message_id`, `associated`, `timestamp`.

---

### Privacy (1 event)

#### privacy.update

Privacy settings changed. Not persisted.

```json
{
  "settings": {
    "group_add": "contacts",
    "last_seen": "everyone",
    "online": "match_last_seen",
    "profile_photo": "contacts",
    "status": "contacts",
    "read_receipts": "enabled",
    "call_add": "known"
  },
  "changed_fields": ["group_add", "profile_photo"],
  "timestamp": 1741361650
}
```

Fields: `settings` (map of current privacy settings), `changed_fields` (which fields changed in this update), `timestamp`.

Setting keys: `group_add`, `last_seen`, `online`, `profile_photo`, `status`, `read_receipts`, `call_add`. Values vary: `"everyone"`, `"contacts"`, `"nobody"`, `"match_last_seen"`, `"enabled"`, `"disabled"`, `"known"`.

---

### Sync (2 events)

#### history.sync

History sync received from WhatsApp server (typically after connecting a new device). Not persisted.

```json
{
  "sync_type": "RECENT",
  "conversation_count": 42,
  "message_count": 1500,
  "pushname_count": 38,
  "timestamp": 1741361700
}
```

Fields: `sync_type` (`"RECENT"`, `"FULL"`, `"PUSH_NAME"`), `conversation_count`, `message_count`, `pushname_count`, `timestamp`.

#### sync.preview

Offline sync preview indicating pending events. Not persisted.

```json
{
  "total": 250,
  "messages": 180,
  "receipts": 45,
  "notifications": 25,
  "timestamp": 1741361750
}
```

Fields: `total`, `messages`, `receipts`, `notifications`, `timestamp`.

---

### Media (1 event)

#### media.retry_result

Result of a media download retry. Not persisted.

```json
{
  "message_id": "3EB0ABC123DEF456",
  "chat": "554192464230@s.whatsapp.net",
  "success": true,
  "timestamp": 1741361800
}
```

Fields: `message_id`, `chat`, `success`, `error_code` (present only when `success=false`), `timestamp`.

---

## Event Persistence Summary

| Persisted | Events |
|-----------|--------|
| Yes | message.received, message.sent, message.delivered, message.read, message.played, message.deleted, message.edited, message.undecryptable, call.received, call.accepted, call.rejected, call.ended, call.group_offer, connection.update, instance.banned, group.update, group.joined, contact.update, contact.identity_changed, contact.sync |
| No | message.starred, message.deleted_for_me, presence.update, blocklist.update, chat.pin, chat.archive, chat.mute, chat.read_state, chat.clear, chat.delete, label.edit, label.chat_association, label.message_association, privacy.update, history.sync, sync.preview, media.retry_result |

## Key Behaviors

- **Delivery semantics**: At-least-once. Deduplicate by `event_id`.
- **DB-before-webhook**: For message.delivered/read/played/deleted/edited, the tenant DB row is updated BEFORE the webhook is dispatched.
- **Global vs per-instance**: Omit `instance_id` for global webhooks. Avoid creating both global and per-instance webhooks for the same URL (causes duplicates).
- **HTTPS only**: By default. Override with `WEBHOOK_ALLOW_INSECURE_HTTP=true`. Restrict domains with `WEBHOOK_ALLOWED_DOMAINS`/`WEBHOOK_BLOCKED_DOMAINS`.
- **Retry**: Exponential backoff (1s, 2s, 4s, 8s, 16s), max 3 retries.
- **Phone field**: Present in all events except connection.update, instance.banned, and group.update. For LID-based chats, resolved from the identity registry when possible.
