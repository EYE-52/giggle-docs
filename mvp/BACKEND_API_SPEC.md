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
- **Auth source:** Backend JWT issued by `POST /auth/exchange` after Google login via Auth.js / NextAuth.
- **Security model:** All product APIs require bearer JWT auth.
- **Public exceptions:** `GET /health` and `GET /ready` can stay unauthenticated.
- **Identity key used across APIs:** `userId` (primary) with `providerAccountId` retained for backward compatibility.
- **Identity injection:** middleware verifies JWT and attaches decoded identity to request context.

### Auth Header
- `Authorization: Bearer <session-token-or-jwt>`

### Auth Middleware (implemented)
- `requireApiAuth()` -> validates bearer token and required identity claims.
- `requireSquadMemberAccess()` -> loads squad context and ensures requester belongs to the squad.
- `requireSquadLeaderAccess()` -> loads squad context and ensures requester is the squad leader.

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
- **Auto-delete behavior:** If the leaving member is the last member, squad is deleted automatically.
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

## 1.6 Get My Squad Context

- **Endpoint:** `GET /squads/me`
- **Purpose:** Return current authenticated user's squad state (or no active squad).
- **Auth:** required (`requireApiAuth`)

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "inSquad": true,
    "squadId": "sq_01JY...",
    "squadCode": "GIG-882",
    "status": "idle",
    "member": {
      "memberId": "mem_01JY...",
      "userId": "usr_01JY...",
      "providerAccountId": "108923456789012345678",
      "displayName": "Himanshu",
      "role": "leader",
      "ready": true,
      "joinedAt": "2026-03-25T10:00:00.000Z"
    },
    "leaderMemberId": "mem_01JY...",
    "members": []
  }
}
```

When user has no squad:
```json
{
  "ok": true,
  "data": {
    "inSquad": false
  }
}
```

### Errors
- `401 UNAUTHORIZED`

---

## 1.7 Start Squad Search (Leader Only)

- **Endpoint:** `POST /squads/:squadId/search`
- **Purpose:** Move squad from `idle` to `searching`.
- **Auth:** required (`requireApiAuth` + leader-only access)

### Preconditions
- Requester is a squad member and leader.
- Squad status is `idle`.
- Squad has at least `MIN_MEMBERS_TO_SEARCH` members (env-configurable).
- All members are `ready: true`.

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 LEADER_ONLY`
- `404 SQUAD_NOT_FOUND`
- `409 INVALID_SQUAD_STATE`
- `409 NOT_ENOUGH_MEMBERS`
- `409 NOT_READY_TO_SEARCH`

---

## 1.8 Cancel Squad Search (Leader Only)

- **Endpoint:** `POST /squads/:squadId/search/cancel`
- **Purpose:** Move squad from `searching` to `idle`.
- **Auth:** required (`requireApiAuth` + leader-only access)

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 LEADER_ONLY`
- `404 SQUAD_NOT_FOUND`
- `409 NOT_IN_SEARCH`

---

## 1.9 Kick Member (Leader Only)

- **Endpoint:** `POST /squads/:squadId/members/:memberId/kick`
- **Purpose:** Remove target member from squad.
- **Auth:** required (`requireApiAuth` + leader-only access)

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 LEADER_ONLY`
- `404 SQUAD_NOT_FOUND`
- `404 MEMBER_NOT_FOUND`
- `409 LEADER_CANNOT_BE_KICKED`

---

## 1.10 Promote Member to Leader (Leader Only)

- **Endpoint:** `POST /squads/:squadId/members/:memberId/promote`
- **Purpose:** Transfer leadership to another member.
- **Auth:** required (`requireApiAuth` + leader-only access)

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `403 LEADER_ONLY`
- `404 SQUAD_NOT_FOUND`
- `404 MEMBER_NOT_FOUND`
- `409 ALREADY_LEADER`

---

## 1.11 Get Agora Lobby Token

