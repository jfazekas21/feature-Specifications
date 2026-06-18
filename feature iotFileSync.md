# Folder Sync — Feature Specification

**Status:** Draft
**Owner:** Jonathan, Haven Lighting
**Target device:** ESP32-S3 lighting controller
**Last updated:** 2026-06-18

---

## 1. Feature Overview

Folder Sync keeps a set of files on a RAM-constrained IoT controller (ESP32-S3) in step with an authoritative copy held on the server. The design optimizes for one thing above all: **minimizing the number of HTTP transactions** by maximizing the useful information carried in each payload and by pushing all comparison logic onto the device.

The system is built around a single per-device **manifest file** that the server maintains. The manifest lists every file the device should hold, along with each file's name, size, version, and a compact CRC tree used for change detection. The device's only routine contact with the server is a tiny poll for the manifest's version number. When that version has not moved, the device is fully synced and no further calls are made. When it has moved, the device fetches the changed portion of the manifest and then pulls only the changed bytes of the affected files using standard HTTP byte-range requests.

This turns the server into an essentially static file host. There is no per-device drill-down protocol, no server-side tree-walking, and no speculative pre-staging logic. The server's job is to keep manifests current and to serve byte ranges; everything else happens on the device.

### Design goals

- Minimize HTTP transactions. Prefer two calls carrying excess data over twelve calls to sync one small change.
- Keep the device's working memory footprint tiny. The device never holds a full manifest or a full file in RAM.
- Make append the fast path, since ~80% of changes are pure appends.
- Detect adds, deletes, appends, and mid-file edits, including many occurring simultaneously.
- Consolidate shared files server-side so the bulk of the file set is computed and served once for the whole fleet.

### Scale and limits

| Parameter | Value | Notes |
|---|---|---|
| Maximum files per device | 1500 | Mix of shared and device-specific |
| Maximum file size | 3 MB | Binary MB (3,145,728 bytes) |
| Minimum byte-range download | 8 KB | Device download floor; also the chunk floor |
| Device response RAM ceiling | 1000 bytes | **Non-negotiable** hardware constraint |
| CRC width | 16-bit | Change detection only — see §8 |
| Max CRC entries per file | 64 | Fixed-width manifest row — see §4 |

---

## 2. Architecture

### 2.1 The manifest is the system

The server maintains, **per device**, an authoritative manifest. The recommended form is a **SQLite `.db` file** with fixed-width columns and one row per file. SQLite is chosen deliberately: it lets the RAM-constrained device answer "what is the CRC for file 47?" by reading a few pages off storage, without ever loading the whole manifest into memory. A flat blob would force a full scan; the `.db` form does not.

The manifest is itself **file zero** — it has a name, size, version, and CRC tree, and it syncs with the exact same machinery as every other file. When twelve files change, only their rows change, so only part of the manifest changes, and the device fetches only the changed portion.

### 2.2 Shared vs. custom split

A single per-device manifest would re-duplicate the CRC data for shared files — the bulk of the 1500 — into every device's manifest, regenerating and storing essentially the same data across the whole fleet and discarding the benefit of sharing.

The manifest is therefore split into two:

- **Shared manifest** — identical across the entire fleet, generated once, cacheable (CDN-friendly), fetched identically by every device. This is where the heavy chunk vectors live. Because it is fleet-wide static, its size stops mattering.
- **Custom manifest** — small, per-device, covering only that device's unique files.

The device's initial poll checks **two** version numbers, one per manifest.

### 2.3 Transaction types

The whole protocol reduces to three kinds of HTTP call:

1. **Version poll** — tiny request returning the shared and custom manifest version numbers. In steady state (two years of silence) this is the only call, and it costs almost nothing.
2. **Manifest fetch** — when a version moves, a byte-range fetch of the changed portion of the relevant manifest.
3. **File range fetch** — one HTTP `Range` GET per changed file, pulling only the changed bytes.

This is about as few calls as the problem physically allows: you must download changed bytes no matter what, and everything else collapses into the poll and the manifest fetch.

---

## 3. The CRC Tree

Each file is represented by a tree:

- **Chunk CRCs** — a 16-bit CRC for each chunk of the file (chunk size derived from file size, see §4).
- **File CRC** — a rollup over the file's content, used as the primary "did this file change" signal.

The device maintains its own cached CRC tree for every file it holds, persisted across power cycles (RAM or a local `.db`). A factory-fresh or cache-corrupted device rebuilds the tree by scanning every file — see the cold-start gate in §8.

> **Rollup width:** Use at least CRC-32 at the file and manifest-master levels even though chunk leaves are 16-bit. A 16-bit rollup risks a changed file producing an unchanged file hash, in which case the device never starts drilling down. This is a correctness concern, not a performance one.

---

## 4. Chunk Sizing — Fixed-Width Vector

Every file is represented by a **fixed-width vector of at most 64 sixteen-bit CRC values** (≤ 128 bytes of chunk vector per row). Uniform rows keep the `.db` columns fixed-width, make manifest size predictable, and simplify the firmware parser, which never handles a variable-length field.

