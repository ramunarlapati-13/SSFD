# FFSD - Fire Fighter Safety Device

<p align="center">
  <strong>Real-time firefighter vitals, geofence intelligence, incident replay, and recovery support.</strong><br/>
  Multi-unit operational dashboard built with React Native + Expo + Firebase Realtime Database.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Expo-SDK%2055-000020?logo=expo&logoColor=white" />
  <img src="https://img.shields.io/badge/React%20Native-0.83-61DAFB?logo=react&logoColor=white" />
  <img src="https://img.shields.io/badge/Firebase-Realtime%20DB-FFCA28?logo=firebase&logoColor=black" />
  <img src="https://img.shields.io/badge/TypeScript-5.9-3178C6?logo=typescript&logoColor=white" />
</p>

---

## Project Overview

FFSD (Fire Fighter Safety Device) is an end-to-end telemetry and monitoring application for multi-unit field fire operations. It provides incident commanders with live response tracking, real-time vitals diagnostics, and recovery tools.

The system connects field-wearable devices (e.g. ESP32/Arduino streaming sensors like DHT11, MPU6050, GPS) with a React Native mobile dashboard via Firebase Realtime Database. The application ensures robust tracking and fast alert response, even in low-connectivity situations.

---

## Core Capabilities

| Category | Capability | Description |
|---|---|---|
| **Fleet Monitoring** | Multi-unit fleet tracking | Simultaneously track multiple units (e.g., `firefighter_01`, `firefighter_02`) with real-time status summary cards, quick selection, and active filters. |
| | Vitals & telemetry panels | Displays live temperature, relative humidity, environmental gas concentration (PPM), and movement state (MOVING / STILL). |
| | Status mapping | Automatically translates incoming `device_state` fields into operational classifications: `NORMAL`, `WARNING`, `EMERGENCY`, `SOS`, or `OFFLINE`. |
| **Resilience & Offline** | Dual-channel sync | Combines Firebase Realtime listeners (`onValue`) with a 1-second active polling fallback (`get`) for robust state synchronization during poor cellular network coverage. |
| | Freshness detection | Automatically marks units as `OFFLINE` if no heartbeat timestamp updates are received within 60 seconds. Supports both millisecond and second-based timestamps. |
| | Offline map style | Switches map styles dynamically to a styling package containing zero external HTTP tile requirements when cellular network connectivity fails. |
| **Alarms & Alerts** | Critical alert handler | Generates interactive critical alarm modals, loops device vibration patterns, and triggers a continuous audio alarm on `EMERGENCY`, `SOS`, or fall conditions. |
| | Auto-dismissal | Automatically clears active modal alarms and vibrations if the unit's telemetry normalizes (state returns to `NORMAL` and fall is cleared). |
| | Cooldown protection | Employs a per-unit, per-alert type cooldown (12 seconds) to prevent sound and notification spam during data jitter. |
| **Geofencing** | Dynamic boundaries | Pulls safe-zone limits and hazard-zone boundaries from Firebase (`config/geofence_zones`), supporting safe-zone breaches and danger-zone intrusion alerts. |
| | Fallback zones | Automatically loads hardcoded local geofence defaults (Command Safe Zone, Heat Pockets, Radiation Pockets) if database configurations are unavailable or invalid. |
| **Analysis & Recovery**| Incident logging | Persists periodic snapshots to the database (`incident_history/{deviceId}/{timestamp}`) every 15 seconds for historical auditing and debriefing. |
| | Interactive replay | Includes an incident playback timeline with 1h/3h/6h time window filters, 0.5x to 4x speed scaling, scrubber sliders, and step-by-step frame controls. |
| | Breadcrumb backtracking | Draws interactive trail history breadcrumbs (depths of last 20, 50, or 100 points) to trace firefighter movement paths for rescue operations. |
| **User Interface** | Interactive map | Embeds a MapLibre GL WebView with vector mapping, 3D building extrusions, satellite imagery overlay toggles, and centering controls. |
| | Navigation routing | Provides a shortcut to instantly open native iOS Maps or Google Maps on Android to navigate directly to the selected firefighter's coordinates. |
| | Runtime themes | Supports toggleable dark/light themes for optimal visibility across high-sunlight or low-light operational environments. |

