# Haven Game Day Technology

| | |
|---|---|
| **Version** | 0.6 |
| **Status** | Draft |
| **Last updated** | 2026-06-30 |
| **Owner** | Jonathan, Haven Lighting |
| **Target / scope** | 9 Series, 9 Series Pro, Q Series, Q Series Pro, X Series (full-color outdoor) |
| **Classification** | Internal |

> Status values: `Draft` · `In Review` · `Approved` · `Deprecated`.
> Bump **Version** and add a **Revision History** row on every edit.

---

*Haven Game Day Technology brings your team's colors to life — automatically. Built into every Haven full-color outdoor lighting system, this Sports AI feature connects your favorite teams directly to your lights and responds to live game events in real time, so you never have to manually set a scene on game day again.*

## 1. Overview

Game Day Technology is a free, built-in feature included with all Haven full-color outdoor lighting systems: **9 Series**, **9 Series Pro**, **Q Series**, **Q Series Pro**, and **X Series**. Once a user links their favorite professional or college sports teams in the Haven app, the system takes over — automatically triggering lighting scenes, light shows, and effects before, during, and after every game.

Setup is straightforward: select your teams, and Haven handles the rest. Every linked team is pre-populated with a complete set of default scenes and reactions (see §7), so the feature works out of the box and can be customized at any time. Behavior across head-to-head matchups is governed by the location's **Play Mode** (§3).

## 2. Goals & Non-Goals

- **Goals** — automatically trigger team-colored lighting scenes (plus Pro light shows and X Series effects) around live game events, with zero manual setup once teams are linked.
- **Non-Goals** — manual scene authoring; sports not covered by the Sports AI data source; indoor or non-full-color product lines.

---

## 3. Play Mode

**Play Mode** is a single per-location setting that determines how Game Day behaves when a linked team's matchup involves a second team. It replaces the former boolean **ENABLE RIVALRY MODE** toggle. It is a single-select picker with three options; the default is **Normal Mode**.

| Mode | Engages lead-following when… | Opponent scenes/reactions sourced from |
|---|---|---|
| **Normal Mode** | Never | n/a — each linked team runs independently |
| **Rivalry Mode** | Two of the user's linked teams play **each other** | User configuration (both teams must be linked) |
| **Beast Mode** | A linked team plays **anyone** | User config for the linked team; **server defaults (§7)** for the unlinked opponent |

Beast Mode's trigger condition is a strict superset of Rivalry Mode's: Rivalry requires *both* teams linked; Beast requires only *one*.

### 3.1 In-app descriptions (UI microcopy)

The settings UI must describe each mode clearly enough that the behavioral difference is obvious from the text, not just the name:

> **Normal Mode** — Plays only the teams and scenes you've set up. Each linked team runs on game day exactly as you configured it.
>
> **Rivalry Mode** — When two of your linked teams play each other, your lights follow the live score and switch to whoever's winning. *Both teams must be linked.*
>
> **Beast Mode** — Follows any matchup your linked team is in — even against a team you haven't added. Your lights track the live score and switch to whoever's leading. The opposing team's colors and reactions come from Haven's built-in defaults.

### 3.2 Explainer dialog

Editing the Play Mode option opens a **full-page, information-only dialog** that presents each mode with a visual graphic description of how it works. The dialog does **not** change the selection — it is purely explanatory; the actual mode is chosen on the settings screen. *(Source of the visual assets is TBD — see Open Questions.)*

---

## 4. Lighting Scenes

Every linked team has its own set of scenes. All compatible systems include five scenes that fire at key moments throughout the game:

| Scene | Trigger | Notes |
|---|---|---|
| **Pre-Game Scene** | Activates before the game begins | Configurable lead time (minutes); default **30 minutes** before game time |
| **Game Time Scene** | Runs throughout the duration of the game | — |
| **Team Win Scene** | Triggers when the team wins | — |
| **Team Lose Scene** | Triggers when the team loses | Not used in Rivalry/Beast Mode (§10) |
| **Event Over Scene** | Per-team scene shown after the game concludes | Interaction with the location-level Event Over Scene/Time setting (§9.3–9.4) is unresolved — see Appendix A-2 / Open Question #7 |

