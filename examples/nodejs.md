# Node.js Integration Examples

## Setup

```bash
npm install axios express
```

```javascript
// biazap.js — Minimal BiaZap API client
const axios = require('axios');

class BiaZap {
  constructor({ baseUrl, token, apiKey }) {
    this.client = axios.create({
      baseURL: baseUrl || 'https://biazap.biasofia.com',
      headers: apiKey
        ? { 'X-API-Key': apiKey }
        : { 'Authorization': `Bearer ${token}` },
    });
  }

  // --- Messages ---

  async sendText(instanceId, to, text, opts = {}) {
    return this.client.post(`/v1/instances/${instanceId}/messages/text`, {
      to, text, ...opts,
    }, { headers: { 'Idempotency-Key': crypto.randomUUID() } });
  }

  async sendImage(instanceId, to, mediaId, caption = '') {
    return this.client.post(`/v1/instances/${instanceId}/messages/image`, {
      to, media_id: mediaId, caption,
    }, { headers: { 'Idempotency-Key': crypto.randomUUID() } });
  }

  async sendDocument(instanceId, to, mediaId, fileName) {
    return this.client.post(`/v1/instances/${instanceId}/messages/document`, {
      to, media_id: mediaId, file_name: fileName,
    }, { headers: { 'Idempotency-Key': crypto.randomUUID() } });
  }

  async sendReaction(instanceId, to, messageId, emoji) {
    return this.client.post(`/v1/instances/${instanceId}/messages/reaction`, {
      to, message_id: messageId, emoji,
    }, { headers: { 'Idempotency-Key': crypto.randomUUID() } });
  }

  // --- Instances ---

  async listInstances() {
    return this.client.get('/v1/instances');
  }

  async createInstance(name) {
    return this.client.post('/v1/instances', { name });
  }

  async connectInstance(instanceId) {
    return this.client.post(`/v1/instances/${instanceId}/connect`);
  }

  async getQR(instanceId) {
    return this.client.get(`/v1/instances/${instanceId}/qr`);
  }

  // --- Contacts ---

  async checkNumbers(instanceId, phones) {
    return this.client.post(`/v1/instances/${instanceId}/contacts/check`, { phones });
  }

  // --- Chats ---

  async listChats(instanceId) {
    return this.client.get(`/v1/instances/${instanceId}/chats`);
  }

  async getMessages(instanceId, chatJid, { limit = 50, page = 1 } = {}) {
    return this.client.get(`/v1/instances/${instanceId}/chats/${chatJid}/messages`, {
      params: { limit, page },
    });
  }

  // --- Media ---

  async uploadMedia(instanceId, filePath) {
    const FormData = require('form-data');
    const fs = require('fs');
    const form = new FormData();
    form.append('file', fs.createReadStream(filePath));
    return this.client.post(`/v1/instances/${instanceId}/media`, form, {
      headers: form.getHeaders(),
    });
  }

  // --- Webhooks ---

  async createWebhook(url, events = '*', secret = '') {
    return this.client.post('/v1/webhooks', { url, events, secret });
  }
}

module.exports = BiaZap;
```

## Webhook Receiver (Express)

```javascript
const express = require('express');
const crypto = require('crypto');

const app = express();
app.use(express.json());

const WEBHOOK_SECRET = process.env.BIAZAP_WEBHOOK_SECRET;

// Verify HMAC signature
function verifySignature(req) {
  const signature = req.headers['x-biazap-signature'];
  const timestamp = req.headers['x-biazap-timestamp'];
  if (!signature || !timestamp) return false;

  // Check timestamp skew (max 5 minutes)
  const age = Math.abs(Date.now() / 1000 - parseInt(timestamp));
  if (age > 300) return false;

  const payload = `${timestamp}.${JSON.stringify(req.body)}`;
  const expected = 'sha256=' + crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(payload)
    .digest('hex');

  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}

app.post('/webhook/biazap', (req, res) => {
  if (WEBHOOK_SECRET && !verifySignature(req)) {
    return res.status(401).json({ error: 'invalid signature' });
  }

  const { type, data, instance_id } = req.body;

  switch (type) {
    case 'message.received':
      console.log(`[${instance_id}] Message from ${data.phone}: ${data.content}`);
      // Auto-reply example
      // biazap.sendText(instance_id, data.from, 'Thanks for your message!');
      break;

    case 'message.delivered':
      console.log(`Messages delivered: ${data.message_ids.join(', ')}`);
      break;

    case 'message.read':
      console.log(`Messages read: ${data.message_ids.join(', ')}`);
      break;

    case 'connection.update':
      console.log(`Instance ${instance_id} status: ${data.status}`);
      break;

    case 'call.received':
      console.log(`Call from ${data.phone}, ID: ${data.call_id}`);
      break;
  }

  res.status(200).json({ ok: true });
});

app.listen(3000, () => console.log('Webhook server on port 3000'));
```

## Real-Time Events (SSE)

```javascript
const EventSource = require('eventsource');

const es = new EventSource(
  `https://biazap.biasofia.com/v1/instances/${instanceId}/events?events=message.received`,
  { headers: { 'Authorization': `Bearer ${token}` } }
);

es.addEventListener('message.received', (e) => {
  const event = JSON.parse(e.data);
  console.log(`From ${event.data.phone}: ${event.data.content}`);
});

es.addEventListener('connection.update', (e) => {
  const event = JSON.parse(e.data);
  console.log(`Connection: ${event.data.status}`);
});

es.onerror = (err) => console.error('SSE error:', err);
```

## Complete Flow: Register → Connect → Send

```javascript
const BiaZap = require('./biazap');

async function main() {
  // 1. Register company
  const { data: reg } = await axios.post('https://biazap.biasofia.com/v1/auth/register', {
    company_name: 'My App',
    email: 'dev@myapp.com',
    password: 'SecurePass123!',
  });
  const token = reg.token;

  // 2. Create client
  const bz = new BiaZap({ token });

  // 3. Create instance
  const { data: inst } = await bz.createInstance('production');
  const instanceId = inst.id;

  // 4. Connect (get QR code)
  await bz.connectInstance(instanceId);
  const { data: qr } = await bz.getQR(instanceId);
  console.log('Scan QR:', qr.qr_code); // base64 QR image

  // 5. Wait for connection (poll or use SSE)
  // ...

  // 6. Send message
  const { data: msg } = await bz.sendText(instanceId, '5511999999999', 'Hello from my app!');
  console.log('Task ID:', msg.task_id);

  // 7. Set up webhook
  await bz.createWebhook('https://myapp.com/webhook/biazap', '*', 'my-secret');
}
```
