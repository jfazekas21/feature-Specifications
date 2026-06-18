# Feature: IoT File Upload — On-Demand Debug Log Upload (ESP32-S3 → Azure Blob Storage)

**Version:** 0.5 · **Last updated:** 2026-06-18

## 1. Overview & Goals
Enable a support engineer or admin to pull debug logs — Wi-Fi, Bluetooth, Ethernet performance logs — off a specific ESP32-S3 device on a home network, for manual case debugging. The Azure backend orchestrates the process securely, tolerates unreliable home connectivity via resumable uploads, guarantees unique file names with timestamps, and never overwrites existing files.

The device side is intentionally minimal: it supports exactly **two commands** (see §3). The backend handles all orchestration, authorization, and storage.

### Goals
- Command-driven, two-phase flow: list available files, then upload a chosen subset.
- One unique blob per file/session, with timestamps in the path.
- Resumable uploads for flaky home networks.
- Scale to thousands of devices.
- Metadata for retrieval and tracking.
- Security via short-lived SAS tokens; devices never hold account keys.
- Direct device-to-blob uploads to minimize server load.

### Non-Goals
- File processing / transformation pipelines (handled downstream).
- Real-time telemetry streaming (separate spec).
- Cross-fleet log analytics (manual, per-device debugging is the v1 use case).

## 2. Triggers & Actors
The same command payload can reach the device through any of three channels, dispatched by different actors, all handled by one transport-agnostic command handler on the device:
- **Installer, on-site** → BLE from the mobile app.
- **Support/admin** → Azure IoT Hub direct method.
- **App** → HTTP response.

BLE is a **trigger only** — the device always uploads to Azure Blob Storage directly over its own Wi-Fi/Ethernet connection. The phone never relays log data. Per-channel authorization (different trust levels per actor) is deferred — see §8.

## 3. Device Command Interface
The device implements exactly two commands. Both accept a JSON payload and are independent of the channel they arrive on. The two are asymmetric:
- **`ListFiles` is request/response** — its return payload (the list of available files) is essential.
- **`UploadFiles` is fire-and-forget during the transfer** — no per-file handshakes or acknowledgments. After the device receives it, the subsequent communication is just the HTTP file uploads to the Blob Storage URL, followed by a single **completion callback** (one HTTP POST) when the whole run finishes.

### 3.1 `ListFiles` — harvest available files
Device scans its filesystem (SPIFFS / LittleFS) and returns the set of files available for upload. This response is required.

Response (device → backend):
```json
{
  "deviceId": "dev-abc123",
  "files": [
    { "filename": "wifi-debug.log",     "path": "/logs/wifi-debug.log",     "size": 184320 },
    { "filename": "bt-debug.log",        "path": "/logs/bt-debug.log",       "size": 40960  },
    { "filename": "eth-perf.log",        "path": "/logs/eth-perf.log",       "size": 12288  }
  ]
}
```
The backend/app shows this list so the admin can select which files to pull.

### 3.2 `UploadFiles` — upload a selected array to Blob Storage (blind / fire-and-forget)
Command (backend → device): an array of file paths plus the target Blob Storage URL (a SAS-scoped base URL), and a `callbackUrl` for the completion POST. The command can be delivered as the payload of an HTTP response (or any channel). During the run the device sends no per-file acks — it just performs the HTTP uploads to Blob Storage. When the run finishes, it sends one completion callback (see §3.3).

Command payload:
```json
{
  "command": "UploadFiles",
  "deviceId": "dev-abc123",
  "blobBaseUrl": "https://account.blob.core.windows.net/container?<sasToken>",
  "pathPrefix": "logs/dev-abc123/2026/06/18/",
  "timestamp": "2026-06-18T13:45:22Z",
  "callbackUrl": "https://api.example.com/uploads/complete",
  "files": [
    { "path": "/logs/wifi-debug.log", "logType": "wifi" },
    { "path": "/logs/eth-perf.log",   "logType": "eth"  }
  ]
}
```
For each file the device composes the destination blob name (see §4) and uploads it as a block blob with retry/resume (see §5). No status is sent during the run; when all files are processed the device fires a single completion callback (§3.3). The backend can also independently confirm results by observing Blob Storage (see §6).

