# Python Integration Examples

## Setup

```bash
pip install requests flask
```

```python
# biazap.py — Minimal BiaZap API client
import uuid
import requests

class BiaZap:
    def __init__(self, base_url="https://api.biazap.com", token=None, api_key=None):
        self.base_url = base_url.rstrip("/")
        self.session = requests.Session()
        if api_key:
            self.session.headers["X-API-Key"] = api_key
        elif token:
            self.session.headers["Authorization"] = f"Bearer {token}"

    def _idem_headers(self):
        return {"Idempotency-Key": str(uuid.uuid4())}

    # --- Messages ---

    def send_text(self, instance_id, to, text, **kwargs):
        return self.session.post(
            f"{self.base_url}/v1/instances/{instance_id}/messages/text",
            json={"to": to, "text": text, **kwargs},
            headers=self._idem_headers(),
        ).json()

    def send_image(self, instance_id, to, media_id, caption=""):
        return self.session.post(
            f"{self.base_url}/v1/instances/{instance_id}/messages/image",
            json={"to": to, "media_id": media_id, "caption": caption},
            headers=self._idem_headers(),
        ).json()

    def send_document(self, instance_id, to, media_id, file_name=""):
        return self.session.post(
            f"{self.base_url}/v1/instances/{instance_id}/messages/document",
            json={"to": to, "media_id": media_id, "file_name": file_name},
            headers=self._idem_headers(),
        ).json()

    def send_location(self, instance_id, to, latitude, longitude, name=""):
        return self.session.post(
            f"{self.base_url}/v1/instances/{instance_id}/messages/location",
            json={"to": to, "latitude": latitude, "longitude": longitude, "name": name},
            headers=self._idem_headers(),
        ).json()

    def send_reaction(self, instance_id, to, message_id, emoji):
        return self.session.post(
            f"{self.base_url}/v1/instances/{instance_id}/messages/reaction",
            json={"to": to, "message_id": message_id, "emoji": emoji},
            headers=self._idem_headers(),
        ).json()

    # --- Instances ---

    def list_instances(self):
        return self.session.get(f"{self.base_url}/v1/instances").json()

    def create_instance(self, name):
        return self.session.post(f"{self.base_url}/v1/instances", json={"name": name}).json()

    def connect(self, instance_id):
        return self.session.post(f"{self.base_url}/v1/instances/{instance_id}/connect").json()

    def get_qr(self, instance_id):
        return self.session.get(f"{self.base_url}/v1/instances/{instance_id}/qr").json()

    # --- Contacts ---

    def check_numbers(self, instance_id, phones):
        return self.session.post(
            f"{self.base_url}/v1/instances/{instance_id}/contacts/check",
            json={"phones": phones},
        ).json()

    # --- Chats ---

    def list_chats(self, instance_id):
        return self.session.get(f"{self.base_url}/v1/instances/{instance_id}/chats").json()

    def get_messages(self, instance_id, chat_jid, limit=50, page=1):
        return self.session.get(
            f"{self.base_url}/v1/instances/{instance_id}/chats/{chat_jid}/messages",
            params={"limit": limit, "page": page},
        ).json()

    # --- Media ---

    def upload_media(self, instance_id, file_path):
        with open(file_path, "rb") as f:
            return self.session.post(
                f"{self.base_url}/v1/instances/{instance_id}/media",
                files={"file": f},
            ).json()

    # --- Webhooks ---

    def create_webhook(self, url, events="*", secret=""):
        return self.session.post(
            f"{self.base_url}/v1/webhooks",
            json={"url": url, "events": events, "secret": secret},
        ).json()
```

## Webhook Receiver (Flask)

```python
import hmac
import hashlib
import time
import json
from flask import Flask, request, jsonify

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret"

def verify_signature(req):
    signature = req.headers.get("X-BiaZap-Signature", "")
    timestamp = req.headers.get("X-BiaZap-Timestamp", "")
    if not signature or not timestamp:
        return False

    # Check timestamp skew (max 5 minutes)
    if abs(time.time() - int(timestamp)) > 300:
        return False

    payload = f"{timestamp}.{req.get_data(as_text=True)}"
    expected = "sha256=" + hmac.new(
        WEBHOOK_SECRET.encode(), payload.encode(), hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(signature, expected)

@app.route("/webhook/biazap", methods=["POST"])
def webhook():
    if WEBHOOK_SECRET and not verify_signature(request):
        return jsonify(error="invalid signature"), 401

    event = request.json
    event_type = event.get("type")
    data = event.get("data", {})
    instance_id = event.get("instance_id")

    if event_type == "message.received":
        print(f"[{instance_id}] From {data['phone']}: {data.get('content', '')}")
        # Handle message (auto-reply, save to DB, etc.)

    elif event_type == "message.delivered":
        print(f"Delivered: {data['message_ids']}")

    elif event_type == "connection.update":
        print(f"Instance {instance_id}: {data['status']}")

    elif event_type == "call.received":
        print(f"Call from {data['phone']}")

    return jsonify(ok=True)

if __name__ == "__main__":
    app.run(port=3000)
```

## Real-Time Events (SSE)

```python
import sseclient  # pip install sseclient-py
import requests
import json

def listen_events(instance_id, token, events_filter=None):
    url = f"https://api.biazap.com/v1/instances/{instance_id}/events"
    params = {}
    if events_filter:
        params["events"] = ",".join(events_filter)

    response = requests.get(
        url, stream=True, params=params,
        headers={"Authorization": f"Bearer {token}", "Accept": "text/event-stream"},
    )
    client = sseclient.SSEClient(response)

    for event in client.events():
        if event.data:
            payload = json.loads(event.data)
            print(f"[{payload['type']}] {payload['data']}")

# Usage
listen_events("inst-1", token, ["message.received", "connection.update"])
```

## Complete Flow

```python
from biazap import BiaZap

# 1. Login
import requests
resp = requests.post("https://api.biazap.com/v1/auth/login", json={
    "email": "dev@myapp.com", "password": "SecurePass123!"
})
token = resp.json()["token"]

# 2. Create client
bz = BiaZap(token=token)

# 3. Create and connect instance
inst = bz.create_instance("production")
instance_id = inst["id"]
bz.connect(instance_id)

# 4. Get QR code
qr = bz.get_qr(instance_id)
print(f"Scan QR: {qr['qr_code']}")  # base64 image

# 5. Send message (after QR scan)
result = bz.send_text(instance_id, "5511999999999", "Hello from Python!")
print(f"Task: {result['task_id']}")

# 6. Send image
media = bz.upload_media(instance_id, "photo.jpg")
bz.send_image(instance_id, "5511999999999", media["media_id"], "Check this!")

# 7. Set up webhook
bz.create_webhook("https://myapp.com/webhook", "*", "my-secret")
```