---

## Screenshots

<p align="center">
  <img src="./docs/screenshots/normal_dashboard.jpg" width="280" alt="Normal Dashboard" />
  <img src="./docs/screenshots/warning_state.jpg" width="280" alt="Warning Dashboard" />
  <img src="./docs/screenshots/critical_alert.jpg" width="280" alt="Critical Alert Modal" />
</p>

---

## System Architecture

```text
       ┌──────────────────────────────┐
       │     Wearable IoT Nodes       │ (Streams: Temp, Humidity, Gas PPM,
       │   (firefighter_01..N)        │  GPS Coords, Movement, Status, Fall)
       └──────────────┬───────────────┘
                      │  Publish (Wi-Fi / Cellular)
                      ▼
       ┌──────────────────────────────┐
       │  Firebase Realtime Database  │
       │                              │
       │   ├─ firefighter_xx/         │ <-- Live telemetry
       │   ├─ incident_history/       │ <-- 15s interval snapshots
       │   └─ config/geofence_zones   │ <-- Geofence configurations
       └──────────────┬───────────────┘
                      │  Sync & Poll
                      ▼
       ┌──────────────────────────────┐
       │     FFSD Mobile App          │ (React Native + Expo)
       │                              │
       │   ├─ Live Dashboard Vitals   │ <-- Status breakdown cards
       │   ├─ MapLibre GL WebView     │ <-- 3D trails, geofences & offline maps
       │   ├─ Critical Alarm / Audio  │ <-- Loop alarms & vibration
       │   └─ Historical Replay Mode  │ <-- 1h/3h/6h playback scrubber
       └──────────────────────────────┘
```

---

## Firebase Data Schema

### 1. Live Telemetry
Under root `/{deviceId}`:
```json
{
  "device_state": "NORMAL",
  "temperature": 32.5,
  "humidity": 45.0,
  "gas_ppm": 25,
  "fall_detected": false,
  "movement": "MOVING",
  "gps": {
    "lat": 16.508948,
    "lng": 80.658042
  },
  "gps_status": "OK",
  "dht_status": "OK",
  "ts": 1711700000000
}
```
*Supported Heartbeat Freshness keys: `ts`, `timestamp`, `lastUpdated`, `last_update`, `updatedAt`.*

### 2. Geofence Configuration
Under root `/config/geofence_zones`:
```json
{
  "zone_1": {
    "id": "safe-zone-1",
    "name": "Command Safe Zone",
    "type": "SAFE",
    "center": {
      "lat": 16.508948,
      "lng": 80.658042
    },
    "radiusMeters": 500
  },
  "zone_2": {
    "id": "danger-zone-1",
    "name": "Radiation Pocket B",
    "type": "DANGER",
    "center": {
      "lat": 16.5064,
      "lng": 80.6544
    },
    "radiusMeters": 120
  }
}
```

### 3. Incident History Snaps
Under root `/incident_history/{deviceId}/{timestamp}`:
```json
{
  "ts": 1711700000000,
  "lat": 16.508948,
  "lng": 80.658042,
  "temperature": 32.5,
  "humidity": 45.0,
  "gas": 25,
  "falling": false,
  "movement": "MOVING",
  "status": "NORMAL"
}
```

---

## Project Structure

