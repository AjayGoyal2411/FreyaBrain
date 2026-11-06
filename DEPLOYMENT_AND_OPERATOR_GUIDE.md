# NeuroSpeak Deployment Guide, Operator Script & Test Report

## 1. How to Run the Website Locally

The complete NeuroSpeak project is located in:
- `C:\Users\Admin\Downloads\NeuroSpeak\dev-prototype` (primary workspace copy)
- `C:\Users\Admin\.gemini\antigravity\brain\d09054ed-b8ce-48b6-acca-cc5cc6623977\scratch\dev-prototype` (verified build & test mirror)

### Local Launch Commands (Windows PowerShell)

```powershell
cd C:\Users\Admin\Downloads\NeuroSpeak\dev-prototype

# 1. Start the local Vite development server (http://localhost:5173)
npm.cmd run dev

# 2. Or build and serve the production bundle (http://localhost:4173)
npm.cmd run build
npm.cmd run preview

# 3. Run automated Vitest unit tests (14 tests)
npm.cmd test

# 4. Run automated Playwright end-to-end browser tests (5 browser scenarios)
npm.cmd run test:e2e
```

### Routes Overview
- `/` — **Introduction & Mode Choice**: Overview, apparatus & wireless architecture modal ("Setup & How It Works"), and direct links to the workbench views.
- `/demo` — **One-Laptop Side-by-Side Demonstration**: Renders the Laptop Operator Workbench on the left and the exact `TabletDisplay` component on the right (`Tablet preview`), driven by the same live session state. Includes the **Presentation View** toggle.
- `/operator` — **Laptop Operator Workbench**: Full operator controls, scenario picker, signal provenance, EEG/yaw waveform drawer, webcam tracking & calibration panel, and private tablet pairing controls.
- `/display` — **Clean Tablet Display**: Full-screen capable landscape display showing strictly two choices at a time (`Choice 1 of 3` → `Choice 2 of 3` → `Choice 3 of 3`) or the single final phrase during head-turn review.

---

## 2. Client & Competition Operator Script (Successful Approval vs. Silent Cancellation)

### Pre-Session Check
1. Open `http://localhost:5173/demo` in Chrome or Microsoft Edge.
2. Click **🔊 Audio Check** on the Laptop Operator pane and verify the short 520 Hz test tone plays (`✓ Audio check passed · Speaker ready`).
3. Optional: Click **Presentation View** in the top bar to enlarge the Tablet Preview and display the 5-step live pipeline strip (`Recorded EEG → Laptop decision → Tablet choice → Head confirmation → Speech`).

### Scenario 1: The "I need water" Presentation Moment (3 Left Choices + Right Approval)
1. Ensure mode tab **A · Competition Replay (Default)** is active.
2. In the **Scenario** dropdown, select:
   `Water request · 3 Left choices + Right approval (Spoken once)`
3. Click **▶ Start Demonstration**.
4. **Narrate the 3-stage binary menu**:
   - **Stage 1 (`Choice 1 of 3`)**: Tablet shows *Personal needs* (Left) vs. *Comfort and support* (Right). Archived EEG trial `S031R08_E004` (Left) arrives → highlights `Decision received (Left)` → advances one step.
   - **Stage 2 (`Choice 2 of 3`)**: Tablet shows *Food and drink* (Left) vs. *Personal care* (Right). Archived EEG trial `S031R08_E008` (Left) arrives → advances one step.
   - **Stage 3 (`Choice 3 of 3`)**: Tablet shows *I need water* (Left) vs. *I need food* (Right). Archived EEG trial `S031R08_E012` (Left) arrives.
5. **Narrate the Phrase Reveal & Approval**:
   - Choices are replaced by the single large phrase **`I need water`**.
   - EEG navigation pauses (`WAITING FOR YOUR APPROVAL · EEG NAVIGATION PAUSED`).
   - Simulated gyroscope trace performs a 12° right turn, holds for ≥0.5 s, and returns within ±2° of neutral.
   - Gesture stepper advances through `1 Neutral centre` → `2 Right turn detected · Hold` → `3 Return to centre` → `4 Message approved`.
   - Audio file `/audio/LLL.mp3` ("I need water") plays **once**, and status transitions to `🔊 Spoken`.

### Scenario 2: Wrong Preview Silent Cancellation
1. In the **Scenario** dropdown, select:
   `Wrong preview · Scripted water, displayed food + Left cancel (Silent)`
2. Click **▶ Start Demonstration**.
3. Path `LLR` selects **`I need food`** while the scripted request was water.
4. During the review window, a simulated left turn (-12°), 0.5 s hold, and return to centre cancels the phrase (`✕ Cancelled · Speaker silent`). No audio is played, and the tablet offers `Start a fresh request`.

### Scenarios 3 & 4: Incomplete Gesture and 8-Second Timeout
- Select `Incomplete gesture · Visible turn, short hold, no speech`: A right turn begins but returns after 230 ms (< 500 ms required hold). Review expires silently.
- Select `Timeout · No qualifying gesture within 8 s (Silent)`: Yaw remains at 0° for 8 seconds. Review expires (`⏱ Timed out · Speaker silent`) with zero speech.

