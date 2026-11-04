# AVENIX API Reference

Source analysed: `https://github.com/UpolDev/AVENIX`\
Version: `1.3.0` · Node.js / Express 5 · Prisma · Socket.IO

This document describes the HTTP routes registered in `src/app.js` and
implemented in `src/routes/*.js`, plus the health, root and docs
endpoints. Responses below are representative examples inferred from the
source code; IDs, timestamps, tokens and database fields will differ at
runtime.

## 1. Base URL, headers and authentication

Set your deployed API origin:

``` bash
export BASE_URL="https://upolzrv-avenix.onrender.com"
```

After a successful login OTP verification or email registration
verification, save the returned JWT:

``` bash
export TOKEN="PASTE_JWT_HERE"
```

Authenticated endpoints use:

``` http
Authorization: Bearer YOUR_JWT
Content-Type: application/json
```

In cURL, use `-H "Authorization: Bearer $TOKEN"` and
`-H "Content-Type: application/json"`.

**Important:** registration and login are two-step email OTP flows.
`/api/auth/login` does not return a JWT; `/api/auth/verify-login-otp`
does. `/api/auth/verify-email` returns a JWT after registration
verification.

------------------------------------------------------------------------

## 2. Public / system endpoints

### GET `/api`

No parameters or authentication.

``` bash
curl -s "$BASE_URL/api"
```

Example response:

``` json
{"name":"AVENIX","version":"1.3.0","docs":"/api/docs","health":"/api/health"}
```

### GET `/api/health`

Checks database connectivity. No authentication.

``` bash
curl -i "$BASE_URL/api/health"
```

Example success:

``` json
{"ok":true,"service":"AVENIX-api","version":"1.3.0","database":"connected","timestamp":"2026-11-04T12:00:00.000Z"}
```

Database unavailable returns HTTP 503:

``` json
{"ok":false,"service":"AVENIX-api","database":"unavailable"}
```

### GET `/api/docs`

Returns the endpoint index from the running app. No authentication.

``` bash
curl -s "$BASE_URL/api/docs"
```

Response contains `name`, `version`, `authentication`, `voice`, and an
`endpoints` object grouping route strings by feature.

------------------------------------------------------------------------

## 3. Authentication --- `/api/auth`

### POST `/api/auth/register`

Public. Creates a pending registration and emails a 6-digit OTP. User
row is created only after OTP verification.

Body: - `username` (required): 3--24 chars, letters/numbers/underscore -
`email` (required): valid email - `password` (required): 8--128 chars -
`displayName` (optional): up to 50 chars

``` bash
curl -X POST "$BASE_URL/api/auth/register" \
  -H "Content-Type: application/json" \
  -d '{"username":"alex_01","email":"alex@example.com","password":"StrongPass123","displayName":"Alex"}'
```

Example HTTP 202:

``` json
{"success":true,"stage":"EMAIL_VERIFICATION","email":"alex@example.com","expiresInSeconds":600,"message":"A 6-digit verification code was sent. Submit it to finish creating your AVENIX account."}
```

### POST `/api/auth/verify-email`

Public. Completes registration.

Body: `email`, `otp` (exactly 6 digits).

``` bash
curl -X POST "$BASE_URL/api/auth/verify-email" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com","otp":"123456"}'
```

Example HTTP 201:

``` json
{"success":true,"message":"AVENIX account created and email verified.","user":{"id":"usr_123","username":"alex_01","email":"alex@example.com","displayName":"Alex","avatarUrl":null,"role":"USER"},"token":"eyJ...","tokenType":"Bearer"}
```

### POST `/api/auth/resend-otp`

Public. Resends registration OTP.

Body: `email`.

``` bash
curl -X POST "$BASE_URL/api/auth/resend-otp" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com"}'
```

Example:

``` json
{"success":true,"message":"A new registration code has been sent."}
```

May return HTTP 429 during resend cooldown, or 503 if email delivery
fails.

### POST `/api/auth/login`

Public. Checks email/password and sends login OTP; does not issue JWT
yet.

Body: `email`, `password`.

``` bash
curl -X POST "$BASE_URL/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com","password":"StrongPass123"}'
```

Example HTTP 202:

``` json
{"success":true,"stage":"LOGIN_OTP","email":"alex@example.com","expiresInSeconds":600,"message":"A login verification code was sent to your registered email."}
```

### POST `/api/auth/verify-login-otp`

