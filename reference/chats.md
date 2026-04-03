# BiaZap API -- Chats, Contacts & Presence Reference

Base URL: `https://biazap.biasofia.com` (production) or `http://localhost:8088` (development).

All endpoints in this document require authentication (any role: owner, admin, or agent) via JWT or API key.

## Chats

### List Chats

```bash
curl "http://localhost:8088/v1/instances/{instanceId}/chats?page=1&limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

Query params: `page` (default 1), `limit` (default 20, max 100).

Response `200`:
```json
{
  "data": [
    {
      "jid": "5541999999999@s.whatsapp.net",
      "name": "Joao",
      "last_message": {
        "id": "3EB0ABC123",
        "content": "Hello!",
        "type": "text",
        "timestamp": "2026-03-07T12:00:00Z",
        "from_me": false
      }
    }
  ],
  "total": 25,
  "page": 1,
  "limit": 20
}
```

### Get Chat Detail

```bash
curl http://localhost:8088/v1/instances/{instanceId}/chats/{chatId} \
  -H "Authorization: Bearer $TOKEN"
```

`chatId` is the JID (e.g. `5541999999999@s.whatsapp.net` or `120363012345678901@g.us`).

Response `200`:
```json
{
  "jid": "5541999999999@s.whatsapp.net",
  "name": "Joao",
  "push_name": "Joao Silva",
  "business_name": "",
  "full_name": "Joao Silva",
  "picture_url": "https://..."
}
```

### Message History

```bash
curl "http://localhost:8088/v1/instances/{instanceId}/chats/{chatId}/messages?page=1&limit=50" \
  -H "Authorization: Bearer $TOKEN"
```

Query params: `page` (default 1), `limit` (default 20, max 100).

Response `200`:
```json
{
  "data": [
    {
      "id": 1,
      "external_id": "ext_xxxx",
      "instance_id": "84c2e480-...",
      "direction": "inbound",
      "remote_jid": "5541999999999@s.whatsapp.net",
      "message_type": "text",
      "content": "Hello!",
      "status": "delivered",
      "whatsapp_id": "3EB0ABC123",
      "media_id": "",
      "quoted_msg_id": "",
      "queued_at": null,
      "sent_at": "2026-03-07T12:00:01Z",
      "delivered_at": "2026-03-07T12:00:02Z",
      "read_at": null,
      "failed_at": null,
      "fail_reason": ""
    }
  ],
  "total": 50,
  "page": 1,
  "limit": 20
}
```

### Get Single Message

```bash
curl http://localhost:8088/v1/instances/{instanceId}/message/{messageId} \
  -H "Authorization: Bearer $TOKEN"
```

`messageId` can be the `external_id` (UUID) or `whatsapp_id`. The API tries both automatically. Returns `404` if not found.

### Chat Actions

Perform actions on a chat: archive, unarchive, pin, unpin, mute, unmute, clear.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/chats/{chatId}/action \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"action": "pin"}'
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `action` | string | yes | `archive`, `unarchive`, `pin`, `unpin`, `mute`, `unmute`, `clear` |
| `duration` | int64 | no | Duration in seconds (for `mute`) |

### Mark Read

Mark specific messages or all messages in a chat as read.

```bash
# Specific messages
curl -X POST http://localhost:8088/v1/instances/{instanceId}/mark-read \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"chat_jid": "5541999999999@s.whatsapp.net", "message_ids": ["3EB0ABC123"]}'

# All messages
curl -X POST http://localhost:8088/v1/instances/{instanceId}/mark-read \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"chat_jid": "5541999999999@s.whatsapp.net", "mark_all": true}'
```

### Mark Unread

Mark a conversation as unread. Syncs the unread badge across all connected WhatsApp devices.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/mark-unread \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"chat_jid": "5541999999999@s.whatsapp.net"}'
```

## Contacts

### List Contacts

```bash
curl "http://localhost:8088/v1/instances/{instanceId}/contacts?page=1&limit=20" \
  -H "Authorization: Bearer $TOKEN"
```

