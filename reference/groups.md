# BiaZap API -- Groups Reference

Base URL: `https://biazap.biasofia.com` (production) or `http://localhost:8088` (development).

All group endpoints require authentication (any role: owner, admin, or agent) via JWT or API key. Group modification operations (update, settings, participants, invite link) require the instance to be a group admin on WhatsApp.

Brazilian phone numbers with the extra 9 digit are normalized automatically in all participant fields.

## Create Group

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Sales Team", "participants": ["5541999999999", "5541988887777"]}'
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Group name (max 25 characters) |
| `participants` | []string | yes | Phone numbers or JIDs of initial participants |

Response `201`: the created group object.

## List Groups

```bash
curl http://localhost:8088/v1/instances/{instanceId}/groups \
  -H "Authorization: Bearer $TOKEN"
```

Response `200`:
```json
{
  "data": [
    {
      "jid": "120363012345678901@g.us",
      "name": "Sales Team",
      "participants_count": 5
    }
  ],
  "total": 3
}
```

## Get Group Detail

```bash
curl http://localhost:8088/v1/instances/{instanceId}/groups/{groupId} \
  -H "Authorization: Bearer $TOKEN"
```

`groupId` is the group JID (e.g. `120363012345678901@g.us`).

Response `200`:
```json
{
  "jid": "120363012345678901@g.us",
  "name": "Sales Team",
  "description": "Sales department group",
  "participants": [
    {"jid": "5541999999999@s.whatsapp.net", "is_admin": true},
    {"jid": "5541988887777@s.whatsapp.net", "is_admin": false}
  ],
  "is_announce": false,
  "is_locked": false
}
```

## Update Group

Update group name, description, or picture. Requires group admin.

```bash
curl -X PATCH http://localhost:8088/v1/instances/{instanceId}/groups/{groupId} \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "New Group Name", "description": "Updated description"}'
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | no | New name (max 25 characters) |
| `description` | string | no | New description |
| `picture` | string | no | New photo (base64 JPEG) |

## Group Settings

Control who can send messages and edit group info. Requires group admin.

```bash
curl -X PATCH http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/settings \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"announce": true, "locked": false}'
```

| Field | Type | Description |
|-------|------|-------------|
| `announce` | bool | `true` = only admins can send messages |
| `locked` | bool | `true` = only admins can edit group info |

## Participants

### Add Participants

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/participants/add \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"participants": ["5541999999999", "5541977776666"]}'
```

Response `200`:
```json
{
  "results": [
    {"jid": "5541999999999@s.whatsapp.net", "status": "added"}
  ]
}
```

### Remove Participants

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/participants/remove \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"participants": ["5541999999999"]}'
```

### Promote to Admin

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/participants/promote \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"participants": ["5541999999999"]}'
```

### Demote from Admin

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/participants/demote \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"participants": ["5541999999999"]}'
```

All participant operations return `{"results": [{"jid": "...", "status": "added|removed|promoted|demoted"}]}`.

## Invite Links

### Get Invite Link

```bash
curl http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/invite-link \
  -H "Authorization: Bearer $TOKEN"
```

Response `200`:
```json
{"invite_link": "https://chat.whatsapp.com/AbCdEfGhIjKl"}
```

### Revoke Invite Link

Revokes the current link and generates a new one.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/invite-link/revoke \
  -H "Authorization: Bearer $TOKEN"
```

Response `200`:
```json
{"invite_link": "https://chat.whatsapp.com/NoVoLiNk1234"}
```

## Join Group

Join a group using an invite link.

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups/join \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"invite_link": "https://chat.whatsapp.com/AbCdEfGhIjKl"}'
```

Response `200`:
```json
{"group_jid": "120363012345678901@g.us"}
```

## Leave Group

```bash
curl -X POST http://localhost:8088/v1/instances/{instanceId}/groups/{groupId}/leave \
  -H "Authorization: Bearer $TOKEN"
```

Response `200`:
```json
{"left": true}
```
