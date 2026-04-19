# MeshCore–Reticulum Bridge Node: Technical Specification v4

**Status:** Draft for community review  
**Supersedes:** meshcore_bridge_spec_v2.md  
**Companion document:** [LXMF as an Interoperability Plane for MeshCore Wide-Area Bridging](https://github.com/artbotterell/CoreNet) (Botterell, KD6O)  
**Upstream firmware base:** [RTNode-2400](https://github.com/GrayHatGuy/RTNode-2400) (GrayHatGuy / jrl290), microReticulum (attermann), RNode Firmware (markqvist)  
**WIP hardware reference:** [awesome-meshcore open hardware](https://github.com/samuk/awesome-meshcore/blob/main/open_hardware.md)  
**Author** Word guessing machine with input from samuk
**License:** GPL-3.0 (inherited)

---

## Overview

This specification describes a dual-MCU MeshCore–Reticulum bridge node consisting of:

- **MCU 1 (MeshCore side):** Seeed XIAO nRF52840 + Wio SX1262, running MeshCore firmware with a bridge companion layer
- **MCU 2 (Reticulum side):** Seeed XIAO ESP32-S3 + Wio SX1280, running [RTNode-2400](https://github.com/GrayHatGuy/RTNode-2400) (microReticulum boundary node)
- **Interconnect:** UART with RTS/CTS hardware flow control

[RTNode-2400](https://github.com/GrayHatGuy/RTNode-2400) provides a working, tested microReticulum boundary node implementation on ESP32-S3 with empirically measured RAM usage of ~29% (94 KB of 327 KB) in boundary mode. This eliminates the need to implement Reticulum from scratch and is the primary architectural change from v2. The CoreNet project ([github.com/artbotterell/CoreNet](https://github.com/artbotterell/CoreNet)) defines the LXMF interoperability plane this bridge implements. WIP carrier hardware is tracked at [awesome-meshcore open hardware](https://github.com/samuk/awesome-meshcore/blob/main/open_hardware.md).

The bridge logic is split across both MCUs according to ownership. On MCU 2 we own RTNode-2400 and develop it directly: the UART interface, LXMF identity mapping, manifest storage, SPI flash driver, and any expansions to the path/announce tables are all first-party RTNode-2400 features. On MCU 1 we do not modify MeshCore; the bridge companion layer operates exclusively within the interface that MeshCore's companion protocol exposes, without touching the core routing stack.

**Relationship to CoreNet architecture:** CoreNet describes a Bridge Daemon running on an always-on host (Pi, server, VM) that connects to a companion radio over serial/BLE/TCP. This node is an embedded implementation of that same pattern: MCU 1 is the companion radio, MCU 2 running RTNode-2400 is the bridge daemon, and the UART link between them replaces the host serial/BLE/TCP connection. The CoreNet safety invariants — no advert propagation, zero-hop ingress, one gateway contact, pull-based manifests, position opt-in — are all implemented here at the MCU layer rather than in software on a host.

**The governing constraint throughout:** the local 868 MHz mesh behaves identically whether or not the bridge is present. The bridge is invisible to local nodes except as a single gateway contact. The mesh works when the bridge is down.

---

## 1. Hardware

### 1.1 Bill of Materials

| Component | Part | Role |
|---|---|---|
| MCU 1 | Seeed XIAO nRF52840 | MeshCore host, bridge companion layer |
| MCU 1 radio | Seeed Wio SX1262 shield | 868 MHz MeshCore last-mile |
| MCU 2 | Seeed XIAO ESP32-S3 (8 MB PSRAM variant) | RTNode-2400 / microReticulum |
| MCU 2 radio | Seeed Wio SX1280 shield | 2.4 GHz RNode interface to Reticulum backbone |
| Interconnect | UART + RTS/CTS | MCU 1 ↔ MCU 2 bridge link |

**Note on MCU 2 radio:** RTNode-2400 on XIAO ESP32-S3 uses the SX1280 operating at 2.4 GHz as its sole backbone interface (confirmed working, boundary mode verified per upstream README — the "2400" in RTNode-2400 refers to this band). WiFi is not used; all Reticulum backbone connectivity is carried over the SX1280 LoRa link. The SX1280 operates in `MODE_ACCESS_POINT` within RTNode-2400's interface abstraction. MCU 1's SX1262 operates at 868 MHz and has no RF conflict with the SX1280 on MCU 2.

**Note on PSRAM:** The XIAO ESP32-S3 Plus Sense board has 8 MB PSRAM. RTNode-2400 boundary mode uses TLSF allocation from PSRAM when available. Use the PSRAM variant; the non-PSRAM variant has been confirmed to fall back to ~170 KB internal SRAM which is tighter.

### 1.2 Backbone Transport

MCU 2 connects to the Reticulum backbone exclusively via the SX1280 LoRa radio link. There is no WiFi or TCP backbone interface in this configuration. RTNode-2400 operates with a single SX1280 interface in `MODE_ACCESS_POINT`, forwarding unroutable packets to adjacent Reticulum nodes over RF. The backbone channel, spreading factor, and uplink peer are configured in RTNode-2400's interface settings.

**RF link budget note:** The SX1280 at 2.4 GHz has a shorter free-space range than sub-GHz LoRa at equivalent transmit power. Ensure the backbone RF uplink to the nearest Reticulum node is link-budgeted for the deployment site. This is the primary range constraint on the backbone side; the local 868 MHz MeshCore mesh is unaffected.

### 1.3 Interconnect Wiring

```
MCU 1 nRF52840              MCU 2 ESP32-S3
──────────────              ──────────────
TX   D6  ──────────────►    RX   D7
RX   D7  ◄──────────────    TX   D6
CTS  D8  ◄──────────────    RTS  D9
RTS  D9  ──────────────►    CTS  D8
3V3      ────────────────   3V3   (if powered from MCU1; check current budget)
GND      ────────────────   GND
```

**Flow control:** MCU 1 asserts CTS HIGH when:
- SX1262 TX_DONE interrupt has not yet been received (radio actively transmitting), or
- Local receive queue depth exceeds 12 packets (75% of 16-packet buffer)

MCU 2 halts UART output immediately on CTS HIGH. MCU 2 must buffer at least 2 packets in RAM to absorb CTS assertion latency — do not assume instantaneous halt.

**Baud rate:** 115200. The bottleneck is 868 MHz airtime (~5 kbps effective), not UART.

---

## 2. Firmware Overview

### 2.1 MCU 2 — RTNode-2400 (Our Firmware)

MCU 2 runs RTNode-2400, which we develop and own. The base `seeed_xiao_esp32s3_boundary` build provides the microReticulum boundary node foundation. Bridge-specific features — the UART interface, manifest storage, SPI flash driver, and LXMF mapping — are implemented directly within RTNode-2400 as first-party code. The existing boundary node behaviour is retained unchanged:

- SX1280 LoRa interface in `MODE_ACCESS_POINT`: sole backbone interface; blocks backbone announces from propagating back to RF
- Path table capped at 48 entries; hash list at 32; announce queue at 4 (expand using SPI flash — see §6.4)
- Backbone announces cached to flash, served on demand when local nodes request paths
- Default route forwarding: unroutable packets forwarded to backbone over SX1280

**MCU 2 memory budget (empirical, from RTNode-2400 README):**

```
XIAO ESP32-S3 boundary build:
  RAM:         28.8% used  (94,492 bytes of 327,680)
  Internal flash: ~27% used  (~1.77 MB of ~6.5 MB usable, partition-table dependent)
```

**External SPI flash:** The carrier board provides an additional 8 MB SPI flash chip accessible to the ESP32-S3. This is entirely separate from the ESP32-S3's internal flash and is available for bridge storage use — see §6.3 and §6.4.

Headroom is comfortable. The UART shim adds negligible RAM overhead.

### 2.2 MCU 1 — MeshCore Companion Firmware + Bridge Layer

MCU 1 runs upstream MeshCore **companion** firmware which we do not and cannot modify. The companion firmware exposes a well-documented binary frame protocol over USB serial (protocol version 8, last updated Feb 2026), which is the intended programmatic interface for external applications. The bridge layer on MCU 1 implements this protocol as a client — it is, from MeshCore's perspective, the companion app.

**Companion protocol USB serial framing:**
```
Outbound (radio → bridge):  0x3E ('>')  +  length[2] little-endian  +  frame
Inbound  (bridge → radio):  0x3C ('<')  +  length[2] little-endian  +  frame
Max frame size: 255 bytes. Protocol documented at:
https://github.com/meshcore-dev/MeshCore/wiki/Companion-Radio-Protocol
```

This is entirely separate from the MCU 1 ↔ MCU 2 bridge UART link defined in §3. MCU 1 uses two serial interfaces: one facing MeshCore (companion protocol over USB serial pins) and one facing MCU 2 (bridge UART with RTS/CTS, §3). Confirm these are on separate UART peripherals on the nRF52840.

The bridge layer responsibilities, mapped to companion protocol commands:
- On boot: register gateway contact via `CMD_ADD_UPDATE_CONTACT` (0x09), type = `ADV_TYPE_REPEATER` (2)
- Receive inbound messages: listen for `PUSH_CODE_MSG_WAITING` (0x83), pull with `CMD_SYNC_NEXT_MESSAGE` (0x0A)
- Deliver inbound LXMF payloads to local nodes via `CMD_SEND_RAW_DATA` (0x19) with `path_len = 0` (zero-hop direct delivery, equivalent to hop count 0)
- Observe newly heard nodes via `PUSH_CODE_ADVERT` (0x80) and `PUSH_CODE_NEW_ADVERT` (0x8A); push manifest entries to MCU 2
- Manage CTS flow control on the MCU 1 ↔ MCU 2 UART link

---

## 3. UART Framing Protocol

MeshCore lacks KISS framing. The following length-prefixed frame format is used on the UART link. It is an inter-MCU transport envelope, not MeshCore's wire format.

```
Byte offset   Length   Field
───────────   ──────   ─────
0             1        Magic 0xAA
1             1        Magic 0x55
2             1        Packet type (see below)
3–4           2        Payload length, big-endian uint16 (max 512)
5–N           N        Payload
N+1–N+2       2        CRC-16/CCITT (polynomial 0x1021, init 0xFFFF)
                       computed over: type byte, len_h, len_l, and all payload bytes
                       i.e. every byte from offset 2 through offset N inclusive
```

If LEN > 512, emit error frame type 0xFF with error code 0x03 and return to WAIT_MAGIC1 immediately. Do not wait for the declared number of bytes — a corrupted length field will otherwise stall the receiver indefinitely.

**Payload size constraints by packet type:**

| Type | Fixed payload size | Notes |
|---|---|---|
| 0x01 | 64 + N bytes | 32-byte source pubkey + 32-byte dest pubkey + MeshCore payload (N ≤ 448) |
| 0x02 | 32 + N bytes | 32-byte dest pubkey + decrypted payload (N ≤ 480) |
| 0x03 | 0 bytes | Hash request carries no payload |
| 0x04 | 16 bytes | LXMF destination hash |
| 0x05 | 106 bytes | Manifest entry (fixed, see §5.4) |
| 0x06 | 1 byte | Echo of acknowledged packet type |
| 0xFF | 1 byte | Error code |

MeshCore packets are bounded by the SX1262 LoRa MTU. In practice MeshCore payloads do not exceed ~200 bytes; the 448-byte limit for type 0x01 provides margin. Implementations must not allocate static buffers of 512 bytes per frame — size buffers to the per-type maximum.

**Packet types:**

| Type | Direction | Meaning |
|---|---|---|
| 0x01 | MCU1 → MCU2 | MeshCore packet for LXMF delivery via backbone |
| 0x02 | MCU2 → MCU1 | LXMF payload for local delivery |
| 0x03 | MCU1 → MCU2 | Gateway contact registration (LXMF hash request) |
| 0x04 | MCU2 → MCU1 | Gateway LXMF hash response |
| 0x05 | MCU1 → MCU2 | Manifest entry push (local node seen) |
| 0x06 | Both | Delivery receipt / ACK |
| 0xFF | Both | Error frame (1-byte error code in payload) |

**Error codes:**

| Code | Meaning |
|---|---|
| 0x01 | CRC failure |
| 0x02 | Destination node not local |
| 0x03 | Payload length exceeds maximum |
| 0x04 | Unknown packet type |

**Receiver state machine (both MCUs):**

```
WAIT_MAGIC1 → recv 0xAA → WAIT_MAGIC2
WAIT_MAGIC2 → recv 0x55 → READ_TYPE
            → recv other → WAIT_MAGIC1
READ_TYPE   → store type, advance → READ_LEN_H
READ_LEN_H  → store len_h, advance → READ_LEN_L
READ_LEN_L  → len = (len_h<<8)|len_l
            → if len > 512: emit 0xFF/0x03, goto WAIT_MAGIC1
            → else: advance → READ_PAYLOAD
READ_PAYLOAD→ accumulate len bytes → READ_CRC_H
READ_CRC_H  → store crc_h → READ_CRC_L
READ_CRC_L  → crc = (crc_h<<8)|crc_l
            → verify CRC-16/CCITT over [type, len_h, len_l, payload]
            → CRC OK:   dispatch(type, payload), goto WAIT_MAGIC1
            → CRC FAIL: emit 0xFF/0x01, goto WAIT_MAGIC1
```

Do not use timeouts for framing. CRC failure is the recovery mechanism.

---

## 4. Addressing and Identity

### 4.1 Bridge LXMF Identity

On boot, MCU 1 sends a type 0x03 frame to MCU 2 requesting the bridge's LXMF destination hash. MCU 2 responds with type 0x04 containing the 16-byte hash derived from RTNode-2400's Reticulum identity (Curve25519 public key, generated on first boot and persisted in ESP32-S3 NVS).

MCU 1 uses this hash as the `lxmf_hash` field in the gateway contact it registers on the local MeshCore mesh. If MCU 2 is unreachable at boot, MCU 1 retries every 30 seconds. MCU 1 does not register a gateway contact until it has received a valid type 0x04 response.

### 4.2 MeshCore Node → LXMF Destination Mapping

Each MeshCore node reachable through this bridge needs a stable LXMF destination hash, derived deterministically from its MeshCore Ed25519 public key. This derivation is performed **once** when a manifest entry is received (type 0x05 on MCU 2, or on first hearing a node on MCU 1), and the result is stored in the SPI flash manifest as a precomputed lookup table entry. It is never recomputed at packet delivery time — inbound LXMF delivery is an O(1) manifest lookup, not a per-packet HKDF operation.

**⚠ INTEROPERABILITY CONFLICT — SEE OPEN QUESTION 1 BEFORE IMPLEMENTING ⚠**

CoreNet specifies that LXMF destination hashes should be derived via **standard Ed25519→Curve25519 conversion** (the same method Reticulum itself uses), with HKDF as a fallback only where the scalar is inaccessible. This implementation uses HKDF because the nRF52840 nrf_crypto API does not expose the Ed25519 scalar. If any other bridge implementation uses direct Ed25519→Curve25519 conversion (as CoreNet prefers), the derived destination hashes will be different and cross-bridge delivery will silently fail. This must be resolved with the CoreNet authors before first deployment. The derivation method must be the same across all bridge implementations.

**Current implementation (HKDF fallback — subject to change pending resolution of open question 1):**

```
Inputs:
  IKM  = meshcore_ed25519_pubkey        (32 bytes, raw little-endian)
  Salt = /* SEE CONSTANT BELOW */       (32 bytes)
  Info = "lxmf-destination-v1"          (UTF-8, no null terminator)
  L    = 16 bytes output

Result:
  lxmf_dest_hash = HKDF-SHA256(IKM, Salt, Info, 16)   (16 bytes, stored directly)
```

HKDF-SHA256 with L=16 produces the destination hash directly. It is called only once per node, not per packet.

**Salt value — protocol constant, hardcoded in both MCUs:**

The salt is SHA-256 of the ASCII string `meshcore-lxmf-bridge-v1`. This is computed once offline by the protocol authors and frozen. Both MCU 1 and MCU 2 firmware embed it as a 32-byte literal constant. It is never computed at runtime.

```c
/* Protocol constant — do not modify */
static const uint8_t LXMF_BRIDGE_SALT[32] = {
    /* SHA-256("meshcore-lxmf-bridge-v1") */
    /* TO BE FILLED BY PROTOCOL AUTHORS BEFORE FIRST RELEASE */
    /* Compute offline: echo -n "meshcore-lxmf-bridge-v1" | sha256sum */
    0x00, 0x00, /* ... 32 bytes ... */
};
```

This constant is only relevant if HKDF is confirmed as the canonical interop method (open question 1). If direct Ed25519→Curve25519 conversion is adopted, this constant is not used. Do not publish or freeze this constant until open question 1 is resolved.

**Reverse mapping** (LXMF destination hash → MeshCore pubkey and display name) is stored as a flat lookup table in the SPI flash manifest. On MCU 2, inbound LXMF delivery scans this table by lxmf_dest_hash. A node not present in the manifest cannot be delivered to — see open question 8.

---

## 5. MCU 1 Bridge Layer

### 5.1 Gateway Contact Registration

After receiving the LXMF hash from MCU 2 (type 0x04), MCU 1 sends `CMD_ADD_UPDATE_CONTACT` (0x09) to register the gateway contact:

```
CMD_ADD_UPDATE_CONTACT {
  code:         0x09
  public_key:   bytes(32)        // bridge Reticulum identity public key (from type 0x04)
  type:         0x02             // ADV_TYPE_REPEATER
  flags:        0x00
  out_path_len: 0x00             // direct — bridge is one hop away
  out_path:     bytes(64) zeros
  adv_name:     "[GW-UK1]\0..."  // null-padded to 32 bytes
  last_advert:  uint32           // current epoch time
}
```

**Do not:**
- Set `out_path_len` > 0 (the bridge is directly reachable, not via a multi-hop path)
- Force link quality to maximum (dishonest, causes routing pathologies)
- Register more than one gateway contact regardless of how many remote nodes are reachable
- Re-register on every boot if the contact and public key are unchanged in persistent storage

Re-register only if the MCU 2 Reticulum identity changes (factory reset). Re-send `CMD_SEND_SELF_ADVERT` (0x07) every 4 hours to keep the gateway contact alive on the local mesh.

### 5.2 Inbound Delivery (Backbone → Local, MCU2 → MCU1)

```
Receive type 0x02 frame from MCU 2
Verify CRC
Extract from payload:
  dest_meshcore_pubkey[32]
  meshcore_payload[N]
Find local MeshCore node matching dest_meshcore_pubkey in companion contacts list
If found:
  Send CMD_SEND_RAW_DATA (0x19) to MeshCore:
    path_len = 0x00      // zero-hop: delivers direct, no re-flooding
    path     = (empty)
    payload  = meshcore_payload[N]
  Send type 0x06 ACK to MCU 2
If not found:
  Send type 0xFF error 0x02 to MCU 2
```

`path_len = 0` is mandatory. It causes MeshCore to deliver the packet as a direct transmission without inserting it into the flood mesh, preventing re-flooding by any repeater that receives it. This is the companion protocol equivalent of hop count 0.

### 5.3 Outbound Handling (Local → Backbone, MCU1 → MCU2)

The bridge layer is the MeshCore companion app on MCU 1. Local nodes address messages to the gateway contact's public key; MeshCore delivers these to the bridge layer as `PUSH_CODE_MSG_WAITING` (0x83) push notifications.

```
Receive PUSH_CODE_MSG_WAITING (0x83) from MeshCore companion
Send CMD_SYNC_NEXT_MESSAGE (0x0A) to pull message
Receive RESP_CODE_CONTACT_MSG_RECV_V3 (0x10):
  pubkey_prefix[6]      // first 6 bytes of sender's public key
  path_len              // hop count
  txt_type
  sender_timestamp
  text[N]
Resolve full sender pubkey from companion contacts list using pubkey_prefix
Build type 0x01 payload:
  sender_pubkey[32] || gateway_pubkey[32] || text[N]
Send to MCU 2 via UART (honour CTS)
Continue CMD_SYNC_NEXT_MESSAGE until RESP_CODE_NO_MORE_MESSAGES (0x0A)
```

Note: `pubkey_prefix` in the companion protocol is only 6 bytes. The bridge layer resolves the full 32-byte sender public key by matching against the companion contacts list retrieved via `CMD_GET_CONTACTS` (0x04). If no match is found, the message is held until the next contacts sync, which is triggered by any `PUSH_CODE_ADVERT` (0x80).

### 5.4 Manifest Push

When the companion firmware notifies the bridge layer of a newly heard or updated node via `PUSH_CODE_ADVERT` (0x80) or `PUSH_CODE_NEW_ADVERT` (0x8A), the bridge layer fetches the full contact record via `CMD_GET_CONTACTS` (0x04) and pushes a type 0x05 manifest entry to MCU 2:

```
Type 0x05 payload (106 bytes):
  Byte 0:       Version = 0x01
  Bytes 1–32:   MeshCore Ed25519 public key (raw)
  Bytes 33–48:  LXMF destination hash (derived per Section 4.2)
  Bytes 49–80:  Display name (UTF-8, null-padded to 32 bytes)  ← CoreNet: up to 32 bytes
  Bytes 81–84:  Last seen Unix timestamp (uint32 big-endian)
  Bytes 85–92:  Region tag (ASCII, null-padded to 8 bytes, e.g. "UK-WAL\0\0")
  Byte  93:     Propagation scope (0x00 = local, 0x01 = region, 0x02 = global; default 0x01)
  Byte  94:     Opt-in flags (bit 0: position shared)
  Bytes 95–98:  Latitude (int32 microdegrees, 0 if not opted in)
  Bytes 99–102: Longitude (int32 microdegrees, 0 if not opted in)
  Bytes 103–106: Position quantisation radius in metres (uint32, min 3000)
```

**Propagation scope** defaults to `0x01` (region). A bridge with scope `region` only interoperates with remote bridges whose region tag matches its own. Scope `global` opts into unrestricted interoperability. Scope `local` disables all WAN forwarding for this node's traffic regardless of other settings. This implements CoreNet's region-scoping requirement. A node that has never explicitly set scope is treated as `region` — consistent with CoreNet's "safe by default" principle.

MCU 2 stores these entries and serves them as the contact manifest when queried by remote bridges. MCU 1 sends a manifest entry:
- When a new node is first heard
- When a known node's last_seen advances by more than 10 minutes
- On bridge boot (replay all known local nodes to resync MCU 2)

MCU 1 removes a node from its local manifest and sends an update with last_seen = 0 when a node has not been heard for 24 hours.

### 5.5 Admin Command Filtering

CoreNet explicitly prohibits bridging of admin, radio configuration, and raw RF commands. The bridge layer on MCU 1 must not forward the following companion protocol commands received via any path to the MeshCore radio, and must not expose them to MCU 2 via the UART link:

| Command | Code | Reason blocked |
|---|---|---|
| `CMD_SET_RADIO_PARAMS` | 0x0B | Radio configuration is local only |
| `CMD_SET_RADIO_TX_POWER` | 0x0C | Radio configuration is local only |
| `CMD_SEND_LOGIN` | 0x1A | Admin access to repeaters/room servers must not be remotely initiated |
| `CMD_FACTORY_RESET` | 0x33 | Destructive; must never be remotely triggerable |
| `CMD_SET_DEVICE_PIN` | 0x25 | Security credential; local only |
| `CMD_EXPORT_PRIVATE_KEY` | 0x17 | Key material; must never traverse any network path |

Any type 0x01 UART frame arriving from MCU 2 whose embedded payload matches any of these command codes after LXMF decapsulation must be silently dropped and logged. MCU 2 does not forward these; this is a defence-in-depth check on MCU 1.

### 5.6 ACK Synthesis

CoreNet specifies that ACKs are "synthesised from LXMF delivery receipts." When MCU 2 receives a delivery confirmation from the Reticulum layer for an outbound LXMF message, it sends a type 0x06 ACK frame to MCU 1. MCU 1 then sends `CMD_SEND_SELF_ADVERT` is not the right mechanism — instead MCU 1 must synthesise a MeshCore ACK back to the original sender.

**Current gap:** the MeshCore companion protocol does not expose a command to inject a synthetic ACK for a previously sent message. `PUSH_CODE_SEND_CONFIRMED` (0x82) is a push notification from the radio to the app, not a command from the app to the radio. There is no `CMD_SEND_ACK` command in the current protocol.

Until this is available upstream, delivery confirmation to the local sender is not possible. The bridge operates on best-effort semantics: the local node sends to the gateway contact and receives no delivery confirmation. This is the same behaviour as sending to an out-of-range direct contact. Document this limitation clearly in any user-facing bridge documentation.

---

## 6. MCU 2 UART Interface (RTNode-2400 Feature)

The UART interface is a first-party RTNode-2400 feature implemented as a new interface type within microReticulum alongside the existing SX1280 LoRa interface.

### 6.1 What the UART Interface Does

The UART interface operates as a full microReticulum interface type. It:

1. Reads framed packets from UART (type 0x01 from MCU 1)
2. Unpacks source/destination MeshCore pubkeys and payload
3. Derives LXMF destination hash for the destination pubkey (per Section 4.2)
4. Constructs an LXMF message addressed to the derived destination
5. Injects into the microReticulum send pipeline (same path as any outbound LXMF message)

Inbound (backbone → local):
1. microReticulum delivers an LXMF message addressed to a locally-registered MeshCore node destination
2. UART interface extracts destination MeshCore pubkey (reverse-mapped from LXMF hash via manifest)
3. Builds type 0x02 frame with dest pubkey and decrypted payload
4. Sends to MCU 1 via UART (honour CTS)

### 6.2 Type 0x03 / 0x04 — LXMF Hash Handshake

On receiving type 0x03 from MCU 1, MCU 2 responds with type 0x04:

```
Type 0x04 payload (16 bytes):
  bridge_lxmf_dest_hash[16]   — derived from RTNode-2400's Reticulum identity
```

This hash is stable across reboots (RTNode-2400 persists its Reticulum identity in NVS). It changes only on factory reset.

### 6.3 Manifest Storage and Service

MCU 2 stores manifest entries received via type 0x05 frames on the external SPI flash (see §6.4). It serves these as an LXMF resource at the CoreNet-defined destination aspect `meshcore` / `bridge` when queried by remote bridges. The full destination naming follows CoreNet's scheme: application `"meshcore"`, aspect `"bridge"` for manifest queries, aspect `"node"` for individual node destinations.

**Manifest entry format on disk:** identical to the type 0x05 payload (106 bytes per entry), stored as a flat array. The 8 MB SPI flash supports far more than needed — the practical limit is the local mesh size, not storage. The implementation should nevertheless enforce a configurable cap (default 512 entries, ~54 KB) to bound manifest query response time.

**Manifest signing:** MCU 2 signs the serialised manifest with its Reticulum identity private key before serving. Remote bridges verify the signature against the bridge's known public key (obtainable from the Reticulum announce).

### 6.4 External SPI Flash — Storage Layout

The carrier board provides 8 MB of SPI flash accessible to the ESP32-S3 via SPI. This storage is entirely separate from the ESP32-S3's internal flash. RTNode-2400 uses internal flash for its NVS identity store and announce cache; the SPI flash is unused by the base RTNode-2400 firmware and is available exclusively for bridge data. No changes to RTNode-2400's internal partition table are required.

**SPI bus:** The carrier board provides separate CS pins for the SPI flash and the Wio SX1280 shield. CS pin assignment is confirmed resolved — verify against the carrier schematic before firmware bring-up and document the pin assignments in the project repository.

**Proposed SPI flash layout (LittleFS, 8 MB):**

| Region | Size | Contents |
|---|---|---|
| Manifest store | 1 MB | Contact manifest + LXMF→pubkey lookup table (106 bytes × up to 9,930 entries) |
| Inbound delivery queue | 6 MB | Per-destination packet queue for 868 MHz rate limiting — see §6.5 |
| Reserved | 1 MB | Future use |

### 6.5 Inbound Rate Mismatch — Delivery Queue

The SX1280 backbone at 2.4 GHz can deliver packets to MCU 2 faster than the 868 MHz SX1262 on MCU 1 can drain them to local nodes. The CTS flow control on the UART link (§1.3) only manages the inter-MCU link; it does not prevent the backbone from accumulating a backlog that exceeds MCU 1's radio airtime budget.

MCU 2 maintains a per-destination inbound delivery queue on SPI flash in the 6 MB region allocated above. When an LXMF message arrives from the backbone:

```
Receive LXMF message addressed to known local destination
If MCU 1 is ready (CTS not asserted, queue for this dest is empty):
    Forward immediately via type 0x02 frame
Else:
    Append to per-destination queue on SPI flash
    Record: arrival timestamp, dest pubkey, payload

Drain loop (runs whenever CTS is deasserted):
    For each destination with queued packets (oldest first):
        Dequeue one packet
        Send type 0x02 to MCU 1
        Wait for type 0x06 ACK before dequeueing next packet for same destination
```

**Queue limits:**

| Parameter | Value | Rationale |
|---|---|---|
| Max queue depth per destination | 16 packets | Bounds per-node SPI flash consumption |
| Max total queued packets | 512 | ~6 MB at ~12 KB average entry with metadata |
| Max queue age | 10 minutes | Stale packets dropped on drain; mesh DM semantics do not guarantee delivery |

When the per-destination queue is full, the oldest queued packet for that destination is dropped (head-drop) to make room for the new arrival. This preserves recency and bounds SPI flash wear. MCU 2 does not notify the sender of drops; this matches MeshCore's existing best-effort DM semantics.

**SPI flash wear:** at typical MeshCore mesh traffic rates (tens of messages per day per node), queue write cycles are negligible against the SPI flash endurance rating.

---

## 7. RTNode-2400 Interface Modes — Retained

The existing interface mode logic in RTNode-2400 already implements the flood mitigation the bridge requires. We retain it without change:

| Interface | Mode | Effect |
|---|---|---|
| SX1280 LoRa (MCU2) | `MODE_ACCESS_POINT` | Sole backbone interface; blocks backbone announces from re-propagating to RF |
| UART interface (new) | `MODE_ACCESS_POINT` | Blocks backbone announces from crossing to MeshCore side |

The UART interface must use `MODE_ACCESS_POINT` so that backbone Reticulum announces do not propagate to MCU 1 and from there onto the 868 MHz MeshCore mesh. This is the Reticulum-layer enforcement of the "no advert propagation" principle; MCU 1's hop-count-zero ingress is the MeshCore-layer enforcement.

---

## 8. Protocol Boundary Table

| Traffic type | 868 MHz RF | MCU1 bridge layer | UART | MCU2 UART interface | SX1280 backbone |
|---|---|---|---|---|---|
| Local→Local DM | ✓ direct | Passes through | ✗ | ✗ | ✗ |
| Local→Remote DM | Received by MCU1 | Intercepts | type 0x01 → | LXMF outbound | RF |
| Remote→Local DM | Delivered, hop=0 | Delivers | ← type 0x02 | LXMF inbound | RF |
| Node advertisements | Local only | ✗ never | ✗ never | ✗ never | ✗ never |
| Position data | Local only | ✗ unless opt-in | type 0x05 manifest | Flash storage | Manifest on request |
| Backbone Reticulum announces | ✗ blocked | ✗ | ✗ | MODE_AP blocks | Received only |
| Bridge gateway contact | Single advert | Registers after 0x04 | type 0x03/0x04 | Hash served | ✗ |
| Manifest queries | ✗ never | ✗ | ✗ | Serves on request | RF |

---

## 9. Failure Modes

| Failure | Local mesh behaviour | Bridge behaviour |
|---|---|---|
| MCU 2 power loss | Unaffected | Gateway contact goes stale after MeshCore 4h timeout; queued inbound packets lost |
| SX1280 backbone RF link lost | Unaffected | MCU 2 buffers outbound; retries with exponential backoff (RTNode-2400 built-in); no backbone delivery until RF restored |
| UART CRC errors >5% over 60s | Unaffected | MCU 1 logs, backs off drip rate; MCU 2 continues normally |
| Gateway contact stale | Local nodes stop routing to bridge | MCU 1 re-advertises every 4h |
| MCU 1 power loss | Local mesh unaffected | MCU 2 continues as RTNode; inbound queue accumulates on SPI flash until MCU 1 recovers |
| Manifest stale (MCU 2 reboots) | Unaffected | MCU 1 replays all known nodes via type 0x05 on next boot |
| Inbound queue full (per-destination) | Unaffected | Head-drop: oldest queued packet for that destination dropped silently; best-effort semantics |
| SPI flash failure | Unaffected | MCU 2 continues as plain RTNode-2400 boundary node; manifest and delivery queue unavailable; MCU 1 notified to suppress manifest pushes |

---

## 10. Implementation Path

### Phase 1 — Hardware bring-up
- Flash RTNode-2400 `seeed_xiao_esp32s3_boundary` to MCU 2
- Configure SX1280 backbone channel and uplink peer in RTNode-2400 interface settings
- Verify MCU 2 connects to a backbone Reticulum node over SX1280 RF and announces on Reticulum
- Verify SX1280 LoRa interface initialises in `MODE_ACCESS_POINT`
- UART link with framing and CRC-16/CCITT verified between MCUs
- Type 0x03/0x04 LXMF hash handshake working

### Phase 2 — Gateway contact
- MCU 1 registers single gateway contact on 868 MHz mesh
- Verified: local MeshCore nodes can see and address the gateway
- Verified: hop count is 1, link quality is not spoofed

### Phase 3 — Outbound path (local → backbone)
- MCU 1 intercepts packets to gateway, sends type 0x01 to MCU 2
- MCU 2 UART interface receives, looks up LXMF destination from manifest, sends via Reticulum
- Verify delivery at a remote Reticulum node

### Phase 4 — Inbound path (backbone → local)
- MCU 2 receives LXMF message for local MeshCore node
- UART interface sends type 0x02 to MCU 1 (or queues to SPI flash if CTS asserted)
- MCU 1 delivers to local node with hop count 0
- Verify: packet not re-flooded

### Phase 5 — Manifest
- MCU 1 pushes type 0x05 manifest entries on node discovery
- MCU 2 stores in LittleFS and serves as LXMF resource
- Remote bridge fetches manifest and resolves display names
- Position opt-in and quantisation (min 3 km radius)

### Phase 6 — Hardening
- Seen-message cache on MCU 2: 64-entry LRU keyed on LXMF message hash (SHA-256 of source_hash + destination_hash + timestamp + payload_hash)
- Rate limiting: max 10 inbound deliveries per local node per minute at MCU 1
- MCU 1 LED: gateway contact registered (solid), backbone active (blink), error (fast blink)
- HKDF salt published as protocol constant in project repository

---

## 11. Open Questions

1. **LXMF destination derivation method — critical interoperability conflict with CoreNet.** CoreNet specifies that LXMF destination hashes should be derived via standard Ed25519→Curve25519 conversion (Reticulum's own method), with HKDF as a fallback only where the scalar is inaccessible. This implementation uses HKDF because the nRF52840 nrf_crypto API does not expose the Ed25519 scalar. If another bridge implementation uses direct conversion and this one uses HKDF, derived hashes will differ and cross-bridge delivery will silently fail. Three paths exist: (a) confirm with CoreNet authors that HKDF is an acceptable canonical method for constrained MCU implementations and document it as such; (b) investigate whether the nRF52840 can perform Ed25519→Curve25519 via a lower-level API or by building against a different crypto library (libsodium, TweetNaCl); (c) move destination derivation entirely to MCU 2 (ESP32-S3), which has no such constraint. This is the highest-priority open question and blocks all interoperability testing.

2. **UART interface integration point in microReticulum.** Since we own RTNode-2400, the integration point is a design decision, not a feasibility question. The cleanest approach is implementing the UART interface as a new `InterfaceImpl` alongside the existing SX1280 interface within microReticulum's abstraction layer. This design should be documented and reviewed before starting Phase 3 implementation.

3. **HKDF salt publication.** The salt value in §4.2 must be computed and published only after open question 1 is resolved — if direct Ed25519→Curve25519 conversion is adopted, the salt is not needed. If HKDF is confirmed canonical, the salt must be frozen and published before any interoperability testing.

4. **Manifest query protocol.** The spec states manifests are served as LXMF resources at CoreNet destination `meshcore` / `bridge` on request. The request packet format, response format for multi-entry manifests (stream of 106-byte records vs length-prefixed list), and pagination for large manifests are not yet defined. This must be specified before Phase 5 and should be coordinated with CoreNet to ensure compatibility with software-daemon bridge implementations.

5. **`contact_type = ADV_TYPE_REPEATER` side effects.** The gateway contact is registered as `ADV_TYPE_REPEATER` (2), the closest available type. MeshCore's routing behaviour toward a REPEATER-typed contact needs to be characterised — specifically whether it affects path selection, flooding scope, or message TTL. A dedicated `ADV_TYPE_BRIDGE` should be proposed upstream to CoreNet and MeshCore jointly. Confirm behaviour before Phase 2 field testing.

6. **6-byte pubkey prefix resolution.** `RESP_CODE_CONTACT_MSG_RECV_V3` provides only the first 6 bytes of the sender's public key. The bridge layer resolves the full 32-byte key from the companion contacts list. If two local nodes share the same 6-byte prefix (unlikely but possible), resolution is ambiguous. Document the collision handling policy.

7. **ACK synthesis — companion protocol gap.** CoreNet requires ACKs to be synthesised from LXMF delivery receipts. The MeshCore companion protocol has no `CMD_SEND_ACK` command — `PUSH_CODE_SEND_CONFIRMED` is a push notification from radio to app, not the reverse. Until MeshCore adds this capability upstream, confirmed delivery cannot be signalled back to the local sender. This should be raised with the MeshCore core team as a companion protocol extension request, referencing CoreNet's requirement. In the meantime, document the best-effort limitation in user-facing bridge documentation.

8. **Inbound delivery to unknown local node.** If an LXMF message arrives on MCU 2 addressed to a MeshCore node with no manifest entry, the UART interface returns error 0x02 and the message is silently dropped. Define whether a best-effort retry hold queue should be implemented on MCU 2, or whether silent drop is acceptable.

9. **Region scope enforcement.** §5.4 adds a propagation scope byte to the manifest entry (local / region / global, default region). MCU 2 must enforce this at the LXMF routing layer: do not forward outbound messages to remote bridges outside the node's declared scope, and do not accept inbound messages for nodes declared as `local`. The enforcement mechanism in the UART interface and LXMF pipeline needs to be specified before Phase 5.

10. **Channel message bridging.** CoreNet specifies channel messages should be forwarded to bridges subscribed to the same channel secret. This implementation has no channel message support. Channel bridging requires: channel secret → LXMF destination derivation, subscription management, and a new UART frame type. Out of scope for v3 but should be added to the Phase 4 or 5 implementation path for v4.

11. **Privacy notice publication.** CoreNet requires bridges to publish a privacy notice alongside manifests stating retention, precision, and sharing policy. The format and delivery mechanism for an embedded node publishing a machine-readable privacy notice over LXMF is not defined. Coordinate with CoreNet on a minimal notice schema suitable for constrained devices.

12. **SPI flash failure mode.** If the external SPI flash is unreadable at boot, MCU 2 must continue as a functional RTNode-2400 boundary node and notify MCU 1 to suppress manifest pushes. Define the detection and fallback behaviour, and add a type 0xFF error subcode for this condition.

13. **Power budget and boot sequencing.** The 3V3 shared rail in §1.3 requires a current budget check — no figures are given. Characterise peak current for ESP32-S3 + SX1280 + SPI flash at boot, confirm the rail can supply both MCUs simultaneously, and validate that the 30-second MCU 1 retry interval is sufficient for MCU 2 boot completion.
