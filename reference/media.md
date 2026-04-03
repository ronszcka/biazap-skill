# Media — Upload & Download

## Upload Media

Upload a file to get a `media_id` for use in message endpoints.

### Multipart Form Upload

```
POST /v1/instances/{instanceId}/media
Content-Type: multipart/form-data
Authorization: Bearer <token>
```

**Form fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | file | Yes (or `url`) | File to upload |
| `url` | string | Yes (or `file`) | URL to download and upload |

**Response (201):**
```json
{
  "media_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "mime_type": "image/jpeg",
  "file_name": "photo.jpg",
  "file_size": 245760
}
```

### From URL

```bash
curl -X POST "https://biazap.biasofia.com/v1/instances/{id}/media" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/image.jpg"}'
```

## Download Media

```
GET /v1/instances/{instanceId}/media/{mediaId}
Authorization: Bearer <token>
```

Returns the raw file with appropriate `Content-Type` header.

```bash
curl -o photo.jpg "https://biazap.biasofia.com/v1/instances/{id}/media/{mediaId}" \
  -H "Authorization: Bearer $TOKEN"
```

## Size Limits

| Type | Max Size |
|------|----------|
| Image | 16 MB |
| Video | 64 MB |
| Audio | 16 MB |
| Document | 64 MB |
| Sticker | 500 KB |

## Using media_id in Messages

After uploading, use the `media_id` in message endpoints:

```bash
curl -X POST "https://biazap.biasofia.com/v1/instances/{id}/messages/image" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "to": "5511999999999",
    "media_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "caption": "Check this out!"
  }'
```

## Inbound Media

When receiving media messages via webhooks, the `message.received` event includes:
- `media_id` — Use with the download endpoint
- `mime_type` — File MIME type
- `file_name` — Original file name (documents)
- `file_size` — Size in bytes
- `duration` — Duration in seconds (audio/video)
