# Giggle Phase 1: Agora Design Overview

This document explains how Agora should be used in Giggle Phase 1 to deliver a working squad lobby demo with live audio/video and a clean path toward matchmaking.

---

## 1. What Agora Is

Agora is a real-time communications platform. For Giggle, the important part is that Agora handles the hard media layer:

- camera capture
- microphone capture
- publishing local audio/video
- subscribing to remote audio/video
- low-latency delivery over Agora's real-time network
- device/network adaptation during calls

For Giggle, Agora is not your product logic. It is the transport and media engine underneath your product.

Your app still owns:

- auth
- squad creation
- squad membership
- ready state
- leader permissions
- matchmaking state
- invite/join experience
- channel assignment rules

Short version:

- Giggle backend decides who should talk to whom.
- Agora makes the talking actually work.

---

## 2. Why Agora Fits Giggle

Giggle is not a static meeting product. It needs fast, disposable, dynamic rooms with good enough quality and low friction.

Agora helps because it gives you:

- fast browser-based group video/audio
- good performance under unstable consumer networks
- a clean channel model that maps well to squads and encounters
- token-based auth so random users cannot join arbitrary channels
- React/Web SDK support already compatible with your frontend stack

This is especially useful for Giggle because your app has two different live communication modes:

1. private squad lobby
2. encounter room after matchmaking

Agora channels map naturally to both.

---

## 3. What Agora Products To Use

### Required for Phase 1

#### A. Agora Video Calling / RTC SDK

Use this for actual audio/video.

In your repo, this is already aligned with:

- `agora-rtc-sdk-ng`
- `agora-rtc-react`

Use cases in Giggle:

- leader and invited member can see/hear each other in the squad lobby
- later, multiple users from two squads can join a shared encounter room

#### B. Agora Token Authentication

Use a backend-generated RTC token for every join.

Why:

- prevents clients from joining arbitrary channels using only app ID
- lets your backend control who can join which squad/meeting channel
- keeps Giggle's own auth model in control

Your backend should issue a token only after:

- JWT auth succeeds
- user is confirmed squad member
- requested channel matches allowed squad/meeting state

### Optional Later, Not Needed for Phase 1

#### C. Agora Signaling

Agora Signaling can handle presence, metadata, lightweight events, and channel attributes.

But for your current MVP, you do not need it yet.

Why not yet:

- you already have a Node backend for squad state
- your current lobby/member/ready/search flows are easier to keep on your own backend
- introducing Signaling now adds another state layer before the media path is proven

Use your own backend first for:

- squad membership
- ready state
- leader actions
- matchmaking transitions

Consider Agora Signaling later for:

- presence sync
- channel metadata
- richer realtime lobby events

#### D. Cloud Recording

Not needed for Phase 1.

#### E. Moderation / Extensions / Effects

Not needed for Phase 1.

Add later when the core loop works.

---

## 4. Recommended Phase 1 Scope

Your target Phase 1 flow is:

1. user logs in and creates a squad
2. user shares invite
3. invited user logs in
4. invited user joins the squad
5. both users can see/hear each other in the squad lobby
6. leader can hit "start matchmaking" after both are ready

This means Phase 1 is mainly a **private squad video lobby**, not full encounter switching.

That is the right scope.

Do not overbuild Phase 1.

You only need to prove:

- auth works
- squad membership works
- RTC join works
- multi-user lobby video works
- leader-only state changes work

---

## 5. Core Architecture

### 5.1 Responsibility Split

#### Giggle Web

- handles UI
- gets session/backend JWT
- calls your squad APIs
- gets Agora token from backend
- joins Agora channel
- renders local and remote video

#### Giggle Server

- validates Giggle JWT
- validates squad membership
- decides allowed Agora channel name
- generates Agora RTC token
- returns token + channel info
- controls ready/search state

#### Agora

- transports live audio/video
- emits join/leave/media events
- handles network adaptation

---

## 6. Channel Strategy

Use deterministic channel naming.

### Phase 1 Lobby Channel

For the squad lobby, use one RTC channel per squad.

Suggested naming:

- `lobby_<squadId>`

Examples:

- `lobby_sq_1712230000_abc123`

Why this works:

- all current squad members join the same private lobby channel
- easy to validate on backend
- easy to leave later when moving to encounter room

### Phase 2 Encounter Channel

Later, when matchmaking exists, generate a new meeting channel:

- `meet_<meetingId>`

Then both squads leave their lobby channel and join the encounter channel.

---

## 7. Phase 1 End-to-End Flow With Agora

### Step 1. User logs in

- Google login via Auth.js
- frontend exchanges session user for backend JWT
- backend JWT is used for protected Giggle APIs

### Step 2. Leader creates squad

- `POST /api/squads/create`
- backend creates squad and leader member record
- frontend gets `squadId` and `squadCode`

### Step 3. Leader opens lobby

Frontend should then request Agora join credentials for the lobby channel.

Suggested API:

- `POST /api/agora/lobby-token`

Request:

```json
{
  "squadId": "sq_..."
}
```

Response:

```json
{
  "ok": true,
  "data": {
    "appId": "AGORA_APP_ID",
    "channelName": "lobby_sq_...",
    "rtcToken": "...",
    "uid": 123456
  }
}
```

