# BiaZap API -- Instance Reference

Base URL: `https://biazap.biasofia.com` (production) or `http://localhost:8088` (development).

All instance endpoints require **Owner** or **Admin** role via JWT (`Authorization: Bearer <token>`) or API key (`X-API-Key: bza_...`).

## Instance Status Lifecycle

```
CREATED --> CONNECTING --> QR_PENDING --> CONNECTED
                                            |
                                        DISCONNECTED
                                            |
                                         BANNED
```

| Status | Description |
|--------|-------------|
| `CREATED` | Instance exists but has never connected |
| `CONNECTING` | Connection to WhatsApp in progress |
| `QR_PENDING` | Waiting for QR code scan |
| `CONNECTED` | Active WhatsApp session |
| `DISCONNECTED` | Session preserved, can reconnect without new QR |
| `BANNED` | Temporary ban from WhatsApp (see `ban_expiry`, `ban_reason`) |
| `TIMED_OUT` | QR codes expired without scan |

## Create Instance

```bash
curl -X POST http://localhost:8088/v1/instances \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Main Support"}'
```

Response `201`:
```json
{
  "id": "84c2e480-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "name": "Main Support",
  "status": "DISCONNECTED",
  "phone": "",
  "created_at": "2026-03-07T10:00:00Z",
  "updated_at": "2026-03-07T10:00:00Z"
}
```

Name must be 3-80 characters and unique within the company. Instance count is limited by the company plan.

## List Instances

```bash
curl http://localhost:8088/v1/instances \
  -H "Authorization: Bearer $TOKEN"
```

Returns an array of instance objects with fields: `id`, `name`, `status`, `phone`, `connected_at`, `last_seen`, `profile_name`, `profile_pic_url`, `last_error`, `ban_expiry`, `ban_reason`, `uptime`, `created_at`, `updated_at`.

## Get Instance

```bash
curl http://localhost:8088/v1/instances/{instanceId} \
  -H "Authorization: Bearer $TOKEN"
```

Returns a single instance object (same shape as list items). Returns `404` if not found.

## Connect Instance

Initiates a WhatsApp connection. This triggers QR code generation for pairing.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/connect \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"qr_retries": 3}'
```

`qr_retries` (optional, 1-3, default 1): how many QR code cycles to attempt before timing out.

## QR Code Flow

### Option 1: Polling (GET)

Fetch the current QR code. Returns `400` if no QR is pending.

```bash
curl http://localhost:8088/v1/instances/{instanceId}/qr \
  -H "Authorization: Bearer $TOKEN"
```

Response `200`:
```json
{
  "qr_code": "2@abc123...",
  "qr_base64": "iVBORw0KGgo...",
  "instance_id": "84c2e480-..."
}
```

`qr_base64` is a PNG image ready for display: `<img src="data:image/png;base64,${qr_base64}">`.

### Option 2: SSE Stream (recommended)

Real-time QR code stream. The server pushes new QR codes as they are generated (~20s each, up to 6 per session).

```bash
curl -N http://localhost:8088/v1/instances/{instanceId}/qr/stream \
  -H "Authorization: Bearer $TOKEN"
```

Events:

| Event | Description |
|-------|-------------|
| `connected` | Stream started (sent immediately) |
| `code` | New QR code available. Fields: `qr` (raw string), `qr_base64` (PNG base64), `mime_type` |
| `success` | User scanned the QR, instance paired |
| `timeout` | All QR codes expired without scan |
| `error` | Error generating QR |

If using a reverse proxy (Nginx/Apache), disable compression and buffering for SSE endpoints to avoid `ERR_INCOMPLETE_CHUNKED_ENCODING`.

## Pairing Code Flow

Alternative to QR: generates a numeric code to pair via phone settings.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/pairing-code \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"phone": "5541999999999"}'
```

Response `200`:
```json
{
  "instance_id": "84c2e480-...",
  "pairing_code": "SG3M-82AE"
}
```

To pair: WhatsApp Settings > Linked Devices > Link a Device > Link with phone number. Enter the code.

## Disconnect Instance

Disconnects from WhatsApp but preserves the session (can reconnect without a new QR).

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/disconnect \
  -H "Authorization: Bearer $TOKEN"
```

## Restart Instance

Restarts the WhatsApp connection (disconnect + reconnect).

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/restart \
  -H "Authorization: Bearer $TOKEN"
```

## Logout Instance

Logs out from WhatsApp and deletes the session. The instance will need to pair again via QR or pairing code.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/logout \
  -H "Authorization: Bearer $TOKEN"
```

## Delete Instance

Permanently removes the instance and all associated data.

```bash
curl -X DELETE http://localhost:8088/v1/instances/{instanceId} \
  -H "Authorization: Bearer $TOKEN"
```

Response `204` (no body).