### 3.3 Completion callback (device → backend)
When the `UploadFiles` run finishes, the device sends one HTTP POST to `callbackUrl` summarizing the outcome. On success the payload is compact; on error it carries as much diagnostic detail as the device can provide.

Success:
```json
{
  "deviceId": "dev-abc123",
  "status": "ok",
  "blobBaseUrl": "https://account.blob.core.windows.net/container/logs/dev-abc123/2026/06/18/",
  "completedAt": "2026-06-18T13:47:10Z",
  "filesRequested": 2,
  "filesUploaded": 2
}
```

Error (as detailed as possible):
```json
{
  "deviceId": "dev-abc123",
  "status": "error",
  "blobBaseUrl": "https://account.blob.core.windows.net/container/logs/dev-abc123/2026/06/18/",
  "completedAt": "2026-06-18T13:47:10Z",
  "filesRequested": 2,
  "filesUploaded": 1,
  "firmwareVersion": "1.4.2",
  "freeHeap": 184320,
  "network": { "type": "wifi", "rssi": -78 },
  "files": [
    {
      "path": "/logs/wifi-debug.log",
      "blobName": "2026-06-18T13-45-22-wifi-debug.log",
      "status": "ok",
      "size": 184320,
      "bytesUploaded": 184320,
      "blocksCommitted": 1,
      "durationMs": 5120
    },
    {
      "path": "/logs/eth-perf.log",
      "blobName": "2026-06-18T13-45-22-eth-debug.log",
      "status": "error",
      "size": 12288,
      "bytesUploaded": 4096,
      "blocksCommitted": 1,
      "attempts": 3,
      "errorStage": "put-block",
      "httpStatus": 403,
      "azureErrorCode": "AuthenticationFailed",
      "message": "SAS token expired before block 2 of 3 was committed",
      "lastAttemptAt": "2026-06-18T13:46:58Z"
    }
  ]
}
```

Field notes:
- `status` (top level): `"ok"` only if every requested file uploaded fully; otherwise `"error"`.
- `blobBaseUrl`: echoed back so the backend can correlate the callback with the session/blobs.
- Per-file `errorStage`: where it failed — e.g., `read-file`, `put-block`, `put-blocklist`, `dns`, `tls`, `connect`, `timeout`.
- `httpStatus` / `azureErrorCode`: the actual response from Blob Storage when available (e.g., `403`/`AuthenticationFailed`, `409`/`BlobAlreadyExists`), so the cause is unambiguous.
- Device context (`firmwareVersion`, `freeHeap`, `network`) is included on error to aid diagnosis without a second round-trip.
- The callback remains best-effort — it can be lost on a flaky network — so the backend still reconciles against Blob Storage (see §6).

## 4. Blob Naming & Organization (overwrite prevention)
```
logs/{deviceId}/{year}/{month}/{day}/{timestamp}-{logType}-debug.log
```
Example: `logs/dev-abc123/2026/06/18/2026-06-18T13-45-22-wifi-debug.log`

- Path prefixes act as virtual folders for browsing in Azure Portal / Storage Explorer.
- The `{timestamp}` and `{pathPrefix}` are supplied by the **server** in the `UploadFiles` command, so naming is server-controlled and consistent.
- Device ID + timestamp + log type guarantees uniqueness and prevents overwrites — sufficient for manual case debugging (no separate case ID needed).

## 5. Handling Unreliable Connections (Resumable Block Blob Upload)
A basic `Put Blob` is all-or-nothing. Use **block blobs** (`Put Block` + `Put Block List`) for resume support.

Implementation notes (Arduino / ESP-IDF with `esp_http_client` or similar):
- Block size: up to 4 MB (PSRAM + large flash on this hardware makes this comfortable).
- Generate unique, fixed-length, base64-encoded Block IDs (e.g., `block-0001` padded).
- Persist progress (committed block list + offset) in NVS / LittleFS for restart/resume.

