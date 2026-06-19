# Add Controller

| | |
|---|---|
| **Version** | 0.1 |
| **Status** | Draft |
| **Last updated** | 2026-06-19 |
| **Owner** | Jonathan, Haven Lighting |
| **Target / scope** | Haven platform — mobile app controller provisioning (Haven Lighting 2.5, Haven Aura 3.0) |
| **Classification** | Confidential |

> Status values: `Draft` · `In Review` · `Approved` · `Deprecated`.
> Bump **Version** and add a **Revision History** row on every edit.

---

## About This Document

This document specifies the **Add Controller** feature: the end-to-end flow by which a user provisions a Haven lighting controller to a location using the mobile app. It covers all device starting states, connectivity types, app/firmware combinations under test, and the full set of test scenarios — including happy-path provisioning, re-add after network changes, re-add after firmware version changes, incomplete connection handling, Wi-Fi password auditing via mobile logs, the Wi-Fi password recommendation feature, and Ethernet-connected controllers.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [Reference: Devices, Firmware, and Apps](#3-reference-devices-firmware-and-apps)
4. [Device Starting States](#4-device-starting-states)
5. [Connectivity Types](#5-connectivity-types)
6. [Flow Overview](#6-flow-overview)
7. [Test Section 1: Add Device from Hard Server Delete State](#7-test-section-1-add-device-from-hard-server-delete-state)
8. [Test Section 2: Add Device from One-Blink (Unassigned) State](#8-test-section-2-add-device-from-one-blink-unassigned-state)
9. [Test Section 3: Re-Add Device After Wi-Fi Network Change](#9-test-section-3-re-add-device-after-wi-fi-network-change)
10. [Test Section 4: Re-Add Device After Firmware Upgrade or Downgrade](#10-test-section-4-re-add-device-after-firmware-upgrade-or-downgrade)
11. [Test Section 5: Add Device — Incomplete Connection (Device Never Goes Online)](#11-test-section-5-add-device--incomplete-connection-device-never-goes-online)
12. [Test Section 6: Mobile Log Verification — Wi-Fi Password Auditing](#12-test-section-6-mobile-log-verification--wi-fi-password-auditing)
13. [Test Section 7: Wi-Fi Password Recommendation Feature](#13-test-section-7-wi-fi-password-recommendation-feature)
14. [Test Section 8: Add Device — Ethernet-Connected Controllers](#14-test-section-8-add-device--ethernet-connected-controllers)
15. [Open Questions](#15-open-questions)
16. [Revision History](#revision-history)

---

## 1. Overview

The Add Controller flow is the process by which a user provisions a Haven lighting controller to a location using the mobile app. The controller must be discovered, assigned credentials, and confirmed online before it is considered successfully added. Several starting states and connectivity configurations must all be handled correctly.

This document captures both the feature specification and the full test plan, covering all combinations of device firmware versions, mobile app versions, platforms, device starting states, and connectivity types.

## 2. Goals & Non-Goals

**Goals**

- Successfully provision a controller to a location from any valid starting state (Hard Server Delete, One-Blink/Unassigned, Two-Blink/SSID Missing, Three-Blink/Password Changed).
- Ensure the API key is properly assigned and the device performs its global device announce HTTP call upon connection.
- Re-add a previously configured device after a Wi-Fi network change without losing existing configuration (events, scenes, schedules, groups, channel/zone assignments).
- Re-add a device after a firmware upgrade or downgrade without data loss.
- Capture the Wi-Fi password entered during provisioning in server-side mobile logs for debugging purposes.
- Surface a Wi-Fi credential recommendation when adding a second (or subsequent) device to a location where a prior device successfully connected.
- Support Ethernet-connected controllers with and without a Wi-Fi backup network configured.

**Non-Goals**

- Modifying events, scenes, schedules, groups, or channel/zone assignments as part of the re-add flow — existing configuration must be preserved, not managed.
- Account-level credential management — Wi-Fi credential recommendation is scoped per location session.
- Automatic firmware version selection during provisioning.

## 3. Reference: Devices, Firmware, and Apps

### Device Versions

| Label | Firmware | Notes |
|---|---|---|
| V6 Production | 6.02.70 (4.18) | Shipping production firmware |
| V6 Release Candidate | 6.02.73 (5.23) | Release candidate firmware |
| V7 | Current V7 build | — |

### Mobile App Versions

| Label | Platform |
|---|---|
| Haven Lighting 2.5 | iOS |
| Haven Lighting 2.5 | Android |
| Haven Aura 3.0 | iOS |
| Haven Aura 3.0 | Android |

## 4. Device Starting States

- **Hard Server Delete** — device has been factory reset via push buttons AND deleted from the server via the Blue Portal Server Delete function. Device exists in no database anywhere.
- **One-Blink (Unassigned)** — device exists in the server database but is not assigned to any location. Device is in one-blink mode, ready to receive credentials.
- **Two-Blink (SSID Missing)** — device is assigned to a location but its Wi-Fi SSID is no longer reachable.
- **Three-Blink (Password Changed)** — device is assigned to a location but its Wi-Fi password has changed.

## 5. Connectivity Types

- **Wi-Fi** — standard wireless connection.
- **Ethernet only** — no Wi-Fi configured.
- **Ethernet + Wi-Fi backup** — hard-wired with optional Wi-Fi backup credentials entered during setup.

## 6. Flow Overview

1. Place the controller into the appropriate starting state.
2. Open the mobile app and begin the Add Device flow.
3. The app discovers the controller and transmits the location assignment and network credentials.
4. The controller connects to the server via the configured network.
5. The server assigns an API key to the device record.
6. The device performs its global device announce HTTP call.
7. The mobile app confirms the device is online and the flow is complete.

For re-add scenarios (Two-Blink, Three-Blink, firmware change), all existing location data — channel/zone assignments, events, schedules, scenes, groups — must remain intact and continue to function correctly after reconnection.

---

## 7. Test Section 1: Add Device from Hard Server Delete State

**Setup:** Place the controller into a hard server delete state by performing a factory reset on the device using its push buttons, then navigating to the Blue Portal web page on the server and selecting **Server Delete** for that device. The device should no longer exist in any database. Confirm the device's LED state indicates it is ready to be provisioned. Attempt to add the device to a location using the mobile app.

### Test Cases

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

**Pass Criteria:** Device successfully added to the location. Device goes online and connects to the server. API key is properly assigned. Device performs its global device announce HTTP call upon connection.

---

## 8. Test Section 2: Add Device from One-Blink (Unassigned) State

**Setup:** Confirm that the controller exists in the server database but is not assigned to any location. This state can be reached by removing/unassigning the device from its location in the app or portal without performing a server delete. The device should be in one-blink mode, indicating it is ready to receive credentials. Attempt to add the device to a location using the mobile app.

### Test Cases

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

**Pass Criteria:** Device successfully added to the location. Device goes online and connects to the server. API key is properly assigned. Device performs its global device announce HTTP call upon connection.

---

## 9. Test Section 3: Re-Add Device After Wi-Fi Network Change

**Setup:** The controller is already added to a location with a full set of configured events, scenes, schedules, groups, and channel/zone assignments. Simulate a Wi-Fi network change by either renaming the SSID (producing a two-blink state) or changing the Wi-Fi password (producing a three-blink state). Perform the add device process from the mobile app to reconnect the controller to its existing location.

This section requires two sub-scenarios for each device/app combination.

### Sub-Scenario A: Two-Blink (SSID Missing / Network Renamed)

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

### Sub-Scenario B: Three-Blink (Password Changed)

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

**Pass Criteria:** Device successfully reconnects to its existing location. After reconnection, verify all of the following are intact and functioning correctly:

- All channel and zone assignments are unchanged — no table relationships were modified
- All events execute as configured prior to the network change
- All schedules execute as configured prior to the network change
- All scenes execute as configured prior to the network change
- All groups execute as configured prior to the network change

---

## 10. Test Section 4: Re-Add Device After Firmware Upgrade or Downgrade

**Setup:** This section targets V6 devices specifically, as residual state on the server or device may behave differently across firmware versions. Begin with a device that is already assigned to a location with a full configuration (events, scenes, schedules, groups, channel/zone assignments). Perform a firmware upgrade or downgrade, then perform the re-add device process (the device remains assigned to its location; re-add is used to reconnect it). Verify the system behaves correctly after the version change.

This section requires two sub-scenarios.

### Sub-Scenario A: Firmware Upgrade (6.02.70 → 6.02.73)

| Device | App | Platform |
|---|---|---|
| V6 upgraded to RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 upgraded to RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 upgraded to RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 upgraded to RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |

### Sub-Scenario B: Firmware Downgrade (6.02.73 → 6.02.70)

| Device | App | Platform |
|---|---|---|
| V6 downgraded to Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 downgraded to Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 downgraded to Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 downgraded to Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |

**Pass Criteria:** Device successfully re-adds to its existing location after the firmware version change. All channel/zone assignments, events, schedules, scenes, and groups remain intact and execute correctly. No table relationships were modified as a result of the firmware change and re-add.

---

## 11. Test Section 5: Add Device — Incomplete Connection (Device Never Goes Online)

**Setup:** Begin the add device process through the mobile app. Allow the app to complete its side of the provisioning flow. However, deliberately prevent the controller from successfully connecting to the server (e.g., by providing an incorrect Wi-Fi password or keeping it off the network). Observe and verify server-side database state.

### Test Cases

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

**Pass Criteria and Verification Points:**

- Verify that an API key was properly assigned to the device record on the server
- Verify the device's location assignment in the database reflects the intended state
- If the device eventually comes online later, verify it successfully connects and performs its global device announce HTTP call
- Verify: if the device comes online *before* the mobile app completes its final registration step, does the system enter a conflicted or inconsistent state? Document observed behavior.

---

## 12. Test Section 6: Mobile Log Verification — Wi-Fi Password Auditing

**Setup:** Perform an add device flow through the mobile app, entering a Wi-Fi password (correct or incorrect). After the provisioning attempt, access the server-side mobile logs to verify that the password entered by the user was captured and is readable for debugging purposes.

### Test Cases

| Device | App | Platform | Password Entry |
|---|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS | Entered correctly |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android | Entered correctly |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS | Entered correctly |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android | Entered correctly |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS | Entered incorrectly |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android | Entered incorrectly |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS | Entered incorrectly |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android | Entered incorrectly |
| V7 | Haven Aura 3.0 | iOS | Entered incorrectly |
| V7 | Haven Aura 3.0 | Android | Entered incorrectly |

**Pass Criteria:** The server's mobile log records clearly contain the exact Wi-Fi password string as typed by the user during the add device flow, including any typos or incorrect characters, providing a reliable debugging audit trail.

---

## 13. Test Section 7: Wi-Fi Password Recommendation Feature

**Setup:** Add a first controller to a location using a known, correct Wi-Fi SSID and password. Once the first device successfully connects to the server, wait at least 25 seconds. Then begin the add device process for a second controller at the same location. Verify that the app displays the recommended Wi-Fi SSID and password that was used successfully by the first device.

Extend this test to a third and fourth device added sequentially, each time waiting at least 25 seconds after the previous device successfully connects before beginning the next add device flow.

### Test Cases — First to Second Device (25-second wait)

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

### Test Cases — Third and Fourth Device Sequential Add

| Device | App | Platform | Notes |
|---|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS | 3rd and 4th device recommendation present |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android | 3rd and 4th device recommendation present |
| V7 | Haven Aura 3.0 | iOS | 3rd and 4th device recommendation present |
| V7 | Haven Aura 3.0 | Android | 3rd and 4th device recommendation present |

**Pass Criteria:**

- Server captures the SSID and password from the first device's successful connection within 25 seconds
- The recommended SSID and password appear in the app when beginning the second device add
- The recommended password string matches exactly — character for character — what was entered for the first device
- Recommendation appears consistently in both Haven Lighting 2.5 and Haven Aura 3.0 on both platforms
- Recommendation continues to appear for the third and fourth device adds in a sequential provisioning session

---

## 14. Test Section 8: Add Device — Ethernet-Connected Controllers

**Setup:** This section covers controllers that are connected via hard Ethernet rather than Wi-Fi. Two sub-scenarios must be tested: devices with no Wi-Fi backup configured, and devices where the user optionally enters a Wi-Fi backup network during the add device flow. Note that the Wi-Fi password recommendation feature does not apply to Ethernet-only devices, as no Wi-Fi credentials are entered or confirmed.

### Sub-Scenario A: Ethernet Only (No Wi-Fi Backup)

The user proceeds through the add device flow in the mobile app and explicitly declines the option to configure a Wi-Fi backup network. The device connects to the server via Ethernet only.

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

### Sub-Scenario B: Ethernet with Wi-Fi Backup Configured

The user proceeds through the add device flow in the mobile app and opts to enter a Wi-Fi backup network and password. The device connects to the server via Ethernet, with Wi-Fi backup credentials stored.

| Device | App | Platform |
|---|---|---|
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Lighting 2.5 | Android |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | iOS |
| V6 Production (6.02.70 / 4.18) | Haven Aura 3.0 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Lighting 2.5 | Android |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | iOS |
| V6 RC (6.02.73 / 5.23) | Haven Aura 3.0 | Android |
| V7 | Haven Lighting 2.5 | iOS |
| V7 | Haven Lighting 2.5 | Android |
| V7 | Haven Aura 3.0 | iOS |
| V7 | Haven Aura 3.0 | Android |

**Pass Criteria:**

- Ethernet-only device (Sub-Scenario A) successfully adds to location and connects to server
- API key is properly assigned for Ethernet-connected device
- Device performs its global device announce HTTP call upon connection
- Ethernet + Wi-Fi backup device (Sub-Scenario B) successfully adds to location with backup credentials stored
- Verify that Wi-Fi password recommendation feature is NOT triggered for Ethernet-only devices, as no confirmed Wi-Fi credentials were entered

---

## 15. Open Questions

| # | Question | Owner |
|---|---|---|
| 1 | What is the expected app UX when the device comes online before the mobile app completes its final registration step (incomplete connection race condition, §11)? | — |
| 2 | Is the 25-second credential capture window for the password recommendation feature (§13) configurable server-side, or hardcoded? | — |
| 3 | Does the Wi-Fi password recommendation feature apply when re-adding a device after a Two-Blink or Three-Blink state, or only during initial provisioning? | — |

---

## Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 2026-06-19 | Jonathan | Initial draft — converted Add Device test plan into standard feature specification template |

---

*End of Add Controller Specification v0.1*