Public. Verifies login OTP and returns JWT.

Body: `email`, `otp`.

``` bash
curl -X POST "$BASE_URL/api/auth/verify-login-otp" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com","otp":"123456"}'
```

Example:

``` json
{"success":true,"message":"Login successful.","user":{"id":"usr_123","username":"alex_01","email":"alex@example.com","displayName":"Alex","avatarUrl":null,"role":"USER"},"token":"eyJ...","tokenType":"Bearer"}
```

### POST `/api/auth/resend-login-otp`

Public. Requires credentials again and resends login OTP.

Body: `email`, `password`.

``` bash
curl -X POST "$BASE_URL/api/auth/resend-login-otp" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com","password":"StrongPass123"}'
```

Example:

``` json
{"success":true,"message":"A new login code has been sent."}
```

### GET `/api/auth/me`

Bearer token required. Gets current user's profile.

``` bash
curl "$BASE_URL/api/auth/me" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"user":{"id":"usr_123","username":"alex_01","email":"alex@example.com","emailVerified":true,"displayName":"Alex","bio":null,"avatarUrl":null,"role":"USER","status":"ONLINE","customStatus":null,"lastSeenAt":"2026-11-04T12:00:00.000Z","createdAt":"2026-11-01T10:00:00.000Z"}}
```

### POST `/api/auth/logout`

Bearer token required. Sets status offline. Client must discard its JWT;
this endpoint does not revoke the token server-side.

``` bash
curl -X POST "$BASE_URL/api/auth/logout" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"message":"Logged out on this client. Discard the bearer token."}
```

### POST `/api/auth/forgot-password`

Public. Requests password-reset OTP. Response intentionally does not
reveal whether the email exists.

Body: `email`.

``` bash
curl -X POST "$BASE_URL/api/auth/forgot-password" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com"}'
```

Example HTTP 202:

``` json
{"success":true,"stage":"PASSWORD_RESET_OTP","message":"If an active account exists for this email, a password reset code has been sent."}
```

### POST `/api/auth/resend-forgot-password-otp`

Public. Body: `email`.

``` bash
curl -X POST "$BASE_URL/api/auth/resend-forgot-password-otp" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com"}'
```

Example:

``` json
{"success":true,"message":"If an active account exists, a new password reset code has been sent."}
```

### POST `/api/auth/reset-password`

Public. Body: `email`, `otp`, `newPassword` (8--128 chars).

``` bash
curl -X POST "$BASE_URL/api/auth/reset-password" \
  -H "Content-Type: application/json" \
  -d '{"email":"alex@example.com","otp":"123456","newPassword":"NewStrongPass456"}'
```

Example:

``` json
{"success":true,"message":"Password reset successful. Log in again using your new password; a login code will be required."}
```

### POST `/api/auth/request-account-deletion`

Bearer token required. Sends account-deletion confirmation OTP.

``` bash
curl -X POST "$BASE_URL/api/auth/request-account-deletion" \
  -H "Authorization: Bearer $TOKEN"
```

Example HTTP 202:

``` json
{"success":true,"stage":"ACCOUNT_DELETE_OTP","message":"A confirmation code has been sent to your registered email."}
```

### POST `/api/auth/resend-account-deletion-otp`

Bearer token required. No body.

``` bash
curl -X POST "$BASE_URL/api/auth/resend-account-deletion-otp" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"message":"A new account deletion code has been sent."}
```

### POST `/api/auth/confirm-account-deletion`

Bearer token required. Body: `otp`.

``` bash
curl -X POST "$BASE_URL/api/auth/confirm-account-deletion" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"otp":"123456"}'
```

Example:

``` json
{"success":true,"message":"Your account has been deactivated and personal profile fields anonymized. You have been signed out."}
```

------------------------------------------------------------------------

## 4. Users --- `/api/users`

All user endpoints require Bearer token.

### GET `/api/users/search?q=...`

Query parameter: - `q` --- search username/display name; minimum 2
chars, maximum 40 chars; returns up to 20 users.

``` bash
curl -G "$BASE_URL/api/users/search" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "q=alex"
```

Example:

``` json
{"users":[{"id":"usr_456","username":"alexander","displayName":"Alex","avatarUrl":null}]}
```

### PATCH `/api/users/me`

Updates only supplied profile fields.

