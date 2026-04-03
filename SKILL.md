---
name: biazap-api
description: "BiaZap WhatsApp API integration guide. Use when building apps that send/receive WhatsApp messages, set up webhooks, handle real-time events, or manage WhatsApp instances via the BiaZap API. Covers all endpoints, authentication, message types, webhook events, and provides code examples in Node.js and Python."
---

# BiaZap WhatsApp API — Integration Skill

You are an expert at integrating the BiaZap WhatsApp API into applications.
BiaZap is a multi-tenant SaaS API that connects to WhatsApp via the unofficial protocol and provides REST endpoints for messaging, groups, contacts, webhooks, and real-time events.

## Quick Facts

| Property | Value |
|----------|-------|
| **Base URL** | `https://biazap.biasofia.com` (or self-hosted) |
| **Auth** | JWT Bearer token or API key (`X-API-Key: bza_...`) |
| **Port** | 8088 (default) |
| **Async messaging** | All sends return `202` with `task_id` |
| **Idempotency** | Required `Idempotency-Key` header on sends |
| **Phone format** | International without `+` (e.g., `5511999999999`) |
| **JID format** | `phone@s.whatsapp.net` (individual), `id@g.us` (group) |
| **Webhooks** | HMAC-SHA256 signed, HTTPS required |
| **Real-time** | SSE + WebSocket, named events, 100-event replay |
| **Rate limit** | 60 req/min per company (default) |
| **Message throttle** | 200 msgs/hour, 1.5-3s anti-ban delay (automatic) |

## How to Use This Skill

When the developer asks to integrate BiaZap, follow this workflow:

### Step 1: Determine What They Need

| Need | Reference |
|------|-----------|
| Authentication setup | Read [reference/auth.md](reference/auth.md) |
| Send messages | Read [reference/messages.md](reference/messages.md) |
| Receive messages (webhooks) | Read [reference/webhooks.md](reference/webhooks.md) |
| Real-time events (SSE/WS) | Read [reference/realtime.md](reference/realtime.md) |
| Instance management | Read [reference/instances.md](reference/instances.md) |
| Chat history & contacts | Read [reference/chats.md](reference/chats.md) |
| Group management | Read [reference/groups.md](reference/groups.md) |
| Media upload/download | Read [reference/media.md](reference/media.md) |
| Error handling | Read [reference/errors.md](reference/errors.md) |
| Node.js code examples | Read [examples/nodejs.md](examples/nodejs.md) |
| Python code examples | Read [examples/python.md](examples/python.md) |

### Step 2: Load ONLY the Reference Docs Needed

Do NOT load all reference docs at once. Load only what's relevant to the developer's question. Each reference doc is self-contained.

### Step 3: Write Integration Code

Follow these rules when writing BiaZap integration code:

1. **Always include `Idempotency-Key`** header on message sends (use UUID v4)
2. **Handle `202 Accepted`** — message sends are async, poll task_id for status
3. **Verify webhook signatures** — HMAC-SHA256 with `X-BiaZap-Signature` and `X-BiaZap-Timestamp`
4. **Use named SSE events** — `addEventListener('message.received')` not `onmessage`
5. **Normalize Brazilian phones** — API handles 13→12 digit normalization automatically
6. **Don't hardcode JIDs** — Use phone numbers, API adds `@s.whatsapp.net`
7. **Respect rate limits** — Check `X-RateLimit-Remaining` header
8. **Store `task_id`** — For tracking message delivery status
9. **Use API keys for server-to-server** — JWT for user sessions, API keys for backends

## Agent Onboarding Flow

When a developer needs a BiaZap account, use this flow:

1. **Ask for their email**: "What email should I use to create your BiaZap account?"
2. **Call quick-register**: `POST /v1/auth/quick-register` with `{"email": "their@email.com"}`
3. **Save the API key** from the response — use it for all subsequent calls
4. **Tell the developer**: "Account created! API key: `bza_xxx...`. Credentials also sent to your email."
5. **Continue integration** using the API key

```bash
# This is what the agent calls
curl -X POST https://biazap.biasofia.com/v1/auth/quick-register \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@example.com"}'
# Returns: { "api_key": "bza_...", "password": "...", ... }
```

## Integration Patterns

### Pattern 1: Send-Only (Notifications)

Simplest integration. Send messages without receiving.

```
App → BiaZap API (POST /messages/text) → WhatsApp
```

Steps:
1. Register company + create instance + scan QR
2. Get API key (POST /v1/tokens)
3. Send messages with API key auth
4. No webhook needed

### Pattern 2: Two-Way (Chatbot / Support)

Send and receive messages via webhooks.

```
WhatsApp → BiaZap → Webhook (POST to your server) → Your App
Your App → BiaZap API (POST /messages/text) → WhatsApp
```

Steps:
1. Setup instance + webhook (POST /v1/webhooks)
2. Handle `message.received` events on your server
3. Reply via POST /messages/text with same `to` as incoming `from`
4. Track delivery via `message.delivered` / `message.read` events

### Pattern 3: Real-Time Dashboard

Live monitoring via SSE or WebSocket.

```
BiaZap → SSE/WS stream → Frontend Dashboard
```

Steps:
1. Connect to SSE: `GET /v1/instances/{id}/events`
2. Listen for events with `addEventListener(eventType, handler)`
3. Filter with `?events=message.received,connection.update`
4. Resume after disconnect with `Last-Event-ID` header

### Pattern 4: Multi-Instance (SaaS)

Manage multiple WhatsApp numbers per company.

