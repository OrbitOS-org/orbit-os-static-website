# Show HN: Orbit OS – an Android-like runtime for embedded Linux, without Docker

---

Every embedded team rebuilds the same infrastructure. GPIO handling. OTA updates. Remote access. Application lifecycle. Secure APIs. Over and over again.

We decided to fix that. Without Docker.

---

## The Problem

If you've worked on embedded Linux, you know the pattern:

- Project starts. You write the app logic.
- Then you spend weeks building the platform layer nobody talks about — device management, remote access, OTA updates, hardware abstraction, process supervision.
- Then the next project starts. You do it again.

Balena solved part of this with Docker containers on edge devices. It works — but containers weren't designed for hardware that boots in seconds, runs on 256MB of RAM, and needs to survive in the field for years without intervention. You're running a Docker daemon, overlay filesystems, and container images just to blink an LED.

We think Docker is the wrong abstraction for embedded. So we built a different one.

---

## What We Built

**Orbit OS** is a secure edge operating system for embedded Linux devices. It runs on top of existing Linux distributions and turns devices into managed compute nodes — with a unified API for hardware, networking, AI, and system services.

The core is **Gravity RT**, a native execution runtime inspired by Android. Instead of containers, apps are distributed as signed `.orb` packages — think APK for edge devices.

Nothing executes outside Gravity RT.

---

## The Architecture

```
Hardware
└── Linux (Debian base)
    └── Gravity RT
        ├── System Server (Go binary) — always running, minimal footprint
        ├── gRPC / UDS  — internal .orb app communication
        ├── gRPC / TCP mTLS — external SDK, CLI, mobile apps
        ├── JVM — lazy loaded, only when a Java .orb is running
        ├── Python VM — lazy loaded, only when a Python .orb is running
        └── TFLite (C++) — AI inference, lazy loaded
```

**System Server** is written in Go — static binary, minimal RSS, always active. It's the process parent for every `.orb` — if an app crashes, it restarts it. If the System Server itself falls, systemd brings it back up.

**Two gRPC APIs, two trust boundaries:**

- **UDS** for internal `.orb` apps — only accessible locally on the device, apps cannot escape
- **TCP mTLS** for external tools — mutual TLS authentication, no open ports, no implicit trust

**Lazy loading everywhere** — the JVM, Python VM, and TFLite only start when an app that needs them is launched. On a 512MB device, this matters.

---

## The .orb Package Format

A `.orb` is the unit of deployment in Orbit OS. Here's the real structure of our Face Recognition app (6MB total, including 3 TFLite models):

```
.
├── bin
│   └── face_recognition_client      ← native compiled binary
├── data
│   ├── blaze_face_full_range.tflite ← face detection model
│   ├── face_landmark.tflite         ← facial landmarks model
│   └── mobilefacenet.tflite         ← face identification model
├── icon.svg
├── lib
├── manifest.json
└── META-INF
    ├── CERT.crt
    ├── CERT.SF
    ├── CERT.sig
    └── MANIFEST.MF
```

And the manifest:

```json
{
  "package_id": "org.orbit-os.service.face-recognition",
  "version": "0.3.0",
  "name": "Edge AI - Face Recognition",
  "type": "binary",
  "architecture": "arm64",
  "entry_point": "face_recognition_client",
  "git_commit": "d1d9709",
  "permissions": [
    "SystemService/*",
    "CameraService/*",
    "AiService/*",
    "AppHubService/*"
  ]
}
```

Notice what's **not** in the permissions: `NetworkService`. This app has no network access — by design, enforced by the runtime. The face recognition data never leaves the device. Privacy enforced at the architecture level, not by policy.

Gravity RT reads the manifest at launch, validates the signature chain, and only exposes the declared APIs to the app. Nothing more.

Each `.orb` also gets a persistent data directory at `/data/orb/<app-id>/` that survives both app restarts and full runtime OTA updates. The runtime never deletes app data automatically.

---

## Real-Time Development — No Deploy Loop

This is the part that changes how embedded development feels.

**Traditional embedded workflow:**
```
Write code → Compile → Flash → Boot → See error → Repeat
```