Body fields: `displayName`, `bio`, `avatarUrl`, `bannerUrl`,
`customStatus`, `status`. Status must be `ONLINE`, `IDLE`, `DND`, or
`INVISIBLE`. Limits: displayName 50, bio 500, customStatus 120 chars.

``` bash
curl -X PATCH "$BASE_URL/api/users/me" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"displayName":"Alex Dev","bio":"Building things","customStatus":"Coding","status":"ONLINE"}'
```

Example:

``` json
{"user":{"id":"usr_123","username":"alex_01","displayName":"Alex Dev","bio":"Building things","avatarUrl":null,"bannerUrl":null,"customStatus":"Coding","status":"ONLINE"}}
```

### GET `/api/users/:id/cosmetics`

Path parameter: `id` = target user ID.

``` bash
curl "$BASE_URL/api/users/usr_123/cosmetics" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"cosmetics":[{"id":"cos_1","name":"Blue Frame","type":"PROFILE_FRAME","isActive":true}]}
```

------------------------------------------------------------------------

## 5. Friends --- `/api/friends`

All endpoints require Bearer token.

### POST `/api/friends/request/:userId`

Path: `userId` = receiver user ID. No body.

``` bash
curl -X POST "$BASE_URL/api/friends/request/usr_456" \
  -H "Authorization: Bearer $TOKEN"
```

Example HTTP 201:

``` json
{"request":{"id":"fr_123","senderId":"usr_123","receiverId":"usr_456","status":"PENDING","createdAt":"2026-11-04T12:00:00.000Z"}}
```

### GET `/api/friends/requests`

Lists pending requests received by current user.

``` bash
curl "$BASE_URL/api/friends/requests" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"requests":[{"id":"fr_123","senderId":"usr_456","receiverId":"usr_123","status":"PENDING","sender":{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null}}]}
```

### POST `/api/friends/accept/:requestId`

Path: `requestId`. No body.

``` bash
curl -X POST "$BASE_URL/api/friends/accept/fr_123" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### POST `/api/friends/reject/:requestId`

Path: `requestId`. No body.

``` bash
curl -X POST "$BASE_URL/api/friends/reject/fr_123" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### GET `/api/friends`

Lists current user's friends.

``` bash
curl "$BASE_URL/api/friends" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"friends":[{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null}]}
```

### DELETE `/api/friends/:userId`

Path: `userId` = friend user ID.

``` bash
curl -X DELETE "$BASE_URL/api/friends/usr_456" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

------------------------------------------------------------------------

## 6. Direct messages --- `/api/dm`

All endpoints require Bearer token.

### GET `/api/dm`

Lists current user's DM and group-DM conversations.

``` bash
curl "$BASE_URL/api/dm" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"conversations":[{"id":"ch_dm1","name":"direct-message","type":"DM","participants":[{"id":"usr_123","username":"alex_01","displayName":"Alex","avatarUrl":null,"status":"ONLINE"},{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null,"status":"IDLE"}],"lastMessage":{"id":"msg_1","content":"Hey!","createdAt":"2026-11-04T12:00:00.000Z","authorId":"usr_456"}}]}
```

### POST `/api/dm`

Creates or reuses a DM. Body: - `userIds` --- array of 1--9 other user
IDs; do not include your own ID. Two participants total creates a DM;
more creates a group DM.

``` bash
curl -X POST "$BASE_URL/api/dm" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"userIds":["usr_456","usr_789"]}'
```

Example HTTP 201:

``` json
{"success":true,"conversation":{"id":"ch_group1","name":"group-message","type":"GROUP_DM","participants":[{"id":"usr_123","username":"alex_01","displayName":"Alex","avatarUrl":null},{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null},{"id":"usr_789","username":"lee","displayName":"Lee","avatarUrl":null}]}}
```

Existing one-to-one DM may return HTTP 200 with `"existing":true`.

### GET `/api/dm/:channelId`

Path: `channelId` = DM channel ID. Must be a participant.

``` bash
curl "$BASE_URL/api/dm/ch_dm1" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"conversation":{"id":"ch_dm1","name":"direct-message","type":"DM","participants":[{"id":"usr_123","username":"alex_01","displayName":"Alex","avatarUrl":null},{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null}]}}
```

------------------------------------------------------------------------

## 7. Servers and channels --- `/api/servers`

All endpoints require Bearer token. Server modification, channel
management and member mute operations are owner-only unless stated
otherwise.

### POST `/api/servers`

Creates a server and default `general` TEXT + `General Voice` VOICE
channels.

Body: `name` (2--80 chars), optional `description` (max 1000).

``` bash
curl -X POST "$BASE_URL/api/servers" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"My Community","description":"A place to chat"}'
```

Example HTTP 201:

``` json
{"server":{"id":"srv_123","name":"My Community","description":"A place to chat","ownerId":"usr_123","channels":[{"id":"ch_text","name":"general","type":"TEXT","position":0},{"id":"ch_voice","name":"General Voice","type":"VOICE","position":1}],"_count":{"members":1}}}
```

### GET `/api/servers`

Lists servers current user joined.

``` bash
curl "$BASE_URL/api/servers" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"servers":[{"id":"srv_123","name":"My Community","ownerId":"usr_123","channels":[],"_count":{"members":1},"myMembership":{"joinedAt":"2026-11-04T12:00:00.000Z","mutedAt":null,"mutedUntil":null}}]}
```

### GET `/api/servers/:id`

Path: `id` = server ID. Membership required.

``` bash
curl "$BASE_URL/api/servers/srv_123" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"server":{"id":"srv_123","name":"My Community","ownerId":"usr_123"},"channels":[{"id":"ch_text","name":"general","type":"TEXT"}],"memberCount":1,"myMembership":{"joinedAt":"2026-11-04T12:00:00.000Z","mutedAt":null,"mutedUntil":null}}
```

### PATCH `/api/servers/:id`

Owner only. Optional body fields: `name`, `description`, `iconUrl`,
`bannerUrl`. URLs must be HTTPS; name 2--80 chars; description max 1000.

``` bash
curl -X PATCH "$BASE_URL/api/servers/srv_123" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"AVENIX Community","description":"Official community","iconUrl":"https://cdn.example.com/icon.png"}'
```

Example:

``` json
{"server":{"id":"srv_123","name":"AVENIX Community","description":"Official community","iconUrl":"https://cdn.example.com/icon.png","ownerId":"usr_123"}}
```

### DELETE `/api/servers/:id`

Owner only. No body.

``` bash
curl -X DELETE "$BASE_URL/api/servers/srv_123" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### POST `/api/servers/:id/join`