**Tie / drawn final.** If a game ends in a draw (e.g. soccer / World Cup), the **Game Time Scene** continues to play and neither the Team Win nor Team Lose Scene fires. *(Rivalry/Beast behavior on a drawn final — whose Event Over Scene runs when there is no winner — is still open; see Open Question #6.)*

---

## 5. Pro Animated Light Shows

**9 Series Pro** and **Q Series Pro** systems unlock three exclusive animated light show events ("Pro Systems Only" in the app) in addition to the standard scenes:

| Light Show | Trigger | Notes |
|---|---|---|
| **Game Start Light Show** | Kickoff / tip-off / first pitch | All supported sports |
| **Score Light Show** | Every time the team scores | Not available for basketball |
| **Team Win Light Show** | Final whistle / end of game win | All supported sports |

These animated shows are exclusive to Pro-tier systems and are not available on standard 9 Series or Q Series units.

---

## 6. X Series Effects

**X Series** systems unlock three effect events ("X Series Only" in the app), mirroring the Pro light shows but using X Series effects (e.g. "Diwali Effect · 10 sec"):

| Effect | Trigger | Notes |
|---|---|---|
| **Game Start Effect** | Kickoff / tip-off / first pitch | All supported sports |
| **Score Effect** | Every time the team scores | Not available for basketball |
| **Team Win Effect** | Final whistle / end of game win | All supported sports |

These effects are exclusive to X Series systems and are not available on the 9 Series, Q Series, or Pro units.

> The relationship between these X Series effects and the general effect library (sparkle / wave / cascade / marquee) introduced in §8 is unresolved — see Appendix A-4.

---

## 7. Preset Teams & Server-Side Defaults

Haven maintains a **server-side default library**: every team in every supported league has predefined default scenes, reactions, light shows, and effects, with lighting already mapped to the team's colors. These defaults are the baseline for the feature in **every** Play Mode:

- **Normal / Rivalry Mode** — every linked team is pre-populated with these defaults on add. The user can change or replace any individual response, but there is never a "missing scene" state.
- **Beast Mode** — when the opposing team is **not** configured by the user, the system uses that team's server defaults for its scenes, reactions, win scene, and light shows.

When a Preset Team is added to a location, it arrives with:

- A **Pre-Game Scene** in the team's colors, set to fire 30 minutes before the game
- A **Game Time Scene** in the team's colors
- A **Team Win Scene** in the team's colors
- A **Score** reaction (effect / light show) in the team's colors (15-second burst)
- A **Game Start** reaction (effect / light show) in the team's colors (15-second burst)
- A **Team Win** reaction (effect / light show) in the team's colors (15-second burst)
- An **Event Over Scene** (warm white) that fires 30 minutes after the game ends

All of these use built-in preset scenes and effects — no database scenes or database effects required. A Preset Team can be used as-is or customized after adding.

> Whether reactions in the default library are tier-gated (Pro/X Series only) when fired in Beast Mode, and whether the warm-white Event Over default coexists with the location-level Event Over setting, are open — see Appendix A-3 and A-6.

---

## 8. Lighting Options & Sources

For each configurable game moment, a response can be drawn from any of the following sources:

- **Preset scenes** — built-in lighting configurations (e.g. "Team Blue", "Celebrate", "Dim")
- **Database scenes** — scenes built in the Haven dashboard
- **Effects** — built-in dynamic patterns: sparkle, wave, cascade, or marquee (each with a configurable duration)
- **Database effects** — custom effects built in the Haven dashboard (each with a configurable duration)
- **Light shows** — pre-choreographed sequences from the location's library

> v0.5 gates light shows to Pro tiers and effects to X Series, with standard 9/Q Series receiving scene-switching only. The merged source presents effects and light shows as generally configurable for any system. This tier question is unresolved — see Appendix A-3. Until reconciled, the tier rules in §5, §6, §10.4, and §14 remain authoritative.

---

## 9. Game Day Settings (Location-Level)

Game Day behavior is configured per location on the **Game Day Settings** screen. These settings apply to the whole location, independent of which teams are linked.

> The configuration *surface* (mobile app vs. admin dashboard) and *audience* (homeowner vs. venue admin) differ between sources — see Appendix A-1.

### 9.1 Execute Sporting Event Window

A daily **Start** / **Stop** time window in the location's local timezone (default **6:00 AM – 11:59 PM**). It functions as a **curfew**:

- Game Day scenes, light shows, and effects only **fire** within this window; events that would fall outside it are suppressed (this prevents flashing lights late at night and early in the morning).
- If a game runs late and **game-end + Event Over Time** (§9.4) would place the Event Over fire time **outside** the window, the Event Over action **does not fire**.
- The curfew gates **future** actions only. The Stop time never reaches back to change a scene that is already showing. A scene that fired legitimately inside the window (including a held Event Over = None scene, see §9.3) **stays in its existing state past the Stop time** until some other command changes it.

### 9.2 Play Mode

The location's Play Mode (Normal / Rivalry / Beast) is selected here. See §3 for full behavior and the explainer dialog.

### 9.3 Event Over Scene

A picker controlling what the location does once a game has finished. The list shows, in order:

- **None** — no scheduled Event Over action. The **Team Win Scene** that fires at the final whistle simply **holds indefinitely** until a new command arrives at the location. The timer (§9.4) is **hidden** when None is selected.
- **Resume Schedule** — return to whatever the location's regular schedule would be doing at that time.
- **All scenes configured for the location** — select a specific scene to hold instead.

When **Resume Schedule** or a **specific scene** is selected, the **Event Over Time** timer (§9.4) is shown.

### 9.4 Event Over Time

A timer value, default **30 minutes**, shown only when Event Over Scene is set to **Resume Schedule** or a specific scene. It defines how long after the game ends the Event Over Scene selection is applied. At **game-end + Event Over Time** (and provided that moment falls **inside** the execution window, §9.1), the server either:

- executes the **selected scene**, or
- if **Resume Schedule** is selected, **returns the location to its normal schedule** — the server looks back at the schedule table to determine what would have executed that day at the current time and applies it.

**Off-event check.** Before resuming, the server must **first check for off events** — if the schedule would have the location **off** at that time, that takes precedence and the lights are not turned on. This reuses the existing schedule/actions table and priority-resolution logic (see the Presets spec).

### 9.5 Push Notifications

Five independent on/off toggles (all **off** by default) send a push notification when the corresponding event occurs. Notifications are delivered via the Haven mobile app to any user who has the location set as their default.

| Toggle | Fires on | Copy |
|---|---|---|
| **Push Notification — Pre Game** | Pre-game window begins | "Game Day! Your team's game is about to start." |
| **Push Notification — Game Time** | Game starts | "Game Time! The game has started." |
| **Push Notification — Team Score** | A tracked team scores | "Score! Your team just scored." |
| **Push Notification — Team Won** | A tracked team wins | "Victory! Your team won!" |
| **Push Notification — Team Lose** | A tracked team loses | "Game Over. Better luck next time." |

---

## 10. Rivalry & Beast Mode — Event Flow

When the active Play Mode (§3) engages lead-following — Rivalry (both teams linked) or Beast (one team linked, opponent via server defaults) — the lights follow the **currently winning** team and switch as the lead changes.

### 10.1 Team priority

Lead-following uses the **existing global team-priority list** already maintained in the app, where every linked team is ranked into a single ordered list. The **higher-priority team** in a matchup is whichever opponent ranks higher.

In **Beast Mode**, the unlinked opponent is not on the priority list; the user's linked team is therefore treated as the higher-priority team for game-start, tie, and event-over purposes.

### 10.2 Event flow

| Moment | Scene shown | Reaction fired |
|---|---|---|
| **Game start** | Higher-priority team's **Game Time Scene** | Higher-priority team's **Game Start** light show/effect |
| **Score (any team)** | **Currently-winning** team's **Game Time Scene** | **Scoring** team's **Score** light show/effect |
| **Tie / before first score** | Higher-priority team's **Game Time Scene** | — |
| **Final whistle (win)** | **Actual winner's Team Win Scene** | **Actual winner's Team Win** light show/effect |
| **After Event Over Time (§9.4)** | Per location-level Event Over setting (§9.3) — see note | — |

**Per-score sequence.** On every scoring event the system (1) evaluates which team is now winning, (2) sends the updated **Game Time Scene** for the currently-winning team, then (3) sends the **score reaction** — the scene update is always sent *before* the reaction. The score reaction is always the one configured (or defaulted) for the **specific team that scored**, regardless of which team currently leads.

**End of game.** Unlike standard Normal-mode play, Rivalry/Beast Mode uses the winner's **Team Win Scene** at the final whistle (along with the winner's Team Win light show/effect). The **Team Lose Scene is never used** in Rivalry/Beast Mode, since both teams belong to the same user. In Beast Mode, if the **unlinked opponent** wins, its **server-default** Team Win Scene and reaction are used.

