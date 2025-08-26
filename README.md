# StampFly Telemetry & Control Protocol (STCP) v0.2

A lightweight, transport-agnostic realtime protocol tailored for StampFly telemetry **and command**. Inspired by best practices but **original** in header layout, message taxonomy, and negotiation. Designed to run over **WebSocket, UDP, ESP-NOW, BLE (GATT/Notifications)** with the **same application frames**.

> Scope: on-air/device frames (ESP) and PC-Bridge mapping. Goal: 50–200 Hz attitude, 5–20 Hz slow telemetry, low jitter, simple reliability for lossy links, extensible TLV.

---

## 0. Design Goals

* **Transport-agnostic**: common frames across WS/UDP/ESP-NOW/BLE.
* **Small & fast**: short header for high-rate streams; optional extended header.
* **Time & order aware**: seq + t\_ms (device clock) for drop/latency accounting.
* **Extensible**: TLV sections, versioning & feature negotiation.
* **Pragmatic reliability**: optional ACK/NACK windows for lossy links; bypass on TCP/WS.
* **Security hooks**: optional HMAC and light encryption.
* **Unified**: telemetry and **low-latency command frames** in one protocol.

---

## 1. Layering & Transports

```
+--------------------+   +------------------+   +------------------+
|   Application      |   | Application      |   | Application      |
|   (STCP Frames)    |   | (STCP Frames)    |   | (STCP Frames)    |
+--------------------+   +------------------+   +------------------+
        |                        |                      |
   WebSocket (TCP)           UDP/IP               ESP-NOW / BLE
```

* Frame bytes are identical regardless of transport.
* For ESP-NOW, each 250-byte MAC frame carries exactly one STCP frame.

---

## 2. Frame Overview

Each STCP frame = **Header** + **Payload** (+ optional TLV trailer). Two header sizes:

* Short header (12B)
* Extended header (TLV)

---

## 3. Message Types (Type field)

|  Type | Name             | Direction | Typical Rate | Payload Schema                |
| ----: | ---------------- | --------- | ------------ | ----------------------------- |
|     0 | **HELLO**        | both      | once/connect | TLV features, UID             |
|     1 | **PONG**         | reply     | on-demand    | TLV RTT data                  |
|     2 | **ATT\_FAST**    | dev→host  | 50–200 Hz    | Fixed 28B (quat,pqr,alt)      |
|     3 | **TEL\_SLOW**    | dev→host  | 5–20 Hz      | CBOR map                      |
|     4 | **EVENT**        | dev→host  | sporadic     | CBOR map                      |
|     5 | **PARAM\_GET**   | host→dev  | request      | CBOR keys                     |
|     6 | **PARAM\_VALUE** | dev→host  | response     | CBOR map                      |
|     7 | **PARAM\_SET**   | host→dev  | command      | CBOR map                      |
|     8 | **CMD\_HIGH**    | host→dev  | sporadic     | CBOR {name,args}              |
|     9 | **ACK/NACK**     | both      | per-policy   | u16 seq, u8 code              |
|    10 | **PING**         | host→dev  | on-demand    | u32 host\_ms                  |
|    11 | **TIMESYNC**     | both      | 1–2 Hz       | TLV sync                      |
|    12 | **LOG\_CHUNK**   | dev→host  | as needed    | binary block                  |
|    13 | **FILE\_XFER**   | both      | as needed    | frag + binary                 |
|    14 | **CMD\_RC**      | host→dev  | 50–500 Hz    | Fixed short payload (see 4.4) |
| 15–31 | **Reserved**     | —         | —            | —                             |

> `CMD_RC` is the new low-latency fixed-size control frame for RC-like commands.

---

## 4. Payload Definitions

### 4.1 `ATT_FAST` (28 bytes)

```
struct AttFast {
  float qx, qy, qz, qw;  // quaternion
  float p, q, r;         // rad/s
  float alt;             // m
}
```

### 4.2 `TEL_SLOW` (CBOR map)

* Keys: `acc`, `mag`, `vxy`, `mode`, `mot`, `batt`, `temp`, `cpu`, …

### 4.3 `CMD_HIGH` (CBOR)

* Higher-level commands: `{ "name":"takeoff", "alt":1.0 }`

### 4.4 `CMD_RC` (16 bytes)

```
struct CmdRC {
  int16_t roll;     // scaled [-32768..32767] → [-1..+1]
  int16_t pitch;
  int16_t yaw;
  int16_t thrust;   // [0..32767] → [0..100%]
  uint16_t aux;     // bitfield (flight mode switches, arm/disarm, etc)
  uint16_t rsv;     // reserved for future
}
```

* Total = 12B + padding = 16B
* Fixed length, no floats, easily fits into ESP-NOW/BLE MTU.
* Host sends at high rate (50–500 Hz). Device applies immediately.
* Device may echo last received `CMD_RC` in telemetry for verification.

---

## 5. Reliability, Prioritization & Security

### 5.1 Command Prioritization

* Devices and bridges MUST **queue commands separately** and service them **ahead of telemetry**.
* On congested links, drop oldest telemetry first; commands are **latest-wins** (replace older unsent commands).
* Recommended service order per cycle: `CMD_*` → `PARAM_*`/`ACK` → `ATT_FAST` → `TEL_SLOW`.

### 5.2 ACK/NACK (optional)

* If **Flags.A=1**, receiver should reply with `ACK/NACK(seq, code)` for **control frames only** (`CMD_*`, `PARAM_*`, `FILE_XFER`).
* No ACK for `ATT_FAST`/`TEL_SLOW`.