Response includes `jid`, `push_name`, `business_name`, `full_name`, `first_name` for each contact.

### Get Contact

```bash
curl http://localhost:8088/v1/instances/{instanceId}/contacts/{contactId} \
  -H "Authorization: Bearer $TOKEN"
```

Returns extended info: `jid`, `push_name`, `business_name`, `full_name`, `first_name`, `status`, `picture_id`, `picture_url`.

### Check WhatsApp Numbers

Verify whether phone numbers have WhatsApp accounts.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/contacts/check \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"numbers": ["5541999999999", "5541988887777"]}'
```

Response `200`:
```json
{
  "instance_id": "84c2e480-...",
  "results": [
    {"query": "5541999999999", "jid": "5541999999999@s.whatsapp.net", "is_in": true, "verified_name": ""},
    {"query": "5541988887777", "jid": "", "is_in": false, "verified_name": ""}
  ]
}
```

### Block / Unblock Contact

Accepts phone number or JID. Brazilian numbers with extra 9 are normalized automatically.

```bash
# Block
curl -X POST http://localhost:8088/v1/instances/{instanceId}/contacts/block \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"contact_id": "5541999999999"}'

# Unblock
curl -X POST http://localhost:8088/v1/instances/{instanceId}/contacts/unblock \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"contact_id": "5541999999999"}'
```

## Presence

### Subscribe to Contact Presence

Start or renew monitoring of a 1:1 contact's presence. Call this when opening a chat view. The lease lasts 300 seconds and is renewed idempotently on repeated calls.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/contacts/{contactId}/presence/subscribe \
  -H "Authorization: Bearer $TOKEN"
```

Accepts phone or JID. Does not accept groups (`@g.us`).

### Get Contact Presence Snapshot

Returns the last known presence/typing state from Redis cache (not MySQL). The snapshot is cached for up to 24 hours. `typing_state` expires after 10 seconds without fresh events.

```bash
curl http://localhost:8088/v1/instances/{instanceId}/contacts/{contactId}/presence \
  -H "Authorization: Bearer $TOKEN"
```

Response `200`:
```json
{
  "contact_jid": "551199999999@s.whatsapp.net",
  "monitoring_active": true,
  "online": true,
  "last_seen": 1741360000,
  "typing_state": "composing",
  "typing_chat": "551199999999@s.whatsapp.net",
  "is_group": false,
  "observed_at": "2026-03-25T15:04:05Z"
}
```

### Set Instance Presence

Set the connected instance's online status or typing indicator.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/presence \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"presence": "composing", "chat_jid": "5541999999999@s.whatsapp.net"}'
```

| Value | Effect |
|-------|--------|
| `available` | Shows "online" |
| `unavailable` | Removes "online" |
| `composing` | Shows "typing..." (requires `chat_jid`) |
| `recording` | Shows "recording audio..." (requires `chat_jid`) |
| `paused` | Removes "typing..." (requires `chat_jid`) |

## Profile

### Get Instance Profile

```bash
# Basic profile
curl http://localhost:8088/v1/instances/{instanceId}/profile \
  -H "Authorization: Bearer $TOKEN"

# Include privacy settings
curl "http://localhost:8088/v1/instances/{instanceId}/profile?include=privacy" \
  -H "Authorization: Bearer $TOKEN"
```

### Update Instance Profile

Supports JSON or multipart (for uploading a profile picture).

```bash
# JSON (name and status only)
curl -X PATCH http://localhost:8088/v1/instances/{instanceId}/profile \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "New Name", "status": "Available 24/7"}'

# Multipart (with picture)
curl -X PATCH http://localhost:8088/v1/instances/{instanceId}/profile \
  -H "Authorization: Bearer $TOKEN" \
  -F "name=New Name" \
  -F "status=Available 24/7" \
  -F "picture=@profile.jpg"
```

Response `200`:
```json
{
  "instance_id": "84c2e480-...",
  "updated": true,
  "fields": ["name", "status"]
}
```