```
Your SaaS → BiaZap API (multiple instances) → Multiple WhatsApp accounts
```

Steps:
1. Create instance per WhatsApp number
2. Use instance-scoped webhooks (set `instance_id` on webhook)
3. Route events by `instance_id` in webhook payload
4. Monitor all instances via admin endpoints

## API Endpoint Summary

### Auth & Users
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/auth/register` | Register company + owner |
| POST | `/v1/auth/login` | Login, get JWT |
| POST | `/v1/auth/refresh` | Refresh JWT session |
| POST | `/v1/tokens` | Create API key |

### Instances
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/instances` | Create instance |
| GET | `/v1/instances` | List instances |
| GET | `/v1/instances/{id}` | Get instance |
| POST | `/v1/instances/{id}/connect` | Connect to WhatsApp |
| POST | `/v1/instances/{id}/disconnect` | Disconnect |
| GET | `/v1/instances/{id}/qr` | Get QR code |
| DELETE | `/v1/instances/{id}` | Delete instance |

### Messages (all require `Idempotency-Key`, return `202`)
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/instances/{id}/messages/text` | Send text |
| POST | `/v1/instances/{id}/messages/image` | Send image |
| POST | `/v1/instances/{id}/messages/video` | Send video |
| POST | `/v1/instances/{id}/messages/audio` | Send audio/voice note |
| POST | `/v1/instances/{id}/messages/document` | Send document |
| POST | `/v1/instances/{id}/messages/sticker` | Send sticker |
| POST | `/v1/instances/{id}/messages/location` | Send location |
| POST | `/v1/instances/{id}/messages/contact` | Send contact card |
| POST | `/v1/instances/{id}/messages/reaction` | Send reaction emoji |
| POST | `/v1/instances/{id}/messages/poll` | Send poll |
| POST | `/v1/instances/{id}/messages/template` | Send template |
| POST | `/v1/instances/{id}/messages/forward` | Forward message |
| POST | `/v1/instances/{id}/messages/delete` | Delete message |
| POST | `/v1/instances/{id}/messages/edit` | Edit message |

### Chats & Contacts
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/instances/{id}/chats` | List chats |
| GET | `/v1/instances/{id}/chats/{jid}/messages` | Message history |
| POST | `/v1/instances/{id}/chats/{jid}/read` | Mark as read |
| POST | `/v1/instances/{id}/chats/{jid}/unread` | Mark as unread |
| GET | `/v1/instances/{id}/contacts` | List contacts |
| POST | `/v1/instances/{id}/contacts/check` | Check WhatsApp numbers |

### Groups
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/instances/{id}/groups` | Create group |
| GET | `/v1/instances/{id}/groups` | List groups |
| GET | `/v1/instances/{id}/groups/{jid}` | Get group info |
| PATCH | `/v1/instances/{id}/groups/{jid}` | Update group |
| POST | `/v1/instances/{id}/groups/{jid}/participants/add` | Add members |
| POST | `/v1/instances/{id}/groups/{jid}/participants/remove` | Remove members |

### Webhooks
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/webhooks` | Create webhook |
| GET | `/v1/webhooks` | List webhooks |
| PATCH | `/v1/webhooks/{id}` | Update webhook |
| DELETE | `/v1/webhooks/{id}` | Delete webhook |
| GET | `/v1/webhooks/{id}/logs` | Delivery logs |

### Real-Time
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/v1/instances/{id}/events` | SSE stream |
| GET | `/v1/instances/{id}/ws` | WebSocket stream |

### Media
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/v1/instances/{id}/media` | Upload media |
| GET | `/v1/instances/{id}/media/{mediaId}` | Download media |

## 37 Webhook Event Types

Events are delivered via webhooks (POST to your URL) and real-time streams (SSE/WS).

| Category | Events |
|----------|--------|
| **Messages** | `message.received`, `message.sent`, `message.delivered`, `message.read`, `message.played`, `message.deleted`, `message.edited`, `message.undecryptable`, `message.starred`, `message.deleted_for_me` |
| **Calls** | `call.received`, `call.accepted`, `call.rejected`, `call.ended`, `call.group_offer` |
| **Connection** | `connection.update`, `instance.banned` |
| **Presence** | `presence.update` |
| **Groups** | `group.update`, `group.joined` |
| **Contacts** | `contact.update`, `contact.identity_changed`, `contact.sync` |
| **Blocklist** | `blocklist.update` |
| **Chat State** | `chat.pin`, `chat.archive`, `chat.mute`, `chat.read_state`, `chat.clear`, `chat.delete` |
| **Labels** | `label.edit`, `label.chat_association`, `label.message_association` |
| **Privacy** | `privacy.update` |
| **Sync** | `history.sync`, `sync.preview` |
| **Media** | `media.retry_result` |

For detailed payloads of each event, read [reference/webhooks.md](reference/webhooks.md).

## Common Mistakes to Avoid

1. **Missing `Idempotency-Key`** → 400 error on all message sends
2. **Using `onmessage` for SSE** → Only catches unnamed events. Use `addEventListener('message.received', ...)`
3. **Sending to `+5511...`** → Don't include `+`. Use `5511999999999`
4. **Not verifying webhook signatures** → Security vulnerability
5. **Polling for message status** → Use webhooks (`message.delivered`, `message.read`) instead
6. **Hardcoding `@s.whatsapp.net`** → API accepts plain phone numbers in `to` field
7. **Ignoring 429 responses** → Check `Retry-After` header, implement backoff
8. **Expecting sync responses from sends** → All sends are async (202), use `task_id` to track