> How the winner's Team Win Scene and the per-team Event Over Scene relate to the location-level Event Over Scene/Time setting (§9.3–9.4) is unresolved — see Appendix A-2 and Open Question #7.

### 10.3 Drawn final

Behavior on a true draw (no winner → no Team Win reaction) is not yet defined for Rivalry/Beast Mode — see Open Question #6.

### 10.4 Tier behavior

Lead-following is available on **all full-color systems**. Scene-switching (Game Time Scene following the lead) works on every tier. The per-event **reactions degrade by tier**:

- **9 Series Pro / Q Series Pro** — fires the corresponding Pro **light shows**.
- **X Series** — fires the corresponding X Series **effects**.
- **9 Series / Q Series (standard)** — scene-switching only; no game-start, score, or team-win reactions.

For basketball, the score reaction is not available (per §5/§6); scene-switching on each lead change still occurs.

---

## 11. Multiple Teams & Conflict Resolution

This section covers a location tracking multiple teams whose games **do not** play each other (the head-to-head case is handled by Rivalry/Beast Mode, §10).

- **Priority order.** Linked teams are ranked into a single ordered priority list (the same list used in §10.1).
- **Suppression.** When a higher-priority team is **active** — its game is scheduled nearby, currently live, or recently concluded — lower-priority team actions are **suppressed entirely** until the higher-priority team's window clears. This prevents conflicting lighting when two linked teams play **different** games at the same time.
- **Disable vs. remove.** A team can be **disabled** without being removed — useful for off-season or temporarily inactive teams. The configuration is retained and resumes when re-enabled. *(Whether a disabled team still counts against the per-tier team cap is open — see Open Questions.)*

