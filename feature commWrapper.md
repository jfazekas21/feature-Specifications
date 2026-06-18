# Socket Communications Wrapper Specification

| | |
|---|---|
| **Protocol Version** | 1.0 |
| **Document Revision** | v24 |
| **Date** | 2026-06-18 |
| **Status** | Active |

### Revision History

| Rev | Date       | Protocol Version | Description                                                                                                                                                                                |
| --- | ---------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| v24 | 2026-06-18 | 1.0              | Transport model reworked. Top-level container is now mandatory on **every** transport, guaranteeing a single parser with no per-transport branching or escape characters. Framing responsibility separated from message structure. The "Container Required" column is replaced by a message-initiation capability table governed by a single rule. UDP added as a transport. OTA chunk transfer given explicit timeout/retransmit and idempotent re-ack semantics. `MENU` documented as the one actor whose payload is a JSON array. |
| v23 | 2026-05-27 | 1.0              | Strict JSON throughout; BLE advertisement structure with company ID `0x10C5`, product ID table, scan response, and device filtering; Device Identity handshake section; Data Types section |

---

## Data Types

The following data types are used for `Variable` and `readOnlyVariable` entries in menu responses. They apply whenever a variable is being retrieved (`get`), written (`set`), or displayed via the menu system.

| `dataType` | Description                              | Range / Notes                                                  |
| ---------- | ---------------------------------------- | -------------------------------------------------------------- |
| `bool`     | Boolean flag                             | `true` or `false`                                              |
| `uint8`    | Unsigned 8-bit integer                   | 0 – 255                                                        |
| `uint16`   | Unsigned 16-bit integer                  | 0 – 65,535                                                     |
| `uint32`   | Unsigned 32-bit integer                  | 0 – 4,294,967,295                                              |
| `int32`    | Signed 32-bit integer                    | −2,147,483,648 – 2,147,483,647                                 |
| `float`    | IEEE 754 single-precision floating-point | Decimal number; may carry `min`, `max`, and `unit`             |
| `string`   | UTF-8 text string                        | May carry a `format` hint (e.g., `"HH:MM:SS"`, `"MM-DD-YYYY"`) |
| `enum`     | Named constant chosen from a fixed set   | Always accompanied by a `values` array listing valid options  |

### Variable Fields by Data Type

| `dataType` | Fields always present | Optional fields |
|------------|-----------------------|-----------------|
| `bool` | `value` | — |
| `uint8` / `uint16` / `uint32` / `int32` | `value` | `min`, `max` |
| `float` | `value` | `min`, `max`, `unit` |
| `string` | `value` | `format` |
| `enum` | `value`, `values` | — |

---

## 1. Overview

### Purpose and Scope
This document defines a lightweight JSON-based messaging wrapper for communication between a host client (e.g., mobile app or server) and an embedded device (e.g., ESP32). All messages are structured around named **actors** (subsystems) and support both synchronous command/response and asynchronous device-initiated messaging.

The wrapper is **transport-independent**. The same message structure is carried unchanged over UART/Serial, BLE, TCP (including Telnet-style sockets), UDP, HTTP, and Azure IoT Hub. A transport may add its own framing, but it never alters the message itself.

### Design Principles
- All messages are valid JSON objects.
- The top-level container key identifies the message class and is **mandatory on every transport**.
- Actors serve as both the command target and the organizational namespace.
- The `ack` payload always reflects the actual applied value, not the requested one.
- **One parser, every transport.** A single parsing function handles every message on every transport. There are no escape characters, no transport-specific container, and no conditional "is the container present?" branch.

### The Single Parser
Every message — regardless of transport — is parsed the same way:

