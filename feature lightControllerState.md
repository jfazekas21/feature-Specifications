# Light & Controller State Management

| | |
|---|---|
| **Version** | 0.2 |
| **Status** | Draft |
| **Last updated** | 2026-06-18 |
| **Owner** | Jonathan, Haven Lighting |
| **Target / scope** | Haven platform — light & controller state management |
| **Classification** | Confidential |

---

## About This Document

This document specifies the architecture for managing and querying the state of lighting controllers and lights in the Haven platform. It covers the two primary data types (light state, controller state), state inference rules, data ingestion sources, telemetry timeline design, and the background polling service.

---

## Table of Contents

1. [Overview](#1-overview)
2. [System Block Diagram](#2-system-block-diagram)
3. [Data Types](#3-data-types)
4. [State Inference Rules](#4-state-inference-rules)
5. [State Ingestion Sources](#5-state-ingestion-sources)
6. [State Confidence Model](#6-state-confidence-model)
7. [Telemetry Timeline](#7-telemetry-timeline)
8. [Background Polling Service](#8-background-polling-service)
9. [API Access Model](#9-api-access-model)
10. [Open Questions](#10-open-questions)
11. [Revision History](#revision-history)

---

## 1. Overview

The goal of this project is to rewrite how the Haven platform handles the state of lighting controllers and the lights they control. The system must:

- Track the current state of every light and every controller at any given moment.
- Record a timeline of state changes for support and diagnostic purposes.
- Know *why* a light or controller changed state — not just what the new state is.
- Accurately model online/offline status based on real signals from devices.
- Support a background service that attempts to recover recently offline controllers.

There are two distinct entities with their own independent state records:

| Entity | Description |
|---|---|
| **Light** | An individual lighting output — color, effect, brightness, on/off status, and the reason for its current state. |
| **Controller** | A network-connected hardware device that manages one or more lights — connectivity, network parameters, and sync state. |

There are two ways to access state information:

| Access Mode | Description | Audience |
|---|---|---|
| **Current State** | The latest known snapshot of a light or controller. | Customers (mobile app) |
| **Timeline** | A chronological log of state changes with timestamps and reasons. | Internal support tools only |

### Non-Goals

- Real-time streaming of state to customers (current-state snapshots only).
- Exposing the telemetry timeline to customers (internal support tooling only).
- Selecting the underlying storage technology — left to the server team (see §10).

---

## 2. System Block Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CONSUMERS                                           │
│                                                                         │
│   ┌──────────────────┐          ┌──────────────────────────────────┐   │
│   │   Mobile App     │          │  Internal Support Portal         │   │
│   │  (Current State) │          │  (Current State + Timeline)      │   │
│   └────────┬─────────┘          └───────────────┬──────────────────┘   │
└────────────┼───────────────────────────────────┼─────────────────────-─┘
             │                                   │
┌────────────┼───────────────────────────────────┼────────────────────────┐
│            │        QUERY / API LAYER           │                        │
│   Current State API                    Timeline API (internal only)      │
│   - GET /lights/{id}/state             - GET /lights/{id}/timeline       │
│   - GET /controllers/{id}/state        - GET /controllers/{id}/timeline  │
└────────────────────────────┬───────────────────────────────────────────-┘
                             │
┌────────────────────────────┴────────────────────────────────────────────┐
│                        STATE STORAGE LAYER                              │
│                                                                         │
│   ┌──────────────────────────┐      ┌──────────────────────────────┐   │
│   │   Current State Store    │      │   Telemetry Timeline Store   │   │
│   │                          │      │                              │   │
│   │  LightState record       │      │  StateChangeEvent records    │   │
│   │  ControllerState record  │      │  (24–48 hr retention window) │   │
│   │  (latest only)           │      │  (delete or archive)         │   │
│   └──────────────────────────┘      └──────────────────────────────┘   │
│                                                                         │
│   Storage technology: server team to select most cost-effective option  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────────────┐
│                     STATE INFERENCE LAYER                               │
│                                                                         │
│  Online/Offline Rules:                                                  │
│  ├─ D2C message received          → ONLINE  (immediate, authoritative)  │
│  ├─ Direct Method success         → ONLINE  (immediate, authoritative)  │
│  ├─ Direct Method timeout (30s)   → OFFLINE (immediate, authoritative)  │
│  ├─ No D2C message in 10 hours    → OFFLINE (inferred)                  │
│  └─ Offline state persists until D2C or successful Direct Method        │
│                                                                         │
│  Light State Confidence Rules:                                          │
│  ├─ Server sends command          → PENDING (unconfirmed)               │
│  ├─ D2C confirms new state        → CONFIRMED                           │
│  └─ Command times out (30s)       → FAILED + controller → OFFLINE       │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────────────┐
│                   STATE INGESTION SOURCES                               │
│                                                                         │
│  ┌───────────────────┐  ┌────────────────────┐  ┌───────────────────┐  │
│  │  D2C Messages     │  │  Direct Methods    │  │  Device Announce  │  │
│  │  (IoT Hub)        │  │  (IoT Hub)         │  │  (HTTP POST)      │  │
│  │                   │  │                    │  │                   │  │
│  │ - Light state     │  │ - Command success  │  │ - IP Address      │  │
│  │ - Change reason   │  │   → ONLINE         │  │ - ETH MAC         │  │
│  │ - Controller RSSI │  │ - Timeout (30s)    │  │ - AP info         │  │
│  │ - Connectivity    │  │   → OFFLINE        │  │ - Connection mode │  │
│  │ → Controller:     │  │ - Pending state    │  │ → Controller      │  │
│  │   ONLINE          │  │   while in-flight  │  │   State update    │  │
│  └───────────────────┘  └────────────────────┘  └───────────────────┘  │
│                                                                         │
│  Change Reason Sources:                                                 │
│  ├─ Server-originated:  Mobile App command, Sports command,             │
│  │                      Service Portal command                          │
│  │                      (reason known at server — logged directly)      │
│  └─ Controller-originated: Scheduled event execution, Push button,     │
│                             Power cycle                                 │
│                             (reason + timestamp come from controller    │
│                              in the D2C message payload)                │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
┌────────────────────────────┴────────────────────────────────────────────┐
│               BACKGROUND POLLING SERVICE                                │
│                                                                         │
│  Target: Controllers that have been offline for less than 72 hours      │
│  Method: refresh_status Direct Method                                   │
│  Rate:   3 controllers per minute (low-overhead, staggered)             │
│  Logic:                                                                 │
│  ├─ Scan controller list for offline controllers                        │
│  ├─ Skip any offline for more than 72 hours                             │
│  ├─ Send refresh_status — if success → ONLINE                           │
│  └─ If timeout → remain OFFLINE, update last-polled timestamp           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Data Types

### 3.1 LightState

Represents the current or historical state of a single light.

| Field | Type | Description |
|---|---|---|
| `lightId` | string | Unique light identifier |
| `controllerId` | string | Parent controller identifier |
| `timestamp` | datetime | UTC timestamp of this state record |
| `isOn` | bool | Whether the light is on or off |
| `brightness` | uint8 | Brightness level (0–100) |
| `onType` | enum | `solidColor` · `whiteColor` · `effect` · `pattern` · `lightShow` · `playlist` |
| `color` | object | Color value (format TBD — HSL, RGB, CCT) |
| `effectName` | string | Name of running effect, pattern, or show if applicable. Null otherwise. |
| `changeReason` | enum | `powerCycle` · `mobileAppCommand` · `sportsCommand` · `pushButton` · `scheduledEvent` |
| `changedByUser` | string | User identifier if reason is `mobileAppCommand` or service portal. Null otherwise. |
| `confidence` | enum | `pending` · `confirmed` · `failed` |

### 3.2 ControllerState

Represents the current or historical state of a single controller.

| Field | Type | Description |
|---|---|---|
| `controllerId` | string | Unique controller identifier |
| `timestamp` | datetime | UTC timestamp of this state record |
| `onlineStatus` | enum | `online` · `offline` |
| `connectionMode` | enum | `wifi` · `eth` |
| `ipAddress` | string | Current IP address |
| `wifiSSID` | string | Connected Wi-Fi network name |
| `wifiPassword` | string | Wi-Fi password (stored securely) |
| `apMacAddress` | string | Access point MAC address |
| `wifiMac` | string | Device Wi-Fi MAC address |
| `ethMac` | string | Device Ethernet MAC address |
| `wifiRssi` | int | Wi-Fi signal strength in dBm |
| `wifiNetworkScan` | object | Result of last Wi-Fi network scan |
| `localTime` | datetime | Local time reported by controller |
| `sunriseTime` | time | Sunrise time for controller's location |
| `sunsetTime` | time | Sunset time for controller's location |
| `tableSyncVersion` | string | Current table sync version |
| `tableSyncCrc` | string | CRC of current synced table |
| `lastSeenAt` | datetime | UTC timestamp of last D2C or successful Direct Method |
| `lastPolledAt` | datetime | UTC timestamp of last background poll attempt |

---

## 4. State Inference Rules

### 4.1 Controller Online/Offline

| Signal | Result | Authority |
|---|---|---|
| D2C message received | `online` | Authoritative — immediate |
| Direct Method response received | `online` | Authoritative — immediate |
| Direct Method timeout (30s) | `offline` | Authoritative — immediate |
| No D2C message in 10 hours | `offline` | Inferred |
| Background poll success | `online` | Authoritative — immediate |
| Background poll timeout | No change — remains `offline` | — |

Offline state persists until an authoritative online signal is received. There is no automatic expiry.

### 4.2 Light State Confidence

| Condition | `confidence` |
|---|---|
| Server has sent a command — no controller response yet | `pending` |
| D2C message confirms light is in expected state | `confirmed` |
| Direct Method to controller timed out (30s) | `failed` |

A `pending` light state should be displayed to the user as unconfirmed in the mobile app. A `failed` state should reflect the last known confirmed state, and the controller should be marked `offline`.

---

## 5. State Ingestion Sources

### 5.1 D2C Messages (IoT Hub — controller-originated)

- Receipt of any D2C message is an authoritative signal that the controller is `online` at that moment.
- D2C messages carry both light state and change reason in the same payload.
- Controller-originated change reasons (`powerCycle`, `scheduledEvent`, `pushButton`) and their timestamps come from the controller in the D2C payload and must be trusted as-is.

### 5.2 Direct Methods (IoT Hub — server-originated)

- A successful response confirms the controller is `online`.
- A timeout (30 seconds) confirms the controller is `offline` and marks any in-flight light state as `failed`.
- When the server sends a lighting command, the light state is immediately set to `pending` with the expected new state.

### 5.3 Device Announce (HTTP POST — controller-originated)

- Fired by the controller on boot or reconnection.
- Provides: IP address, connection mode, Wi-Fi MAC, Ethernet MAC, AP MAC, Wi-Fi SSID.
- All fields from device announce are written directly into ControllerState at the time of receipt.
- Receipt of device announce is also an authoritative `online` signal.

### 5.4 Server-Originated Change Reasons

The server knows the reason for any command it initiates. The following reasons are assigned server-side at the time of command dispatch:

| Source | `changeReason` |
|---|---|
| Mobile app user action | `mobileAppCommand` |
| Sports AI / Live Sports integration | `sportsCommand` |
| Service portal action | `mobileAppCommand` (with user/source tag) |

---

## 6. State Confidence Model

```
Server sends lighting command
         │
         ▼
  LightState.confidence = "pending"
  LightState reflects expected new values
         │
         ├─── D2C confirms new state ──────────► confidence = "confirmed"
         │
         └─── Direct Method timeout (30s) ─────► confidence = "failed"
                                                  controller → offline
                                                  light reverts to last confirmed state
```

---

## 7. Telemetry Timeline

### 7.1 Purpose

The timeline provides a chronological log of all state changes for a light or controller. It is for internal support use only and is not exposed to customers.

### 7.2 StateChangeEvent record

Each state change produces a `StateChangeEvent` record appended to the timeline store. This record contains:

- Entity type (`light` or `controller`)
- Entity ID
- UTC timestamp
- Previous state snapshot
- New state snapshot
- Change reason (for lights)
- Signal source that triggered the state change (D2C, Direct Method, device announce, background poll, server command)
- Confidence level (for lights)

### 7.3 Retention

- Default retention window: 24–48 hours.
- After the window expires: delete or archive — server team to decide based on cost.
- Storage technology: server team to select most cost-effective option (e.g. Cosmos DB TTL, Azure Table Storage, cold archive).

---

## 8. Background Polling Service

### 8.1 Purpose

A low-overhead background service that periodically attempts to contact recently offline controllers to detect recovery.

### 8.2 Rules

| Parameter | Value |
|---|---|
| Method | `refresh_status` Direct Method |
| Target | Controllers with `onlineStatus = offline` |
| Eligibility window | Offline for less than 72 hours |
| Exclusion | Controllers offline for 72 hours or more are ignored |
| Rate | 3 controllers per minute (staggered) |

### 8.3 Behavior

- On success: controller transitions to `online`. `lastPolledAt` updated.
- On timeout: controller remains `offline`. `lastPolledAt` updated. No retry until next poll cycle.
- The poll rate of 3 per minute is a starting point and may be adjusted based on fleet size and observed system load.

---

## 9. API Access Model

| Endpoint | Audience | Description |
|---|---|---|
| `GET /lights/{id}/state` | Mobile app, internal | Current LightState snapshot |
| `GET /controllers/{id}/state` | Mobile app, internal | Current ControllerState snapshot |
| `GET /lights/{id}/timeline` | Internal only | StateChangeEvent log for a light |
| `GET /controllers/{id}/timeline` | Internal only | StateChangeEvent log for a controller |

---

## 10. Open Questions

| # | Question | Owner |
|---|---|---|
| 1 | What storage technology should be used for current state and timeline? (Cosmos DB, Azure Table Storage, other?) | Server team |
| 2 | Should the timeline retention window be 24 or 48 hours? | Server team / product |
| 3 | Should expired timeline records be deleted or archived to cold storage? | Server team (cost-driven) |
| 4 | What is the exact color representation format in LightState? (RGB, HSL, CCT, or a union?) | Firmware / app team |
| 5 | Should `wifiPassword` be included in ControllerState, and if so, how is it stored securely? | Server team / security |
| 6 | What is the exact payload schema of D2C messages carrying light state and change reason? | Firmware team |
| 7 | Should the background polling service be rate-limited globally across all installations or per-installation? | Server team |

---

## Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 2026-06-18 | Jonathan | Initial draft |
| 0.2 | 2026-06-18 | Jonathan | Standardized to common spec template (metadata table, plain title, Non-Goals subsection, Revision History) |

---

*End of Light & Controller State Management Specification v0.2*