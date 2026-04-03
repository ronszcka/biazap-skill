# BiaZap API — Claude Code Skill

A [Claude Code](https://claude.ai/claude-code) skill that teaches Claude how to help developers integrate the **BiaZap WhatsApp API** into any application.

## What This Skill Does

When a developer asks Claude to integrate WhatsApp messaging into their app, this skill provides:

- Complete API reference (93 endpoints, 37 webhook events)
- Ready-to-use code examples (Node.js, Python)
- Integration patterns (send-only, chatbot, real-time dashboard, multi-instance)
- Best practices (HMAC verification, idempotency, rate limiting)
- **Quick account creation** — just provide your email, get an API key instantly

## Install

```bash
npx skills add https://github.com/ronszcka/biazap-skill --skill biazap-api
```

### Alternative methods

```bash
# Via Claude Code CLI
claude skill add github:ronszcka/biazap-skill

# Manual clone (personal — available in all projects)
git clone https://github.com/ronszcka/biazap-skill ~/.claude/skills/biazap-api

# Manual clone (project-only)
git clone https://github.com/ronszcka/biazap-skill .claude/skills/biazap-api
```

### Update

To update the skill to the latest version:

```bash
npx skills add https://github.com/ronszcka/biazap-skill --skill biazap-api
```

Or if manually cloned:

```bash
cd ~/.claude/skills/biazap-api && git pull
```

## Skill Structure

```
SKILL.md                        # Main skill (routing, patterns, endpoint map)
reference/
  auth.md                       # JWT, API keys, quick-register, login
  messages.md                   # 14 message types with curl examples
  webhooks.md                   # 37 events with JSON payload examples
  instances.md                  # Instance lifecycle, QR code, pairing
  chats.md                      # Chat history, contacts, presence, profile
  groups.md                     # Group CRUD, participants, invite links
  media.md                      # Upload/download, size limits
  realtime.md                   # SSE + WebSocket with code examples
  errors.md                     # Status codes, rate limits, throttling
examples/
  nodejs.md                     # Client class + Express webhook + SSE
  python.md                     # Client class + Flask webhook + SSE
```

Claude loads reference docs **on demand** — only what's needed for the developer's question.

## Usage Examples

After installing, just ask Claude:

- *"Create a BiaZap account for me"* (asks your email, calls quick-register)
- *"Integrate WhatsApp messaging into my Node.js app using BiaZap"*
- *"Set up a webhook to receive WhatsApp messages"*
- *"Send an image with caption via BiaZap API"*
- *"Create a real-time WhatsApp dashboard with SSE"*
- *"How do I verify BiaZap webhook signatures?"*

## About BiaZap

BiaZap is a multi-tenant SaaS WhatsApp API built in Go. It provides REST endpoints for messaging, groups, contacts, webhooks, and real-time events via the unofficial WhatsApp protocol.

| Feature | Details |
|---------|---------|
| **Base URL** | `https://biazap.biasofia.com` |
| **Auth** | JWT Bearer or API key (`X-API-Key: bza_...`) |
| **Messaging** | 14 types: text, image, video, audio, document, sticker, location, contact, reaction, poll, template, forward, delete, edit |
| **Events** | 37 webhook event types with HMAC-SHA256 signing |
| **Real-time** | SSE + WebSocket with 100-event replay buffer |
| **Rate limiting** | Per-company, plan-based (default 60 req/min) |

## License

MIT
