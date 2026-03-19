# Look At My Screen

Peer-to-peer screen sharing in a single HTML file. No backend, no signaling server, no install.

Models: Human brain for idea, Sonnet 4.6 for plan, Opus 4.6 for build

---

## How it works (quick example)

```
1. Sharer opens the page, clicks "Start Sharing", picks a window/screen.
   → A unique invite link is generated and shown.

2. Sharer sends the invite link to the viewer (Slack, email, whatever).

3. Viewer opens the link in their browser.
   → A response link is automatically generated.

4. Viewer sends the response link back to the sharer.

5. Sharer clicks the response link.
   → The connection is established. Viewer sees the screen live.
```

No accounts. No server. No data leaves your machine except to the peer.

---

## How it works (technical)

The app uses **WebRTC** for the actual screen stream, but WebRTC requires both sides to exchange connection metadata (an SDP offer/answer) before a direct link can be established. Normally a signaling server handles this. Here, the URL hash does it instead.

**Signaling via URL hash:**

- The sharer's SDP offer is base64-encoded and embedded into a URL hash (`#offer=...`).
- The viewer opens that URL, generates an SDP answer, encodes it the same way, and creates a response URL (`#answer=...`).
- When the sharer opens the response URL, the page detects it is a relay tab and uses the **BroadcastChannel API** to pass the answer to the original sharer tab (same browser, same machine).
- Once both sides have each other's SDP + ICE candidates, WebRTC takes over and establishes a direct peer connection.

**Three page routes, one file:**

| URL hash                      | Role                                                                            |
| ----------------------------- | ------------------------------------------------------------------------------- |
| _(none)_                      | Sharer home — start sharing, manage viewer slots                                |
| `#offer=...&sid=...&idx=...`  | Viewer page — auto-generates and displays the response link                     |
| `#answer=...&sid=...&idx=...` | Relay tab — forwards answer to the sharer tab via BroadcastChannel, then closes |

**ICE candidate gathering** is completed in full before a link is generated (`waitForIceGathering`), so the full connection info is baked into a single link — no trickle ICE, no second round-trip needed.

Multiple viewers are supported; each gets its own offer/answer slot tracked by `idx`.

---

## Technologies

| What                                       | Why                                                                                        |
| ------------------------------------------ | ------------------------------------------------------------------------------------------ |
| **WebRTC** (`RTCPeerConnection`)           | Peer-to-peer video stream of the captured screen                                           |
| **Screen Capture API** (`getDisplayMedia`) | Captures a screen, window, or tab as a `MediaStream`                                       |
| **BroadcastChannel API**                   | Lets the relay tab silently hand the answer back to the open sharer tab without any server |
| **URL hash (`location.hash`)**             | Carries the SDP offer/answer between machines as base64 in the URL                         |
| **`crypto.randomUUID()`**                  | Generates a session ID so a relay tab targets the right sharer tab                         |

No npm. No framework. No build step. Open `index.html` directly in a browser.

---

## Dependencies

**None.** Everything used is a standard browser API available in all modern browsers (Chrome, Edge, Firefox, Safari).

---

## ICE Servers

```js
const ICE_SERVERS = [
  { urls: "stun:stun.l.google.com:19302" },
  { urls: "stun:stun1.l.google.com:19302" },
];
```

These are the **only external connections** the app makes — and they only happen during connection setup, not during the stream.

**What they are:** STUN servers (Session Traversal Utilities for NAT), operated by Google and free for public use.

**What they do:** Most devices sit behind a NAT router and don't know their own public IP address. Before WebRTC can connect two peers directly, each peer asks a STUN server "what is my public IP and port?". The STUN server replies with that information and immediately disconnects. This is called an ICE candidate.

**What they do NOT do:** STUN servers never see your screen, never relay your stream, and have no knowledge of who you are connecting to. The query is a single UDP packet exchange — effectively "what's my public IP?" and nothing more.

**No TURN servers are configured.** TURN servers act as a fallback relay when a direct peer connection cannot be established (e.g., both peers are behind symmetric NAT/strict firewalls). Without TURN, connections in those rare network environments will fail. This is an intentional trade-off to keep everything serverless.

If you want to add TURN for reliability, replace `ICE_SERVERS` with your own TURN credentials at the top of `index.html`.