```text
.
├── App.tsx                     # App entry point, mounts status bar & Dashboard
├── index.ts                    # Expo root component registration
├── app.json                    # Expo config (bundle identifiers, plugin links)
├── package.json                # Project dependencies and script declarations
├── tsconfig.json               # TypeScript compiler options
├── src/
│   ├── components/
│   │   ├── AlarmPlayer.tsx     # Webview-based buzzer audio player (expo-av alternative)
│   │   ├── AnalyticsPanel.tsx  # Telemetry charts, health states, and status counters
│   │   └── MapWrapper.tsx      # MapLibre GL viewer, overlays, and routing actions
│   ├── lib/
│   │   ├── firebase.ts         # Firebase client setup & env checks
│   │   └── types.ts            # TypeScript interfaces (DeviceData, GeofenceZone, etc.)
│   └── screens/
│       └── Dashboard.tsx       # Core screen orchestrating polling, geofences, and alarms
├── assets/                     # App icons and local assets
└── docs/screenshots/           # Markdown image previews
```

---

## Tech Stack

- **Mobile Framework**: React Native (Expo SDK 55)
- **Language**: TypeScript
- **Backend/Real-time DB**: Firebase Database SDK
- **Mapping**: WebView rendering MapLibre GL 3.6.2 (via OpenFreeMap and ArcGIS tiles)
- **Icons**: Lucide React Native
- **Buzzer Audio**: WebView audio injection (MP3 streamer)
- **Device Utilities**: React Native Vibration API

---

## Getting Started

### 1. Prerequisites
- Node.js 18+
- Expo Go application installed on iOS or Android test device

### 2. Installation
Clone the repository, navigate to the React Native project directory, and install dependencies:
```bash
git clone https://github.com/Rsmk27/firefighter-monitoring-device.git
cd firefighter-monitoring-device/FFSD
npm install --legacy-peer-deps
```

### 3. Configure Environment Variables
Create a `.env` file in the project root:
```env
EXPO_PUBLIC_FIREBASE_API_KEY=your_firebase_api_key
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=your_project_id.firebaseapp.com
EXPO_PUBLIC_FIREBASE_PROJECT_ID=your_project_id
EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=your_project_id.firebasestorage.app
EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=your_messaging_sender_id
EXPO_PUBLIC_FIREBASE_APP_ID=your_app_id
EXPO_PUBLIC_FIREBASE_DATABASE_URL=https://your_project_id-default-rtdb.REGION.firebasedatabase.app
```

### 4. Running the Development Server
Start the Expo packager:
```bash
npx expo start
```
Scan the output QR code with **Expo Go** to open the live dashboard.

---

## Reliability & Safety Guardrails

- **Network Resilience**: Incorporates immediate listener streams backed by periodic HTTP GET requests to keep dashboards fresh even if WebSockets are dropped.
- **Fail-safe Geofences**: Guarantees geofence coverage by loading predefined safety coordinates if Firebase DB connections are lost.
- **Timestamp Protection**: Automatically sanitizes timestamp anomalies (e.g. scales seconds-based epoch inputs up to milliseconds-based inputs).
- **Alarm Cooldowns**: Protects coordinators from auditory fatigue by rate-limiting notifications.
- **Security Check**: Enforces environment validations on initialization. Fails immediately and details missing configurations rather than operating silently on fallback keys.

---

## Development Team

1. **R.S. Manikanta**
   - GitHub: [Rsmk27](https://github.com/Rsmk27)
   - LinkedIn: [Srinivasa Manikanta](https://www.linkedin.com/in/srinivasamanikanta/)
2. **G. Sairam**
   - GitHub: [sairamgalam017](https://github.com/sairamgalam017)
   - LinkedIn: [Sairam Galam](https://www.linkedin.com/in/sairam-galam/)
3. **J. Santhosh**
   - GitHub: [chintu-boltey](https://github.com/chintu-boltey)
   - LinkedIn: [Santhosh Juvvanapudi](https://www.linkedin.com/in/santhosh-juvvanapudi-07a871373/)
4. **N. Ramu**
   - GitHub: [ramunarlapati-13](https://github.com/ramunarlapati-13)
   - LinkedIn: [Ramu Narlapati](https://www.linkedin.com/in/ramunarlapati/)

---

## License

Copyright (c) 2026 Power Pulse Team.  
Licensed under the MIT License.