1. Parse the bytes as one JSON object. (The transport's framing, below, tells the parser where that object begins and ends.)
2. The root object contains **exactly one** container key — one of `cmd`, `ack`, `msg`, or `err` — and, optionally, `mID`. The container key selects the message class.
3. Inside the container, read `act` to identify the target/subject actor.
4. The remaining key is the method name or property operation (`set` / `get`); its value is the payload. The payload is a JSON **object** for every actor except `MENU`, whose payload is a JSON **array** (see §9.3).

Because the container is always present, step 2 never branches on transport. A message that omits the container is malformed on every transport, including UART and BLE.

### Transport Framing
Framing — knowing where one message ends and the next begins — is the transport's responsibility and is the *only* thing that varies between transports. It never changes the message structure.

| Transport | Framing mechanism |
|-----------|-------------------|
| UART / Serial | Stream; messages delimited by the framing convention in §1.1 (length prefix or newline). |
| BLE socket | Stream over GATT writes/notifications; external framing per §1.1. |
| TCP / Telnet socket | Stream; framing per §1.1. |
| UDP | One complete message per datagram; the datagram boundary is the frame. |
| HTTP | One complete JSON object per request body and per response body. |
| Azure IoT Hub | One complete JSON object per direct-method payload, per device-to-cloud message, and per cloud-to-device message. |

### 1.1 Stream Framing Convention
On byte-stream transports (UART, BLE, TCP) a frame boundary must be defined so the reader can isolate one JSON object. One of the following applies for the life of a connection; it is chosen per transport, not per message:

- **Newline-delimited:** each message is minified (no embedded newlines) and terminated by a single `\n`. This is the default for UART and Telnet-style sessions.
- **Length-prefixed:** each message is preceded by a fixed-width byte count. Preferred where binary-safe framing is needed.

Datagram transports (UDP) and request/response transports (HTTP, IoT Hub) need no stream framing: the datagram or body *is* the frame.

### Message Initiation by Transport
Whether a given message can be sent **unsolicited** depends on whether its sender can initiate on that transport. This is governed by a single rule:

> **An unsolicited message is possible only when its sender can initiate communication on that transport.**

Synchronous `cmd` → `ack` exchanges work identically on all transports. The rule only constrains *unsolicited* traffic — a server-initiated `cmd` to an idle device, or a device-initiated `msg` / `err`.

| Transport | Persistent bidirectional channel | Server → device `cmd` (unsolicited) | Device → client `msg` / `err` (unsolicited) |
|-----------|----------------------------------|-------------------------------------|---------------------------------------------|
| UART / Serial | Yes | Anytime | Anytime |
| BLE socket | Yes | Anytime | Anytime |
| TCP / Telnet | Yes | Anytime | Anytime |
| UDP | Connectionless; either side may send anytime, **best-effort** | Anytime (best-effort) | Anytime (best-effort) |
| HTTP | No — the **device** is the HTTP client | Not possible directly; the server may only return a `cmd` inside its HTTP response to a device-initiated request | Yes — the device sends `msg` / `err` as an HTTP request body |
| Azure IoT Hub | Yes — the device holds the MQTT/AMQP link | Anytime, via cloud-to-device message or direct method | Yes — as a device-to-cloud message |

Consequences worth stating explicitly:

- Over **HTTP**, the server cannot reach an idle device. A server-originated `cmd` must wait until the device next contacts the server, at which point it rides back in the HTTP response body. Device-originated `msg` / `err` are unaffected: the device is always free to POST them.
- Over **Azure IoT Hub**, both directions of unsolicited traffic are available because the device maintains a persistent connection: direct methods / cloud-to-device carry server-originated `cmd`; device-to-cloud carries `msg` / `err`.
- On **socket-class transports** (UART, BLE, TCP) both directions are open at all times.

---

## 2. BLE Advertisement Data

Before a connection is established, each device broadcasts a standard BLE advertisement packet. The advertisement is not part of the JSON wrapper — it uses fixed BLE AD structures. Its purpose is device discovery and identification only; no configuration data is exchanged here.

The advertisement is split across two packets: a **primary advertising packet** (always broadcast) and a **scan response packet** (returned only when an active scanner sends a `SCAN_REQ`). Together they use a maximum of 43 of the available 62 bytes across both packets.

---

### Primary Advertising Packet (31 bytes max, 25 used)

| Record | AD Type | Value | Bytes |
|--------|---------|-------|-------|
| Flags | `0x01` | LE General Discoverable, BR/EDR not supported (`0x06`) | 3 |
| Shortened Local Name | `0x08` | First 6 characters of the device name | 8 |
| Manufacturer Specific Data | `0xFF` | Company ID + Product ID + Protocol Version + MAC | 14 |
| **Total** | | | **25 / 31** |

The shortened name in `0x08` uses the first 6 characters of the full device name. The complete name is carried in the scan response.

### Manufacturer Specific Data (`0xFF`) — 14 bytes

| Field | Type | Bytes | Notes |
|-------|------|-------|-------|
| Company ID | `uint16` LE | 2 | `0x10C5` — our assigned company ID |
| Product ID | `uint16` LE | 2 | See Product ID table below |
| Protocol Version Major | `uint8` | 1 | JSON wrapper major version (e.g., `1`) |
| Protocol Version Minor | `uint8` | 1 | JSON wrapper minor version (e.g., `0`) |
| MAC Address | `bytes` | 6 | Device BLE MAC — included as a workaround for iOS hiding the MAC at the API level |

The advertised Major/Minor must match the `protocolVersion` reported in the JSON identity handshake (§3). At protocol `1.0`, Major = `1`, Minor = `0`.

### Company ID and Product ID

The company ID `0x10C5` namespaces all our devices, allowing scanners to ignore all third-party BLE traffic. The product ID identifies the specific product variant.

| Product ID | Product |
|------------|---------|
| `1` | SB600 |
| `2` | Model 195 |
| `3` | Model 225 |

---

### Scan Response Packet (31 bytes max, up to 18 used)

Returned only in response to an active `SCAN_REQ`. It carries the full device name, keeping the primary advertising packet compact.

| Record | AD Type | Value | Bytes |
|--------|---------|-------|-------|
| Complete Local Name | `0x09` | Full device name — max 16 characters | up to 18 |
| **Total** | | | **up to 18 / 31** |

---

### Device Filtering

Scanners should identify our devices using this sequence:

1. Check that a Manufacturer Specific record (`0xFF`) is present in the advertisement.
2. Verify the Company ID field equals `0x10C5` — discard all other advertisements silently.
3. Read the Product ID to determine the product variant.
4. Read Protocol Version Major/Minor to confirm payload format compatibility.

---

## 3. Device Identity and Initial Handshake

The initial handshake is a JSON-layer exchange that occurs immediately after a connection is established on a bidirectional transport. It is separate from BLE advertisement data and uses the standard `msg` / `cmd` / `ack` wrapper.

### Device-Initiated Identity Push

Upon connection, the device automatically sends an unsolicited `msg` containing its identity. The client does not need to request this — it arrives as the first message on the connected channel. (This is an unsolicited device-originated message and therefore applies only where the device can initiate; see §1.)

```json
{
  "msg": {
    "act": "Device",
    "Identity": {
      "protocolVersion": "1.0",
      "deviceName": "Model 225 Indicator",
      "firmwareVersion": "2.3.1",
      "ethernetMAC": "AA:BB:CC:DD:EE:FF",
      "wifiMAC": "AA:BB:CC:DD:EE:00"
    }
  }
}
```

### Identity Fields

| Field | Type | Description |
|-------|------|-------------|
| `protocolVersion` | string | Version of this JSON wrapper protocol (e.g., `"1.0"`) |
| `deviceName` | string | Human-readable device name (matches BLE local name) |
| `firmwareVersion` | string | Device firmware version string (e.g., `"2.3.1"`) |
| `ethernetMAC` | string | Ethernet MAC address in `"XX:XX:XX:XX:XX:XX"` format; `"N/A"` if no Ethernet interface |
| `wifiMAC` | string | Wi-Fi MAC address in `"XX:XX:XX:XX:XX:XX"` format; `"N/A"` if no Wi-Fi interface |

### Client-Requested Identity

The client may also explicitly request identity at any time using the standard `cmd` pattern:

```json
{"cmd":{"act":"Device","Identify":{}},"mID":1}
```

```json
{
  "ack": {
    "act": "Device",
    "Identify": {
      "protocolVersion": "1.0",
      "deviceName": "Model 225 Indicator",
      "firmwareVersion": "2.3.1",
      "ethernetMAC": "AA:BB:CC:DD:EE:FF",
      "wifiMAC": "AA:BB:CC:DD:EE:00"
    }
  },
  "mID": 1
}
```

### Versioning

The `protocolVersion` field allows clients to detect incompatible devices and adapt accordingly. Clients should check this field before sending any further commands. A mismatch in major version (e.g., client expects `"2.x"`, device reports `"1.x"`) should be treated as an incompatible device. The major/minor reported here must agree with the BLE advertisement's Protocol Version Major/Minor (§2).

---

## 4. Reserved Words

The following keys are reserved and may not be used as actor names, method names, or property names.

### Top-Level Containers
The only valid root keys in any message. Exactly one appears per message, and it is always present:

| Key | Direction | Description |
|-----|-----------|-------------|
| `cmd` | Client → Device | Command (method call or property get/set) |
| `ack` | Device → Client | Optional response to a `cmd` (see `mID`) |
| `msg` | Device → Client | Async status or telemetry |
| `err` | Device → Client | Async error notification |

### Actor
```
act  —  required in cmd, ack, msg, and err; identifies the target/subject subsystem
```
The `act` field is the first key inside the container. Actor names also define the menu namespace — the actor name and its menu category are the same thing.

### Operations
```
set  —  write one or more properties
get  —  read one or more properties
warn —  inline notification inside an ack (severity + string)
```

### Message Tracking
```
mID  —  optional message ID for ack correlation; carried at the top level of the
        root object, alongside the container key. An ack is sent only when the
        originating cmd carried an mID.
```

---

## 5. Message Structure

### Container Prototype
The container is mandatory. `mID`, when present, is a sibling of the container key at the root of the object.
```json
{"cmd":{"act":"actorName","methodName":{<args>}}}
{"cmd":{"act":"actorName","methodName":{<args>}},"mID":1232}
```

The root object therefore has at most two keys: the one container key, and optionally `mID`. Any key inside the container that is not `act` and not a reserved operation is interpreted as a method name or property name.

### Actor + Method Pattern
The method name is the key; its value is the argument object. An empty object `{}` signals no arguments required.
```json
{"cmd":{"act":"Scale","ZeroScale":{}}}
{"cmd":{"act":"Scale","Calibrate":{"ref_weight":50.0}}}
```

### Property Pattern
```json
{"cmd":{"act":"WIFI","set":{"SSID":"network_name"}}}
{"cmd":{"act":"WIFI","get":["SSID","Password"]}}
```

### Message ID
When `mID` is present on a `cmd`, the device sends a corresponding `ack`. Without `mID`, the command is fire and forget.
```json
{"cmd":{"act":"JFS","Format":{}}}
{"cmd":{"act":"JFS","Format":{}},"mID":1232}
{"ack":{"act":"JFS","Format":{}},"mID":1232}
```

---

## 6. Commands and Responses

### Fire and Forget
```json
{"cmd":{"act":"JFS","Format":{}}}
```

### Tracked Command / ACK Pair
```json
{"cmd":{"act":"JFS","Format":{}},"mID":1232}
{"ack":{"act":"JFS","Format":{}},"mID":1232}
```

### Method with Arguments
Input and output values may differ — the `ack` always reflects the actual applied value:
```json
{"cmd":{"act":"Scale","Setpoint":{"target":500.000002,"unit":"lbs","tolerance":2.5}},"mID":1232}
{"ack":{"act":"Scale","Setpoint":{"target":500,"unit":"lbs","tolerance":2.5}},"mID":1232}
```

### Property Get / Set
```json
{"cmd":{"act":"WIFI","set":{"SSID":"JMF's iPhone","Password":"12345678"}},"mID":1232}
{"ack":{"act":"WIFI","set":{"SSID":"JMF's iPhone","Password":"12345678"}},"mID":1232}

{"cmd":{"act":"WIFI","get":["SSID","Password"]},"mID":1232}
{"ack":{"act":"WIFI","get":{"SSID":"JMF's iPhone","Password":"12345678"}},"mID":1232}
```

---

## 7. Async Messages

Async messages originate from the device without a prior command. Per the initiation rule (§1), they are valid on any transport where the device can initiate: always on socket-class transports (UART, BLE, TCP), as an HTTP request body on HTTP, and as a device-to-cloud message on Azure IoT Hub. The message body is byte-identical regardless of how it is carried.

### Telemetry (`msg`)
```json
{"msg":{"act":"ESP","ESP":{"heap":"21.8k","UART":358,"NTP_PD":1434322}}}
```

### Error Notification (`err`)
```json
{"err":{"act":"WIFI","WIFI":{"errorCode":12,"reason":"Association Leave"}}}
```

---

## 8. Inline ACK Notification

A `warn` key may be included in any `ack` to notify the client of a condition that arose while processing the `cmd`. The protocol describes what happened; the client decides how to present it.

### Severity Levels

| Severity | Meaning |
|----------|---------|
| `info` | Informational only, no action required |
| `warn` | Value was adjusted or a condition exists worth surfacing |
| `error` | Command was rejected or only partially applied |

### Value Clamped to Maximum
```json
{"cmd":{"act":"WIFI","set":{"timeout":2.222222222}},"mID":1232}
{"ack":{"act":"WIFI","set":{"timeout":2.0},"warn":{"severity":"warn","string":"Value clamped to maximum of 2.0"}},"mID":1232}
```

### Command Rejected
```json
{"cmd":{"act":"Scale","Setpoint":{"target":99999,"unit":"lbs","tolerance":2.5}},"mID":1233}
{"ack":{"act":"Scale","Setpoint":{"target":99999,"unit":"lbs","tolerance":2.5},"warn":{"severity":"error","string":"Target exceeds scale capacity"}},"mID":1233}
```

---

## 9. Extended Examples

### 9.1 Scale Telemetry
Device-initiated push of four load cell voltages and current weight values:
```json
{"msg":{"act":"Scale","Scale":{"lc1":"1.832v","lc2":"1.847v","lc3":"1.809v","lc4":"1.821v","gross":"482.6lbs","net":"431.2lbs"}}}
```

### 9.2 Batching Operation
Method call with structured arguments across multiple platforms:
```json
{"cmd":{"act":"Batching","Start":{
    "targets": {"type":"platforms","value":[1,2,4]},
    "config":  {"fn":"accumulate","unit":"lbs","interval":"12sec"},
    "options": {"setpoint":500,"tolerance":2}
}},"mID":1233}
```

### 9.3 Menu System

The menu is a hierarchical directory of actors. Each actor name is both the command target and its menu category — they are the same namespace. The menu system is a specialized expansion of the GET PROPERTIES pattern; no new protocol concepts are introduced.

**Payload shape note.** `MENU` is the one actor whose `ack` payload is a JSON **array** rather than an object — it returns a list of menu entries. The single parser (§1) is unaffected: it still reads the root key, then `act`, then the value behind the remaining key. Parsers must accept that this value is an array when `act` is `MENU`, and an object otherwise. No other actor returns an array payload.

#### Navigation
Request the root menu to discover available actors:
```json
{"cmd":{"act":"MENU","Root":{}},"mID":1232}
{"ack":{"act":"MENU","Root":[
    {"name":"Scale","type":"menuItem"},
    {"name":"WIFI","type":"menuItem"},
    {"name":"OTA","type":"menuItem"}
]},"mID":1232}
```

Drill into an actor to retrieve its variables and methods:
```json
{"cmd":{"act":"MENU","Scale":{}},"mID":1233}
{"ack":{"act":"MENU","Scale":[

    {
        "name":     "Unit",
        "tip":      "Weighing unit",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["lbs", "kg"],
        "value":    "lbs"
    },
    {
        "name":     "Decimal Places",
        "tip":      "Weight display precision",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "max":      4,
        "value":    2
    },
    {
        "name":     "Zero Scale",
        "tip":      "Zero the live weight",
        "type":     "method",
        "params":   {},
        "response": {
            "zeroed": {"dataType":"bool"}
        }
    },
    {
        "name":     "Calibrate",
        "tip":      "Calibrate with a known reference weight",
        "type":     "method",
        "params":   {
            "ref_weight": {"dataType":"float","unit":"lbs","min":1,"max":5000,"default":50.0}
        },
        "response": {
            "correction": {"dataType":"float"},
            "status":     {"dataType":"string"}
        }
    }

],"mID":1233}}
```

#### Menu Item Types

| Type | Description |
|------|-------------|
| `menuItem` | Navigable submenu / actor category |
| `Variable` | Read/write property (`dataType`, `min`, `max`, `value`) |
| `readOnlyVariable` | Display-only value (`dataType`, `value`) |
| `method` | Callable action with `params` and `response` prototypes |

#### Executing a Discovered Method
The menu item defines exactly how to build the `cmd`. The method `name` becomes the cmd key (spaces removed); `params` defines the argument object; `response` predicts the `ack` payload.

Simple method (no params):
```json
{"cmd":{"act":"Scale","ZeroScale":{}},"mID":1234}
{"ack":{"act":"Scale","ZeroScale":{"zeroed":true}},"mID":1234}
```

Advanced method (with params):
```json
{"cmd":{"act":"Scale","Calibrate":{"ref_weight":50.0}},"mID":1235}
{"ack":{"act":"Scale","Calibrate":{"correction":0.023,"status":"OK"}},"mID":1235}
```

#### Response Size and Datagram Transports
A full menu actor response can be large. On stream transports (UART, BLE, TCP) and on HTTP / IoT Hub this is a non-issue — the framing or body carries arbitrary length. On **UDP**, a response that exceeds a single datagram (or the negotiated path MTU) cannot be delivered as one message. Where menu retrieval must run over UDP, either keep individual actor responses within one datagram or use a stream transport for menu operations. The protocol does not define application-level reassembly of a single oversized response.

### 9.4 Firmware OTA Chunk Transfer
Chunked binary file transfer for firmware upgrades. The sender waits for an `ack` on each chunk before sending the next, providing natural flow control without a stream abstraction.

```json
{"cmd":{"act":"OTA","Chunk":{"seq":42,"total":200,"size":512,"data":"3f8a02...c4"}},"mID":1233}
{"ack":{"act":"OTA","Chunk":{"seq":42}},"mID":1233}
```

| Field | Description |
|-------|-------------|
| `seq` | Chunk sequence number |
| `total` | Total number of chunks |
| `size` | Byte count of this chunk |
| `data` | Hex-encoded binary payload |

#### Reliability by Transport
The per-chunk `ack` means OTA carries its own delivery guarantee and does not depend on the transport being reliable:

- **Reliable, ordered transports (UART, BLE, TCP, IoT Hub):** the per-chunk `ack` is pure flow control. Order and delivery are already guaranteed by the transport; the sender simply paces itself to one outstanding chunk.
- **UDP (best-effort):** the per-chunk `ack` becomes the reliability mechanism. The sender MUST start a retransmit timer when it sends a chunk, retransmit the same `seq` if no `ack` arrives before the timer expires, and abort the transfer after a bounded number of retries. Choose a chunk `size` that keeps the datagram within the path MTU to avoid IP fragmentation.

#### Duplicate Handling (Idempotency)
Because a retransmit can cause a chunk to arrive twice (e.g., when an `ack` was lost rather than the chunk), the receiver MUST treat a `seq` it has already applied idempotently: re-send the `ack` for that `seq` and do **not** write the chunk again. This makes the transfer safe under any number of retransmissions.

---

> **Appendix A — Model 225 Indicator: Complete Menu Messages** follows below, unchanged from revision v23. It is an illustrative model of one product's menu tree and is not affected by the transport/framing changes in this revision.

# Appendix A — Model 225 Indicator: Complete Menu Messages

This appendix models the complete menu structure of the Model 225 Desktop Indicator
firmware (`195_225_nxp_app_2`). Each section shows the `cmd` request and the full
`ack` response for that actor. Settings that only appear under certain conditions
include a `condition` field. Read-only values use `"type":"readOnlyVariable"`.

---

## A.1 Root Menu

```json
{"cmd":{"act":"MENU","Root":{}}}

{"ack":{"act":"MENU","Root":[
    {"name":"IndicatorSetup",      "type":"menuItem","tip":"Indicator configuration, clock, and mode settings"},
    {"name":"ScaleSetup",          "type":"menuItem","tip":"Scale measurement and filter settings"},
    {"name":"ScaleCalibration",    "type":"menuItem","tip":"Span, zero, and load cell calibration"},
    {"name":"LoadCellAssignments", "type":"menuItem","tip":"DLC load cell assignment and trim (DLC card only)","condition":"ScaleType=DLC"},
    {"name":"ComSetup",            "type":"menuItem","tip":"Serial, Ethernet, and Wi-Fi communication settings"},
    {"name":"PrinterSetup",        "type":"menuItem","tip":"Ticket printer port and print tab configuration"},
    {"name":"SystemConfig",        "type":"menuItem","tip":"Accumulators, password, DAC, key lockout, and badge reader"},
    {"name":"ModeConfig",          "type":"menuItem","tip":"Settings for the active mode of operation"},
    {"name":"DLCSetup",            "type":"menuItem","tip":"DLC cell diagnostics and SNAP communication (DLC card only)","condition":"ScaleType=DLC"},
    {"name":"ReviewMenu",          "type":"menuItem","tip":"Scale ID and read-only calibration and configuration counters"}
]}}
```

---

## A.2 Indicator Setup

```json
{"cmd":{"act":"MENU","IndicatorSetup":{}}}

{"ack":{"act":"MENU","IndicatorSetup":[

    {
        "name":     "USA",
        "tip":      "Enable USA legal-for-trade mode",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    },
    {
        "name":     "NSC",
        "tip":      "National Scale Count (always N/A)",
        "type":     "readOnlyVariable",
        "dataType": "string",
        "value":    "N/A"
    },
    {
        "name":     "LFT",
        "tip":      "Legal For Trade",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":      "OIML",
        "tip":       "OIML compliance mode",
        "type":      "Variable",
        "dataType":  "bool",
        "value":     false,
        "condition": "USA=false"
    },
    {
        "name":     "TimeFormat",
        "tip":      "Clock display format",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["12H", "24H"],
        "value":    "12H"
    },
    {
        "name":     "DateFormat",
        "tip":      "Date display format",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["MM-DD-YYYY", "DD-MM-YYYY"],
        "value":    "MM-DD-YYYY"
    },
    {
        "name":     "ConsecutiveNumber",
        "tip":      "Starting consecutive ticket number",
        "type":     "Variable",
        "dataType": "uint32",
        "min":      0,
        "value":    1
    },
    {
        "name":      "ClearTare",
        "tip":       "Clear tare on power up",
        "type":      "Variable",
        "dataType":  "bool",
        "value":     false,
        "condition": "CalSealed=YES, USA=YES, LFT=NO"
    },
    {
        "name":      "ClearID",
        "tip":       "Clear ID on power up",
        "type":      "Variable",
        "dataType":  "bool",
        "value":     false,
        "condition": "CalSealed=YES, USA=YES, LFT=NO"
    },
    {
        "name":      "NumberOfScales",
        "tip":       "Number of active scales",
        "type":      "Variable",
        "dataType":  "uint8",
        "min":       1,
        "max":       3,
        "value":     1,
        "condition": "ScaleCard != SingleScaleAnalog"
    },
    {
        "name":      "Totalizer",
        "tip":       "Enable multi-scale totalizer",
        "type":      "Variable",
        "dataType":  "bool",
        "value":     false,
        "condition": "NumberOfScales > 1"
    },
    {
        "name":     "ModeOfOperation",
        "tip":      "Active weighing mode",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Normal", "IDStorage", "Batcher", "DFC", "PackageWeigher", "AxleWeigher", "CheckWeigher", "PWC", "Livestock"],
        "value":    "Normal"
    },
    {
        "name":     "SetTime",
        "tip":      "Set the indicator real-time clock",
        "type":     "method",
        "params":   {
            "time": {"dataType":"string","format":"HH:MM:SS"}
        },
        "response": {
            "time": {"dataType":"string"}
        }
    },
    {
        "name":     "SetDate",
        "tip":      "Set the indicator date",
        "type":     "method",
        "params":   {
            "date": {"dataType":"string","format":"MM-DD-YYYY"}
        },
        "response": {
            "date": {"dataType":"string"}
        }
    }

]}}
```

---

## A.3 Scale Setup

```json
{"cmd":{"act":"MENU","ScaleSetup":{}}}

{"ack":{"act":"MENU","ScaleSetup":[

    {
        "name":     "PrimaryUnits",
        "tip":      "Primary weighing unit",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["lb", "kg"],
        "value":    "lb"
    },
    {
        "name":     "SecondaryUnits",
        "tip":      "Secondary weighing unit",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["lb", "kg"],
        "value":    "kg"
    },
    {
        "name":     "ZeroTracking",
        "tip":      "Automatic zero tracking range in display divisions",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["0.0d", "0.5d", "1.0d", "2.0d", "3.0d"],
        "value":    "0.0d"
    },
    {
        "name":     "ZeroLimit",
        "tip":      "Restrict zero key to within 2% of capacity",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "PowerUpZero",
        "tip":      "Automatically zero on power up",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "SampleRate",
        "tip":      "A/D conversions per second",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      1,
        "max":      16,
        "value":    4
    },
    {
        "name":     "MotionRange",
        "tip":      "Motion detection threshold in display divisions",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      1,
        "max":      99,
        "value":    3
    },
    {
        "name":     "StableCount",
        "tip":      "Consecutive stable readings required before weight is accepted",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      1,
        "max":      99,
        "value":    3
    },
    {
        "name":     "WeightIntervals",
        "tip":      "Single or dual range weighing",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      1,
        "max":      2,
        "value":    1
    },
    {
        "name":     "ScaleType",
        "tip":      "Scale hardware type",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Analog", "DLC", "Remote"],
        "value":    "Analog"
    },
    {
        "name":     "FilterType",
        "tip":      "Digital filter algorithm",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["None", "Average", "Median"],
        "value":    "None"
    },
    {
        "name":     "FilterLevel",
        "tip":      "Filter strength",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      1,
        "max":      9,
        "value":    2
    },
    {
        "name":     "FilterBreak",
        "tip":      "Filter break range in display divisions",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      1,
        "max":      9,
        "value":    1
    },
    {
        "name":      "Interval",
        "tip":       "Display graduation size (single range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "min":       1,
        "value":     10,
        "condition": "WeightIntervals=1"
    },
    {
        "name":      "DecimalPlace",
        "tip":       "Display decimal position (single range)",
        "type":      "Variable",
        "dataType":  "uint8",
        "min":       0,
        "max":       4,
        "value":     0,
        "condition": "WeightIntervals=1"
    },
    {
        "name":      "Capacity",
        "tip":       "Maximum scale capacity (single range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "min":       1,
        "value":     50000,
        "condition": "WeightIntervals=1"
    },
    {
        "name":      "LowInterval",
        "tip":       "Low range graduation size (dual range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "min":       1,
        "value":     10,
        "condition": "WeightIntervals=2"
    },
    {
        "name":      "LowDecimalPlace",
        "tip":       "Low range decimal position (dual range)",
        "type":      "Variable",
        "dataType":  "uint8",
        "min":       0,
        "max":       4,
        "value":     0,
        "condition": "WeightIntervals=2"
    },
    {
        "name":      "LowCapacity",
        "tip":       "Low range capacity (dual range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "min":       1,
        "value":     50000,
        "condition": "WeightIntervals=2"
    },
    {
        "name":      "HighInterval",
        "tip":       "High range graduation size (dual range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "min":       1,
        "value":     10,
        "condition": "WeightIntervals=2"
    },
    {
        "name":      "HighDecimalPlace",
        "tip":       "High range decimal position (dual range)",
        "type":      "Variable",
        "dataType":  "uint8",
        "min":       0,
        "max":       4,
        "value":     0,
        "condition": "WeightIntervals=2"
    },
    {
        "name":      "HighCapacity",
        "tip":       "High range capacity (dual range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "min":       1,
        "value":     100000,
        "condition": "WeightIntervals=2"
    },
    {
        "name":      "PrelimFilterCount",
        "tip":       "Preliminary filter sample count (DLC only)",
        "type":      "Variable",
        "dataType":  "uint8",
        "min":       0,
        "value":     0,
        "condition": "ScaleType=DLC"
    }

]}}
```

---

## A.4 Scale Calibration

```json
{"cmd":{"act":"MENU","ScaleCalibration":{}}}

{"ack":{"act":"MENU","ScaleCalibration":[

    {
        "name":      "SpanWeight",
        "tip":       "Known reference weight used for span calibration (Analog)",
        "type":      "Variable",
        "dataType":  "float",
        "min":       0,
        "condition": "ScaleType=Analog"
    },
    {
        "name":      "SpanCount",
        "tip":       "A/D count at span weight (hardware-derived, read only)",
        "type":      "readOnlyVariable",
        "dataType":  "uint32",
        "condition": "ScaleType=Analog"
    },
    {
        "name":      "ZeroCount",
        "tip":       "A/D count at zero (hardware-derived, read only)",
        "type":      "readOnlyVariable",
        "dataType":  "uint32",
        "condition": "ScaleType=Analog"
    },
    {
        "name":      "LC1",
        "tip":       "Load cell 1 A/D count (single range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "value":     1000,
        "condition": "ScaleType=Analog, WeightIntervals=1"
    },
    {
        "name":      "LC2",
        "tip":       "Load cell 2 A/D count (single range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "value":     1000,
        "condition": "ScaleType=Analog, WeightIntervals=1"
    },
    {
        "name":      "LC3",
        "tip":       "Load cell 3 A/D count (single range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "value":     1000,
        "condition": "ScaleType=Analog, WeightIntervals=1"
    },
    {
        "name":      "LC4",
        "tip":       "Load cell 4 A/D count (single range)",
        "type":      "Variable",
        "dataType":  "uint32",
        "value":     1000,
        "condition": "ScaleType=Analog, WeightIntervals=1"
    },
    {
        "name":      "CalibrateSpan",
        "tip":       "Perform analog span calibration with a known reference weight",
        "type":      "method",
        "params":    {
            "span_weight": {"dataType":"float","unit":"lb","min":1,"max":50000,"default":50.0}
        },
        "response":  {
            "span_count": {"dataType":"uint32"},
            "zero_count": {"dataType":"uint32"},
            "status":     {"dataType":"string"}
        },
        "condition": "ScaleType=Analog"
    },
    {
        "name":      "SmartCalibration",
        "tip":       "Interactive DLC calibration wizard",
        "type":      "method",
        "params":    {},
        "response":  {
            "status": {"dataType":"string"}
        },
        "condition": "ScaleType=DLC"
    },
    {
        "name":      "ZeroCalibration",
        "tip":       "Perform DLC zero calibration",
        "type":      "method",
        "params":    {},
        "response":  {
            "zero_count": {"dataType":"uint32"},
            "status":     {"dataType":"string"}
        },
        "condition": "ScaleType=DLC"
    },
    {
        "name":      "CellTrimming",
        "tip":       "Per-cell trim factor (DLC)",
        "type":      "Variable",
        "dataType":  "float",
        "min":       0.1,
        "max":       10.0,
        "value":     1.0,
        "condition": "ScaleType=DLC"
    },
    {
        "name":      "SpanAdjustment",
        "tip":       "Interactive DLC span adjustment",
        "type":      "method",
        "params":    {
            "span_weight": {"dataType":"float","unit":"lb","min":1,"max":50000,"default":50.0}
        },
        "response":  {
            "correction": {"dataType":"float"},
            "status":     {"dataType":"string"}
        },
        "condition": "ScaleType=DLC"
    }

]}}
```

---

## A.5 Load Cell Assignments (DLC only)

```json
{"cmd":{"act":"MENU","LoadCellAssignments":{}}}

{"ack":{"act":"MENU","LoadCellAssignments":[

    {
        "name":     "CellID",
        "tip":      "Load cell hardware identifier",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "value":    0
    },
    {
        "name":     "CellToScale",
        "tip":      "Scale assignment for this cell",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Scale1", "Scale2", "Scale3"],
        "value":    "Scale1"
    },
    {
        "name":     "CellsPerScale",
        "tip":      "Number of cells assigned to this scale",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "value":    0
    },
    {
        "name":     "CellTrim",
        "tip":      "Individual cell trim factor",
        "type":     "Variable",
        "dataType": "float",
        "min":      0.1,
        "max":      10.0,
        "value":    1.0
    }

]}}
```

---

## A.6 Com Setup

```json
{"cmd":{"act":"MENU","ComSetup":{}}}

{"ack":{"act":"MENU","ComSetup":[
    {"name":"SerialPorts","type":"menuItem","tip":"COM1, COM2, USB-B, OPC1, OPC2 serial port settings"},
    {"name":"Ethernet",   "type":"menuItem","tip":"Ethernet network settings"},
    {"name":"WiFi",       "type":"menuItem","tip":"Wi-Fi network settings"},
    {"name":"BankMode",   "type":"menuItem","tip":"Bank mode configuration"},
    {"name":"ISiteIP",    "type":"menuItem","tip":"iSite IP settings (DLC only)","condition":"ScaleType=DLC"},
    {"name":"SendGross",  "type":"menuItem","tip":"Send gross weight output (non-DLC only)","condition":"ScaleType!=DLC"}
]}}
```

### A.6.1 Serial Ports

```json
{"cmd":{"act":"MENU","SerialPorts":{}}}

{"ack":{"act":"MENU","SerialPorts":[

    {
        "name":     "Type",
        "tip":      "Serial protocol type",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["SMA", "SB600", "Continuous", "Disabled"],
        "value":    "SMA"
    },
    {
        "name":     "BaudRate",
        "tip":      "Serial baud rate",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["1200", "2400", "4800", "9600", "19200", "38400", "57600", "115200"],
        "value":    "9600"
    },
    {
        "name":     "DataBits",
        "tip":      "Number of data bits",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["7", "8"],
        "value":    "8"
    },
    {
        "name":     "StopBits",
        "tip":      "Number of stop bits",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["1", "2"],
        "value":    "1"
    },
    {
        "name":     "Parity",
        "tip":      "Parity bit setting",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["None", "Even", "Odd"],
        "value":    "None"
    },
    {
        "name":     "TransferCondition",
        "tip":      "When to transmit data",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Continuous", "OnPrint", "OnStable", "OnDemand"],
        "value":    "Continuous"
    },
    {
        "name":     "Scale",
        "tip":      "Scale channel to transmit",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["CurrentSelected", "Scale1", "Scale2", "Scale3"],
        "value":    "CurrentSelected"
    },
    {
        "name":     "GrossOnly",
        "tip":      "Transmit gross weight only",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "ManualMode",
        "tip":      "Enable manual print mode",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "MessageSlot",
        "tip":      "Assigned message slot",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Disabled", "Slot1", "Slot2", "Slot3"],
        "value":    "Disabled"
    },
    {
        "name":      "ElectricalInterface",
        "tip":       "Electrical interface standard (COM2 only)",
        "type":      "Variable",
        "dataType":  "enum",
        "values":    ["RS232", "RS485", "RS422"],
        "value":     "RS232",
        "condition": "Port=COM2"
    },
    {
        "name":     "SBHighThreshold",
        "tip":      "SB-series high threshold",
        "type":     "Variable",
        "dataType": "uint32",
        "min":      0,
        "value":    1500
    }

]}}
```

### A.6.2 Ethernet

```json
{"cmd":{"act":"MENU","Ethernet":{}}}

{"ack":{"act":"MENU","Ethernet":[

    {
        "name":     "EthernetEnable",
        "tip":      "Enable Ethernet interface",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "DHCP",
        "tip":      "Use DHCP for IP address assignment",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    },
    {
        "name":      "IPAddress",
        "tip":       "Static IP address",
        "type":      "Variable",
        "dataType":  "string",
        "value":     "192.168.4.138",
        "condition": "DHCP=false"
    },
    {
        "name":      "Gateway",
        "tip":       "Default gateway address",
        "type":      "Variable",
        "dataType":  "string",
        "value":     "192.168.4.1",
        "condition": "DHCP=false"
    },
    {
        "name":      "Subnet",
        "tip":       "Subnet mask",
        "type":      "Variable",
        "dataType":  "string",
        "value":     "255.255.255.0",
        "condition": "DHCP=false"
    },
    {
        "name":     "ServerPortA",
        "tip":      "Server port A",
        "type":     "Variable",
        "dataType": "uint16",
        "min":      1,
        "max":      65535,
        "value":    10010
    },
    {
        "name":     "ServerPortB",
        "tip":      "Server port B",
        "type":     "Variable",
        "dataType": "uint16",
        "min":      1,
        "max":      65535,
        "value":    10011
    },
    {
        "name":     "ServerPortC",
        "tip":      "Server port C",
        "type":     "Variable",
        "dataType": "uint16",
        "min":      1,
        "max":      65535,
        "value":    10012
    },
    {
        "name":     "ClientServerPort",
        "tip":      "Client server port",
        "type":     "Variable",
        "dataType": "uint16",
        "min":      1,
        "max":      65535,
        "value":    10010
    },
    {
        "name":     "PortType",
        "tip":      "Protocol type for all Ethernet ports",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["SB600", "SMA", "Disabled"],
        "value":    "SB600"
    },
    {
        "name":     "PortThreshold",
        "tip":      "SB-series threshold for all Ethernet ports",
        "type":     "Variable",
        "dataType": "uint32",
        "min":      0,
        "value":    1500
    },
    {
        "name":     "MessageSlot",
        "tip":      "Assigned message slot for all Ethernet ports",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Disabled", "Slot1", "Slot2", "Slot3"],
        "value":    "Disabled"
    }

]}}
```

### A.6.3 Wi-Fi

```json
{"cmd":{"act":"MENU","WiFi":{}}}

{"ack":{"act":"MENU","WiFi":[

    {
        "name":     "WiFiEnable",
        "tip":      "Enable Wi-Fi interface",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    },
    {
        "name":     "DHCP",
        "tip":      "Use DHCP for IP address assignment",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    },
    {
        "name":     "SSID",
        "tip":      "Wi-Fi network name",
        "type":     "Variable",
        "dataType": "string",
        "value":    "195"
    },
    {
        "name":     "Password",
        "tip":      "Wi-Fi network password",
        "type":     "Variable",
        "dataType": "string",
        "value":    "12345678"
    },
    {
        "name":     "PortType",
        "tip":      "Protocol type for all Wi-Fi ports",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["SB600", "SMA", "Disabled"],
        "value":    "SB600"
    },
    {
        "name":     "PortThreshold",
        "tip":      "SB-series threshold for all Wi-Fi ports",
        "type":     "Variable",
        "dataType": "uint32",
        "min":      0,
        "value":    1500
    },
    {
        "name":     "MessageSlot",
        "tip":      "Assigned message slot for all Wi-Fi ports",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Disabled", "Slot1", "Slot2", "Slot3"],
        "value":    "Disabled"
    }

]}}
```

### A.6.4 iSite IP (DLC only)

```json
{"cmd":{"act":"MENU","ISiteIP":{}}}

{"ack":{"act":"MENU","ISiteIP":[

    {
        "name":     "SiteOrder",
        "tip":      "Site/Order identifier",
        "type":     "Variable",
        "dataType": "string",
        "value":    ""
    },
    {
        "name":     "DHCP",
        "tip":      "Use DHCP",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    },
    {
        "name":      "IPAddress",
        "tip":       "Static IP address",
        "type":      "Variable",
        "dataType":  "string",
        "value":     "192.168.1.70",
        "condition": "DHCP=false"
    },
    {
        "name":      "Subnet",
        "tip":       "Subnet mask",
        "type":      "Variable",
        "dataType":  "string",
        "value":     "255.255.255.0",
        "condition": "DHCP=false"
    },
    {
        "name":      "Gateway",
        "tip":       "Default gateway",
        "type":      "Variable",
        "dataType":  "string",
        "value":     "192.168.1.1",
        "condition": "DHCP=false"
    },
    {
        "name":     "DNS1",
        "tip":      "Primary DNS server",
        "type":     "Variable",
        "dataType": "string",
        "value":    "8.8.8.8"
    },
    {
        "name":     "DNS2",
        "tip":      "Secondary DNS server",
        "type":     "Variable",
        "dataType": "string",
        "value":    "8.8.4.4"
    },
    {
        "name":     "EnableDNS",
        "tip":      "Enable DNS resolution",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    }

]}}
```

### A.6.5 Send Gross (non-DLC only)

```json
{"cmd":{"act":"MENU","SendGross":{}}}

{"ack":{"act":"MENU","SendGross":[

    {
        "name":     "SendGrossEnable",
        "tip":      "Enable gross weight output",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "GrossWeightPort",
        "tip":      "Port for gross weight output",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Com1", "Com2", "USB", "OPC1", "OPC2"],
        "value":    "Com1"
    }

]}}
```

---

## A.7 Printer Setup

```json
{"cmd":{"act":"MENU","PrinterSetup":{}}}

{"ack":{"act":"MENU","PrinterSetup":[

    {
        "name":     "Port",
        "tip":      "Printer output port",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Com1", "Com2", "USB", "OPC1", "OPC2"],
        "value":    "Com1"
    },
    {
        "name":     "AutoLF",
        "tip":      "Automatically add line feed after each line",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "EndingLF",
        "tip":      "Number of line feeds at end of ticket",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "max":      99,
        "value":    0
    },
    {
        "name":     "EndOfLine",
        "tip":      "Line terminator character (hex)",
        "type":     "Variable",
        "dataType": "string",
        "value":    "0D"
    },
    {
        "name":     "StartOfTicket",
        "tip":      "String sent at start of ticket",
        "type":     "Variable",
        "dataType": "string",
        "value":    ""
    },
    {
        "name":     "EndOfTicket",
        "tip":      "String sent at end of ticket",
        "type":     "Variable",
        "dataType": "string",
        "value":    ""
    },
    {
        "name":     "EndOfTicketLineFeeds",
        "tip":      "Line feeds appended at end of ticket",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "max":      99,
        "value":    0
    },
    {
        "name":     "PrintSlot",
        "tip":      "Active print tab slot",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "value":    0
    },
    {
        "name":     "TimeTab",
        "tip":      "Print tab position for time field (RR.CC = row.column)",
        "type":     "Variable",
        "dataType": "float",
        "value":    1.0
    },
    {
        "name":     "DateTab",
        "tip":      "Print tab position for date field",
        "type":     "Variable",
        "dataType": "float",
        "value":    2.0
    },
    {
        "name":     "ConsecutiveTab",
        "tip":      "Print tab position for consecutive number field",
        "type":     "Variable",
        "dataType": "float",
        "value":    6.0
    },
    {
        "name":     "GrossTab",
        "tip":      "Print tab position for gross weight field",
        "type":     "Variable",
        "dataType": "float",
        "value":    7.0
    },
    {
        "name":     "TareTab",
        "tip":      "Print tab position for tare field",
        "type":     "Variable",
        "dataType": "float",
        "value":    8.0
    },
    {
        "name":     "NetTab",
        "tip":      "Print tab position for net weight field",
        "type":     "Variable",
        "dataType": "float",
        "value":    9.0
    },
    {
        "name":     "GrossAccumTab",
        "tip":      "Print tab position for gross accumulator",
        "type":     "Variable",
        "dataType": "float",
        "value":    0.0
    },
    {
        "name":     "NetAccumTab",
        "tip":      "Print tab position for net accumulator",
        "type":     "Variable",
        "dataType": "float",
        "value":    0.0
    },
    {
        "name":     "IDTab",
        "tip":      "Print tab position for ID field",
        "type":     "Variable",
        "dataType": "float",
        "value":    3.05
    }

]}}
```

---

## A.8 System Config

```json
{"cmd":{"act":"MENU","SystemConfig":{}}}

{"ack":{"act":"MENU","SystemConfig":[

    {"name":"Accumulators","type":"menuItem","tip":"Gross and net accumulators per scale"},
    {"name":"DACOutput",   "type":"menuItem","tip":"Analog DAC output configuration"},
    {"name":"KeyLockout",  "type":"menuItem","tip":"Individual key lockout settings"},
    {"name":"BadgeReader", "type":"menuItem","tip":"Badge reader port and type settings"},
    {"name":"WINVRS",      "type":"menuItem","tip":"WINVRS computer interface settings (if enabled in firmware)"}

]}}
```

### A.8.1 Accumulators

```json
{"cmd":{"act":"MENU","Accumulators":{}}}

{"ack":{"act":"MENU","Accumulators":[

    {
        "name":     "GenAccums",
        "tip":      "Enable gross accumulator",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    },
    {
        "name":     "AccumulatorScale1",
        "tip":      "Accumulated weight for Scale 1 (read only)",
        "type":     "readOnlyVariable",
        "dataType": "float"
    },
    {
        "name":      "AccumulatorScale2",
        "tip":       "Accumulated weight for Scale 2 (read only)",
        "type":      "readOnlyVariable",
        "dataType":  "float",
        "condition": "NumberOfScales >= 2"
    },
    {
        "name":      "AccumulatorScale3",
        "tip":       "Accumulated weight for Scale 3 (read only)",
        "type":      "readOnlyVariable",
        "dataType":  "float",
        "condition": "NumberOfScales >= 3"
    },
    {
        "name":      "TotalizerAccumulator",
        "tip":       "Multi-scale totalizer accumulator (read only)",
        "type":      "readOnlyVariable",
        "dataType":  "float",
        "condition": "Totalizer=true"
    },
    {
        "name":     "Password",
        "tip":      "Setup menu access password",
        "type":     "Variable",
        "dataType": "string",
        "value":    ""
    },
    {
        "name":      "LRPort",
        "tip":       "Remote scale communication port",
        "type":      "Variable",
        "dataType":  "enum",
        "values":    ["Com1", "Com2", "USB", "OPC1", "OPC2"],
        "value":     "Com1",
        "condition": "ScaleType=Remote"
    },
    {
        "name":     "ClearAccumulators",
        "tip":      "Clear all accumulator values",
        "type":     "method",
        "params":   {},
        "response": {
            "cleared": {"dataType":"bool"}
        }
    }

]}}
```

### A.8.2 DAC Output

```json
{"cmd":{"act":"MENU","DACOutput":{}}}

{"ack":{"act":"MENU","DACOutput":[

    {
        "name":     "DACGross",
        "tip":      "Output gross weight (vs net)",
        "type":     "Variable",
        "dataType": "bool",
        "value":    true
    },
    {
        "name":     "DACLowWeight",
        "tip":      "Weight value corresponding to DAC minimum output",
        "type":     "Variable",
        "dataType": "float",
        "value":    0
    },
    {
        "name":     "DACHighWeight",
        "tip":      "Weight value corresponding to DAC maximum output",
        "type":     "Variable",
        "dataType": "float",
        "value":    50000
    },
    {
        "name":     "DACVoltOutput",
        "tip":      "Maximum DAC voltage output",
        "type":     "Variable",
        "dataType": "float",
        "min":      0,
        "max":      10.0,
        "value":    10.0
    },
    {
        "name":     "AdjustHigh",
        "tip":      "DAC high-end trim adjustment",
        "type":     "Variable",
        "dataType": "int32",
        "value":    0
    },
    {
        "name":     "AdjustLow",
        "tip":      "DAC low-end trim adjustment",
        "type":     "Variable",
        "dataType": "int32",
        "value":    0
    },
    {
        "name":     "DACScale",
        "tip":      "Scale channel for DAC output",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Scale1", "Scale2", "Scale3"],
        "value":    "Scale1"
    }

]}}
```

### A.8.3 Key Lockout

```json
{"cmd":{"act":"MENU","KeyLockout":{}}}

{"ack":{"act":"MENU","KeyLockout":[

    {"name":"ZeroKeyLock",    "tip":"Lock Zero key",       "type":"Variable","dataType":"bool","value":false},
    {"name":"TareKeyLock",    "tip":"Lock Tare key",       "type":"Variable","dataType":"bool","value":false},
    {"name":"NetKeyLock",     "tip":"Lock Net key",        "type":"Variable","dataType":"bool","value":false},
    {"name":"PrintKeyLock",   "tip":"Lock Print key",      "type":"Variable","dataType":"bool","value":false},
    {"name":"UnitKeyLock",    "tip":"Lock Unit key",       "type":"Variable","dataType":"bool","value":false},
    {"name":"GreenKeyLock",   "tip":"Lock Green key",      "type":"Variable","dataType":"bool","value":false},
    {"name":"KeypadLock",     "tip":"Lock entire keypad",  "type":"Variable","dataType":"bool","value":false},
    {"name":"IDKeyLock",      "tip":"Lock ID key",         "type":"Variable","dataType":"bool","value":false},
    {"name":"CountKeyLock",   "tip":"Lock Count key",      "type":"Variable","dataType":"bool","value":false},
    {"name":"MemKeyLock",     "tip":"Lock Memory key",     "type":"Variable","dataType":"bool","value":false},
    {"name":"PresetKeyLock",  "tip":"Lock Preset key",     "type":"Variable","dataType":"bool","value":false},
    {"name":"DeleteKeyLock",  "tip":"Lock Delete key",     "type":"Variable","dataType":"bool","value":false},
    {"name":"StartKeyLock",   "tip":"Lock Start key",      "type":"Variable","dataType":"bool","value":false},
    {"name":"DropKeyLock",    "tip":"Lock Drop key",       "type":"Variable","dataType":"bool","value":false},
    {"name":"PauseKeyLock",   "tip":"Lock Pause key",      "type":"Variable","dataType":"bool","value":false},
    {"name":"StopKeyLock",    "tip":"Lock Stop key",       "type":"Variable","dataType":"bool","value":false},
    {"name":"RestartKeyLock", "tip":"Lock Restart key",    "type":"Variable","dataType":"bool","value":false},
    {"name":"DumpKeyLock",    "tip":"Lock Dump key",       "type":"Variable","dataType":"bool","value":false}

]}}
```

### A.8.4 Badge Reader

```json
{"cmd":{"act":"MENU","BadgeReader":{}}}

{"ack":{"act":"MENU","BadgeReader":[

    {
        "name":     "Reader1Port",
        "tip":      "Port for badge reader 1",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Com1", "Com2", "USB", "OPC1", "OPC2"],
        "value":    "Com1"
    },
    {
        "name":     "Reader1Type",
        "tip":      "Badge reader 1 type",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["None", "AWID", "HID", "Proximity"],
        "value":    "None"
    },
    {
        "name":     "Reader2Port",
        "tip":      "Port for badge reader 2",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Com1", "Com2", "USB", "OPC1", "OPC2"],
        "value":    "Com1"
    },
    {
        "name":     "Reader2Type",
        "tip":      "Badge reader 2 type",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["None", "AWID", "HID", "Proximity"],
        "value":    "None"
    },
    {
        "name":     "ThresholdWeight",
        "tip":      "Minimum weight required to trigger badge read",
        "type":     "Variable",
        "dataType": "float",
        "min":      0,
        "value":    0
    },
    {
        "name":      "SiteID",
        "tip":       "Include site ID in badge output",
        "type":      "Variable",
        "dataType":  "bool",
        "value":     false,
        "condition": "Reader1Type=AWID or Reader2Type=AWID"
    }

]}}
```

### A.8.5 WINVRS

```json
{"cmd":{"act":"MENU","WINVRS":{}}}

{"ack":{"act":"MENU","WINVRS":[

    {
        "name":     "Computer1Port",
        "tip":      "Communication port for Computer 1",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Com1", "Com2", "USB", "OPC1", "OPC2"],
        "value":    "Com1"
    },
    {
        "name":     "Computer1Mode",
        "tip":      "Computer 1 operating mode",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Disabled", "Enabled", "Continuous", "Terminal"],
        "value":    "Disabled"
    },
    {
        "name":     "Computer2Port",
        "tip":      "Communication port for Computer 2",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Com1", "Com2", "USB", "OPC1", "OPC2"],
        "value":    "Com1"
    },
    {
        "name":     "Computer2Mode",
        "tip":      "Computer 2 operating mode",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Disabled", "Remote"],
        "value":    "Disabled"
    },
    {
        "name":     "PrintPassthrough",
        "tip":      "Pass print output through to Computer 2",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "TrafficMode",
        "tip":      "Traffic light control mode",
        "type":     "Variable",
        "dataType": "enum",
        "values":   ["Off", "RGR", "GRG"],
        "value":    "Off"
    },
    {
        "name":     "TrafficOnThreshold",
        "tip":      "Weight threshold to activate traffic signal",
        "type":     "Variable",
        "dataType": "float",
        "min":      0,
        "value":    0
    },
    {
        "name":     "TrafficOffThreshold",
        "tip":      "Weight threshold to deactivate traffic signal",
        "type":     "Variable",
        "dataType": "float",
        "min":      0,
        "value":    0
    },
    {
        "name":     "TrafficOffDelay",
        "tip":      "Delay in seconds before traffic signal deactivates",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "value":    1
    },
    {
        "name":     "EnterRelayCommand",
        "tip":      "Relay command string sent on vehicle entry",
        "type":     "Variable",
        "dataType": "string",
        "value":    "3!4"
    },
    {
        "name":     "ExitRelayCommand",
        "tip":      "Relay command string sent on vehicle exit",
        "type":     "Variable",
        "dataType": "string",
        "value":    "1!2"
    },
    {
        "name":     "TrafficDisplay",
        "tip":      "Show traffic status on indicator display",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "DFCEnable",
        "tip":      "Enable Digital Fill Control within WINVRS",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    }

]}}
```

---

## A.9 Mode Config

```json
{"cmd":{"act":"MENU","ModeConfig":{}}}

{"ack":{"act":"MENU","ModeConfig":[
    {"name":"IDStorage",      "type":"menuItem","tip":"ID storage mode settings",              "condition":"ModeOfOperation=IDStorage"},
    {"name":"DFC",            "type":"menuItem","tip":"Digital fill control settings",         "condition":"ModeOfOperation=DFC"},
    {"name":"Batcher",        "type":"menuItem","tip":"Batcher mode settings",                 "condition":"ModeOfOperation=Batcher"},
    {"name":"PackageWeigher", "type":"menuItem","tip":"Package weigher settings",              "condition":"ModeOfOperation=PackageWeigher"},
    {"name":"AxleWeigher",    "type":"menuItem","tip":"Axle weigher settings",                 "condition":"ModeOfOperation=AxleWeigher"},
    {"name":"CheckWeigher",   "type":"menuItem","tip":"Check weigher zone and color settings", "condition":"ModeOfOperation=CheckWeigher"},
    {"name":"PWC",            "type":"menuItem","tip":"Preset weight comparator settings",     "condition":"ModeOfOperation=PWC"},
    {"name":"Livestock",      "type":"menuItem","tip":"Livestock inclinometer settings",       "condition":"ModeOfOperation=Livestock"}
]}}
```

### A.9.1 ID Storage

```json
{"cmd":{"act":"MENU","IDStorage":{}}}

{"ack":{"act":"MENU","IDStorage":[

    {"name":"WeightAlarm",    "tip":"Enable weight alarm",            "type":"Variable","dataType":"bool",   "value":false},
    {"name":"AlarmThreshold", "tip":"Weight alarm trigger threshold", "type":"Variable","dataType":"float",  "value":1000.0},
    {"name":"AlarmTimeOn",    "tip":"Alarm on duration in seconds",   "type":"Variable","dataType":"uint8",  "min":0,"max":99,"value":99},
    {"name":"IDCount",        "tip":"Number of ID prompts",           "type":"Variable","dataType":"uint8",  "min":1,"max":3,"value":1},
    {"name":"Prompt1",        "tip":"Label for ID prompt 1",          "type":"Variable","dataType":"string", "value":"ID"},
    {"name":"Prompt2",        "tip":"Label for ID prompt 2",          "type":"Variable","dataType":"string", "value":"ID2"},
    {"name":"Prompt3",        "tip":"Label for ID prompt 3",          "type":"Variable","dataType":"string", "value":"ID3"}

]}}
```

### A.9.2 DFC (Digital Fill Control)

```json
{"cmd":{"act":"MENU","DFC":{}}}

{"ack":{"act":"MENU","DFC":[

    {"name":"Speed",         "tip":"Fill speed mode",                   "type":"Variable","dataType":"enum",  "values":["SingleSpeed","DualSpeed"],"value":"SingleSpeed"},
    {"name":"GateSequence",  "tip":"Gate actuation sequence",           "type":"Variable","dataType":"enum",  "values":["ABB","ABA","ABBA"],       "value":"ABB"},
    {"name":"AutoTrim",      "tip":"Enable automatic trim adjustment",  "type":"Variable","dataType":"bool",  "value":false},
    {"name":"AutoPrint",     "tip":"Print ticket automatically",        "type":"Variable","dataType":"bool",  "value":false},
    {"name":"MultiDrop",     "tip":"Enable multi-drop mode",            "type":"Variable","dataType":"bool",  "value":false},
    {"name":"DumpGate",      "tip":"Enable dump gate",                  "type":"Variable","dataType":"bool",  "value":false},
    {"name":"AutoDump",      "tip":"Automatically actuate dump gate",   "type":"Variable","dataType":"bool",  "value":false},
    {"name":"Decumulate",    "tip":"Enable decumulate mode",            "type":"Variable","dataType":"bool",  "value":false},
    {"name":"AutoTare",      "tip":"Automatically tare before fill",    "type":"Variable","dataType":"bool",  "value":false},
    {"name":"JogToCutoff",   "tip":"Jog gate to cutoff point",         "type":"Variable","dataType":"bool",  "value":false},
    {"name":"FastCutoff",    "tip":"Fast speed cutoff weight",          "type":"Variable","dataType":"float", "value":0},
    {"name":"FillWeight",    "tip":"Target fill weight",                "type":"Variable","dataType":"float", "value":0},
    {"name":"SlowCutoff",    "tip":"Slow speed cutoff weight",          "type":"Variable","dataType":"float", "value":0},
    {"name":"Trim",          "tip":"Fill trim correction",              "type":"Variable","dataType":"float", "value":0},
    {"name":"DropCount",     "tip":"Number of drops per cycle",         "type":"Variable","dataType":"uint8", "value":0},
    {"name":"ZeroTolerance", "tip":"Zero acceptance tolerance",         "type":"Variable","dataType":"float", "value":0},
    {"name":"GateTimer",     "tip":"Gate actuation timer in ms",        "type":"Variable","dataType":"uint16","value":0},
    {"name":"Chatter",       "tip":"Gate chatter compensation",         "type":"Variable","dataType":"float", "value":0}

]}}
```

### A.9.3 Batcher

```json
{"cmd":{"act":"MENU","Batcher":{}}}

{"ack":{"act":"MENU","Batcher":[

    {"name":"Speed",         "tip":"Batcher speed mode",              "type":"Variable","dataType":"enum",  "values":["SingleSpeed","DualSpeed"],"value":"SingleSpeed"},
    {"name":"GateSequence",  "tip":"Gate actuation sequence",         "type":"Variable","dataType":"enum",  "values":["ABB","ABA","ABBA"],       "value":"ABB"},
    {"name":"AutoTrim",      "tip":"Enable automatic trim",           "type":"Variable","dataType":"bool",  "value":false},
    {"name":"BinCount",      "tip":"Number of bins",                  "type":"Variable","dataType":"uint8", "min":1,"max":8,"value":4},
    {"name":"AutoPrint",     "tip":"Print ticket automatically",      "type":"Variable","dataType":"bool",  "value":false},
    {"name":"DumpGate",      "tip":"Enable dump gate",                "type":"Variable","dataType":"bool",  "value":false},
    {"name":"AutoDump",      "tip":"Automatically actuate dump gate", "type":"Variable","dataType":"bool",  "value":false},
    {"name":"Decumulate",    "tip":"Enable decumulate mode",          "type":"Variable","dataType":"bool",  "value":false},
    {"name":"BatchCount",    "tip":"Number of batches to run",        "type":"Variable","dataType":"uint32","value":0},
    {"name":"ZeroTolerance", "tip":"Zero acceptance tolerance",       "type":"Variable","dataType":"float", "value":0},
    {"name":"GateTimer",     "tip":"Gate actuation timer in ms",      "type":"Variable","dataType":"uint16","value":0},
    {"name":"SettleTimer",   "tip":"Settle timer in ms",              "type":"Variable","dataType":"uint16","value":0}

]}}
```

### A.9.4 Package Weigher

```json
{"cmd":{"act":"MENU","PackageWeigher":{}}}

{"ack":{"act":"MENU","PackageWeigher":[

    {"name":"IDCount",  "tip":"Number of ID prompts",        "type":"Variable","dataType":"uint8",  "min":1,"max":3,"value":1},
    {"name":"Prompt1",  "tip":"Label for prompt 1",          "type":"Variable","dataType":"string", "value":"ID"},
    {"name":"Prompt2",  "tip":"Label for prompt 2",          "type":"Variable","dataType":"string", "value":"ID2"},
    {"name":"Prompt3",  "tip":"Label for prompt 3",          "type":"Variable","dataType":"string", "value":"ID3"},
    {"name":"RetainID", "tip":"Retain ID between weighments","type":"Variable","dataType":"bool",   "value":false}

]}}
```

### A.9.5 Axle Weigher

```json
{"cmd":{"act":"MENU","AxleWeigher":{}}}

{"ack":{"act":"MENU","AxleWeigher":[

    {"name":"AWMode",          "tip":"Axle detection mode",           "type":"Variable","dataType":"enum",  "values":["Auto","Manual"],"value":"Auto"},
    {"name":"AxlePads",        "tip":"Enable axle pad inputs",        "type":"Variable","dataType":"bool",  "value":false},
    {"name":"ThresholdWeight", "tip":"Minimum weight to detect axle", "type":"Variable","dataType":"float", "min":0,"value":500},
    {"name":"StopDelay",       "tip":"Stop delay in ms",              "type":"Variable","dataType":"uint16","value":0},
    {"name":"TotalDelay",      "tip":"Total delay in ms",             "type":"Variable","dataType":"uint16","value":0},
    {"name":"AxleCounter",     "tip":"Starting axle count",           "type":"Variable","dataType":"uint32","min":1,"value":1}

]}}
```

### A.9.6 Check Weigher

```json
{"cmd":{"act":"MENU","CheckWeigher":{}}}

{"ack":{"act":"MENU","CheckWeigher":[

    {"name":"Outputs",         "tip":"Number of output zones",         "type":"Variable","dataType":"uint8",  "min":1,"max":5,"value":3},
    {"name":"AutoPrint",       "tip":"Print ticket automatically",     "type":"Variable","dataType":"bool",   "value":false},
    {"name":"UnderThreshold",  "tip":"Under zone upper boundary",      "type":"Variable","dataType":"float",  "value":500},
    {"name":"LowOKThreshold",  "tip":"Low OK zone upper boundary",     "type":"Variable","dataType":"float",  "value":1000},
    {"name":"HighOKThreshold", "tip":"High OK zone upper boundary",    "type":"Variable","dataType":"float",  "value":1500},
    {"name":"OverThreshold",   "tip":"Over zone lower boundary",       "type":"Variable","dataType":"float",  "value":2000},
    {"name":"ColorUnder",      "tip":"Display color for Under zone",   "type":"Variable","dataType":"enum",   "values":["Red","Yellow","Green","Blue","Pink","White"],"value":"Yellow"},
    {"name":"ColorLowOK",      "tip":"Display color for Low OK zone",  "type":"Variable","dataType":"enum",   "values":["Red","Yellow","Green","Blue","Pink","White"],"value":"Pink"},
    {"name":"ColorHighOK",     "tip":"Display color for High OK zone", "type":"Variable","dataType":"enum",   "values":["Red","Yellow","Green","Blue","Pink","White"],"value":"Blue"},
    {"name":"ColorAcceptOK",   "tip":"Display color for Accept zone",  "type":"Variable","dataType":"enum",   "values":["Red","Yellow","Green","Blue","Pink","White"],"value":"Green"},
    {"name":"ColorOver",       "tip":"Display color for Over zone",    "type":"Variable","dataType":"enum",   "values":["Red","Yellow","Green","Blue","Pink","White"],"value":"Red"}

]}}
```

### A.9.7 Preset Weight Comparators (PWC)

```json
{"cmd":{"act":"MENU","PWC":{}}}

{"ack":{"act":"MENU","PWC":[

    {"name":"NumberOfOutputs","tip":"Number of comparator outputs",          "type":"Variable","dataType":"uint8", "min":1,"value":1},
    {"name":"BalanceOnPrint", "tip":"Balance outputs on print",              "type":"Variable","dataType":"bool",  "value":false},
    {"name":"MonitorZero",    "tip":"Monitor zero between weighments",       "type":"Variable","dataType":"bool",  "value":false},
    {"name":"Threshold",      "tip":"Per-output trip threshold (array)",     "type":"Variable","dataType":"float", "value":0},
    {"name":"Trim",           "tip":"Per-output trim correction (array)",    "type":"Variable","dataType":"float", "value":0},
    {"name":"OutputScale",    "tip":"Scale channel for each output (array)", "type":"Variable","dataType":"enum",  "values":["Scale1","Scale2","Scale3"],"value":"Scale1"}

]}}
```

### A.9.8 Livestock

```json
{"cmd":{"act":"MENU","Livestock":{}}}

{"ack":{"act":"MENU","Livestock":[

    {"name":"Inclinometer","tip":"Enable inclinometer tilt compensation","type":"Variable","dataType":"bool",  "value":false},
    {"name":"SetPitch",    "tip":"Tilt pitch angle in degrees",          "type":"Variable","dataType":"float", "value":0.0},
    {"name":"SetRoll",     "tip":"Tilt roll angle in degrees",           "type":"Variable","dataType":"float", "value":0.0},
    {
        "name":     "DefaultAngleCalibration",
        "tip":      "Run default angle calibration procedure",
        "type":     "method",
        "params":   {},
        "response": {
            "pitch":  {"dataType":"float"},
            "roll":   {"dataType":"float"},
            "status": {"dataType":"string"}
        }
    }

]}}
```

---

## A.10 DLC Setup (DLC card only)

```json
{"cmd":{"act":"MENU","DLCSetup":{}}}

{"ack":{"act":"MENU","DLCSetup":[

    {
        "name":     "CellDiagnostics",
        "tip":      "Real-time load cell diagnostic readings",
        "type":     "method",
        "params":   {},
        "response": {
            "cell_data": {"dataType":"string"}
        }
    },
    {
        "name":     "SnapMediabox",
        "tip":      "Enable SNAP Mediabox communication",
        "type":     "Variable",
        "dataType": "bool",
        "value":    false
    },
    {
        "name":     "SnapID",
        "tip":      "SNAP network node ID",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "value":    0
    },
    {
        "name":     "Channel",
        "tip":      "SNAP communication channel",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "max":      15,
        "value":    15
    }

]}}
```

---

## A.11 Review Menu

```json
{"cmd":{"act":"MENU","ReviewMenu":{}}}

{"ack":{"act":"MENU","ReviewMenu":[

    {
        "name":     "ScaleID",
        "tip":      "Scale identification number",
        "type":     "Variable",
        "dataType": "uint8",
        "min":      0,
        "value":    0
    },
    {
        "name":     "CalibrationCounter",
        "tip":      "Number of calibration events (read only, auto-increments)",
        "type":     "readOnlyVariable",
        "dataType": "uint32",
        "value":    0
    },
    {
        "name":     "ConfigurationCounter",
        "tip":      "Number of configuration events (read only, auto-increments)",
        "type":     "readOnlyVariable",
        "dataType": "uint32",
        "value":    0
    }

]}}
```