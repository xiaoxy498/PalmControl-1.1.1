# PalmControl

> Phone-to-phone remote control for Android. Share one phone's screen, watch and operate it from another phone.
> One APK, two roles · Self-hosted relay · MIT licensed

[![License: MIT](https://img.shields.io/badge/license-MIT-38BDF8.svg)](LICENSE)
[![Android](https://img.shields.io/badge/Android-8.0%2B-3DDC84.svg)](https://developer.android.com)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-339933.svg)](https://nodejs.org)

[中文](README.md) · [Architecture](docs/ARCHITECTURE.md) · [Protocol](docs/PROTOCOL.md) · [AI-assisted deployment (Chinese)](docs/AI-DEPLOY.md) · [Contributing](CONTRIBUTING.md)

---

## Why this exists

Remote control between desktop computers is a solved problem. Remote control between **phones** is not:

- Most tools require both phones to sign in to the same account or go through a vendor cloud.
- Many only let you *watch* the screen, without real touch injection.
- Cloud relays add latency and remove control over your own data, even on a LAN.
- Some require root, ADB, or a PC in the middle.

PalmControl takes a different approach:

- **One APK, both roles** — the same app can act as the controlled device (Host) or the controlling device (Controller).
- **Self-hosted relay** — a small Node.js server (~500 lines) that runs on a Raspberry Pi, NAS, VPS, or Docker.
- **No accounts** — 6-digit pairing code, optional PIN, and manual approval on the controlled device.
- **Real gesture injection** — taps, long-presses, swipes, two-finger gestures, and text input via an AccessibilityService.
- **Works entirely on a LAN** — point both phones at `ws://192.168.x.x:8080/ws`.

Typical use cases: helping a family member with their phone, debugging your spare device, demoing an app to a client, or operating a device that stays at home.

---

## Features

### Host (controlled device)

- One-tap screen sharing via MediaProjection, kept alive by a foreground service
- 6-digit pairing code, valid for 10 minutes
- Optional 4–6 digit PIN (the server only stores a SHA-256 hash)
- Manual approval for every join request; auto-denied after 30 seconds
- Optional unattended mode ("auto accept", off by default)
- Quality presets: 480p / 720p / 1080p
- Frame rate: 10 / 15 / 20 FPS
- Persistent notification with a "stop sharing" action
- Automatically rebuilds capture on orientation change

### Controller (controlling device)

- Connect with server URL + pairing code
- Full-screen, aspect-correct video with letterboxing
- Direct touch: tap, long-press, swipe, two-finger gesture
- Toolbar: Back / Home / Recents / Notifications / Keyboard
- Text input injected into the focused field on the host
- Live stats: FPS, bitrate, resolution
- Clear errors for disconnects, network failures, and host exit

### Relay server

- Pure WebSocket relay — forwards messages, never inspects frames
- Room lifecycle: 10-minute pairing TTL, 60-second pending timeout
- Abuse limits: 2 MB per frame, 16 KB per text message, 120 input events/sec, one controller per room
- Heartbeats, graceful shutdown, `/health` endpoint
- No database, no accounts — restart clears everything
- `Dockerfile` and `docker-compose.yml` included

---

## Architecture

```
┌────────────────────┐        WebSocket        ┌────────────────────┐
│   Host (Android)    │  ───────────────────►  │   Relay Server      │
│  MediaProjection    │   JSON control msgs    │   Node.js + ws      │
│  VirtualDisplay     │   + binary JPEG frames │   rooms / pairing   │
│  AccessibilityService│ ◄───────────────────  │                    │
└────────────────────┘        WebSocket        └────────────────────┘
                                                        ▲
                                                        │ WebSocket
                                                        ▼
                                              ┌────────────────────┐
                                              │ Controller (Android)│
                                              │ decode / draw / touch│
                                              └────────────────────┘
```

- **Video**: MediaProjection → VirtualDisplay → ImageReader → JPEG → binary WebSocket frames.
- **Input**: touches are normalized to 0..1 coordinates, sent as `input.gesture`, and injected by the host's AccessibilityService.
- **Session**: a 6-digit room code identifies the room; after pairing the relay forwards traffic in both directions without decoding it.
- **Why not WebRTC in v1**: no STUN/TURN to operate, one tiny service to self-host, and no heavy native dependencies. H.264/WebRTC is on the roadmap.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/PROTOCOL.md](docs/PROTOCOL.md).

---

## Quick start

### 1. Run the server

> Prefer to delegate? Paste [docs/AI-DEPLOY.md](docs/AI-DEPLOY.md) into any AI assistant and it will clone the server from GitHub, deploy it, configure TLS, and verify it for you.

**Docker**

```bash
git clone https://github.com/<your-name>/palmcontrol.git
cd palmcontrol
docker compose up -d
```

**Node.js**

```bash
cd server
npm install
npm start
# PalmControl relay listening on ws://0.0.0.0:8080/ws
```

**Verify**

```bash
curl http://127.0.0.1:8080/health
# {"ok":true,"rooms":0,"uptimeSec":1}
```

LAN URL example:

```text
ws://192.168.1.10:8080/ws
```

For public access, put a TLS reverse proxy (Nginx / Caddy / Traefik) in front and use `wss://`. The repository intentionally ships no certificates or private keys.

Minimal Nginx example:

```nginx
server {
    listen 443 ssl;
    server_name palm.example.com;

    ssl_certificate     /etc/letsencrypt/live/palm.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/palm.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;
    }
}
```

### 2. Build the Android APK

Requirements:

| Item | Version |
| --- | --- |
| JDK | 17 |
| Android SDK | API 35 (Build-Tools 35.0.0) |
| Gradle | Use the bundled `gradlew` (8.14.6) |
| minSdk | 26 (Android 8.0) |
| targetSdk | 35 (Android 15) |

```bash
cd android
./gradlew :app:assembleDebug
# Output: app/build/outputs/apk/debug/app-debug.apk
```

Install:

```bash
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

You can also open `android/` in Android Studio and run it.

> For a release build, configure your own `signingConfigs` in `android/app/build.gradle.kts`, then run `./gradlew :app:assembleRelease`. No keys are committed.

### 3. Pair and control

1. Install the same APK on both phones.
2. Open **Me** → sign up (username + password, stored on-device only) → log in.
3. Go to **Me → Settings** → turn on **Allow sharing this screen** (off by default). The server URL is prefilled.
4. **Host phone**: open **Devices** → tap **Start sharing this screen** → allow notifications → approve screen capture → enable the accessibility service. A 6-digit code appears.
5. **Controller phone**: open **Remote** → enter the 6-digit code → tap **Start remote control** → enter the PIN if the host set one, otherwise tap **Connect** directly.
6. Approve the "Connection request" dialog on the host.
7. The controller shows the live screen; touch to operate.

Enable accessibility: Settings → Accessibility → Downloaded apps → **PalmControl · Remote control** → On.

---

## Security and privacy

This is remote-control software, so the security model matters:

- **Default public relay** — with no configuration, traffic goes through the author's public server, which is fine for trials. Self-host for privacy so data stays on your machine.
- **PINs are hashed** — the server stores only `SHA-256(pin)`. Pairing codes are 6 digits and expire in 10 minutes.
- **Manual approval by default** — the host must tap "Allow". Auto-accept is opt-in.
- **One controller per room** — no silent spectators.
- **Rate limits** — 2 MB per frame, 16 KB per text message, 120 input events per second.
- **Use `wss://` on the public internet** — `ws://` is plaintext and can be sniffed on untrusted networks.
- **Accessibility is powerful** — it can inject gestures and read window content. Only enable it for an APK you trust, and turn it off when not needed.
- **No accounts, no telemetry, no ads** — nothing is collected.

Please report vulnerabilities privately rather than posting reproducible exploit details in a public issue.

---

## Known limitations

These are Android platform constraints, not bugs:

1. **The screen goes black when the host sleeps** — MediaProjection cannot capture a locked/off display. Keep the host screen awake.
2. **System keys cannot be injected** — accessibility supports global actions (Back/Home/Recents/Notifications), not Power or Volume.
3. **Secure screens are blocked** — banking apps, password managers, and other `FLAG_SECURE` surfaces appear black and reject input.
4. **Screen capture must be re-approved each session** — Android 14+ requires fresh consent for every MediaProjection session.
5. **Sideloaded APKs may need "Restricted settings"** — on Android 13+, allow the accessibility service under App info → Restricted settings.
6. **Quality depends on the network** — JPEG by default, dropping frames under pressure. Not suitable for high-frame-rate game streaming.
7. **No audio, file transfer, or clipboard sync yet.**
8. **Android 8.0+** is required on both devices.

---

## Roadmap

- [ ] H.264 hardware encoding + WebRTC transport
- [ ] Audio playback
- [ ] File transfer
- [ ] Two-way clipboard sync
- [ ] QR-code pairing
- [ ] Read-only multi-viewer mode
- [ ] Trackpad mode for precise control
- [ ] Custom key mapping
- [ ] End-to-end encryption (currently TLS at the transport layer)
- [ ] Built-in TLS and certificate pinning

Discuss priorities in Issues.

---

## Repository layout

```text
palmcontrol/
├── android/                     # Android client (Kotlin, one APK, two roles)
│   ├── app/src/main/java/com/palmcontrol/
│   │   ├── controller/          # remote view (decode / draw / gestures)
│   │   ├── host/                # capture, encoding, input injection
│   │   ├── net/                 # protocol messages and WebSocket client
│   │   ├── service/             # capture foreground service, accessibility service
│   │   ├── ui/                  # home, host, controller, settings
│   │   └── util/                # prefs, URL normalization
│   └── app/src/test/            # JVM unit tests
├── server/                      # Node.js relay
│   ├── src/server.js            # rooms, pairing, forwarding, limits
│   ├── test/server.test.js      # end-to-end protocol tests
│   └── Dockerfile
├── docs/
│   ├── ARCHITECTURE.md
│   └── PROTOCOL.md
├── docker-compose.yml
├── CONTRIBUTING.md
└── LICENSE
```

---

## Tests

```bash
# Server: pairing, PIN, denial, disconnects, health
cd server && npm test

# Android: URL normalization, resolution math
cd android && ./gradlew :app:testDebugUnitTest
```

---

## FAQ

**Can two phones connect directly without a server?**
Not in v1. P2P requires NAT traversal and signaling. You can run the relay on your LAN router/NAS so traffic never leaves the local network.

**Why not ADB?**
ADB needs a PC and USB debugging. Accessibility is the practical no-root option for phone-to-phone use.

**Can the controller see my notifications?**
Yes — it sees the host screen, including the notification shade. Clear sensitive content before sharing.

**Multiple controllers at once?**
No. A second controller is rejected with `host_busy`.

**Why is the image slightly soft?**
The default is 720p at JPEG quality 62. Switch the host to "Sharp", or raise the frame rate if bandwidth allows.

**iOS support?**
No. iOS does not allow sideloading apps that inject touches, and third-party input injection is not permitted.

---

## License

[MIT](LICENSE) © 2026 PalmControl contributors

Keep the copyright notice for reuse or commercial use. Issues and PRs are welcome.