- **Endpoint:** `POST /agora/lobby-token/:squadId`
- **Purpose:** Issue Agora RTC token for squad lobby channel.
- **Auth:** required (`requireApiAuth`, `requireSquadMemberAccess`)

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "memberId": "mem_01JY...",
    "appId": "your-agora-app-id",
    "channelName": "lobby_sq_01JY...",
    "rtcToken": "007eJx...",
    "uid": 123456789,
    "expiresIn": 3600,
    "expiresAt": "2026-04-03T14:00:00.000Z"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 SQUAD_NOT_FOUND`
- `500 AGORA_NOT_CONFIGURED`
- `500 INTERNAL_ERROR`

### Required Environment Variables
- `AGORA_APP_ID`
- `AGORA_APP_CERTIFICATE`
- `AGORA_TOKEN_EXPIRY_SECONDS` (optional, default 3600)

---

## 2) Matchmaking APIs (Phase 2)

Phase 2 uses the squad-state APIs in section 1 for queue entry/exit:
- `POST /squads/:squadId/search` (start search)
- `POST /squads/:squadId/search/cancel` (cancel search)

This section defines the dedicated matchmaking runtime APIs and matching rules.

### 2.0 Runtime Rules (Design Contract)

- Queue strategy: bucket by `region` and `size`.
- Window A (0-8s): same region + same size.
- Window B (8-16s): same region + size difference <= 1.
- Window C (16s+): region fallback + size difference <= 1.
- Anti-repeat cooldown: avoid same squad pair for 10-15 minutes where possible.
- Starvation prevention: older queued squads get priority bias.

---

## 2.1 Get Matchmaking Status

- **Endpoint:** `GET /matchmaking/status/:squadId`
- **Purpose:** Fetch current search or match state for a squad.
- **Intended Backend Function:** `getMatchmakingStatusHandler` -> `getSquadMatchStatus()`
- **Auth:** required (`requireApiAuth`, `requireSquadMemberAccess`)

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "state": "searching",
    "queue": {
      "region": "ap-south-1",
      "size": 2,
      "queuedAt": "2026-03-25T10:05:00.000Z",
      "waitSeconds": 7
    },
    "match": null
  }
}
```

Matched example:
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "state": "matched",
    "queue": null,
    "match": {
      "encounterId": "enc_01JY...",
      "opponentSquadId": "sq_01JY_op",
      "matchedAt": "2026-03-25T10:06:10.000Z"
    }
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 SQUAD_NOT_FOUND`

---

## 2.2 Encounter Handoff Status

- **Endpoint:** `GET /matchmaking/encounters/:encounterId`
- **Purpose:** Return encounter handoff state while clients transition from lobby to encounter room.
- **Intended Backend Function:** `getEncounterHandoffHandler` -> `getEncounterHandoffStatus()`
- **Auth:** required (`requireApiAuth`, `requireSquadMemberAccess`)

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "encounterId": "enc_01JY...",
    "status": "awaiting_ack",
    "squadAId": "sq_01JY_A",
    "squadBId": "sq_01JY_B",
    "ack": {
      "sq_01JY_A": true,
      "sq_01JY_B": false
    },
    "expiresAt": "2026-03-25T10:06:50.000Z"
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 ENCOUNTER_NOT_FOUND`

---

## 2.3 Acknowledge Encounter Join

- **Endpoint:** `POST /matchmaking/encounters/:encounterId/ack`
- **Purpose:** Mark current squad as ready in encounter handoff flow; once both squads ack, state moves to `in_encounter`.
- **Intended Backend Function:** `ackEncounterJoinHandler` -> `ackEncounterJoin()`
- **Auth:** required (`requireApiAuth`, `requireSquadMemberAccess`)

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
    "encounterId": "enc_01JY...",
    "squadId": "sq_01JY...",
    "acknowledged": true,
    "allAcked": false
  }
}
```

### Errors
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 ENCOUNTER_NOT_FOUND`
- `409 ENCOUNTER_EXPIRED`

---

## 2.4 Skip Encounter (Leader Only)

- **Endpoint:** `POST /matchmaking/skip`
- **Purpose:** Leader requests disconnect and rematch.
- **Intended Backend Function:** `skipEncounterHandler` -> `skipAndRequeueSquad()`
- **Auth:** required (`requireApiAuth`, `requireSquadLeaderAccess`)

### Request Body
```json
{
  "squadId": "sq_01JY...",
  "encounterId": "enc_01JY..."
}
```

### Success Response (200)
```json
{
  "ok": true,
  "data": {
    "squadId": "sq_01JY...",
    "previousEncounterId": "enc_01JY...",
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
6. `POST /squads/:squadId/search`
7. `POST /squads/:squadId/search/cancel`
8. `GET /matchmaking/status/:squadId`
9. `GET /matchmaking/encounters/:encounterId`
10. `POST /matchmaking/encounters/:encounterId/ack`
11. `POST /encounters/token`
12. `POST /encounters/disconnect`
13. `GET /health` + `GET /ready`
14. `POST /events` (optional)

This sequence gives a runnable MVP path: lobby -> queue -> match handoff -> video bootstrap.
