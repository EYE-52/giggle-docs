# Giggle MVP Backend API Spec

This document defines the first backend APIs for Giggle MVP, including:
- endpoint purpose
- request/response contracts
- intended backend function/handler name

Scope matches the MVP docs: lobby-first, then matchmaking, then encounter bootstrap.
All product APIs are secured with Google OAuth identity from your existing Auth.js login.

---

## 0) Conventions

### Base URL
- Local: `http://localhost:<server-port>/api`

### Content Type
- Request/response JSON unless otherwise stated.

### Authentication & Identity
- **Auth source:** Existing Google OAuth session (Auth.js / NextAuth).
- **Security model:** All product APIs require authentication.
- **Public exceptions:** `GET /health` and `GET /ready` can stay unauthenticated.
- **Identity key used across APIs:** `providerAccountId` from MongoDB `accounts` collection (`provider: "google"`).
- **Identity injection:** `providerAccountId`, internal `userId`, and display profile are resolved in auth middleware and attached to request context.

### Auth Header
- `Authorization: Bearer <session-token-or-jwt>`

### Auth Middleware (intended)
- `requireApiAuth()` -> validates session/token.
- `attachGiggleIdentity()` -> resolves `{ userId, providerAccountId, name, email, image }`.
- `requireSquadMember()` / `requireSquadLeader()` -> authorization checks by squad role.

### Identity Mapping
```ts
type GiggleIdentity = {
  userId: string; // users._id from MongoDB adapter
  provider: "google";
  providerAccountId: string; // accounts.providerAccountId (primary external user key)
  name?: string;
  email?: string;
  image?: string;
};
```

### Standard Error Response
```json
{
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "Human-readable error",
    "details": {}
  }
}
```

### Standard Success Envelope
```json
{
  "ok": true,
  "data": {}
}
```

### Suggested Core Types
```ts
type SquadRole = "leader" | "member";
type LobbyStatus = "idle" | "searching" | "matched" | "in_encounter";

type SquadMember = {
  memberId: string;
  userId: string;
  providerAccountId: string;
  displayName: string;
  role: SquadRole;
  ready: boolean;
  joinedAt: string; // ISO
};

type Squad = {
  squadId: string;
  squadCode: string; // ex: GIG-882
  status: LobbyStatus;
  members: SquadMember[];
  createdAt: string; // ISO
};
```

---

## 1) Lobby APIs (Phase 1)

## 1.1 Create Squad

- **Endpoint:** `POST /squads/create`
- **Purpose:** Create new squad and assign requester as leader.
- **Intended Backend Function:** `createSquadHandler` -> `createSquad()`
- **Auth:** required (`requireApiAuth`)

### Request Body
```json
{
  "displayName": "Himanshu"
}
```

`displayName` is optional override; if omitted, use authenticated profile name.

