This is the "Move Fast" plan. For the MVP of **Giggle**, we are stripping away everything that isn't core to the "Squad-to-Squad" experience. No logins, no profiles—just a URL and a video feed.

Here is the **MVP Product Blueprint** for your `giggle-docs` repository.

---

# Giggle MVP: Functional Requirements (The "Raw" Demo)

**Goal:** Allow a user to create a link, have a friend join, and then have that pair of friends automatically matched with another pair of friends in a live video call.

---

## 1. Core Component: The "Squad Link" System
To keep it "raw" and frictionless, we use URL-based room entry.
* **The Landing Page:** A single button: `[ Create a Squad ]`.
* **The Session ID:** Clicking the button generates a unique, short-lived ID (e.g., `giggle.social/s/apple-zebra-12`).
* **The Joiner Logic:** Anyone who visits that specific URL is automatically added to that "Squad" in the backend.

## 2. Core Component: The "Lobby" Instance
Before the match happens, the friends need to see each other.
* **Local Media Check:** Access camera/mic and show the user their own face.
* **The Friend Grid:** A simple 2-column layout where the creator sees the joiner(s).
* **The "Find Match" Button:** Only visible to the person who created the link (the "Squad Leader").

## 3. Core Component: The "Matchmaker" Logic
This is the "Brain" in `giggle-server`.
* **The Search Signal:** When the Leader hits "Find Match," the server flags that specific Squad ID as `status: "searching"`.
* **The Collision:** The server checks if there is any other Squad ID with the `searching` status.
* **The Handover:** As soon as two squads are found, the server generates a unique `Match_ID` and pushes it to all 4 (or more) users via WebSockets.

## 4. Core Component: The "Video Instance" (Agora Integration)
This is the actual "Omegle" moment.
* **Auto-Join:** The app listens for the `Match_ID` from the server and instantly connects everyone to that Agora Channel.
* **The 2v2 Grid:** A split-screen layout. 
    * **Left Side:** Your Squad.
    * **Right Side:** The Stranger Squad.
* **The "Skip" Button:** Disconnects from the current `Match_ID` and puts the squad back into the "Searching" state.

---

## MVP Priority List (The Build Order)

| Priority | Component | Why? |
| :--- | :--- | :--- |
| **1** | **Socket Signaling** | You can't match people if the server doesn't know who is online. |
| **2** | **Agora Web SDK** | Getting the "Camera On" and streaming is the biggest technical hurdle. |
| **3** | **Redis Queue** | Managing the list of squads waiting to be matched. |
| **4** | **Basic UI Layout** | A simple CSS grid to show 4+ people without the videos overlapping. |

---

## What we are EXCLUDING (For the Demo)
To stay "Raw," we will skip these entirely for the first 3 weeks:
* **Authentication:** No emails, no passwords. Use `localStorage` to remember a temporary username.
* **Database Persistence:** If the server restarts, all current squads vanish. (Acceptable for an MVP).
* **Profile Pictures:** Use a default avatar or just the video feed.
* **Mobile Optimization:** Focus on the Desktop Chrome experience first.

---

## Step-by-Step Execution for your Repos

1.  **giggle-server:** Build a Node script that handles `create-squad`, `join-squad`, and a basic `match-squad` array.
2.  **giggle-web:** Build a single page that asks for a name, shows the camera, and sends a "Search" signal to the server.
3.  **The Test:** Open two different browsers (Chrome and Brave) on your MacBook. Create a squad in each. Hit "Search" in both. **If you see yourself in both windows, the MVP is a success.**

**Would you like me to generate the "Matchmaker" logic for the server so you can test this on your local machine tonight?**