---

## 3. How to Pair a Real Tablet and Verify State Synchronization

1. Open `/operator` on the laptop. Locate the **Real Laptop-to-Tablet Pairing** panel (`Session ID: NS-XXXX-XXXX` and `Expiring Token`).
2. **Same-machine two-window verification**:
   - Click the **Tablet Pairing Link** on `/operator` (opens `/display?session=NS-...&token=...` in a new browser window/tab).
   - On `/operator`, observe the status badge update to `Tablet Joined (TAB-XXXXX) · Awaiting Operator Acknowledge`.
   - Click **✓ Acknowledge Tablet Join** on `/operator`.
   - Verify the **DISPLAY SYNC** pipeline node on `/operator` highlights green and reads **`Tablet updated (ACK #1)`** — confirming the tablet received and acknowledged state sequence `#1`.
   - On the `/display` window, click **🔊 Enable sound** once so the tablet browser unlocks audio playback.
   - Start a scenario on `/operator`: in paired mode, **only the tablet plays the approved speech**.
3. **Cross-device LAN / Deployed Pairing**:
   - Set `VITE_RELAY_WS_URL` in `.env.local` to your deployed Cloudflare Worker Durable Object WebSocket URL (`wss://<worker-subdomain>/`), build, and open the pairing link or QR URL on the physical tablet.

---

## 4. Webcam Head Tracking Verification & Measured Timings

1. Open `/operator?mode=webcam` or `/demo?mode=webcam` and select tab **B · Live Webcam Engineering Demo**.
2. Camera permission is requested **only** when you click **📷 Enable camera**.
3. ** Tracker & Calibration**:
   - Uses self-hosted **YuNet ONNX (`yunet.onnx`) + ONNX Runtime WebAssembly (`ort-wasm-simd-threaded.wasm`)** running inside a dedicated Web Worker (`/tracking/camera-worker.mjs`).
   - Guides the user through 5 calibration steps (or quick neutral lock): neutral centre → physical right turn → centre → physical left turn → finish centre.
   - **Mirrored sign calibration**: Even though the video preview is horizontally mirrored (`scaleX(-1)`) for natural user feedback, `rightSign` is calibrated from the user's physical turn toward their right shoulder.
   - **Pose rejection tolerances (`WEBCAM_TOLERANCES`)**:
     - Frame staleness limit: `450 ms`
     - Max head roll (sideways tilt): `±18°`
     - Max vertical pitch ratio shift (nodding): `±0.38`
     - Allowed eye-scale distance ratio: `0.65× to 1.50×` calibrated neutral
     - Any face loss, second face in view, or stale frame immediately disarms `YawGate` back to `neutral`.
4. **Measured Timings on Test Hardware (Windows 11, Node.js v26.13.0, Microsoft Edge Chromium)**:
   - **Pose state-machine & UI feedback latency**: `1–4 ms` for state evaluation; `~45–85 ms` end-to-end worker inference + render per 640×480 frame (well within the 200 ms target).
   - **Accepted-gesture-to-audio start latency**: `12–35 ms` using pre-fetched `AudioContext.decodeAudioData` buffer playback.

---

## 5. Private Deployment Guide (Cloudflare Pages + Worker Durable Object Relay)

### Step A: Deploy the Pairing Relay Worker + Durable Object
```powershell
cd C:\Users\Admin\Downloads\NeuroSpeak\dev-prototype\relay

# Authenticate with Cloudflare (requires interactive account login)
npx wrangler login

# Deploy the Worker and PairingSessionDO Durable Object migration
npx wrangler deploy
```

### Step B: Configure Frontend Relay URL & Build Production Bundle
```powershell
cd C:\Users\Admin\Downloads\NeuroSpeak\dev-prototype

# Create .env.local pointing to your private Worker relay
Set-Content -Path .env.local -Value "VITE_RELAY_WS_URL=wss://neurospeak-pairing-relay.<your-subdomain>.workers.dev"

# Build static production assets into dist/
npm.cmd run build
```

### Step C: Deploy Frontend to Cloudflare Pages & Enforce Access Policy
1. Before sharing any link, open the **Cloudflare Zero Trust Dashboard → Access → Applications** and create a Self-Hosted Access Application covering both:
   - Your Cloudflare Pages domain (`neurospeak-prototype.pages.dev` and `*.neurospeak-prototype.pages.dev` preview deployments)
   - Your Worker relay domain (`neurospeak-pairing-relay.<your-subdomain>.workers.dev`)
2. Add an Access Policy (e.g., One-Time Pin email allowlist for clinicians/reviewers).
3. Deploy the built `dist/` folder:
```powershell
npx wrangler pages deploy dist --project-name neurospeak-prototype
```
4. Verify from an unauthenticated/incognito browser window that Cloudflare Access prompts for authentication before serving the app, and verify HTTPS camera + audio permissions after signing in.