Path: `id` = server ID. No body.

``` bash
curl -X POST "$BASE_URL/api/servers/srv_123/join" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"server":{"id":"srv_123","name":"My Community"}}
```

### POST `/api/servers/:id/leave`

Path: `id` = server ID. No body. Owner cannot leave their own server.

``` bash
curl -X POST "$BASE_URL/api/servers/srv_123/leave" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### GET `/api/servers/:id/members`

Path: `id` = server ID. Membership required. Returns at most 100
members.

``` bash
curl "$BASE_URL/api/servers/srv_123/members" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"members":[{"joinedAt":"2026-11-04T12:00:00.000Z","mutedAt":null,"mutedUntil":null,"user":{"id":"usr_123","username":"alex_01","displayName":"Alex","avatarUrl":null,"status":"ONLINE"}}]}
```

### POST `/api/servers/:id/channels`

Owner only. Body: - `name` required, 1--80 chars - `type` optional:
`TEXT` or `VOICE` (default `TEXT`) - `topic` optional, max 240 chars -
`slowmodeSeconds` optional integer 0--21600; must be 0 for VOICE -
`position` optional integer (default 0)

``` bash
curl -X POST "$BASE_URL/api/servers/srv_123/channels" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"announcements","type":"TEXT","topic":"Important updates","slowmodeSeconds":10,"position":2}'
```

Example HTTP 201:

``` json
{"channel":{"id":"ch_announcement","name":"announcements","type":"TEXT","topic":"Important updates","slowmodeSeconds":10,"position":2,"serverId":"srv_123"}}
```

### PATCH `/api/servers/:id/channels/:channelId`

Owner only. Path: `id` server ID, `channelId` channel ID. Optional body
fields: `name`, `topic`, `position`, `slowmodeSeconds`.

``` bash
curl -X PATCH "$BASE_URL/api/servers/srv_123/channels/ch_announcement" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"news","topic":"Latest news","position":3,"slowmodeSeconds":5}'
```

Example:

``` json
{"channel":{"id":"ch_announcement","name":"news","type":"TEXT","topic":"Latest news","position":3,"slowmodeSeconds":5,"serverId":"srv_123"}}
```

### DELETE `/api/servers/:id/channels/:channelId`

Owner only. Path: server ID and channel ID. No body.

``` bash
curl -X DELETE "$BASE_URL/api/servers/srv_123/channels/ch_announcement" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### POST `/api/servers/:id/members/:userId/mute`

