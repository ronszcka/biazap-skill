# BiaZap API - Message Sending Reference

Base URL: `https://biazap.biasofia.com` (prod) | `http://localhost:8088` (dev)

## Common Pattern

- **Auth:** `Authorization: Bearer <jwt>` or `X-API-Key: bza_...`
- **Required header:** `Idempotency-Key: <unique-string>` (prevents duplicates on retry)
- **Response:** `202 Accepted` with `{"status":"queued","task_id":"asynq:task:xxxx","idempotency_key":"..."}`
- **Idempotency:** Same key + instance + operation type returns existing task state, no duplicate

## Recipient Format (`to` field)

- Phone number: `5541999999999` (auto-converted to JID)
- Individual JID: `5541999999999@s.whatsapp.net`
- Group JID: `120363012345678901@g.us`

**Brazilian numbers:** 13-digit `55XX9XXXXXXXX` auto-normalized to 12-digit `55XXXXXXXXXX` (extra 9 stripped).

## Media Source (image/video/audio/document/sticker)

Two mutually exclusive fields -- one is required:
- `media_url` - Public URL to download from
- `media_id` - Pre-uploaded media ID from `POST /v1/instances/{id}/media`

## Shared Optional Fields

These fields are available on most send endpoints (text, image, video, audio, document, sticker, location, contact):
- `quoted_id` (string) - WhatsApp message ID to reply to
- `mentions` ([]string) - JIDs to @mention

## Quick Setup

```bash
export BASE="https://biazap.biasofia.com"
export ID="your-instance-id"
export TOKEN="your-jwt-token"
# Or use: -H "X-API-Key: bza_your_api_key_here"
```

---

## 1. Text -- `POST /v1/instances/{instanceId}/messages/text`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `text` | string | yes | Message content |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/text" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: txt-$(uuidgen)" \
  -d '{"to":"5541999999999","text":"Hello from BiaZap!"}'
```

## 2. Image -- `POST /v1/instances/{instanceId}/messages/image`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `media_url` / `media_id` | string | yes | Image source |
| `caption` | string | no | Image caption |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/image" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: img-$(uuidgen)" \
  -d '{"to":"5541999999999","media_url":"https://example.com/photo.png","caption":"Check this out"}'
```

## 3. Video -- `POST /v1/instances/{instanceId}/messages/video`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `media_url` / `media_id` | string | yes | Video source |
| `caption` | string | no | Video caption |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/video" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: vid-$(uuidgen)" \
  -d '{"to":"5541999999999","media_url":"https://example.com/clip.mp4","caption":"Watch this"}'
```

## 4. Audio -- `POST /v1/instances/{instanceId}/messages/audio`

Sent as PTT (voice note) by default. Non-OGG formats auto-converted to OGG Opus (mono, 16kHz, 32kbps) via ffmpeg. 64-byte waveform generated automatically.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `media_url` / `media_id` | string | yes | Audio source (any format: MP3, OGG, M4A, WAV) |
| `ptt` | bool | no | Default `true` (voice note). `false` = regular audio, no conversion |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/audio" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: aud-$(uuidgen)" \
  -d '{"to":"5541999999999","media_url":"https://example.com/voice.mp3"}'
```

## 5. Document -- `POST /v1/instances/{instanceId}/messages/document`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `media_url` / `media_id` | string | yes | Document source |
| `file_name` | string | no | Display filename in WhatsApp |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/document" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: doc-$(uuidgen)" \
  -d '{"to":"5541999999999","media_url":"https://example.com/report.pdf","file_name":"report-q1.pdf"}'
```

## 6. Sticker -- `POST /v1/instances/{instanceId}/messages/sticker`

Image must be **WebP** format.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `media_url` / `media_id` | string | yes | Sticker source (WebP) |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/sticker" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: stk-$(uuidgen)" \
  -d '{"to":"5541999999999","media_url":"https://example.com/sticker.webp"}'
```

