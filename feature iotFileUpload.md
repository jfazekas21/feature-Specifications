# Feature: IoT File Upload

## Summary
Enable IoT devices to upload files (logs, firmware artifacts, captured media, sensor data dumps) to the platform reliably over constrained networks.

## Goals
- Allow registered IoT devices to upload files to a managed storage backend.
- Support large files via chunked / resumable uploads over intermittent connectivity.
- Authenticate and authorize each upload at the device level.
- Track upload status and surface it to operators.

## Non-Goals
- File processing / transformation pipelines (handled downstream).
- Real-time streaming of telemetry (covered by a separate spec).

## Requirements

### Functional
- Devices request a short-lived upload session/credential before uploading.
- Uploads support resume after disconnect (chunked, idempotent).
- Server validates file size, type, and checksum on completion.
- Metadata (device ID, timestamp, file type, size, checksum) stored alongside the file.
- Operators can list, filter, and download uploaded files.

### Non-Functional
- Max file size: TBD.
- Supported file types: TBD.
- Retention policy: TBD.
- Throughput / concurrency targets: TBD.

## Proposed Design

### Upload Flow
1. Device authenticates and requests an upload session.
2. Server issues a scoped, time-limited upload token (e.g., pre-signed URL or session ID).
3. Device uploads the file in chunks, retrying failed chunks.
4. Device signals completion; server verifies checksum and finalizes.
5. Server records metadata and emits an event for downstream consumers.

### Components
- Device SDK / client: chunking, retry, checksum.
- Upload API: session creation, chunk receipt, finalization.
- Storage backend: object store (e.g., S3-compatible).
- Metadata store: upload records and status.

## Security
- Per-device authentication (mTLS / device tokens).
- Scoped, expiring upload credentials.
- Server-side checksum and size validation.
- Audit logging of upload events.

## Open Questions
- Which storage backend?
- Max file size and supported types?
- Retention and cleanup policy?
- How are devices authenticated (mTLS, JWT, API key)?

## Status
Draft — created 2026-06-18.