---

## 12. Game Day Overrides

On specific calendar dates — holidays, private events, venue buy-outs — staff can **suppress all Game Day automation** without touching the underlying team configuration. Teams, scenes, and schedules remain intact and resume automatically on subsequent game days.

Overrides are managed separately from the Game Day configuration itself, so they can be added and removed without requiring access to the lighting setup.

---

## 13. Reliability & Data

- **Data source.** Live game data is polled from a professional sports data provider, **Broadage**, and updated continuously during live matches.
- **Polling cadence.** Scheduled games are polled every **10 seconds**; live game state is polled multiple times per second.
- **Sleep / wake.** When a sport has no games scheduled, in progress, or recently concluded across yesterday, today, and tomorrow, polling for that sport goes to sleep and automatically resumes at midnight — keeping the system efficient during off-days and bye weeks.
- **Cancellation.** If a game is cancelled, any pending pre-game lighting is automatically cancelled.
- **Logging.** All activity is logged for review and troubleshooting.

**Supported Sports & Leagues**

| Sport | Leagues |
|---|---|
| Basketball | NBA, NCAA Basketball |
| American Football | NFL, NCAA Football |
| Baseball | MLB |
| Hockey | NHL |
| Soccer | MLS, World Cup |

---

## 14. Product Tier Comparison