Backend checks:

- authenticated user exists
- authenticated user is in the squad
- requested squad is valid

Frontend then:

- creates Agora client
- requests camera/mic access
- joins `lobby_<squadId>`
- publishes local tracks

### Step 4. Invitee joins squad

- invitee logs in
- invitee enters invite code or lands on invite/join screen
- `POST /api/squads/join`
- backend adds them to squad

After success, frontend also calls:

- `POST /api/agora/lobby-token`

Invitee joins the same channel:

- `lobby_<squadId>`

Now both users can see and hear each other.

### Step 5. Ready state

Each user toggles ready:

- `POST /api/squads/:squadId/ready`

This state stays in Giggle backend, not Agora.

Agora is only media.

### Step 6. Leader starts matchmaking

- `POST /api/squads/:squadId/search`

For Phase 1 demo, this can simply move squad to `searching` state.

You do not need to implement actual encounter-room switching yet if the goal is only to prove the lobby experience.

---

## 8. What You Need From Agora For Phase 1

### Frontend SDK Features

You will use:

- create RTC client
- create microphone audio track
- create camera video track
- join channel with token
- publish local tracks
- subscribe to remote users
- render remote and local video tracks
- leave channel cleanly on navigation/disconnect

### Backend Token Features

You will use:

- App ID
- App Certificate
- RTC token generation
- channel-scoped token issuance
- user-specific UID binding
- token expiry handling

---

## 9. What Giggle Should Not Use Agora For Yet

Do not move these concerns into Agora in Phase 1:

- squad membership source of truth
- leader permissions
- invite authorization
- matchmaking state
- ready state
- app-level identity

Keep these in Giggle backend.

Reason:

- easier to debug
- fewer moving parts
- cleaner business rules
- better long-term control

Agora should remain the media plane, not your business logic plane.

---

## 10. Recommended Backend Additions For Agora Phase 1

Add a small Agora module in server:

- `src/config/agoraConfig.js`
- `src/services/agoraTokenService.js`
- `src/controllers/agoraController.js`
- `src/routes/agoraRoutes.js`

### Suggested env vars

```bash
AGORA_APP_ID=...
AGORA_APP_CERTIFICATE=...
AGORA_TOKEN_EXPIRY_SECONDS=3600
```

### Suggested endpoint

#### `POST /api/agora/lobby-token`

Purpose:

- issue RTC token for squad lobby channel

Checks:

- JWT valid
- requester is squad member
- squad exists
- requested token is only for `lobby_<squadId>`

Response:

- `appId`
- `channelName`
- `rtcToken`
- `uid`
- `expiresIn`

---

## 11. Recommended Frontend Modules For Agora Phase 1

Suggested frontend structure:

- `src/lib/agora/client.ts`
- `src/lib/agora/tracks.ts`
- `src/lib/agora/joinLobbyChannel.ts`
- `src/components/lobby/LobbyVideoGrid.tsx`
- `src/components/lobby/LocalTile.tsx`
- `src/components/lobby/RemoteTile.tsx`

Responsibilities:

- one place to create Agora client
- one place to create/publish tracks
- one place to join/leave channel
- one place to render participant tiles

Keep UI thin. The real value in Phase 1 is reliable join/publish/subscribe behavior.

---

## 12. Security Model

Use Agora token auth, not app ID alone.

Important rule:

- client never decides its own valid channel without backend approval

Correct flow:

1. client authenticates with Giggle backend
2. backend checks squad membership
3. backend generates token for exact allowed channel
4. client joins with returned token

This matters because Giggle is not a generic meeting app. Access is conditional on squad state.

---

## 13. Token Expiry

For Phase 1, keep it simple:

- issue tokens valid for 1 hour
- if token expires during long session, request a fresh token from backend and renew

You do not need advanced token rotation before the core demo works.

---

## 14. Risks and Practical Notes

### Camera/Mic Permissions

This will be one of the biggest real issues for your demo.

Add a small preflight UI:

- camera granted / blocked
- mic granted / blocked
- selected devices

### Browser Limits

Target desktop Chrome first.

That is already consistent with your MVP rules.

### Identity Mapping

Agora UID should be deterministic per join session, but the true source of access remains Giggle `userId`.

Do not use Agora UID as business identity.

### Debugging

Log:

- squadId
- channelName
- userId
- issued Agora UID
- token issue time

This will save time when remote video does not appear.

---

## 15. Recommended Build Order

### First

Implement private squad lobby video only:

1. create squad
2. join squad
3. issue Agora lobby token
4. join lobby channel
5. publish local tracks
6. subscribe to remote tracks

### Second

Add ready flow and leader-only search state.

### Third

Add matchmaking and switch from:

- `lobby_<squadId>`

to:

- `meet_<meetingId>`

---

## 16. Recommendation Summary

For Giggle Phase 1, use:

- Agora RTC / Video SDK: yes
- Agora token authentication: yes
- Agora Signaling: no, not yet
- Agora recording/moderation/effects: no, not yet

Best Phase 1 architecture:

- Giggle backend owns auth, squad state, ready state, permissions, matchmaking state
- Agora owns live media transport
- one private Agora lobby channel per squad
- leader-only search action stays in Giggle backend

This is the shortest path to a working demo that actually proves the product.