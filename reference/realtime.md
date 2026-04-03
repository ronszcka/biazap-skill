# Real-Time Events — SSE & WebSocket

## Server-Sent Events (SSE)

```
GET /v1/instances/{instanceId}/events
Authorization: Bearer <token>
```

**Query params:**
| Param | Description |
|-------|-------------|
| `events` | Comma-separated event filter (e.g., `message.received,message.sent`) |
| `last_event_id` | Resume from event ID (up to 100 events replayed) |

**Headers:**
- `Last-Event-ID` — Alternative to `last_event_id` query param

**Stream format:**
```
event: message.received
id: 550e8400-e29b-41d4-a716-446655440000
data: {"event_id":"550e8400...","type":"message.received","instance_id":"inst-1","timestamp":"2026-04-03T10:00:00Z","data":{...}}

: heartbeat

event: message.delivered
id: 660e8400-e29b-41d4-a716-446655440001
data: {"event_id":"660e8400...","type":"message.delivered",...}
```

### JavaScript Example

```javascript
const eventSource = new EventSource(
  `https://biazap.biasofia.com/v1/instances/${instanceId}/events?events=message.received,message.sent`,
  { headers: { 'Authorization': `Bearer ${token}` } }
);

// Use addEventListener (NOT onmessage) — SSE uses named events
eventSource.addEventListener('message.received', (e) => {
  const event = JSON.parse(e.data);
  console.log('New message:', event.data.content);
});

eventSource.addEventListener('connection.update', (e) => {
  const event = JSON.parse(e.data);
  console.log('Status:', event.data.status);
});
```

### Node.js Example (eventsource package)

```javascript
import EventSource from 'eventsource';

const es = new EventSource(
  `https://biazap.biasofia.com/v1/instances/${instanceId}/events`,
  { headers: { 'Authorization': `Bearer ${token}` } }
);

es.addEventListener('message.received', (e) => {
  const payload = JSON.parse(e.data);
  console.log(`From ${payload.data.phone}: ${payload.data.content}`);
});
```

## WebSocket

```
GET /v1/instances/{instanceId}/ws
Authorization: Bearer <token>
```

**Query params:** Same as SSE (`events`, `last_event_id`)

**Message format (JSON):**
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "type": "message.received",
  "instance_id": "inst-1",
  "timestamp": "2026-04-03T10:00:00Z",
  "data": { ... }
}
```

### JavaScript Example

```javascript
const ws = new WebSocket(
  `wss://biazap.biasofia.com/v1/instances/${instanceId}/ws?events=message.received`
);

// Auth via first message or header (depends on client library)
ws.onopen = () => console.log('Connected');

ws.onmessage = (e) => {
  const event = JSON.parse(e.data);
  if (event.type === 'message.received') {
    console.log('New message:', event.data.content);
  }
};
```

### Python Example

```python
import asyncio
import websockets
import json

async def listen():
    uri = f"wss://biazap.biasofia.com/v1/instances/{instance_id}/ws"
    headers = {"Authorization": f"Bearer {token}"}
    
    async with websockets.connect(uri, extra_headers=headers) as ws:
        async for message in ws:
            event = json.loads(message)
            print(f"[{event['type']}] {event['data']}")

asyncio.run(listen())
```

## Limits & Behavior

| Property | Value |
|----------|-------|
| Max subscribers per instance | 20 |
| Replay buffer | Last 100 events |
| WebSocket ping interval | 30 seconds |
| Overflow response | HTTP 429 (before upgrade for WS) |

## Event Filtering

Filter server-side to reduce bandwidth:

```
# Only message events
?events=message.received,message.sent,message.delivered

# Only connection + call events  
?events=connection.update,call.received,call.ended

# All events (default — no filter)
(omit events param)
```

## Resuming After Disconnect

Both SSE and WebSocket support resume from the last received event:

```
# SSE — via header
Last-Event-ID: 550e8400-e29b-41d4-a716-446655440000

# SSE — via query param
?last_event_id=550e8400-e29b-41d4-a716-446655440000

# WebSocket — via query param
?last_event_id=550e8400-e29b-41d4-a716-446655440000
```

The server replays up to 100 buffered events that occurred after the given ID.