| Feature | 9 Series / Q Series | 9 Series Pro / Q Series Pro | X Series |
|---|---|---|---|
| Game Day Technology included | ✅ | ✅ | ✅ |
| Teams supported | Up to 3 | Up to 8 | TBD |
| Pre-Game / Game Time / Team Win / Team Lose / Event Over Scenes | ✅ | ✅ | ✅ |
| Game Start / Score / Team Win Light Show (Pro) | ❌ | ✅ | ❌ |
| Game Start / Score / Team Win Effect (X Series) | ❌ | ❌ | ✅ |
| Play Mode: Normal / Rivalry / Beast (scene switching) | ✅ | ✅ | ✅ |
| Rivalry/Beast per-event reactions | ❌ (scenes only) | ✅ (light shows) | ✅ (effects) |
| Cost | Free, built-in | Free, built-in | Free, built-in |

---

## 15. Ideal Use Cases

Game Day Technology is designed for anyone who wants to wear their team pride on their home or business:

- **Homeowners** looking to elevate their game day experience
- **Bars and restaurants** wanting dynamic, crowd-engaging displays
- **College campuses** and student housing
- **Businesses** showing community or team support on game days
- **Split-team households and rivalry games** — with Rivalry and Beast Mode, a head-to-head matchup lights up live with whichever team is leading

---

Haven Game Day Technology turns every game into an experience — no manual setup, no missed moments, just your team's colors exactly when it matters most.

## 16. Open Questions

| # | Question | Status | Owner |
|---|---|---|---|
| 1 | Which sports leagues/data source feed live game events, and what is the event latency? | **Largely resolved** — leagues and source (Broadage) documented in §13; precise end-to-end latency still TBD | Product / backend |
| 2 | How are scene/light-show colors derived per team (team color database)? | **Resolved** — server-side Preset Team default library, colors pre-mapped (§7) | Product |
| 3 | What happens on scheduling conflicts when two linked teams play **different** games simultaneously? | **Resolved** — higher-priority team active suppresses lower-priority actions until its window clears (§11) | Product |
| 4 | Score reaction is excluded for basketball — are there other per-sport exclusions to document? | Open | Product |
| 5 | How many teams does **X Series** support (tier table TBD)? | Open | Product |
| 6 | In Rivalry/Beast Mode, what happens on a true **draw / tie final** (no winner → no Team Win reaction)? Does the higher-priority team's Event Over Scene still run? | Open (narrowed; Normal-mode tie now defined in §4) | Product |
| 7 | How does the **per-team Event Over Scene** (§4) relate to the **location-level Event Over Scene / Event Over Time** (§9.3–9.4)? Which governs at game-end, and is the timer per-team or location-level? | Open — see Appendix A-2 | Product |
| 8 | How is a rivalry/Beast matchup detected from the data feed? | Partially — same Broadage feed (§13); detection of "same matchup" to confirm | Backend |
| 9 | For "Resume Schedule," confirm the off-event check and schedule-table lookback reuse the existing Presets priority-resolution logic exactly (no new conflict logic). | Open | Backend |
| 10 | In **Beast Mode**, when the unlinked opponent (server defaults) scores or wins, do its reactions respect tier gating (§10.4 / Appendix A-3)? | New / open | Product |
| 11 | Source of the **visual graphic assets** for the Play Mode explainer dialog (§3.2). | New / open | Design |
| 12 | Does a **disabled** team (§11) still count against the per-tier team cap? | New / open | Product |

---

## Appendix A — Specification Conflicts to Reconcile

The following conflicts surfaced when merging the *"GameDay — Automated Sports Lighting"* spec (June 2026) into this document. The body above retains the v0.5 / session-decision position; these are flagged for later reconciliation.

