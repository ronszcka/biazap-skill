# Error Handling

## HTTP Status Codes

| Code | Meaning | When |
|------|---------|------|
| `200` | Success | GET, PUT, PATCH, DELETE |
| `201` | Created | POST (webhooks, media, register) |
| `202` | Accepted | Async message operations (queued for delivery) |
| `400` | Bad Request | Invalid JSON, missing required fields |
| `401` | Unauthorized | Missing/invalid token or API key |
| `403` | Forbidden | Insufficient permissions, suspended account |
| `404` | Not Found | Resource doesn't exist |
| `409` | Conflict | Duplicate resource (e.g., duplicate instance name) |
| `422` | Unprocessable | Validation error (e.g., invalid phone format) |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Error | Server error |
| `503` | Service Unavailable | Database or dependency down |

## Error Response Format

```json
{
  "error": "instance not connected",
  "code": "INSTANCE_NOT_CONNECTED"
}
```

Some errors include additional context:

```json
{
  "error": "plan limit reached: maximum 3 instances",
  "code": "PLAN_LIMIT_REACHED"
}
```

## Rate Limiting

Rate limits are per-company, based on the subscription plan.

**Headers on every response:**
| Header | Description |
|--------|-------------|
| `X-RateLimit-Limit` | Requests allowed per window |
| `X-RateLimit-Remaining` | Remaining requests |
| `X-RateLimit-Reset` | Window reset time (Unix timestamp) |
| `Retry-After` | Seconds to wait (on 429 responses) |

**Default:** 60 requests/minute per company.

## Message Throttling

WhatsApp has anti-spam protections. BiaZap enforces:

| Limit | Value |
|-------|-------|
| Hourly cap | 200 messages/hour per instance |
| Warmup period | 50 messages/hour (first 24h after connect) |
| Anti-ban delay | 1.5-3s between messages (automatic) |
| Rate limit pause | Auto-resume with exponential backoff |

When throttled, messages are re-queued (not dropped). The `202` response with `task_id` still applies.

## Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `instance not connected` | WhatsApp not connected | Connect instance, scan QR |
| `idempotency key required` | Missing `Idempotency-Key` header | Add UUID header |
| `plan limit reached` | Too many instances | Upgrade plan or delete unused |
| `company suspended` | Account suspended by admin | Contact support |
| `webhook url must use https` | HTTP URL for webhook | Use HTTPS URL |
| `invalid phone format` | Bad phone number | Use international format without + |

## Idempotency

All async message endpoints require an `Idempotency-Key` header:

```bash
curl -X POST ".../messages/text" \
  -H "Idempotency-Key: unique-uuid-here" \
  -d '{"to": "5511999999999", "text": "Hello"}'
```

- Same key returns the same `task_id` (no duplicate send)
- Keys are scoped per instance + operation type
- Recommended: use UUID v4 for each request