### Success Response (201)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "squadCode": "GIG-882",
    "member": {
      "memberId": "mem_01JY...",
      "userId": "usr_01JY...",
      "providerAccountId": "108923456789012345678",
      "displayName": "Himanshu",
      "role": "leader",
      "ready": false,
      "joinedAt": "2026-03-25T10:00:00.000Z"
    },
    "status": "idle"
  }
}
```

### Errors
- `400 INVALID_REQUEST` (missing/invalid `displayName`)
- `401 UNAUTHORIZED`
- `500 INTERNAL_ERROR`

---

## 1.2 Join Squad by Code

- **Endpoint:** `POST /squads/join`
- **Purpose:** Join existing squad via 6-digit code.
- **Intended Backend Function:** `joinSquadHandler` -> `joinSquadByCode()`
- **Auth:** required (`requireApiAuth`)

### Request Body
```json
{
  "squadCode": "GIG-882",
  "displayName": "Aman"
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "squadCode": "GIG-882",
    "member": {
      "memberId": "mem_01JY...",
      "userId": "usr_01JY...",
      "providerAccountId": "109999999999999999999",
      "displayName": "Aman",
      "role": "member",
      "ready": false,
      "joinedAt": "2026-03-25T10:01:00.000Z"
    },
    "members": [
      {
        "memberId": "mem_01JY_leader",
        "userId": "usr_01JY_leader",
        "providerAccountId": "108923456789012345678",
        "displayName": "Himanshu",
        "role": "leader",
        "ready": true,
        "joinedAt": "2026-03-25T10:00:00.000Z"
      },
      {
        "memberId": "mem_01JY_member",
        "userId": "usr_01JY_member",
        "providerAccountId": "109999999999999999999",
        "displayName": "Aman",
        "role": "member",
        "ready": false,
        "joinedAt": "2026-03-25T10:01:00.000Z"
      }
    ],
    "status": "idle"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `404 SQUAD_NOT_FOUND`
- `409 SQUAD_FULL` (if limit reached)
- `409 SQUAD_CLOSED` (already matched/in encounter and joins are blocked)

---

## 1.3 Get Squad Lobby State

- **Endpoint:** `GET /squads/:squadId`
- **Purpose:** Fetch current lobby state (poll fallback for realtime updates).
- **Intended Backend Function:** `getSquadHandler` -> `getSquadState()`
- **Auth:** required (`requireApiAuth`, `requireSquadMember`)

### Request Params
- Path: `squadId`

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "squadCode": "GIG-882",
    "status": "searching",
    "members": [
      {
        "memberId": "mem_01JY_leader",
        "userId": "usr_01JY_leader",
        "providerAccountId": "108923456789012345678",
        "displayName": "Himanshu",
        "role": "leader",
        "ready": true,
        "joinedAt": "2026-03-25T10:00:00.000Z"
      }
    ],
    "leaderMemberId": "mem_01JY_leader"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 SQUAD_NOT_FOUND`

---

## 1.4 Update Ready State

- **Endpoint:** `POST /squads/:squadId/ready`
- **Purpose:** Mark current member ready/unready in lobby.
- **Intended Backend Function:** `setReadyHandler` -> `setMemberReadyState()`
- **Auth:** required (`requireApiAuth`, `requireSquadMember`)

### Request Body
```json
{
  "ready": true
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "memberId": "mem_01JY_member",
    "providerAccountId": "109999999999999999999",
    "ready": true,
    "readyCount": 2,
    "totalMembers": 3
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 SQUAD_NOT_FOUND`
- `404 MEMBER_NOT_FOUND`
- `409 INVALID_SQUAD_STATE` (if not allowed during encounter)

---

## 1.5 Leave Squad

- **Endpoint:** `POST /squads/:squadId/leave`
- **Purpose:** Member leaves squad; if leader leaves, promote next member.
- **Intended Backend Function:** `leaveSquadHandler` -> `removeMemberAndReassignLeader()`
- **Auth:** required (`requireApiAuth`, `requireSquadMember`)

### Request Body
```json
{}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "leftMemberId": "mem_01JY_member",
    "newLeaderMemberId": "mem_01JY_leader",
    "remainingCount": 2,
    "squadDeleted": false
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 SQUAD_NOT_FOUND`
- `404 MEMBER_NOT_FOUND`

---

## 2) Matchmaking APIs (Phase 2)

## 2.1 Enqueue Squad

- **Endpoint:** `POST /matchmaking/enqueue`
- **Purpose:** Leader places squad in matchmaking queue.
- **Intended Backend Function:** `enqueueSquadHandler` -> `enqueueSquadForMatch()`
- **Auth:** required (`requireApiAuth`, `requireSquadLeader`)

### Request Body
```json
{
  "squadId": "sq_01JY...",
  "region": "ap-south-1"
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "queueStatus": "searching",
    "queuedAt": "2026-03-25T10:05:00.000Z",
    "estimatedWaitSec": 20
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 LEADER_ONLY`
- `409 ALREADY_IN_QUEUE`
- `409 NOT_READY_TO_SEARCH`

---

## 2.2 Dequeue Squad

- **Endpoint:** `POST /matchmaking/dequeue`
- **Purpose:** Leader cancels matchmaking search.
- **Intended Backend Function:** `dequeueSquadHandler` -> `dequeueSquad()`
- **Auth:** required (`requireApiAuth`, `requireSquadLeader`)

### Request Body
```json
{
  "squadId": "sq_01JY..."
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "queueStatus": "idle",
    "removed": true
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 LEADER_ONLY`
- `404 NOT_IN_QUEUE`

---

## 2.3 Get Matchmaking Status

- **Endpoint:** `GET /matchmaking/status/:squadId`
- **Purpose:** Fetch current matchmaking state.
- **Intended Backend Function:** `getMatchmakingStatusHandler` -> `getSquadMatchStatus()`
- **Auth:** required (`requireApiAuth`, `requireSquadMember`)

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "state": "matched",
    "meetingId": "meet_01JY...",
    "opponentSquadId": "sq_01JY_op",
    "matchedAt": "2026-03-25T10:06:10.000Z"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 SQUAD_NOT_FOUND`

---

## 2.4 Skip Encounter

- **Endpoint:** `POST /matchmaking/skip`
- **Purpose:** Leader requests disconnect and rematch.
- **Intended Backend Function:** `skipEncounterHandler` -> `skipAndRequeueSquad()`
- **Auth:** required (`requireApiAuth`, `requireSquadLeader`)

### Request Body
```json
{
  "squadId": "sq_01JY...",
  "meetingId": "meet_01JY..."
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "previousMeetingId": "meet_01JY...",
    "queueStatus": "searching"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 LEADER_ONLY`
- `409 NOT_IN_ENCOUNTER`

---

## 3) Encounter / Video Bootstrap APIs (Phase 3)

## 3.1 Create Agora Token

- **Endpoint:** `POST /encounters/token`
- **Purpose:** Issue short-lived Agora token for a meeting/channel.
- **Intended Backend Function:** `issueAgoraTokenHandler` -> `createRtcToken()`
- **Auth:** required (`requireApiAuth`, `requireSquadMember`)

### Request Body
```json
{
  "meetingId": "meet_01JY...",
  "squadId": "sq_01JY...",
  "role": "publisher"
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "meetingId": "meet_01JY...",
    "agoraChannel": "giggle_meet_01JY...",
    "agoraUid": "109999999999999999999",
    "rtcToken": "<token>",
    "expiresAt": "2026-03-25T11:06:10.000Z"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 MEMBER_NOT_IN_MEETING`
- `404 MEETING_NOT_FOUND`

---

## 3.2 Disconnect from Encounter

- **Endpoint:** `POST /encounters/disconnect`
- **Purpose:** Leave active encounter and return member/squad to lobby flow.
- **Intended Backend Function:** `disconnectEncounterHandler` -> `disconnectMemberFromMeeting()`
- **Auth:** required (`requireApiAuth`, `requireSquadMember`)

### Request Body
```json
{
  "meetingId": "meet_01JY...",
  "squadId": "sq_01JY..."
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "meetingId": "meet_01JY...",
    "memberId": "mem_01JY_member",
    "disconnected": true,
    "squadStatus": "idle"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 MEETING_NOT_FOUND`
- `404 MEMBER_NOT_FOUND`

---

## 3.3 Get Encounter Metadata

- **Endpoint:** `GET /encounters/:meetingId`
- **Purpose:** Fetch metadata required by clients for rendering context.
- **Intended Backend Function:** `getEncounterHandler` -> `getMeetingDetails()`
- **Auth:** required (`requireApiAuth`, `requireSquadMember`)

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "meetingId": "meet_01JY...",
    "leftSquadId": "sq_01JY_A",
    "rightSquadId": "sq_01JY_B",
    "createdAt": "2026-03-25T10:06:10.000Z",
    "status": "active"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 MEETING_NOT_FOUND`

---

## 4) Platform/Ops APIs (Phase 4)

## 4.1 Health Check

- **Endpoint:** `GET /health`
- **Purpose:** Process-level liveness probe.
- **Intended Backend Function:** `healthHandler`

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "status": "up",
    "service": "giggle-server",
    "time": "2026-03-25T10:00:00.000Z"
  }
}
```

---

## 4.2 Readiness Check

- **Endpoint:** `GET /ready`
- **Purpose:** Readiness probe (Redis, signaling, config).
- **Intended Backend Function:** `readyHandler` -> `checkDependenciesReady()`

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "status": "ready",
    "checks": {
      "redis": "ok",
      "agoraConfig": "ok"
    }
  }
}
```

### Error Response (503)
```json
{
  "ok": false,
  "error": {
    "code": "NOT_READY",
    "message": "One or more dependencies are unavailable",
    "details": {
      "redis": "down"
    }
  }
}
```

---

## 4.3 Event Ingestion (Optional but recommended)

- **Endpoint:** `POST /events`
- **Purpose:** Track product events for MVP debugging/analytics.
- **Intended Backend Function:** `ingestEventHandler` -> `recordProductEvent()`
- **Auth:** required (`requireApiAuth`)

### Request Body
```json
{
  "eventName": "squad_enqueued",
  "squadId": "sq_01JY...",
  "metadata": {
    "region": "ap-south-1"
  },
  "occurredAt": "2026-03-25T10:05:00.000Z"
}
```

### Success Response (202)
```json
{
  "ok": true,
  "data": {
    "accepted": true
  }
}
```

---

## 5) Build Order (Implementation Sequence)

1. `POST /squads/create`
2. `POST /squads/join`
3. `GET /squads/:squadId`
4. `POST /squads/:squadId/ready`
5. `POST /squads/:squadId/leave`
6. `POST /matchmaking/enqueue`
7. `POST /matchmaking/dequeue`
8. `GET /matchmaking/status/:squadId`
9. `POST /encounters/token`
10. `POST /encounters/disconnect`
11. `GET /health` + `GET /ready`
12. `POST /events` (optional)

This sequence gives a runnable MVP path: lobby -> queue -> match handoff -> video bootstrap.