Owner only. Path: server ID and target user ID. Body: -
`durationMinutes` optional integer 1--43200; omit or null for indefinite
mute - `reason` optional, truncated to 240 chars

``` bash
curl -X POST "$BASE_URL/api/servers/srv_123/members/usr_456/mute" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"durationMinutes":60,"reason":"Breaking server rules"}'
```

Example:

``` json
{"success":true,"mutedAt":"2026-11-04T12:00:00.000Z","mutedUntil":"2026-11-04T13:00:00.000Z"}
```

### DELETE `/api/servers/:id/members/:userId/mute`

Owner only. Path: server ID and target user ID. No body.

``` bash
curl -X DELETE "$BASE_URL/api/servers/srv_123/members/usr_456/mute" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### POST `/api/servers/:id/invites`

Any server member may create an invite. Body: - `maxUses` optional
integer 1--1000 - `expiresInHours` optional integer 1--720

``` bash
curl -X POST "$BASE_URL/api/servers/srv_123/invites" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"maxUses":10,"expiresInHours":24}'
```

Example HTTP 201:

``` json
{"invite":{"id":"inv_123","code":"AbCdEf123","serverId":"srv_123","creatorId":"usr_123","maxUses":10,"uses":0,"expiresAt":"2026-11-05T12:00:00.000Z","server":{"id":"srv_123","name":"My Community"}}}
```

### POST `/api/servers/invite/:code/join`

Path: `code` = invite code. No body.

``` bash
curl -X POST "$BASE_URL/api/servers/invite/AbCdEf123/join" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"server":{"id":"srv_123","name":"My Community"}}
```

------------------------------------------------------------------------

## 8. Messages --- `/api/channels`

All routes require Bearer token. `:channelId` is a channel ID;
`:messageId` is a message ID.

### GET `/api/channels/:channelId`

Fetches messages. Query: - `limit` optional, default 50, clamped
1--100 - `before` optional message ID cursor

``` bash
curl -G "$BASE_URL/api/channels/ch_text" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "limit=30" \
  --data-urlencode "before=msg_older"
```

Example:

``` json
{"messages":[{"id":"msg_1","content":"Hello!","channelId":"ch_text","authorId":"usr_456","createdAt":"2026-11-04T12:00:00.000Z","author":{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null},"replyTo":null,"reactions":[]}],"nextCursor":"msg_1"}
```

### POST `/api/channels/:channelId`

Sends a message. Body: - `content` text (up to 10000 chars) - `mediaUrl`
optional URL/string - `mediaType` optional media type -
`mediaDurationMs` optional number - `replyToId` optional message ID in
the same channel

At least `content` or `mediaUrl` is required. Voice channels do not
accept text messages.

``` bash
curl -X POST "$BASE_URL/api/channels/ch_text" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"content":"Hello everyone!","replyToId":"msg_parent"}'
```

Example HTTP 201:

``` json
{"message":{"id":"msg_new","content":"Hello everyone!","channelId":"ch_text","authorId":"usr_123","replyToId":"msg_parent","createdAt":"2026-11-04T12:00:00.000Z","author":{"id":"usr_123","username":"alex_01","displayName":"Alex","avatarUrl":null},"replyTo":{"id":"msg_parent","content":"Previous message","authorId":"usr_456","createdAt":"2026-11-04T11:59:00.000Z"},"reactions":[]}}
```

### PATCH `/api/channels/:messageId`

Edits your own message. Body: `content` required, 1--10000 chars after
trim. The route accepts media fields in code but only updates `content`.

``` bash
curl -X PATCH "$BASE_URL/api/channels/msg_new" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"content":"Edited message"}'
```

Example:

``` json
{"message":{"id":"msg_new","content":"Edited message","editedAt":"2026-11-04T12:05:00.000Z","channelId":"ch_text","authorId":"usr_123"}}
```

### DELETE `/api/channels/:messageId`

Unsend your own message; server owner can also unsend messages in their
server.

``` bash
curl -X DELETE "$BASE_URL/api/channels/msg_new" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### POST `/api/channels/:messageId/reactions`

Body: `emoji` (required, 1--32 chars). Toggles the current user's
reaction.

``` bash
curl -X POST "$BASE_URL/api/channels/msg_1/reactions" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"emoji":"🔥"}'
```

