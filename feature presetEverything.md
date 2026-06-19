# Presets

| | |
|---|---|
| **Version** | 0.1 |
| **Status** | Draft |
| **Last updated** | 2026-06-19 |
| **Owner** | Jonathan, Haven Lighting |
| **Target / scope** | Haven platform — preset scenes, schedules & Store distribution |
| **Classification** | Confidential |

> Status values: `Draft` · `In Review` · `Approved` · `Deprecated`.
> Bump **Version** and add a **Revision History** row on every edit.

---

## About This Document

This document specifies the **Presets** feature: pre-built scenes and schedules that require zero manual configuration, automatically adapt to a location's current lighting devices, and are distributed through the existing Store. It covers the core concepts, the two execution paths, default schedule auto-creation, the self-healing model, customization rules, the actions-table lifecycle, priority resolution, the Store subscription flow, Gameday, synchronization timing, and the consolidated test scenarios.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [Core Concepts](#3-core-concepts)
4. [Execution Model](#4-execution-model)
5. [Default / Automatic Schedules](#5-default--automatic-schedules)
6. [Self-Healing](#6-self-healing)
7. [Customization](#7-customization)
8. [Scene–Schedule Relationship (Actions Table Lifecycle)](#8-sceneschedule-relationship-actions-table-lifecycle)
9. [Priority Assignment for Overlapping Schedules](#9-priority-assignment-for-overlapping-schedules)
10. [Store Subscription Flow](#10-store-subscription-flow)
11. [Gameday as a Preset Category](#11-gameday-as-a-preset-category)
12. [Synchronization Timing](#12-synchronization-timing)
13. [Consolidated Test Scenarios](#13-consolidated-test-scenarios)
14. [Resolved Decisions Log](#14-resolved-decisions-log)
15. [Open Questions](#15-open-questions)
16. [Revision History](#revision-history)

---

## 1. Overview

Customers using Haven lighting controllers historically had to manually build a **scene** — a saved set of per-light color/state assignments with one execute method — for every event they wanted to light up for: Christmas, Diwali, Thanksgiving, New Year, birthdays, Valentine's Day, and more. This setup burden repeats for every holiday and event, every year, and does not adapt when the customer's physical lighting setup changes.

**Presets** solves this by giving customers pre-built scenes and schedules that:

- Require zero manual configuration to use.
- Automatically adapt to whatever lights, controllers, and zones currently exist at a location.
- Extend beyond holidays into Gameday (team-color event lighting) and the existing effects Store.

## 2. Goals & Non-Goals

**Goals**

- Provide Haven-authored preset scenes and schedules that work with no manual setup.
- Keep schedule-linked scenes synchronized to the current device population of a location (self-healing), fixing the current defect where newly added lights/zones are silently ignored.
- Distribute presets through the existing Store, subscribable per location — including locations with zero controllers.
- Auto-create baseline on/off schedules at location-creation time.
- Maintain a normalized controller-side actions table that always contains exactly what the current schedule table references.

**Non-Goals**

- Manual-vs-scheduled execution conflict resolution — existing controller logic already handles overlapping schedule ranges (§9).
- Actions-table capacity and flash management — handled by the existing table-sync implementation.
- Migration/versioning of the existing installed base.
- Account-level subscription fan-out — everything is scoped per location (§10).

## 3. Core Concepts

### 3.1 Scene

A named set of per-light color/state assignments with one execute method. Two kinds:

- **Preset scene** — provided by Haven (e.g. "Christmas," "Fourth of July," "Boy Birthday Party"). Lives in the same scene list as customer scenes and has its own execute method. Does not compete with or override customer-created scenes of the same name/topic — the customer can have both.
- **Custom / customized scene** — either fully customer-created, or a customer-edited copy of a preset scene saved under a **new scene ID** with no reference back to the original preset.

### 3.2 Schedule (a.k.a. Event)

A row that ties a date/time range to a scene ID. Legacy name: "event" — both terms refer to the same entity; "schedule" is the current terminology.

- A schedule is just a **lookup / pointer**: date range + scene ID reference.
- Created once (manually, or auto-generated per §5). After creation it is **not** maintained or auto-adjusted — the customer can edit or delete it freely, and it is not treated as special once it exists.

### 3.3 Store

Existing concept currently used for lighting **effects**. Extends to cover **scenes** and **schedules**:

- Customer can browse and subscribe to scenes/schedules from the Store **at onboarding, before adding any controller**.
- Subscribing creates the corresponding rows for the customer immediately (see §10).

## 4. Execution Model

There are two fundamentally different execution paths. This distinction is the single most important thing to test correctly.

### 4.1 Server-Side, On-Demand Execution (NOT saved locally)

Applies to:

- Store effect execution.
- **Preset scene execution** — when a customer taps "execute" on a preset scene directly, not via a schedule.

Behavior:

- The preset scene library is just a list of names (Fourth of July, Memorial Day, Christmas, etc.) — no persisted command set.
- On execute, the **server generates an array of raw lighting commands on demand** (e.g. "light A → red," "light B → blue") and sends them directly to the controller.
- These are **not** "execute scene ID X" commands — they are fully resolved, explicit lighting commands.
- Requires the controller to be **online**.
- Nothing is written to the controller's local storage as a result of this execution path.

### 4.2 Controller-Side, Locally-Stored Execution (synced to flash)

Applies to:

- **Preset schedules** (and any scene linked to a schedule).

Behavior:

- When a customer subscribes to a schedule (e.g. the "Christmas" schedule), the schedule row **and** the full resolved scene definition (all lighting actions) are synced down and persisted in the controller's local flash database, in a table called the **actions table**.
- The actions table has a column for scene ID, so the controller can look up exactly what to do.
- The controller has a real-time clock and evaluates the schedule table **every minute**, independent of internet connectivity.
- On a schedule trigger: the controller looks up the schedule table → finds the scene ID → looks up the actions table → executes the stored lighting actions locally.
- This path **must work fully offline.**

### 4.3 Test Distinction (explicit)

| Action | Path | Expected controller behavior |
|---|---|---|
| Execute preset scene "Christmas" from the scene list in-app | Server-side, on-demand | Controller receives raw ON commands (explicit per-light colors); requires online controller |
| Manually execute a schedule from the Schedules/Events page | Controller-side, local | Controller resolves scene ID internally from its own actions table and executes — works offline |

## 5. Default / Automatic Schedules

Every new location automatically gets baseline schedules created at location-creation time, **before any controller is added** (hardcoded, server-side logic):

| Trigger | Action | Points to |
|---|---|---|
| Sunset | ON | Default "on" preset scene |
| Midnight | OFF | Default "off" preset scene |
| Sunrise | OFF | **Same** "off" preset scene as midnight |

Notes:

- Multiple schedules can point to the same scene ID — this is explicitly fine and expected (shared scene IDs across schedules).
- This requires no customer setup and no controller to exist yet.

## 6. Self-Healing

This is a deliberately **asymmetric** rule. Self-healing applies to **scenes** (preset and customized), but **not** to schedules.

### 6.1 Preset Scenes — Self-Healing: YES

- Preset scenes are defined against the **current device population** of a location, not a fixed snapshot.
- When a light/controller/zone is **added**, all preset scenes **that are linked to a schedule** at that location are automatically updated to include it — no customer action required.
- When a light/controller/zone is **removed**, those same preset scenes automatically reflow to exclude it.
- **Scope of healing:** Only preset scenes tied to a schedule need to be kept synchronized to the controller's actions table. (Preset scenes not linked to any schedule are resolved on-demand server-side per §4.1 and don't need local sync.)
- **Timing:** Healing/synchronization is triggered **at the time of the device change event**, not deferred until the scene is next executed — because the controller may be offline at execution time and the schedule must still fire correctly using current data. Sync happens "as soon as possible" after the device change.
- **Normalization:** If multiple schedules point to the same scene ID, the scene is updated once (normalized), not once per schedule. All schedules referencing that scene ID benefit immediately once the actions table is updated.

### 6.2 Customer-Customized Scenes — Self-Healing: YES

- Customized scenes (§7.1 copies with a new scene ID) **also self-heal**, same as preset scenes.
- **Current behavior being replaced:** today, newly added lights/zones are silently ignored by existing customer-created scenes. This is the specific defect Presets must fix — for both preset scenes and customer-customized scenes.
- **New requirement:** when a light/zone is added to a location, server-side logic must automatically incorporate that device into customer-created/customized scenes that are linked to a schedule, and must intelligently assign it an initial color/state rather than leaving it unconfigured.
- **Color/state inference logic by device series** (best-effort, not perfect — any reasonable default is an improvement over the current "ignore it" behavior):

| Series | Inference rule |
|---|---|
| **X-Series** | New light mirrors/matches whatever the existing X-Series lights in that scene are doing. |
| **Q-Series** | New light matches the color of other lights in the scene (cross-series color matching). |
| **L-Series** | Best-effort guess; if no clear inference is available, default to the **first color** already present in the scene. |

This inference logic applies anywhere a scene needs to absorb a newly added device — i.e., it is the general mechanism behind "self-healing," not a separate feature.

### 6.3 Preset Schedules — Self-Healing: NO

- Once a preset schedule is created (via Store subscription or default auto-creation), it is a static, standalone row.
- It is **not** automatically updated when devices are added or removed.
- The customer can manually edit any field on it at any time: name, date range, and **which scene it points to.**
- The system does not monitor or treat these rows as special after creation.

## 7. Customization

### 7.1 Customizing a Preset Scene

- Customer can edit a preset scene or save a copy of it.
- Saving a customized copy generates a **brand-new scene ID** with **no reference back** to the original preset scene ID.
- Customized scenes are only persisted to the actions table **if and only if** they are tied to an event/schedule.

### 7.2 Customizing a Preset Schedule

- There is no separate "customized copy" concept for schedules.
- Editing a preset schedule **directly modifies that schedule row** (name, date range, linked scene ID, etc.) — there is no fork/copy behavior.
- The customer can delete or modify it freely; the system does not treat it specially compared to any other schedule.

## 8. Scene–Schedule Relationship (Actions Table Lifecycle)

| Scenario | Behavior |
|---|---|
| Customer edits a schedule to point to a different scene | Tables re-sync (debounced ~4–5 min), actions table updated |
| Old scene no longer referenced by any schedule after the change | Old scene **may be removed** from the actions table during that sync |
| Customer deletes a schedule | The scene it pointed to **is deleted from the actions table** |
| …unless another schedule still references that same scene ID | Then the scene is **retained** (not deleted) — actions table is purged/rebuilt to contain exactly what's needed by the remaining schedule table; no orphans, no premature deletes |
| Two schedules point to the same scene ID | Fully supported — shared scene IDs across many schedules is expected and fine |
| General principle | The actions table should always be purged/normalized to contain only what's referenced by current schedule table rows — never more, never less |

**Schedule edit UI note:** On the schedule edit form, the primary field is "which scene do you want to execute?" — the customer can pick any preset scene or any of their own custom scenes from this dropdown.

## 9. Priority Assignment for Overlapping Schedules

- Customers can stack overlapping default categories (e.g. "Everyday On," "Winter On," "Christmas") without manually setting priority.
- **Priority is hardcoded by category, not inferred from date-range logic:**
  - Christmas overrides Winter.
  - Winter overrides Everyday (default/fallback, plain white).
  - i.e., more specific/seasonal/holiday categories always outrank the general default.
- **Implementation:** the schedule table includes a **priority column**. Conflict resolution at runtime (when multiple schedules are simultaneously "in range") is **already handled by existing controller logic** based on this column — not a new component to build.

## 10. Store Subscription Flow

- **Scoping:** Subscriptions are per-location. All resulting scene and schedule rows link directly to a location ID — there is no account-level subscription that fans out across a customer's multiple locations.
- Natural onboarding flow: the customer is prompted to select Store scenes/schedules **after** their first add-device/add-controller flow.
- However, subscribing must also be possible on a **location with zero controllers** — important for making location creation itself frictionless.
- Subscribing to a preset schedule (e.g. "Fourth of July") causes the **server to create that row** in the schedule table for the customer immediately upon subscription — not deferred to the next relevant date/season.
- Default on/off schedules (§5) are auto-created the same way, without the customer needing to go through the Store at all.

## 11. Gameday as a Preset Category

- **Previously:** the customer selects a favorite team, then manually builds scenes/effects for each game event (game started, scoring play, etc.) — the same heavy-setup problem as holidays.
- **With Presets:** the customer selects a favorite team and the system **auto-creates and assigns all event scenes** (scoring, game start, etc.) using that team's colors, with no manual scene-building.
- Gameday becomes another **preset content category**, following the same Store-distribution and (where applicable) self-healing rules as holiday presets.

## 12. Synchronization Timing

| Event | Sync behavior |
|---|---|
| Edit that changes a schedule's linked scene | Synchronized to controller tables with a **debounce of ~4–5 minutes**, not instantaneous |
| Device-driven self-healing update to a scene | Synchronized **as soon as possible** after the device change event (not debounced the same way) — because schedules must fire correctly even while the controller is offline, so the authoritative data must be current before that risk window |
| Controller schedule evaluation loop | Runs **every minute**, checking the schedule table for anything due, then jumping to the actions table to resolve and execute |

## 13. Consolidated Test Scenarios

1. **Execute preset scene directly (online controller)** — expect raw, explicit lighting ON commands sent from the server; nothing written to the local actions table as a result.
2. **Manually execute a schedule from the Schedules page** — expect the controller to resolve the scene ID from its own local actions table and execute; verify this works with the controller offline from the cloud (but locally clocked).
3. **New location created, no controller yet** — verify sunset-ON, midnight-OFF, sunrise-OFF schedules are auto-created; verify midnight-OFF and sunrise-OFF reference the identical scene ID.
4. **Add a light to a location with an existing schedule-linked preset scene** — verify the scene auto-updates to include the new light, the actions-table sync happens immediately (not deferred to next execution), and this occurs without any customer action.
5. **Remove a light/controller from a location** — verify linked preset scenes drop the reference to the removed device immediately, and the actions table reflects this without customer action.
6. **Add/remove a device where the affected preset scene is NOT linked to any schedule** — verify no actions-table sync occurs (only schedule-linked scenes need local sync).
6a. **Add a light to a location with an existing schedule-linked customer-created/customized scene** — verify the new light is automatically added to that scene (not ignored, per the previously broken behavior) and assigned a color/state per the §6.2 inference rules.
6b. **Add an X-Series light to a scene already containing X-Series lights** — verify the new light's assigned state matches the existing X-Series lights' state.
6c. **Add a Q-Series light to a scene** — verify the new light's color matches the color already used by other lights in the scene.
6d. **Add an L-Series light to a scene with no clear inference signal** — verify the new light defaults to the first color already present in the scene.
7. **Customer edits a preset schedule's date range or name** — verify no auto-adjustment occurs afterward; verify the edit persists and the system treats it as an ordinary row going forward.
8. **Customer edits a preset schedule to point to a different scene** — verify the ~4–5 min debounced sync occurs; verify the old scene is removed from the actions table if and only if no other schedule references it.
9. **Two schedules reference the same scene ID; delete one schedule** — verify the scene is NOT deleted from the actions table (still referenced by the remaining schedule).
10. **Delete the last schedule referencing a given scene ID** — verify the scene IS deleted from the actions table.
11. **Customer customizes (saves a copy of) a preset scene** — verify a new scene ID is generated with no link back to the original preset; verify it is only persisted to the actions table if attached to a schedule.
12. **Overlapping default schedules (Everyday/Winter/Christmas) all active on the same date** — verify hardcoded priority resolves correctly: Christmas > Winter > Everyday.
13. **Store subscription at onboarding, before any controller exists** — verify schedule/scene rows are created server-side immediately upon subscription, independent of device presence.
14. **Store subscription after first controller add** — verify the same immediate row-creation behavior.
15. **Gameday team selection** — verify all required event scenes (game start, scoring, etc.) are auto-created and correctly color-mapped to the selected team, without manual scene building.
16. **Multiple schedules sharing one scene ID, then a device is added** — verify the scene is updated once (normalized) and all referencing schedules immediately benefit from the updated scene without per-schedule duplication of work.
17. **Customer picks a scene from the schedule-edit dropdown** — verify both preset scenes and customer custom scenes are selectable as valid targets.
18. **All devices/lights removed from a location** — verify the affected scene's action set becomes empty (no-op) but the scene **remains visible** in the mobile app's scene list; verify the linked schedule does not error out when it fires with nothing to execute.

## 14. Resolved Decisions Log

| Decision | Resolution |
|---|---|
| Customized scene self-healing | Customer-customized scenes self-heal identically to preset scenes, using the §6.2 color-inference rules by device series. This also fixes the current defect where newly added lights/zones are silently ignored by existing customer scenes. |
| All devices removed from a location | The scene becomes a no-op action set (no lighting commands to execute) but **remains visible in the mobile app** — it is not hidden or deleted just because the location has zero devices. |
| Scoping model | Everything is scoped **per location**, not per account. All scene and schedule tables link directly to a location ID. (No account-level fan-out — a multi-location customer subscribes per location.) |
| Manual vs. scheduled execution conflict | Out of scope / no new work needed — existing controller logic already handles overlapping schedule ranges correctly. |
| Actions table capacity | Out of scope / no new work needed — table sync and flash capacity are already handled by the existing implementation; not a constraint for Presets. |
| Existing installed base migration/versioning | Out of scope / no new work needed for this spec. |

## 15. Open Questions

| # | Question | Owner |
|---|---|---|
| — | None outstanding. All previously open items have been resolved or explicitly marked out of scope (see §14). | — |

---

## Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 2026-06-19 | Jonathan | Initial draft — incorporated Presets requirements & test specification into standard template |

---

*End of Presets Specification v0.1*