Chunk size **scales up with file size** so the 64-entry cap always holds, and no chunk is ever smaller than the 8 KB download floor (localizing finer than you can fetch is pointless). Chunk size is **not stored** — the device derives it from the file size using the same switch the server used.

| File size | Chunk size | Resulting entries |
|---|---|---|
| ≤ 8 KB | (none — file CRC only) | 1 |
| 8 KB – 512 KB | 8 KB | 2 – 64 |
| 512 KB – 3 MB | 64 KB | 9 – 48 |

Worked checks (binary units): 512 KB ÷ 8 KB = 64 (cap, exactly); 3 MB ÷ 64 KB = 48; 2 MB ÷ 64 KB = 32; 1 MB ÷ 64 KB = 16. Every file lands at ≤ 64 entries.

An optional 16 KB tier can be inserted in the middle for finer mid-size granularity; the two-tier-plus-trivial form above is the lean version.

> **Consequence accepted:** a 3 MB file localizes changes to 64 KB rather than 8 KB, so a mid-file edit re-fetches up to 64 KB (~100–200 ms of over-fetch on a few-Mbps link, on the ~20% of changes that aren't appends). Appends still resolve to a single tail range regardless of chunk size. This is a cheap price for fixed-width rows.

> **Critical correctness rule:** because chunk size is derived from file size, the device must compute it **the same way the server did**, or the vector misaligns. The tier switch is a **shared, versioned constant**. Retuning the boundaries is effectively a manifest-format change, and every device must agree on which version it is parsing.

---

## 5. Sync Flow

### 5.1 Narrative

1. The device polls the server for the shared and custom manifest version numbers.
2. If both match the device's cached versions, the device is synced. **Stop.**
3. If a version moved, the device byte-range-fetches the changed portion of that manifest and verifies it against the manifest CRC.
4. The device compares the manifest rows against its local file state to classify each file: **unchanged**, **new**, **deleted**, **appended**, or **mid-file edit**.
5. Deleted files are removed locally. For each new/changed file, the device performs the per-file update (§5.2).
6. After each file update, the device recomputes its file CRC and confirms it matches the manifest.

### 5.2 Per-file update — size-first / optimistic append

Because ~80% of changes are pure appends, append is the fast path and avoids chunk-tree drill-down entirely when possible:

1. Compare the manifest's file **size** against the local size.
2. **Server larger →** optimistically assume append. Issue one `Range` GET for `[local_EOF, server_EOF)`, append, recompute file CRC.
   - **CRC now matches →** append confirmed, done (one fetch).
   - **CRC mismatch →** it was also a mid-file edit; fall back to step 3.
3. **Drill-down →** compare local vs. manifest chunk CRCs, identify differing chunks, `Range`-GET each differing chunk, recompute and confirm.
4. **New file →** fetch whole (or by chunks if large), verify file CRC.

Size comparison is the cheapest possible pre-filter for "did anything change here."

### 5.3 Swim-lane

```
DEVICE                                   SERVER
  |                                         |
  |  GET /sync/versions  (tiny poll)        |
  |---------------------------------------->|
  |   { shared_ver, custom_ver }            |
  |<----------------------------------------|
  |                                         |
  | versions unchanged? --> DONE            |
  |                                         |
  | version moved:                          |
  |  GET manifest (Range: changed portion)  |
  |---------------------------------------->|
  |   manifest bytes                        |
  |<----------------------------------------|
  |                                         |
  | verify manifest CRC                     |
  | classify rows:                          |
  |   unchanged / new / deleted /           |
  |   appended / mid-file edit              |
  |                                         |
  | delete removed files locally            |
  |                                         |
  | per changed file:                       |
  |   size-first append attempt             |
  |    GET file (Range: tail)               |
  |---------------------------------------->|
  |     tail bytes                          |
  |<----------------------------------------|
  |   recompute file CRC                    |
  |    match? --> done                      |
  |    mismatch --> chunk drill-down        |
  |     GET file (Range: differing chunk)   |
  |---------------------------------------->|
  |       chunk bytes                       |
  |<----------------------------------------|
  |   recompute & confirm                   |
  |                                         |
  | update cached versions + CRC tree       |
  |                                         |
```

---

## 6. Mass-Change Fallback

The level-by-level drill-down is optimized for a small number of changes. The spec explicitly allows many simultaneous changes — e.g. ten deletes plus six mid-file edits plus twelve appends in one window. With the single-manifest design, all of those are still resolved by **one manifest fetch**, regardless of how many files moved, because the manifest is authoritative for the entire file set in one document. This is the main reason the manifest approach is preferred over per-file tree-walking: mass change does not produce a transaction storm.

When the manifest itself has changed extensively, the device may re-fetch it whole rather than range-fetching scattered changes (see §7).

---

## 7. Manifest-Format Caveat (`.db` does not append cleanly)

The tail-Range fast path depends on changes being appends. A SQLite file is paged internally: inserting or updating rows rewrites b-tree pages scattered through the file, bumps the header, and touches the freelist. So a manifest change produces byte changes **sprinkled across the file**, and the "compare size, fetch the tail" shortcut that works for content files does **not** work for the manifest itself — the device will see a mid-file diff and drill down on the manifest.

Three options, in rough order of preference:

1. **Keep the manifest small enough** that re-downloading it whole on any change is cheap. Often viable if the heavy chunk vectors live in the cacheable shared manifest.
2. **Use a log-structured / append-only manifest format** so manifest changes are themselves tail-appends.
3. **Accept drill-down on the manifest** and treat it as the one file that does not get the append optimization.

Decide this consciously rather than discovering it in testing. (Open item, §9.)

---

## 8. Where the Design Can Fail

**16-bit CRC is change detection, not authentication.** A 16-bit CRC has 65,536 values, so any single chunk comparison has a ~1-in-65,536 chance of colliding — a chunk changed but its CRC matches, and the device believes it is synced when it is not. Across 384 chunks × 1500 files × continuous polling over years across a fleet, this is not hypothetical. Mitigation: keep chunk leaves at 16-bit (the storage/round-trip trade is real) but use **CRC-32 at the file and manifest-master levels** so a changed file always forces a drill-down. Do not market 16-bit CRC as tamper-protection; there is no malicious-modification defense here and none was requested.

**Cold start depends on an unmeasured number.** The device caches its CRC tree across power cycles, but a factory-fresh or cache-corrupted device must rebuild by scanning every file. If a full 3 MB CRC scan takes two minutes rather than tens of milliseconds, a 1500-file cold scan runs into hours and the "device quickly knows every file's CRC" premise collapses. Treat the design as provisional until the firmware measurement in §10 lands.

**"Fill every byte" is wasteful in steady state.** For two years between bursts, every poll returns "versions unchanged" and there is nothing legitimate to anticipate. Density pays off only when there is a real, version-backed change to communicate. Gate it: lean response in steady state, dense payloads only when a manifest version has moved.

**Deletes and new files require authoritative membership.** A pure hash tree cannot express "file 47 was deleted" or "file 1601 is new." The manifest must be **authoritative for presence** — if a row is absent, the file must not exist on the device. This is designed in via the manifest, not bolted on; if membership were ever inferred from hashes alone, deleted files would linger forever.

---

## 9. Open Questions

1. **Manifest format** — SQLite `.db` (random-access friendly, but pages scatter on write) vs. an append-only/log-structured format (syncs as a tail-append, but needs a query strategy on-device). See §7.
2. **Chunk-leaf CRC width** — 16-bit keeps a maxed file's vector at 128 bytes (capped); the collision risk in §8 may argue for wider leaves at a storage cost. Confirm 16-bit leaves are acceptable given CRC-32 rollups.
3. **Tier boundaries** — the §4 thresholds (8 KB / 512 KB / 3 MB; optional 16 KB tier) are tunable. Lock the values and assign a format version before firmware integration.
4. **Future transports** — design assumes HTTP as primary. Provisioning for IoT-hub and WebSocket sync is anticipated but not yet specified.

---

## 10. Appendix — Required Firmware Measurement (Design Gate)

**The firmware team must measure how long the ESP32-S3 takes to calculate a 16-bit CRC across a full 3 MB file, end to end (including flash read, not just CRC math).**

This is a gating dependency for the entire design flow. If the answer is ~10 ms, frequent rescans, incremental boot verification, and aggressive caching are all viable. If it is closer to two minutes, the entire cold-start and rescan strategy must change.

Analytical expectation (to be confirmed, not assumed):

- **CRC compute** is O(total bytes) and essentially chunk-size-independent. A table-driven CRC-16 on the LX7 at 240 MHz (~4–8 cycles/byte → ~30–60 MB/s) puts a 3 MB scan near ~50–100 ms of pure compute.
- **Flash read dominates.** LittleFS over SPI flash is realistically low-single-digit MB/s sequential, so reading 3 MB is the actual bottleneck — likely 5–10× the compute. Align reads to the LittleFS block (~4 KB) for efficiency.
- **Decouple I/O buffer from CRC chunk.** Stream the file through a small 4 KB flash-aligned I/O buffer while accumulating the CRC over the logical chunk window. This keeps RAM tiny and flash reads efficient independent of the manifest chunk granularity.

The measured number feeds directly into the chunk-leaf width decision (§9.2) and the cold-start risk (§8).

---

## Appendix B — Reference Figures

- 3 MB ÷ 8 KB = **384** chunks (binary units). At 2 bytes each, 768 bytes for a full 8 KB-chunk vector — which is why the fixed 64-entry cap (§4) exists.
- 3 MB ÷ 64 KB = **48** chunks — the large-file tier, comfortably under the 64 cap.
- 16-bit CRC value space = **65,536** → ~1-in-65,536 per-chunk collision probability (§8).