Example:

``` json
{"success":true,"added":true,"reactions":[{"emoji":"🔥","userId":"usr_123"}]}
```

### DELETE `/api/channels/:messageId/reactions/:emoji`

Path: `messageId`, URL-encoded `emoji`. Removes current user's reaction.

``` bash
curl -X DELETE "$BASE_URL/api/channels/msg_1/reactions/%F0%9F%94%A5" \
  -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"reactions":[]}
```

### POST `/api/channels/:messageId/bump`

Bumps a message in a server channel. No body. One bump per user/message.

``` bash
curl -X POST "$BASE_URL/api/channels/msg_1/bump" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"bumpCount":4,"bumpedAt":"2026-11-04T12:00:00.000Z"}
```

### GET `/api/channels/:channelId/bumped`

Lists up to 50 bumped messages in a channel.

``` bash
curl "$BASE_URL/api/channels/ch_text/bumped" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"messages":[{"id":"msg_1","content":"Important!","bumpedAt":"2026-11-04T12:00:00.000Z","bumpCount":4,"author":{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null}}]}
```

### GET `/api/channels/:channelId/pinned`

Lists up to 100 pinned messages.

``` bash
curl "$BASE_URL/api/channels/ch_text/pinned" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"messages":[{"id":"msg_1","content":"Read this","pinnedAt":"2026-11-04T12:00:00.000Z","pinnedById":"usr_123","author":{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null}}]}
```

### POST `/api/channels/:messageId/pin`

Server owner only. No body.

``` bash
curl -X POST "$BASE_URL/api/channels/msg_1/pin" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"message":{"id":"msg_1","pinnedAt":"2026-11-04T12:00:00.000Z","pinnedById":"usr_123"}}
```

### DELETE `/api/channels/:messageId/pin`

Server owner only. No body.

``` bash
curl -X DELETE "$BASE_URL/api/channels/msg_1/pin" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### POST `/api/channels/:channelId/read`

Marks channel as read. No body.

``` bash
curl -X POST "$BASE_URL/api/channels/ch_text/read" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"lastReadAt":"2026-11-04T12:00:00.000Z"}
```

### GET `/api/channels/:channelId/read-states`

Lists read states for channel members.

``` bash
curl "$BASE_URL/api/channels/ch_text/read-states" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"readStates":[{"user":{"id":"usr_123","username":"alex_01","displayName":"Alex","avatarUrl":null},"lastReadAt":"2026-11-04T12:00:00.000Z"}]}
```

------------------------------------------------------------------------

## 9. Notifications --- `/api/notifications`

All routes require Bearer token.

### GET `/api/notifications`

Query: `limit` optional (default 30, 1--100), `before` optional
notification ID cursor.

``` bash
curl -G "$BASE_URL/api/notifications" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "limit=20"
```

Example:

``` json
{"notifications":[{"id":"ntf_1","userId":"usr_123","type":"FRIEND_REQUEST","content":"@sam sent you a friend request.","isRead":false,"createdAt":"2026-11-04T12:00:00.000Z"}],"unreadCount":1,"nextCursor":"ntf_1"}
```

### PATCH `/api/notifications/:id/read`

Marks one notification as read. Path: `id`.

``` bash
curl -X PATCH "$BASE_URL/api/notifications/ntf_1/read" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

### POST `/api/notifications/read-all`

Marks all current user's notifications as read. No body.

``` bash
curl -X POST "$BASE_URL/api/notifications/read-all" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"updated":8}
```

### DELETE `/api/notifications/:id`

Deletes one of your notifications.

``` bash
curl -X DELETE "$BASE_URL/api/notifications/ntf_1" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true}
```

------------------------------------------------------------------------

## 10. Privacy --- `/api/privacy`

All routes require Bearer token.

### GET `/api/privacy/blocks`

Lists users blocked by current user.

``` bash
curl "$BASE_URL/api/privacy/blocks" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"blockedUsers":[{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null,"blockedAt":"2026-11-04T12:00:00.000Z"}]}
```

### POST `/api/privacy/blocks/:userId`

Blocks target user. Path: `userId`. No body. Also removes friendship and
pending friend requests between the two users.

``` bash
curl -X POST "$BASE_URL/api/privacy/blocks/usr_456" -H "Authorization: Bearer $TOKEN"
```

Example HTTP 201:

``` json
{"success":true,"message":"User blocked."}
```

