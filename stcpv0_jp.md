# StampFly Telemetry & Control Protocol (STCP) v0.2

StampFly 用に設計された軽量リアルタイム通信プロトコルです。**テレメトリ**と**操縦コマンド**を一つの共通仕様で扱えることを目的としています。WebSocket, UDP, ESP-NOW, BLE(GATT通知) などの多様なトランスポートに対応します。

> 適用範囲：機体上のフレーム構造およびPCブリッジでの取り扱い。目標は、姿勢テレメトリ 50–200 Hz、スロー系テレメトリ 5–20 Hz、コマンド 50–500 Hz。

---

## 0. 設計目標

* **トランスポート非依存**：異なる物理層でも同一フレームを利用。
* **軽量**：12Bショートヘッダと短い固定ペイロード。
* **時間管理**：シーケンス番号 + 時刻差分で遅延・欠落を把握。
* **拡張性**：TLVやCBORで柔軟に拡張可能。
* **信頼性**：ACK/NACKは必要に応じて、損失の多いリンクでは有効化。
* **セキュリティ**：HMACや暗号化のフックを定義。
* **統合**：テレメトリと操縦コマンドを統一仕様で扱う。

---

## 1. レイヤリングとトランスポート

```
+------------------+   +------------------+   +------------------+
| Application      |   | Application      |   | Application      |
| (STCP Frames)    |   | (STCP Frames)    |   | (STCP Frames)    |
+------------------+   +------------------+   +------------------+
        |                    |                      |
   WebSocket(TCP)         UDP/IP               ESP-NOW / BLE
```

* トランスポートごとにフレーミング方法は異なるが、STCPフレームの内容は同一。
* ESP-NOWでは1 MACフレームに1 STCPフレームを格納。

---

## 2. フレーム概要

* 各STCPフレーム = ヘッダ + ペイロード + (任意のTLV)。
* ヘッダはショート(12B)または拡張(TLV)。

---

## 3. メッセージ種別

| Type | 名称           | 向き    | 周期       | ペイロード           |
| ---: | ------------ | ----- | -------- | --------------- |
|    0 | HELLO        | 双方向   | 接続時      | 機能・UID          |
|    1 | PONG         | 応答    | 随時       | RTT情報           |
|    2 | ATT\_FAST    | 機体→地上 | 50–200Hz | 姿勢(Quat,pqr,高度) |
|    3 | TEL\_SLOW    | 機体→地上 | 5–20Hz   | CBOR地図          |
|    4 | EVENT        | 機体→地上 | 不定期      | CBORイベント        |
|    5 | PARAM\_GET   | 地上→機体 | 要求       | CBORキー          |
|    6 | PARAM\_VALUE | 機体→地上 | 応答       | CBORマップ         |
|    7 | PARAM\_SET   | 地上→機体 | 指令       | CBORマップ         |
|    8 | CMD\_HIGH    | 地上→機体 | 不定期      | 高レベルコマンド        |
|    9 | ACK/NACK     | 双方向   | 随時       | seqとコード         |
|   10 | PING         | 地上→機体 | 随時       | ホスト時刻           |
|   11 | TIMESYNC     | 双方向   | 1–2Hz    | 同期TLV           |
|   12 | LOG\_CHUNK   | 機体→地上 | 随時       | ログ断片            |
|   13 | FILE\_XFER   | 双方向   | 随時       | ファイル断片          |
|   14 | CMD\_RC      | 地上→機体 | 50–500Hz | RC型コマンド         |

---

## 4. ペイロード定義

### 4.1 ATT\_FAST (28B)

姿勢クォータニオン4要素 + 角速度3要素 + 高度1要素。

### 4.2 TEL\_SLOW

CBOR形式。例: `batt`, `mode`, `mot`, `cpu`。

### 4.3 CMD\_HIGH

CBORコマンド例: `{ "name":"takeoff", "alt":1.0 }`

### 4.4 CMD\_RC (16B)

```
struct CmdRC {
  int16_t roll;
  int16_t pitch;
  int16_t yaw;
  int16_t thrust;
  uint16_t aux;   // フライトモードやスイッチ
  uint16_t rsv;
}
```

* 短く固定長。ESP-NOWやBLEに適合。
* 高速周期(50–500Hz)で送信。

---

## 5. 信頼性・優先度・セキュリティ

* **優先度**：CMD > PARAM/ACK > ATT\_FAST > TEL\_SLOW。
* **ACK/NACK**：制御系(CMD, PARAM)のみ必須。テレメは不要。
* **暗号化**：Flags.Cで有効化。ChaCha20などを想定。
* **デッドマン**：一定時間CMD\_RCが来なければフェイルセーフ動作。

---

## 6. バージョン交渉

* HELLOで機能・MTU・CMD\_RC対応有無を通知。
* 未対応ならCMD\_HIGHへフォールバック。

---

## 7. PCブリッジでの扱い

* JSONに変換してWebブラウザへ配信。

```json
{
  "src_id":1,
  "type":"ATT_FAST",
  "quat":[...],
  "omega":[...],
  "alt":1.23
}
```

* ログ保存はParquet形式を推奨。

---

## 8. まとめ

* CMD\_RC: RCスティック相当の低レイテンシ指令。
* CMD\_HIGH: 高レベル制御コマンド。
* 両者を統合的に扱う軽量プロトコル。

---

## 9. C/C++ テンプレート

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
  stcp_hdr_t h;
  stcp_hdr_init(&h, STCP_CMD_RC16, src, g_seq++, 2, sizeof(pl));
  uint8_t buf[sizeof(h)+sizeof(pl)];
  memcpy(buf,&h,sizeof(h));
  memcpy(buf+sizeof(h),&pl,sizeof(pl));
  // transport send
}
```

---

## 10. ライセンスとIPR

本仕様はStampFlyプロジェクト向けに独自に設計されたものです。既存プロトコル（CRTP, SBUS, MAVLinkなど）のコピーではありません。

### ライセンス

MIT License として公開します。商用・学術問わず自由に利用・改変・再配布可能です。