**Orbit OS workflow:**
```
Code runs on your laptop
  → Calls GPIO API       → pin on the real device toggles
  → Calls AiService API  → TFLite runs on the device's hardware
  → Calls CameraService  → live frame from the device camera
  → Calls NetworkService → configures the device's network interface
  → Logs stream back instantly
  → Iterate — change code, call again
  → When it works: package as .orb, sign, publish to Store
```

Your code runs on your development machine. Your code connects to the real device over the network.

Running on your laptop, your code calls device APIs and you can remotely activate GPIO pins, run AI inference, configure networking, access the camera, and use any other device capability — all through the same API that production apps use. No packaging, no deploying, no rebooting.

No SSH required. Dev Mode is enabled in Settings (Android-style), which activates the SDK certificate for passwordless mTLS auth — the device warns you explicitly about the security tradeoff when you enable it.

This is what Chrome DevTools did for web development — bring the real execution environment close to the developer. We think embedded deserves the same.

---

## OTA Updates — Built on OSTree

System updates use **OSTree** — the same technology behind ChromeOS, Fedora Silverblue, and Automotive Grade Linux. Domains where a failed update is unacceptable.

OSTree keeps the system partition immutable. Updates are atomic — they either apply completely or don't apply at all. Instant rollback if something goes wrong.

The update flow:

```
Orbit OS Store generates a .orbit file
└── Sent to device via gRPC: SystemUpdate(OtaFile)
    └── OSTree installs into internal repo
        └── Boot process is killed (takes the whole child chain with it)
            └── Boot relaunches — new version active
                └── All .orb apps restart automatically
```

This is a soft restart — the Linux base doesn't reboot, only the Gravity RT chain. Fast and safe.

**Two separate update types:**

| | `.orb` | `.orbit` |
|---|---|---|
| What | Individual app | Full runtime |
| Generated by | Developers | Orbit OS team only |
| Impact | Only that app | Full runtime restart |
| Android equivalent | APK | OS update |

Full `.orbit` update size: **21MB**.

---

## Edge AI — First-Class, Not an Add-On

TFLite is included in the OS base, compiled in C++ with direct access to ARM NEON acceleration. Any `.orb` can use AI inference without installing anything extra.

We ran the same Face Recognition `.orb` — same package, no changes — on two completely different devices.

### Raspberry Pi 4 (ARM Cortex-A72, no AI hardware)

```
Device:         Raspberry Pi 4
SoC:            Broadcom BCM2711 (Cortex-A72)
AI hardware:    None — CPU inference only

                    Idle          With live face recognition
CPU Usage:          2.3%          18.2%
RAM Used:           139 MB        224 MB
SoC Thermal:        50.2°C        57.5°C
Identification:     —             93.9% confidence
```

### Arduino UNO Q (Qualcomm QRB2210, Hexagon DSP)

```
Device:         Arduino UNO Q
SoC:            Qualcomm QRB2210 (Kryo-V2, 4 cores @ 2016 MHz)
AI hardware:    Hexagon DSP + Adreno GPU

                    With live face recognition
CPU Usage:          1.9%
RAM Used:           507 MB / 1.70 GB
SoC Thermal:        44.4°C
Uptime:             788,881s (~9 days continuous)
```

### What this means

Same `.orb`. Same Store. Two completely different hardware profiles. Dramatically different results.

On the Raspberry Pi 4, TFLite runs inference on the CPU — 18% load, temperature climbs to 57°C. It works well.

On the Arduino UNO Q, TFLite automatically delegates inference to the Qualcomm Hexagon DSP. The CPU sits at **1.9%** while the dedicated AI hardware does all the work. The SoC stays at **44.4°C** — barely warm — running face recognition continuously for 9 days straight.

The developer wrote zero hardware-specific code. The runtime figures it out.

This is what "Edge AI first-class" means in practice — not just that AI works on edge devices, but that the runtime transparently leverages whatever AI acceleration the hardware provides. ARM NEON on Cortex-A. Hexagon DSP on Qualcomm. The same `.orb` runs optimally on both.

---

## AppHub — A Browser Desktop for Your Device

Every Orbit OS device runs an **AppHub** — a web portal that aggregates the UIs of all installed apps into a single authenticated entry point.

Think of it as a desktop OS, but in the browser, running on the device itself.

Any `.orb` with a web interface registers itself with the AppHub via `RegisterWebUI(addr, route)`. From that point on, it's accessible at a dedicated path:

```
http://<device-ip>/apphub  (login)
    └── /portflux           → PortFlux traffic routing UI
    └── /relay4             → Relay Controller UI
    └── /face-recognition   → Face Recognition UI
    └── /mqtt               → MQTT Broker UI
    └── ...
```

One login. All your apps. No browser extensions, no client software, no scattered ports. Just a URL and a password.

The installed app icons appear in the AppHub just like apps on a home screen. It's the same mental model — except it runs on a Raspberry Pi and is accessible from any device on your local network.

---

## MCP Server — Your Device as an AI Agent Tool

We have an MCP Server `.orb` in the Store (5 stars). It exposes your device's full Gravity RT API as an MCP tool, which means AI agents — Claude, or any MCP-compatible client — can directly control physical hardware:

```
AI Agent (cloud)
└── MCP Protocol
    └── MCP Server .orb on device
        └── Gravity RT
            └── GPIO / Camera / Sensors / Relays / BLE / ...
```

Edge AI running locally on the device. Cloud AI orchestrating through MCP. Both at the same time.

---

## The SDK Ecosystem

All SDKs are generated from the same `.proto` files — consistent API across every language.

| Language | Status |
|---|---|
| Go | Available at launch |
| Kotlin | Available (used in our Android app) |
| Java | Beta soon |
| Python | Q3 |
| C++ | Q4 |

The **CLI tool** (ships with the Go SDK) connects to the device API directly — install packages, remove packages, stream logs. No SSH.

The **Android app** (sideload for now, Play Store soon) has two modes: browse the Store online, or connect directly to local devices via IP + credentials + mTLS.

Third-party developers can generate their own Kotlin or Swift SDKs from our `.proto` files and build custom mobile apps for their specific Orbit OS devices.

---

## Security Model

```
Production (Dev Mode OFF):    mTLS certificate + user/pass
Development (Dev Mode ON):    mTLS certificate only (SDK official cert)
```

The SDK ships with an official Orbit OS mTLS certificate. In Dev Mode, this certificate alone grants access — no password friction during development. The device explicitly informs you of the security implication when you enable it.

In production, Dev Mode is off. The extra layer comes back automatically.

Zero compromise on production security. Zero friction in development.

All `.orb` apps run as **non-privileged processes** — no root access, no direct hardware access. Every system call goes through Gravity RT, which enforces the permissions declared in the manifest. An app cannot reach anything it didn't ask for — and cannot escalate privileges at runtime.

---

## Numbers

- Installer: **25MB** `.run` (installs JVM and dependencies via apt once)
- Full OTA update: **21MB**
- Face Recognition app: **6MB** (3 TFLite models included)

**Raspberry Pi 4** (Broadcom BCM2711, no AI hardware):
- Runtime idle RAM: **139MB** / CPU: **2.3%**
- Face recognition live: **224MB RAM** / **18.2% CPU** / **57.5°C**

**Arduino UNO Q** (Qualcomm QRB2210, Hexagon DSP):
- Runtime + face recognition live: **507MB RAM** / **1.9% CPU** / **44.4°C**
- Uptime at time of measurement: **788,881s (~9 days)**

---

## What's Available Now

The **Store** is live at store.orbit-os.org with apps already available:

- IOFlow — peripheral control and testing
- Edge AI – Face Recognition
- Edge AI – Smart Image Detection
- **MCP Server** — AI agent integration (★ 5.0)
- Mochi MQTT Broker
- RPI 4-Channel Relay Controller (★ 5.0)
- WireShield VPN
- PortFlux — traffic routing
- Webcam Preview
- Serial Console

The **Go SDK and installer** are launching next month. Community Edition targets the following hardware on Debian 13:

- Raspberry Pi 3 / 4 / 5 / Zero 2W
- Arduino UNO Q (Qualcomm QRB2210)

---

## What We're Looking For

- **Developers** who want to try real-time embedded development without the Docker overhead
- **Hardware makers** interested in certifying devices for the Orbit OS ecosystem
- **Feedback** on the architecture, the package model, the security decisions

We're a team of three. We've been building this for a while. The Community Edition launches next month and we'd love early testers who aren't afraid to break things and tell us what's wrong.

Forum: forum.orbit-os.org
Website: orbit-os.org
Store: store.orbit-os.org