### DELETE `/api/privacy/blocks/:userId`

Unblocks a user you blocked.

``` bash
curl -X DELETE "$BASE_URL/api/privacy/blocks/usr_456" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"message":"User unblocked."}
```

### GET `/api/privacy/bookmarks`

Query: `limit` optional (default 30, 1--100), `before` optional bookmark
ID cursor.

``` bash
curl -G "$BASE_URL/api/privacy/bookmarks" \
  -H "Authorization: Bearer $TOKEN" \
  --data-urlencode "limit=30"
```

Example:

``` json
{"bookmarks":[{"id":"bm_1","userId":"usr_123","messageId":"msg_1","createdAt":"2026-11-04T12:00:00.000Z","message":{"id":"msg_1","content":"Remember this","author":{"id":"usr_456","username":"sam","displayName":"Sam","avatarUrl":null},"channel":{"id":"ch_text","name":"general","type":"TEXT","serverId":"srv_123"}}}],"nextCursor":"bm_1"}
```

### POST `/api/privacy/bookmarks/:messageId`

Bookmarks an accessible, non-deleted message. Path: `messageId`.

``` bash
curl -X POST "$BASE_URL/api/privacy/bookmarks/msg_1" -H "Authorization: Bearer $TOKEN"
```

Example HTTP 201:

``` json
{"success":true,"bookmarked":true}
```

### DELETE `/api/privacy/bookmarks/:messageId`

Removes your bookmark.

``` bash
curl -X DELETE "$BASE_URL/api/privacy/bookmarks/msg_1" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"bookmarked":false}
```

------------------------------------------------------------------------

## 11. Reports --- `/api/reports`

All routes require Bearer token.

### POST `/api/reports`

Body: - `reason` required: `SPAM`, `HARASSMENT`, `IMPERSONATION`,
`INAPPROPRIATE_CONTENT`, `SCAM`, `OTHER` - `targetUserId` optional -
`messageId` optional - At least one of `targetUserId` or `messageId` is
required - `details` optional, max 2000 chars

``` bash
curl -X POST "$BASE_URL/api/reports" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"reason":"SPAM","targetUserId":"usr_456","details":"Repeated unsolicited spam."}'
```

Example HTTP 201:

``` json
{"success":true,"report":{"id":"rep_123","status":"PENDING","createdAt":"2026-11-04T12:00:00.000Z"}}
```

------------------------------------------------------------------------

## 12. Media --- `/api/media`

All routes require Bearer token. Upload uses `multipart/form-data`, not
JSON.

### POST `/api/media/upload`

Multipart fields: - `file` required --- file field; max size defaults to
25 MiB; one file - `purpose` optional --- `chat` (default), `voice`,
`avatar`, `profile-banner`, `server-icon`, `server-banner` - `channelId`
optional --- checked for `voice` purpose - `serverId` --- required for
`server-icon` / `server-banner`, and caller must own server -
`expiresInSeconds` optional --- clamped to 60--2,592,000 seconds

Accepted MIME types: audio/webm, audio/ogg, audio/mpeg, audio/mp4,
audio/aac, video/mp4, video/webm, image/jpeg, image/png, image/webp,
image/gif.

``` bash
curl -X POST "$BASE_URL/api/media/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@./photo.png" \
  -F "purpose=chat"
```

Example HTTP 201:

``` json
{"asset":{"id":"med_123","url":"http://localhost:3000/media/550e8400-e29b-41d4-a716-446655440000.png","mimeType":"image/png","mediaType":"IMAGE","sizeBytes":48231,"expiresAt":null},"loop":false}
```

Voice upload example:

``` bash
curl -X POST "$BASE_URL/api/media/upload" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@./voice.webm;type=audio/webm" \
  -F "purpose=voice" \
  -F "channelId=ch_text" \
  -F "expiresInSeconds=86400"
```

### DELETE `/api/media/:id`

Path: `id` = media asset ID owned by current user. Marks media for
cleanup; it is not an immediate physical deletion.

``` bash
curl -X DELETE "$BASE_URL/api/media/med_123" -H "Authorization: Bearer $TOKEN"
```

Example:

``` json
{"success":true,"message":"Media queued for cleanup"}
```

------------------------------------------------------------------------

## 13. Socket.IO real-time events

The backend also exposes Socket.IO on the same server. This is not a
REST/cURL API. Connect with a JWT:

