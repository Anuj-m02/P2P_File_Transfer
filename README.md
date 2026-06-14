# ⚡ P2P Web Share

> Direct browser-to-browser file transfer — no uploads, no cloud, no middleman.

![WebRTC](https://img.shields.io/badge/WebRTC-DataChannel-00d4ff?style=flat-square)
![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-WebSockets-009688?style=flat-square&logo=fastapi)
![No Server Storage](https://img.shields.io/badge/File%20Data-Never%20Touches%20Server-00ff88?style=flat-square)
[![Live Demo](https://img.shields.io/badge/Live%20Demo-Vercel-black?style=flat-square&logo=vercel)](https://p2p-file-transfer-six.vercel.app/)
[![Backend](https://img.shields.io/badge/Backend-Railway-8B5CF6?style=flat-square&logo=railway)](https://p2pfiletransfer-production-8fff.up.railway.app/)

---

## Live Demo

| | URL |
|---|---|
| **Frontend** | [https://p2p-file-transfer-six.vercel.app/](https://p2p-file-transfer-six.vercel.app/) |
| **Backend status** | [https://p2pfiletransfer-production-8fff.up.railway.app/](https://p2pfiletransfer-production-8fff.up.railway.app/) |

No setup required — open the frontend URL in two browser tabs (or share it with a friend) and start transferring files immediately.

---

## What is P2P Web Share?

P2P Web Share lets two browsers exchange files directly over a WebRTC DataChannel — the signaling server only brokers the connection handshake and never sees, stores, or relays any file data. Once both peers are connected, all bytes flow peer-to-peer at full network speed with SHA-256 integrity verification on every transfer.

---

## Features

- **Room-based pairing** — host creates a 6-character room code, guest joins with it; no accounts needed
- **True P2P transfer** — files stream directly browser-to-browser via RTCDataChannel; the server is blind to file contents
- **Chunked streaming** — files split into 16 KB frames with backpressure flow control (handles large files without memory overload)
- **SHA-256 integrity verification** — sender hashes the file before sending; receiver verifies after full reassembly
- **Live progress indicators** — real-time transfer %, speed (KB/s · MB/s), bytes remaining, and ETA
- **Disconnect & interrupt recovery** — 5-second grace period on network blips; mid-transfer interrupts detected and reported cleanly
- **Auto-download** — received file is reassembled in-browser and saved directly to the guest's Downloads folder

---

## How It Works

```
┌──────────────────────────────────────────────────────────────────────┐
│                     SIGNALING ONLY (no file data)                    │
│                                                                      │
│  Browser A (Host)        FastAPI Server        Browser B (Guest)     │
│       │                       │                       │              │
│       │── create room ───────>│                       │              │
│       │<─ room_id: "ABC123" ──│                       │              │
│       │                       │<── join room ─────────│              │
│       │<─ peer_joined ────────│                       │              │
│       │── SDP offer ─────────>│──── SDP offer ───────>│              │
│       │<─ SDP answer ─────────│<─── SDP answer ───────│              │
│       │── ICE candidates ────>│──── ICE candidates ──>│              │
│       │<─ ICE candidates ─────│<─── ICE candidates ───│              │
│       │                       │                       │              │
└───────│───────────────────────│───────────────────────│──────────────┘
        │                       ✗ file data never here  │
        │                                               │
        │   ╔═══════════════════════════════════════╗   │
        └───╢   WebRTC DataChannel  (direct P2P)    ╟───┘
            ║   file_info → binary chunks → done    ║
            ╚═══════════════════════════════════════╝
```

**Transfer protocol (DataChannel messages):**
1. Sender computes SHA-256 hash of the file (Web Crypto API)
2. Sends `file_info` JSON → `{ type, name, size, hash }`
3. Streams raw `ArrayBuffer` chunks (16 KB each) with flow-control drain loop
4. Sends `file_complete` JSON sentinel
5. Receiver reassembles chunks → triggers browser download → verifies SHA-256

---

## Tech Stack

| Layer | Technology |
|---|---|
| P2P Transport | WebRTC `RTCDataChannel` (`binaryType: "arraybuffer"`) |
| ICE / STUN | Google public STUN — `stun.l.google.com:19302` |
| File Integrity | Web Crypto API — `crypto.subtle.digest("SHA-256", ...)` |
| Frontend | HTML5, CSS3, Vanilla JavaScript (zero frameworks) |
| Signaling Server | Python 3.10+, FastAPI, WebSockets (`python-websockets`) |
| Frontend Hosting | Vercel |
| Backend Hosting | Railway |
| Fonts | Orbitron, Share Tech Mono (Google Fonts) |

---

## Getting Started

### Option A — Use the live deployment (no setup needed)

1. Open [https://p2p-file-transfer-six.vercel.app/](https://p2p-file-transfer-six.vercel.app/) in your browser
2. Share the link with your peer — they open the same URL
3. Follow the [Usage](#usage) steps below

The frontend automatically connects to the Railway backend — nothing to configure.

---

### Option B — Run locally

#### Prerequisites

- Python 3.10 or newer
- A modern browser (Chrome 90+, Firefox 90+, Edge 90+)
- [Live Server](https://marketplace.visualstudio.com/items?itemName=ritwickdey.LiveServer) VS Code extension **or** any static file server

#### 1 — Clone the repo

```bash
git clone https://github.com/Anuj-m02/P2P_File_Transfer.git
cd P2P_File_Transfer
```

#### 2 — Start the signaling server

```bash
cd backend

# Create and activate a virtual environment
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # macOS / Linux

# Install dependencies
pip install -r requirements.txt

# Copy the example env file (edit if you need a different port or origin)
copy .env.example .env         # Windows
# cp .env.example .env         # macOS / Linux

# Start the server
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

The signaling server is now running at `http://localhost:8000`.

#### 3 — Open the frontend

Open `frontend/index.html` with **Live Server** (right-click → *Open with Live Server* in VS Code).

> The WebSocket URL in `websocket.js` defaults to `ws://localhost:8000/ws` for local use. When deployed, it points to the Railway backend URL.

---

## Usage

### Host (the person sending a file)

1. Open the app and click **Connect**
2. Click **Create Room** — a 6-character room code appears
3. Share that code with your peer via any channel (chat, message, etc.)
4. Wait for the peer to join — the WebRTC badge turns green
5. Drag & drop a file onto the drop zone, or click **Choose File**
6. Click **Send File ↑** — progress bar, speed, and ETA update live
7. "Transfer Complete ✓" appears when done

### Guest (the person receiving a file)

1. Open the app and click **Connect**
2. Enter the room code you received and click **Join**
3. Wait for WebRTC to connect (green badge)
4. The progress bar appears automatically once the host starts sending
5. The file downloads to your **Downloads** folder when complete
6. SHA-256 result is shown — ✅ Verified or ❌ Corrupted

---

## Security & Privacy

| Property | Detail |
|---|---|
| **Server blind** | The FastAPI server only relays SDP and ICE messages — it never receives, buffers, or logs any file bytes |
| **In-transit encryption** | WebRTC DataChannel is encrypted with DTLS-SRTP by the browser automatically |
| **File integrity** | SHA-256 hash computed in the sender's browser, verified in the receiver's browser after full reassembly |
| **No persistence** | No file data is written to disk on the server; room state lives only in memory and is discarded on disconnect |
| **No accounts** | No sign-up, no cookies, no tracking |

---

## Deployment

### Frontend — Vercel

The `frontend/` folder is deployed as a static site on Vercel.

- Live URL: [https://p2p-file-transfer-six.vercel.app/](https://p2p-file-transfer-six.vercel.app/)
- Auto-deploys on every push to `main`
- No build step — plain HTML/CSS/JS served directly

### Backend — Railway

The `backend/` folder runs as a Python/FastAPI service on Railway.

- Live URL: [https://p2pfiletransfer-production-8fff.up.railway.app/](https://p2pfiletransfer-production-8fff.up.railway.app/)
- Start command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
- Auto-deploys on every push to `main`

### Monitoring endpoints

| Endpoint | Response |
|---|---|
| `GET /` | `{"status":"ok","active_connections":N,"active_rooms":N,"uptime":"...","version":"1.0.0"}` |
| `GET /health` | `{"status":"healthy"}` |
| `GET /stats` | `{"active_connections":N,"active_rooms":N,"uptime":"...","version":"1.0.0"}` |

---

## Project Structure

```
P2P_File_Transfer/
├── backend/
│   ├── app/
│   │   ├── main.py               # FastAPI app — HTTP endpoints + /ws WebSocket
│   │   ├── websocket_manager.py  # Tracks active WebSocket connections
│   │   ├── room_manager.py       # Room registry — create, join, peer routing
│   │   └── config.py             # HOST, PORT, ALLOWED_ORIGINS from .env
│   ├── requirements.txt
│   └── .env.example              # Environment variable template
│
├── frontend/
│   ├── index.html                # Single-page app shell
│   ├── css/
│   │   └── style.css             # Cyberpunk / gaming UI theme
│   └── js/
│       ├── ui.js                 # Central DOM module (window.ui.*)
│       ├── websocket.js          # WebSocket client + message bus
│       ├── crypto.js             # AES-GCM stub (Phase 8)
│       ├── hash.js               # SHA-256 via Web Crypto API
│       ├── webrtc.js             # RTCPeerConnection + DataChannel lifecycle
│       ├── transfer.js           # Chunked send/receive, progress, integrity
│       └── room.js               # Room lifecycle, signaling message routing
│
├── .gitignore
└── README.md
```

---

## Environment Variables

Copy `backend/.env.example` to `backend/.env` and adjust as needed:

| Variable | Default | Description |
|---|---|---|
| `HOST` | `0.0.0.0` | Interface to bind |
| `PORT` | `8000` | Port to listen on |
| `ALLOWED_ORIGINS` | `http://localhost:5500,...` | CORS whitelist (comma-separated) |

On Railway, `PORT` is injected automatically by the platform.

---

##Build by Anuj Maurya