## 7. Location -- `POST /v1/instances/{instanceId}/messages/location`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `latitude` | float | yes | Latitude |
| `longitude` | float | yes | Longitude |
| `name` | string | no | Place name |
| `address` | string | no | Address text |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/location" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: loc-$(uuidgen)" \
  -d '{"to":"5541999999999","latitude":-25.4284,"longitude":-49.2733,"name":"Downtown","address":"Main St"}'
```

## 8. Contact (vCard) -- `POST /v1/instances/{instanceId}/messages/contact`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `contact_name` | string | yes | Contact display name |
| `contact_phone` | string | yes | Contact phone number |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/contact" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: cnt-$(uuidgen)" \
  -d '{"to":"5541999999999","contact_name":"Support Team","contact_phone":"5541988887777"}'
```

## 9. Reaction -- `POST /v1/instances/{instanceId}/messages/reaction`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Chat JID where the message is |
| `message_id` | string | yes | WhatsApp message ID to react to |
| `emoji` | string | yes | Emoji (empty string to remove reaction) |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/reaction" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: rct-$(uuidgen)" \
  -d '{"to":"5541999999999","message_id":"3EB0ABC123DEF456","emoji":"\ud83d\udc4d"}'
```

## 10. Poll -- `POST /v1/instances/{instanceId}/messages/poll`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `question` | string | yes | Poll question |
| `options` | []string | yes | Answer options (minimum 2) |
| `max_selections` | int | no | Max selections per voter (default: 1) |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/poll" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: poll-$(uuidgen)" \
  -d '{"to":"5541999999999","question":"Best day?","options":["Monday","Tuesday","Wednesday"],"max_selections":1}'
```

## 11. Template (Buttons) -- `POST /v1/instances/{instanceId}/messages/template`

Requires **WhatsApp Business** account.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Recipient |
| `body_text` | string | yes | Main message text |
| `footer_text` | string | no | Footer text |
| `buttons` | []object | yes | 1-3 buttons: `{"text":"Label","id":"btn_id"}` |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/template" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: tpl-$(uuidgen)" \
  -d '{"to":"5541999999999","body_text":"Choose:","buttons":[{"text":"Buy","id":"btn_buy"},{"text":"Cancel","id":"btn_cancel"}]}'
```

## 12. Forward -- `POST /v1/instances/{instanceId}/messages/forward`

Forwards an existing message to another chat. Works with all types.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Destination recipient |
| `forward_chat_id` | string | yes | JID of chat containing the original message |
| `message_id` | string | yes | WhatsApp ID of message to forward |

```bash
curl -X POST "$BASE/v1/instances/$ID/messages/forward" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: fwd-$(uuidgen)" \
  -d '{"to":"5541988888888","forward_chat_id":"5541999999999@s.whatsapp.net","message_id":"3EB0ABC123DEF456"}'
```

## 13. Delete -- `DELETE /v1/instances/{instanceId}/messages/{messageId}`

Deletes a sent message ("delete for everyone").

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `to` | string | yes | Chat JID |
| `message_id` | string | yes | WhatsApp message ID |

```bash
curl -X DELETE "$BASE/v1/instances/$ID/messages/3EB0ABC123DEF456" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: del-$(uuidgen)" \
  -d '{"to":"5541999999999","message_id":"3EB0ABC123DEF456"}'
```

## 14. Edit -- `PUT /v1/instances/{instanceId}/messages/{messageId}/edit`

Edits a sent text message. Only outbound text messages, within ~15 min of sending. The `messageId` in the URL is the WhatsApp message ID, not the internal ID.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `content` | string | yes | New text content |

```bash
curl -X PUT "$BASE/v1/instances/$ID/messages/3EB0ABC123DEF456/edit" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -H "Idempotency-Key: edit-$(uuidgen)" \
  -d '{"content":"Updated message text"}'
```

Errors: 400 if content empty, not outbound, not text, or already deleted. 404 if not found.

---

## Message Lifecycle

```
queued -> sent -> delivered -> read -> played (view-once)
                                  |
                               deleted
```

Webhook events: `message.sent`, `message.delivered`, `message.read`, `message.played`, `message.deleted`, `message.edited`.
