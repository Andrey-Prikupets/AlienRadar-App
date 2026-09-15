# 📡 AlienRadar — Telegram Mini App (TMA)

[![Telegram WebApp](https://img.shields.io/badge/Telegram-Mini_App-24A1DE?style=for-the-badge&logo=telegram&logoColor=white)](https://core.telegram.org/bots/webapps)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.4-3178C6?style=for-the-badge&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![Preact](https://img.shields.io/badge/Preact-10.20-673AB8?style=for-the-badge&logo=preact&logoColor=white)](https://preactjs.com/)
[![Vite](https://img.shields.io/badge/Vite-5.2-646CFF?style=for-the-badge&logo=vite&logoColor=white)](https://vitejs.dev/)
[![MQTT over WSS](https://img.shields.io/badge/MQTT-WSS_Paho-660066?style=for-the-badge&logo=eclipsemosquitto&logoColor=white)](https://www.hivemq.com/)

A sleek, ultra-responsive **Telegram Mini App (TMA)** designed for the **AlienRadar** mmWave human detection & tracking system. Built with Preact, TypeScript, and HTML5 Canvas, this app connects directly to MQTT brokers over Secure WebSockets (WSS) to render real-time radar scopes, velocity vectors, motion trails, and intrusion alarms directly inside Telegram chats on mobile and desktop.

---

## 🌟 Key Features

- **🎮 Real-Time Radar Scope**: High-framerate 120° sector canvas recreating the iconic *Alien* motion tracker aesthetic with real-time target blips, velocity direction arrows, and fading track histories (5-second trailing decay).
- **☁️ Global Cloud Connectivity**: Connects to public or private cloud brokers (e.g. HiveMQ Cloud, EMQX Cloud, AWS IoT, Mosquitto) using TLS-encrypted WebSockets (`wss://`, typically port `8884` / `8084`).
- **⚡ Dynamic On-Demand Stream Leases**: Implements a client lease protocol (`alienradar/{devId}/cmd`) that commands the ESP32 to switch to high-frequency radar streaming (10–20 Hz) only while the user has the Mini App open, conserving ESP32 CPU and cellular/Wi-Fi bandwidth.
- **🔍 Multi-Device Auto-Discovery**: Automatically listens on `alienradar/+/availability` to detect and list all online AlienRadar hardware on your broker via retained MQTT status, allowing one-tap switching between multiple sensors.
- **🔊 Web Audio Radar Audio**: Synthesizes authentic radar acoustic blips in real-time matching target distance and approach velocity directly in the browser via the Web Audio API.
- **🎨 Visual Themes**:
  - **CRT Phosphor Green**: Classic sci-fi radar phosphor glow.
  - **Cyberpunk Amber**: High-contrast amber display.
  - **Modern Blue**: Clean industrial aesthetic.
- **☁️ Telegram CloudStorage Integration**: Stores broker credentials, SSL settings, and UI preferences in `Telegram.WebApp.CloudStorage`, automatically syncing across all your devices with seamless fallback to `localStorage`.
- **🔗 Zero-Config Deep Linking**: Supports URL hash parameters (`#dev=DEVICE_ID&host=BROKER_HOST&port=PORT`) so Telegram Bots can open the Mini App pre-configured for a specific device with zero manual typing.
- **📦 Single-File Production Artifact**: Compiles into a self-contained, high-performance static bundle via `vite-plugin-singlefile`, ready for deployment to GitHub Pages, Cloudflare Pages, or Vercel.

---

## 🏗️ Architecture & Data Flow

```
┌─────────────────────────┐           ┌────────────────────────────────┐
│   AlienRadar Hardware   │           │       MQTT Cloud Broker        │
│    (ESP32 + LD2450)     │           │  (HiveMQ / EMQX / Mosquitto)   │
└───────────┬─────────────┘           └───────────────┬────────────────┘
            │                                         │
            │  1. LWT Availability (Retained):        │
            │     alienradar/{devId}/availability     │
            ├────────────────────────────────────────►│
            │                                         │
            │  2. Event-Driven Changes (Retained):    │
            │     alienradar/{devId}/presence         │
            │     alienradar/{devId}/alarm_state      │
            ├────────────────────────────────────────►│
            │                                         │
            │  3. WSS Connection (Port 8884):         │
            │                                         │◄──────┐
            │  4. Mini App publishes Lease Ping:      │       │
            │     alienradar/{devId}/cmd              │       │
            │◄────────────────────────────────────────┤       │
            │                                         │       │
            │  5. On-Demand Target Stream (2-10 Hz):  │       │
            │     alienradar/{devId}/targets          │       │
            ├────────────────────────────────────────►│       │
            │                                         │       │
            │                                         │  6. Real-Time WSS
            │                                         │     Target Stream
            │                                         ▼       │
            │                         ┌───────────────────────┴────────┐
            │                         │   AlienRadar Mini App (TMA)    │
            │                         │    (Preact + HTML5 Canvas)     │
            │                         │   Rendered inside Telegram     │
            └─────────────────────────┴────────────────────────────────┘
```

> [!TIP]
> **Zero Broker Clutter**: When no viewer has the Mini App open, the ESP32 produces **zero continuous traffic**. It only publishes retained single-word status messages when presence or alarm state changes.

---

## 📡 MQTT Protocol Specification

### 1. Availability & Discovery Topic: `alienradar/{devId}/availability`
Published with MQTT **retained** flag upon connect and via **Last Will and Testament (LWT)** upon disconnect. Mini App subscribes to wildcard `alienradar/+/availability` to discover online radars.
- Payload: `online` or `offline`

### 2. State Change Topics (Retained, Event-Driven Only)
Published **only when values change** (retained):
- `alienradar/{devId}/presence`: `detected` | `clear`
- `alienradar/{devId}/alarm_state`: `disarmed` | `armed` | `triggered`

### 3. On-Demand Target Telemetry: `alienradar/{devId}/targets`
Streamed **only while an active viewer lease is held** (or during an active alarm):
```json
{
  "ts": 124580,
  "targets": [
    {
      "id": 1,
      "x": 350,
      "y": 1820,
      "speed": -240,
      "active": true
    }
  ],
  "presence": true,
  "alarm": false
}
```
- `ts`: ESP32 uptime timestamp in ms.
- `x`: Lateral distance in mm (negative = left, positive = right).
- `y`: Radial distance in mm forward.
- `speed`: Target velocity in mm/s (negative = approaching, positive = receding).
- `presence`: `true` if any target is currently active.
- `alarm`: `true` if an active target is inside the defined intrusion zone.

### 4. Stream Lease Command: `alienradar/{devId}/cmd`
Sent periodically (every 15–20s) by the Mini App to request or extend high-frequency streaming:
```json
{
  "action": "stream",
  "lease": 30
}
```

---

## 🚀 Quick Start (Local Development)

### Prerequisites
- Node.js 18+ or 20+
- `pnpm` (recommended), `npm`, or `corepack`

### Setup
```bash
# Clone repository
git clone https://github.com/Andrey-Prikupets/AlienRadar-App.git
cd AlienRadar-App

# Install dependencies
pnpm install

# Start local Vite development server
pnpm dev
```
Open `http://localhost:5173` in your browser.

---

## 🛠️ Building for Production

To create the optimized, self-contained single-file bundle:

```bash
pnpm build
```
The output will be generated at `dist/index.html`.

On Windows, you can also run the root automation script:
```cmd
build-miniapp.cmd
```

---

## 🤖 Deploying as a Telegram Mini App

### Step 1: Host the Static Application
Deploy the contents of `dist/` to any HTTPS static host:
- **GitHub Pages**: Push `dist/index.html` to a `gh-pages` branch.
- **Cloudflare Pages / Vercel**: Connect repository, set build directory to `dist`.

> [!IMPORTANT]
> Telegram Mini Apps **must** be served over HTTPS.

### Step 2: Register with BotFather
1. Open [@BotFather](https://t.me/BotFather) in Telegram.
2. Send `/newapp` and select your Telegram bot.
3. Provide an App title, description, and preview image.
4. Set the **Web App URL** to your hosted URL (e.g., `https://your-domain.com/index.html`).
5. Choose a short name (e.g. `radar`). Your Mini App link will be `t.me/YourBot/radar`.

### Step 3: Deep Linking with Pre-Configured Settings
You can generate direct launch buttons from your bot using URL hash parameters to bypass manual user configuration:
```
https://your-domain.com/index.html#dev=F91978&host=xyz123.s1.eu.hivemq.cloud&port=8884&ssl=1
```
When tapped, the Mini App will automatically configure the target broker, port, and device ID, establishing an immediate radar connection.

---

## ⚙️ Configuration Settings

Inside the Mini App, tap the **Settings (Gear) icon** to configure:

| Field | Description | Default |
|---|---|---|
| **Broker Host** | Secure WebSocket hostname of your MQTT broker | — |
| **Port** | WSS Port (`8884` for HiveMQ Cloud, `8084` for EMQX) | `8884` |
| **Device ID** | ESP32 MAC/Hardware ID (e.g. `F91978`) | — |
| **Username** | MQTT client username | — |
| **Password** | MQTT client password | — |
| **Use SSL/TLS** | Connect via `wss://` (required for Telegram Mini Apps) | `true` |
| **Theme** | Color scheme (`CRT Green`, `Cyberpunk Amber`, `Consumer Blue`) | `CRT Green` |
| **Mute Sound** | Toggle client-side acoustic radar blips | `false` |

---

## 🔒 Security & Privacy

- All MQTT credentials configured manually are stored **exclusively on the client side** (encrypted in Telegram CloudStorage or browser `localStorage`).
- All communication with the broker occurs over **WSS (TLS 1.2/1.3)**.
- No analytics, external trackers, or telemetry scripts are included.

---

## 📄 License

Distributed under the MIT License. See `LICENSE` for more information.