**A-1 · Configuration surface & primary audience.** v0.5 configures Game Day in the **Haven mobile app**, framed for **homeowners and businesses**. The merged spec configures via the **Haven admin dashboard**, framed for **venue admins / operations staff**. Decide whether these are one surface, two surfaces for two audiences, or a single consumer+commercial offering with parity. Affects every "where is this set" reference and the §9 framing.

**A-2 · Event Over delay scope (per-team vs. location-level).** v0.5 §9.3–9.4 defines Event Over Scene and Event Over Time as **location-level** (one timer, default 30 min, None / Resume Schedule / scene tri-state, off-event check). The merged spec makes the event-over scene and its delay **per team** (Preset Team default = warm-white scene, 30 min after game end). Reconcile whether Event Over is per-team, per-location, or layered (location default overridable per team). Tied to Open Question #7.

**A-3 · Tier gating of effects & light shows.** v0.5 gates **light shows → Pro tiers** and **effects → X Series**, with standard 9/Q Series getting scene-switching only (§5, §6, §10.4, §14). The merged spec presents effects and light shows as **generally configurable per moment for any system**, and Preset Teams ship score / game-start / win reactions with no tier qualifier (§7, §8). Reconcile whether reactions are universally available or remain tier-gated — this also drives Open Question #10.

**A-4 · Effect model & naming.** v0.5 illustrates X Series effects as named themed effects (e.g. "Diwali Effect · 10 sec"). The merged spec defines built-in effects as **sparkle / wave / cascade / marquee** (15-second bursts) plus database (custom) effects. Determine whether these are the same effect system described differently or two distinct concepts, and standardize default durations (10 sec vs. 15 sec).

**A-5 · Optional responses vs. guaranteed defaults.** The merged spec states each game-moment response is **optional** (a moment may have no response). The default-library principle (§7) holds that every linked team is **pre-populated**, so no moment is ever empty. Reconcile whether a user can clear a response to "none," and what fires if a moment has neither a configured nor a default response.

**A-6 · Event Over default behavior.** v0.5's Event Over default leans on **Resume Schedule + off-event check + schedule lookback**. The merged spec's Preset Team default Event Over is a fixed **warm-white scene fired 30 min after game end**. Decide the out-of-box Event Over default.

---

## Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 2026-06-18 | Jonathan | Initial draft |
| 0.2 | 2026-06-18 | Jonathan | Standardized to common spec template (metadata table, numbered sections, Goals/Non-Goals, Open Questions, Revision History) |
| 0.3 | 2026-06-19 | Jonathan | Added status-values note under metadata and closing footer for cross-document consistency |
| 0.4 | 2026-06-30 | Jonathan | Added Rivalry Mode sub-feature; reconciled scene/show names to the live app (Game Time / Team Win / Team Lose / Event Over); added X Series Effects category and X Series to scope |
| 0.5 | 2026-06-30 | Jonathan | Documented location-level Game Day Settings screen (Execute Sporting Event Window, push notifications); added Event Over Scene (Resume Schedule + location scenes) and Event Over Time (default 30 min) with off-event-check resume logic |
| 0.6 | 2026-06-30 | Jonathan | Introduced **Play Mode** (Normal / Rivalry / Beast) replacing the Rivalry boolean, with UI microcopy and information-only explainer dialog; made Event Over Scene a tri-state (None / Resume Schedule / scene) with conditional timer and "None holds Team Win Scene indefinitely" behavior; clarified curfew semantics (gates future actions only); added server-side default library / Preset Teams (§7); merged in lighting-option sources (§8), multiple-team conflict resolution / priority suppression (§11), Game Day Overrides (§12), and Reliability & Data incl. Broadage + supported leagues (§13); updated Rivalry/Beast event flow to use the winner's Team Win Scene; resolved Open Questions #2 and #3 (and largely #1); added **Appendix A** logging conflicts with the merged spec |

---

*End of Haven Game Day Technology Specification v0.6*