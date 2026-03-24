This is the definitive **Giggle MVP Product Blueprint**. This document is designed to be the "source of truth" in your `giggle-docs` repository, focusing on a raw, functional demo that proves the core "Squad-to-Squad" concept without any unnecessary overhead.

---

# 🚀 Giggle MVP: Core Product Blueprint

**Project Name:** Giggle  
**Concept:** Social discovery for squads. Don't meet strangers alone; bring your friends.  
**MVP Goal:** A "zero-friction" web demo where a group of friends can form a squad via a 6-digit code and be matched into a 2v2 or 3v3 video instance with another squad.

---

## 1. User Journey (The "Raw" Flow)

1.  **Entry:** User arrives at `giggle.social`. They see two buttons: `[Create a Squad]` and `[Join with Code]`.
2.  **Squad Creation:** * **Leader:** Clicks "Create," gets a 6-digit **Squad Code** (e.g., `GIG-882`), and enters the **Lobby**.
    * **Friends:** Enter the 6-digit code on the home screen and instantly appear in the Leader's Lobby.
3.  **The Lobby:** All squad members see their own cameras. The Leader sees a `[Find a Match]` button once at least one friend has joined.
4.  **The Match:** * Leader hits "Find a Match." The entire squad enters the global queue.
    * The system pairs them with another searching squad.
5.  **The Encounter:** Both squads are instantly moved into a shared video room.
    * **Layout:** Your squad on the left, their squad on the right.
    * **Action:** A `[Skip]` button allows the Leader to find a new match.

---

## 2. Functional Components (Prioritized)

### A. Home & Lobby System
* **Code Generator:** A backend utility to create unique, non-colliding 6-digit alphanumeric strings.
* **Stateful Lobby:** A real-time synchronized view where the list of "Connected Friends" updates instantly as people enter/leave the code-room.
* **Media Permissions:** A "Pre-flight" check to ensure Camera and Microphone are active before allowing a user to join a squad.

### B. The Matchmaking Brain
* **The Queue:** A Redis-backed list that holds `Squad_IDs` currently looking for an encounter.
* **Pairing Logic:** A simple FIFO (First-In-First-Out) algorithm that pulls the first two available squads from the queue and assigns them a shared `Meeting_ID`.
* **Sync-Move:** A WebSocket broadcast that tells all clients in both squads to "Switch to Video Room: X" at the exact same millisecond.

### C. Video Instance Engine (Agora)
* **Dynamic Channel Joining:** Automatically connecting to an Agora channel based on the `Meeting_ID`.
* **Dynamic Grid:** A CSS-grid that scales based on the number of people. (e.g., If it's a 2-man squad meeting a 3-man squad, the grid handles 5 video feeds).
* **Manual Disconnect:** Logic to gracefully exit a channel and return to the "Lobby" state without refreshing the page.

---

## 3. Technical Constraints & Omissions (The "Keep it Simple" Rules)

* **Google OAuth Required:** Users must be signed in with Google; API identity is based on `providerAccountId` from the Auth.js MongoDB `accounts` collection.
* **Secured APIs:** All product APIs are authenticated/authorized server-side (leader-only and squad-member checks where needed).
* **Ephemeral Product State:** Squad rooms/matchmaking can remain in-memory/Redis for MVP; auth/account data persists in MongoDB via Auth.js.
* **Desktop-First:** Optimized for Chrome/Safari on desktop to avoid the complexities of mobile browser camera constraints.
* **No Moderation (For Demo):** The initial demo assumes a "trusted" group of testers. AI moderation will be added in Phase 2.

---

## 4. Success Metric for MVP
**"The 2-Browser Test":**
> Successfully opening two tabs on a MacBook, creating two separate squads (4 'users' total), hitting "Find Match," and seeing all four video streams interacting in a single shared instance without a page reload.

---

## 5. Backend API Contract
Detailed request/response contracts and intended backend handler functions are documented in:

`BACKEND_API_SPEC.md`

---

### Next Step for Giggle
With this document finalized, your next move is to build the **Signaling Server** in the `giggle-server` repo. 

