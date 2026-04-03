# BiaZap API — Claude Code Skill

A [Claude Code](https://claude.ai/claude-code) skill that teaches Claude how to help developers integrate the **BiaZap WhatsApp API** into any application.

## What This Skill Does

When a developer asks Claude to integrate WhatsApp messaging into their app, this skill provides:

- Complete API reference (93 endpoints, 37 webhook events)
- Ready-to-use code examples (Node.js, Python)
- Integration patterns (send-only, chatbot, real-time dashboard, multi-instance)
- Best practices (HMAC verification, idempotency, rate limiting)

## Install

```bash
# Add to your project
claude skill add github:ronszcka/biazap-api-skill
```

Or manually clone to your skills directory:

```bash
git clone https://github.com/ronszcka/biazap-api-skill ~/.claude/skills/biazap-api
```

## Skill Structure

```
SKILL.md                        # Main skill (routing, patterns, endpoint map)
reference/
  auth.md                       # JWT, API keys, register, login
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

- *"Integrate WhatsApp messaging into my Node.js app using BiaZap"*
- *"Set up a webhook to receive WhatsApp messages"*
- *"Send an image with caption via BiaZap API"*
- *"Create a real-time WhatsApp dashboard with SSE"*
- *"How do I verify BiaZap webhook signatures?"*

## About BiaZap

BiaZap is a multi-tenant SaaS WhatsApp API built in Go. It provides REST endpoints for messaging, groups, contacts, webhooks, and real-time events via the unofficial WhatsApp protocol.

- **API**: REST with JWT/API key auth
- **Messaging**: 14 message types (text, image, video, audio, document, sticker, location, contact, reaction, poll, template, forward, delete, edit)
- **Events**: 37 webhook event types with HMAC-SHA256 signing
- **Real-time**: SSE + WebSocket with 100-event replay buffer
- **Rate limiting**: Per-company, plan-based

## License

MIT