``` javascript
import { io } from "socket.io-client";

const socket = io(BASE_URL, {
  auth: { token: YOUR_JWT }
});
```

Authentication failure emits a connection error such as
`Authentication required`, `Invalid token`, or `Account unavailable`.

Client → server events implemented:

  -----------------------------------------------------------------------
  Event                   Payload                 Purpose
  ----------------------- ----------------------- -----------------------
  `server:join`           `serverId` string       Join server room if
                                                  member

  `server:leave`          `serverId` string       Leave server room

  `channel:join`          `channelId` string      Join accessible
                                                  channel/DM room

  `channel:leave`         `channelId` string      Leave channel room

  `voice:join`            `channelId` string      Join a server VOICE
                                                  channel

  `voice:leave`           `channelId` string      Leave voice room

  `voice:offer`           `{channelId, signal}`   Relay WebRTC offer

  `voice:answer`          `{channelId, signal}`   Relay WebRTC answer

  `voice:ice-candidate`   `{channelId, signal}`   Relay ICE candidate
  -----------------------------------------------------------------------

Server → client events emitted by the source include:

-   `server:updated`, `server:deleted`, `server:member-joined`,
    `server:member-left`, `server:member-muted`, `server:member-unmuted`
-   `channel:created`, `channel:updated`, `channel:deleted`,
    `channel:read`
-   `message:new`, `message:updated`, `message:deleted`,
    `message:reactions`, `message:bumped`, `message:pinned`,
    `message:unpinned`
-   `voice:peers`, `voice:participant-joined`, `voice:participant-left`,
    `voice:state`, and relayed `voice:offer`, `voice:answer`,
    `voice:ice-candidate`

Voice implementation note: source code only handles WebRTC signaling.
The client must implement WebRTC peer connections and configure
STUN/TURN or an SFU for actual audio/video transport.

------------------------------------------------------------------------

## 14. Common errors and implementation notes

Typical error bodies include:

``` json
{"error":"Authentication required"}
{"error":"Invalid or expired token"}
{"error":"Channel access denied"}
{"error":"Requested record not found"}
{"error":"A record with this value already exists"}
{"error":"Invalid request data","details":[]}
{"error":"Internal server error"}
```

Common HTTP statuses: - `400` invalid body, invalid OTP, invalid
parameter or invalid operation - `401` missing/invalid/expired JWT or
invalid login - `403` authenticated but not allowed (permissions,
membership, owner-only action) - `404` resource not found or
inaccessible - `409` duplicate username/email, duplicate friend request,
or duplicate bump - `429` rate limit / OTP resend cooldown / channel
slowmode - `503` database health failure or email delivery failure -
`500` unexpected server error

Implementation details observed in source: 1. Global rate limit defaults
to 240 requests per 15 minutes; auth routes have a separate 25 requests
per 15 minutes limit. 2. JSON body size defaults to 128kb. 3. JWT
lifetime defaults to 7 days. 4. OTP defaults: 10-minute expiry, 5
attempts, 60-second resend wait. 5. Media max size defaults to 25 MiB.
6. `/api/auth/logout` only updates presence; it does not invalidate an
already-issued JWT. 7. `/api/channels/:messageId` PATCH accepts
media-related properties in the handler but only persists `content`. 8.
`/api/channels/:messageId/pin` and channel management are
server-owner-only in this implementation. 9.
`GET /api/channels/:channelId` cursor response uses the first item in
the reversed response array as `nextCursor`; test pagination carefully
in your client. 10. The media endpoint returns a public URL. For
production, the README recommends private object storage and signed
URLs. 11. The README explicitly says production voice/video requires a
WebRTC client plus TURN/SFU infrastructure.

## 15. Source files reviewed

-   `src/app.js` --- mounts route groups, global middleware, `/api`,
    `/api/docs`, `/api/health`
-   `src/server.js` --- Socket.IO authentication, rooms, voice signaling
-   `src/middleware/auth.js` --- JWT authentication and admin helper
-   `src/routes/auth.js`
-   `src/routes/users.js`
-   `src/routes/friends.js`
-   `src/routes/dm.js`
-   `src/routes/servers.js`
-   `src/routes/messages.js`
-   `src/routes/notifications.js`
-   `src/routes/privacy.js`
-   `src/routes/reports.js`
-   `src/routes/media.js`

This reference documents routes found in the current `main` branch
source reviewed for this analysis. It does not claim that example IDs or
response values are live production data.