```cpp
// Pseudocode / key flow for one file in the UploadFiles array
const size_t BLOCK_SIZE = 4 * 1024 * 1024;
String blobUrl = blobBaseUrl_with_path;  // base + pathPrefix + composed blob name

std::vector<String> committedBlocks;
for (size_t offset = 0; offset < fileSize; offset += BLOCK_SIZE) {
    size_t chunkSize = min(BLOCK_SIZE, fileSize - offset);
    String blockId = base64Encode(padded(offset / BLOCK_SIZE));

    if (alreadyUploaded(blockId)) continue;  // resume: skip committed blocks

    int retries = 0;
    while (retries < 3) {
        if (uploadBlock(blobUrl + "&comp=block&blockid=" + blockId, fileChunk, chunkSize)) {
            committedBlocks.push_back(blockId);
            saveProgress();
            break;
        }
        retries++;
        delay(exponentialBackoff(retries));  // 1s, 2s, 4s
    }
}

// Commit: assemble blocks and set metadata
String blockListXml = buildBlockListXml(committedBlocks);
httpPut(blobUrl + "&comp=blocklist", blockListXml, headersWithMetadata);
```

### Metadata Headers (sent on Put Block List)
- `x-ms-meta-deviceid: dev-abc123`
- `x-ms-meta-devicename: LivingRoomSensor`
- `x-ms-meta-logtype: wifi`
- `x-ms-meta-uploadtime: 2026-06-18T13:45:22Z`
- `x-ms-version: 2024-11-04` (or latest supported)
- `x-ms-tags: ...` — optional blob index tags; lower priority since v1 is per-device retrieval, not cross-fleet querying.

### Resume Logic
- On reboot/retry, query committed vs. uncommitted blocks (`GET ...?comp=blocklist&blocklisttype=all`) and re-upload only what's missing.
- Exponential backoff + max retries per block.
- Azure IoT Hub's native file-upload feature is a managed alternative (handles SAS + completion notifications).

## 6. Server-Side Responsibilities (Azure Functions / App Service / IoT Hub)
- Dispatch `ListFiles`, receive and surface the file list to the admin/app.
- For `UploadFiles`: mint a short-lived SAS scoped to the target container/prefix, build the command payload (base URL, path prefix, timestamp, `callbackUrl`, selected files), and send it.
- Expose the `callbackUrl` endpoint to receive the device's completion POST (`status` ok/error + echoed blob URL).
- Confirm outcomes primarily by **observing Blob Storage** — Event Grid `BlobCreated` or listing the device's prefix and comparing against the requested array — since the completion callback is best-effort and can be lost. Files that never appear within an expected window are treated as failed/incomplete.
- Optional post-upload enrichment via Event Grid `BlobCreated` or a periodic job (`Set Blob Metadata` for exact receive time, etc.).

## 7. Security & Scaling
- **SAS tokens**: scoped to a blob prefix (e.g., write to `logs/{deviceId}/*`), short expiry (30–60 min), generated server-side per upload session.
- **One container recommended**: partition by device-ID prefix. A single storage account handles thousands of devices; split only if approaching ~20k requests/sec.
- **Authentication**: Azure AD / Managed Identity on the server; devices never receive account keys.
- **Reliability**: block blobs + retries; enable soft delete + versioning on the container.

## 8. Error Handling & Monitoring
- **Device**: log failures locally; retry blocks/files within the current `UploadFiles` run; send a single completion POST (`ok`/`error`) at the end. A failed file is also simply absent from Blob Storage.
- **Server**: treat the completion callback as a fast-path signal, but reconcile against Blob Storage (blob present + expected size) since the callback can be lost. Re-issue `UploadFiles` for any files that didn't land. Azure Monitor / Application Insights for backend-side observability.
- **Storage**: diagnostics, versioning, soft delete.
- **Edge cases**: partial uploads (resume), duplicate commands (idempotency via server timestamp/session), large logs (block-size tuning).

## 9. Open Questions
- Retention / cleanup policy for uploaded logs?
- Per-channel authorization model (installer BLE vs. support direct method vs. app HTTP) — how does the device verify a command is legitimate? (Deferred.)
- Does `ListFiles` need filtering (by log type, size, date) or is the full list fine?
- Max log file size and default block size?
- Native IoT Hub file upload vs. custom SAS flow? (Not being decided yet.)

## Status
Draft — updated 2026-06-18.