### 5.3 HMAC & Encryption (optional)

* If **Flags.H=1**, append Trailer TLV `HMAC` (type `0x80`, len=16 or 32) over **header+payload+TLVs** with key selected by `KEY_ID`.
* If **Flags.C=1**, encrypt payload (and TLV trailer except HMAC) with stream cipher (XSalsa20/ChaCha20) using `KEY_ID` and per-frame nonce derived from `SrcID:Seq`.

### 5.4 Failsafe & Deadman

* Devices MUST enter failsafe if **no command frame** (`CMD_RC`) is received within `cmd_timeout_ms`.
* Host should periodically interleave `PING` or `HELLO` keepalives; devices may ignore these for flight control purposes.

---

## 6. Versioning & Negotiation

* HELLO handshake advertises support for `CMD_RC` (bitflag).
* If not supported, fall back to `CMD_HIGH` CBOR setpoints.

---

## 7. PC-Bridge Mapping

* On ingress, PC-Bridge wraps raw STCP → JSON for browsers:

```json
{
  "src_id": 1,
  "seq": 10234,
  "host_ms": 1735256123456,
  "offset_ms": -8.4,
  "type": "ATT_FAST",
  "payload": { "quat":[qx,qy,qz,qw], "omega":[p,q,r], "alt":1.23 }
}
```

* For commands, the bridge MAY expose a **low-latency socket** or priority queue and throttle telemetry when necessary.
* Logs: Parquet schema (flat): `src_id, host_ms, offset_ms, type, seq, qx..qw, pqr, alt, cmd_* ...`.

---

## 8. Summary of Control in STCP

* **CMD\_RC**: fast, compact, RC-like stick commands (binary fixed payload).
* **CMD\_HIGH**: slower, expressive high-level commands (CBOR).
* Both coexist in one unified protocol with telemetry frames.

---

## 9. C/C++ Templates (ESP-IDF)

```c
// stcp.h (minimal)
#pragma once
#include <stdint.h>

#define STCP_V1 1

enum stcp_type {
  STCP_HELLO=0, STCP_PONG=1, STCP_ATT_FAST=2, STCP_TEL_SLOW=3,
  STCP_EVENT=4, STCP_PARAM_GET=5, STCP_PARAM_VALUE=6, STCP_PARAM_SET=7,
  STCP_CMD_GENERIC=8, STCP_ACK=9, STCP_PING=10, STCP_TIMESYNC=11, STCP_LOG_CHUNK=12,
  STCP_FILE_XFER=13,
  STCP_CMD_RC16=20, STCP_CMD_RATE16=21, STCP_CMD_ATT_Q=22, STCP_CMD_SYS=23,
};

typedef struct __attribute__((packed)) {
  uint8_t vt;      // V:3 bits | Type:5 bits
  uint8_t flags;   // A,E,H,C,X (bit0..)
  uint16_t seq;    // BE
  uint16_t src;    // BE
  uint16_t tms_d;  // BE
  uint16_t len;    // BE payload length
  uint16_t rsv;    // BE
} stcp_hdr_t;

static inline void stcp_hdr_init(stcp_hdr_t* h, uint8_t type, uint16_t src, uint16_t seq, uint16_t tms_d, uint16_t len) {
  h->vt = (uint8_t)((STCP_V1 & 0x07) << 5) | (type & 0x1F);
  h->flags = 0; h->seq = __builtin_bswap16(seq); h->src = __builtin_bswap16(src);
  h->tms_d = __builtin_bswap16(tms_d); h->len = __builtin_bswap16(len); h->rsv = 0;
}

// ===== Command payloads =====
typedef struct __attribute__((packed)) { int16_t roll, pitch, yaw, thr; uint8_t mode_aux, rsv; } stcp_cmd_rc16_t;
typedef struct __attribute__((packed)) { int16_t p, q, r, T; } stcp_cmd_rate16_t;
typedef struct __attribute__((packed)) { uint16_t qx, qy, qz, qw, T, rsv; } stcp_cmd_att_q_t; // float16 encoded

typedef struct __attribute__((packed)) {
  float qx, qy, qz, qw; // BE float → convert when writing
  float p, q, r;        // rad/s
  float alt;            // m
} stcp_att_fast_t;
```

```c
// send_cmd_rc16.c (sketch)
#include "stcp.h"
#include <string.h>

static uint16_t g_seq;

void send_cmd_rc16(uint16_t src, int16_t roll, int16_t pitch, int16_t yaw, int16_t thr, uint8_t mode, uint8_t aux){
  stcp_cmd_rc16_t pl = { __builtin_bswap16(roll), __builtin_bswap16(pitch), __builtin_bswap16(yaw), __builtin_bswap16(thr), (uint8_t)((mode<<4)|(aux&0x0F)), 0 };
  stcp_hdr_t h; stcp_hdr_init(&h, STCP_CMD_RC16, src, g_seq++, 2, sizeof(pl));
  uint8_t buf[sizeof(h)+sizeof(pl)]; memcpy(buf,&h,sizeof(h)); memcpy(buf+sizeof(h),&pl,sizeof(pl));
  // transport send
}
```

---

## 10. License & IPR Note

This STCP specification is an **original work** for StampFly projects. It references generic protocol design patterns and does **not replicate** proprietary layouts (e.g., CRTP, SBUS, MAVLink).

### License

The STCP specification and reference implementations are released as **open source** under the MIT License. You are free to use, modify, and distribute this specification and its reference code in both academic and commercial contexts, provided that the license notice is retained.

---
