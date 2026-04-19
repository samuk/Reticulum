# MeshCore–Reticulum Bridge Node: Technical Specification v5

**Status:** Draft for community review  
**Supersedes:** meshcore_bridge_spec_v2.md  
**Companion document:** [LXMF as an Interoperability Plane for MeshCore Wide-Area Bridging](https://github.com/artbotterell/CoreNet) (Botterell, KD6O)  
**Upstream firmware base:** [RTNode-2400](https://github.com/GrayHatGuy/RTNode-2400) (GrayHatGuy / jrl290), microReticulum (attermann), RNode Firmware (markqvist)  
**WIP hardware reference:** [awesome-meshcore open hardware](https://github.com/samuk/awesome-meshcore/blob/main/open_hardware.md)  
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

### 4.2 CoreNet DM Routing Model

**The HKDF derivation in earlier versions of this spec was solving the wrong problem and is removed.**

The CoreNet Python reference implementation does not derive per-node LXMF destination hashes from MeshCore Ed25519 keys. Individual MeshCore nodes do not have LXMF destinations. The bridge is the LXMF endpoint. DM routing works as follows:

```
Outbound (@KD6O@SEA hello):
  1. Parse @callsign@region prefix from DM text
  2. Look up peer router "SEA" by name in peer registry → get peer identity_hash
  3. Send LXMF propagated to meshcore.node.<peer_identity_hash>
     f[01]: "hello"
     f[FB]: {target_callsign: "KD6O", source_callsign: "W5XYZ", source_router: "LAX"}
  4. Receiving bridge at SEA looks up "KD6O" in its local contact registry
  5. Delivers as DM to local node with [@W5XYZ@LAX] prefix
```

The MCU 2 contact registry therefore stores:
- **Local contacts:** MeshCore pubkey → display_name (callsign), `lxmf_hash = ""` (not used for local)
- **Remote contacts:** display_name (callsign) → `lxmf_hash` (the peer router's identity hash, not a per-node hash), `router_name`

The `lxmf_hash` on a remote contact is the **peer router's identity hash**, populated when the bridge receives a manifest from that peer. It is not derived from the remote node's MeshCore key.

**Implications for the spec:**

- The type 0x05 UART manifest frame's `lxmf_dest_hash` field (bytes 33–48) is the peer router's LXMF identity hash, not a per-node derived value. All nodes served by the same remote bridge share the same `lxmf_hash` (their bridge's identity hash).
- Open question 1 (HKDF vs Ed25519→Curve25519 derivation) is **resolved and closed** — no per-node LXMF hash derivation is needed.
- The SPI flash manifest lookup for inbound delivery no longer scans by a derived hash — it scans by `target_callsign` from the LXMF `f[FB]` metadata.
- The HKDF salt constant (`LXMF_BRIDGE_SALT`) is not needed and should be removed from the firmware.

**Peer router identity discovery:** MCU 2 learns a peer router's identity hash from its Reticulum announce on the backbone. When a peer announce arrives, MCU 2 stores the router name → identity_hash mapping, which MCU 1 can request via a new type 0x07 UART frame (see §3 packet types — add in next revision).

**Callsign → local node resolution:** When MCU 2 receives an inbound LXMF message with `target_callsign`, it looks up the callsign in its local manifest to find the MeshCore pubkey, then sends a type 0x02 frame to MCU 1.

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

The encoding table specifies inbound text messages use the `ContactMsgRecvV3` encoding delivered as `propagated` LXMF. On MCU 1, this means the bridge layer receives a type 0x02 frame from MCU 2 and delivers it to the local node using `CMD_SEND_TXT_MSG` (0x02) — not `CMD_SEND_RAW_DATA`. This preserves message type information and ensures the local client displays the message correctly rather than as an opaque raw payload.

```
Receive type 0x02 frame from MCU 2
Verify CRC
Extract from payload:
  dest_meshcore_pubkey[32]
  text[N]
Find local MeshCore node matching dest_meshcore_pubkey in companion contacts list
If found:
  Send CMD_SEND_TXT_MSG (0x02) to MeshCore:
    txt_type        = 0x00       // TXT_TYPE_PLAIN
    attempt         = 0x00
    sender_timestamp = current epoch (bridge-synthesized)
    pubkey_prefix   = dest_meshcore_pubkey[0:6]
    text            = text[N]
  path_len in the delivery is 0xFF (direct — not flood-mode)
  Send type 0x06 ACK to MCU 2
If not found:
  Send type 0xFF error 0x02 to MCU 2
```

**Note on path_len:** `CMD_SEND_TXT_MSG` does not take a path_len parameter directly — the companion radio determines routing. To prevent re-flooding, the bridge must ensure the recipient is registered as a direct contact (`out_path_len = -1` or `0xFF`) in the companion contacts list. MCU 2 should set this when registering local nodes. Confirm this is sufficient to prevent re-flood — see open question 6.

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
  Bytes 33–48:  Peer router LXMF identity hash (16 bytes) — the bridge's own identity hash
                for local nodes (used by remote bridges to route replies back);
                for remote contacts, the peer router's identity hash from its Reticulum announce
  Bytes 49–80:  Display name / callsign (UTF-8, null-padded to 32 bytes)
  Bytes 81–84:  Last seen Unix timestamp (uint32 big-endian)
  Bytes 85–92:  Router name / region tag (ASCII, null-padded to 8 bytes, e.g. "UK1\0\0\0\0\0")
  Byte  93:     Propagation scope (0x00 = local, 0x01 = region, 0x02 = global; default 0x01)
  Byte  94:     Opt-in flags (bit 0: wide-area visible, bit 1: position shared, bit 2: telemetry)
  Bytes 95–98:  Latitude (int32 microdegrees, 0 if not opted in)
  Bytes 99–102: Longitude (int32 microdegrees, 0 if not opted in)
  Bytes 103–106: Position quantisation radius in metres (uint32, min 1000; recommended default 10000 = ~1km)
```

**Primary lookup keys on MCU 2:**
- Outbound: `router_name` → peer router identity hash → LXMF destination
- Inbound: `target_callsign` → local node `pubkey[32]` → type 0x02 frame delivery

The `display_name` field doubles as the CoreNet callsign. Case-insensitive matching is required per CoreNet spec §5.2.

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

The CoreNet encoding table specifies: `Ack` → `receipt` transport → "4-byte tag echoed as LXMF message hash; no separate LXMF message needed." This means ACK is handled at the Reticulum transport layer as a delivery receipt, not as a separate application message. When MCU 2 submits an outbound LXMF message to the propagation pipeline, Reticulum will deliver a receipt when the message is accepted by a propagation node; a stronger confirmation arrives when the destination node retrieves the message.

**MCU 2 behaviour:** On receiving a Reticulum delivery receipt for an outbound message, MCU 2 sends a type 0x06 ACK frame to MCU 1 carrying the 4-byte message tag.

**MCU 1 behaviour:** The MeshCore companion protocol still has no `CMD_SEND_ACK` command. `PUSH_CODE_SEND_CONFIRMED` (0x82) cannot be driven from the app side. MCU 1 receives the type 0x06 ACK and logs it, but cannot signal confirmed delivery back to the original local sender. The bridge operates on best-effort semantics for the local-to-remote direction.

This is a known limitation to document in user-facing bridge documentation. It should be raised with the MeshCore core team as a companion protocol extension request.

---

## 6. MCU 2 UART Interface (RTNode-2400 Feature)

The UART interface is a first-party RTNode-2400 feature implemented as a new interface type within microReticulum alongside the existing SX1280 LoRa interface.

### 6.1 What the UART Interface Does

The UART interface operates as a full microReticulum interface type. It handles outbound and inbound message translation between the UART binary framing and CoreNet-compliant LXMF messages.

**Outbound (MCU 1 → backbone):**

1. Reads type 0x01 frame from UART; unpacks `sender_pubkey[32]`, `gateway_pubkey[32]`, `text[N]`
2. **Parse `@callsign@region` prefix** from `text[N]` using the grammar in CoreNet spec §5.2:
   - Pattern: `@<callsign>@<router-name>` at start of message, optional trailing space
   - Extract `target_callsign`, `router_name`, `body` (text after prefix)
   - If no valid prefix: treat as a router command (`who`, `channels`, `bridge-status`, etc.) or return error DM
3. Look up `router_name` in peer registry → get `peer_identity_hash`
   - If not found: send error DM back via type 0x02: `[CoreNet] Unknown router: @<router-name>`
4. Resolve `sender_callsign` from local manifest by matching `sender_pubkey`
5. Construct LXMF `propagated` message:
   - Destination: `meshcore.node.<peer_identity_hash_hex>`
   - `f[01]`: message body (text with prefix stripped)
   - `f[FB]`: `{target_callsign: "KD6O", source_callsign: "W5XYZ", source_router: "UK1"}`
6. Submit to microReticulum LXMF propagation pipeline

**`propagated` transport is mandatory.** The message is addressed to the peer bridge, not the individual node.

**Inbound (backbone → MCU 1):**

1. microReticulum delivers a `propagated` LXMF message to the bridge's own `meshcore.node.<own_hash>` destination
2. Extract from `f[FB]`: `target_callsign`, `source_callsign`, `source_router`; from `f[01]`: message body
3. Look up `target_callsign` in SPI flash manifest → get local MeshCore `pubkey[32]`
   - If not found: log and discard — remote bridge addressed a node not in this bridge's manifest
4. Format inbound text: `[@<source_callsign>@<source_router>] <body>` (CoreNet spec §9.2)
5. Build type 0x02 frame: `dest_meshcore_pubkey[32] || formatted_text[N]`
6. Send to MCU 1 via UART (honour CTS)

### 6.2 Type 0x03 / 0x04 — LXMF Hash Handshake

On receiving type 0x03 from MCU 1, MCU 2 responds with type 0x04:

```
Type 0x04 payload (16 bytes):
  bridge_lxmf_dest_hash[16]   — derived from RTNode-2400's Reticulum identity
```

This hash is stable across reboots (RTNode-2400 persists its Reticulum identity in NVS). It changes only on factory reset.

### 6.3 Manifest Storage and Service

MCU 2 stores manifest entries received via type 0x05 frames on the external SPI flash (see §6.4). It serves these via LXMF at CoreNet destination `meshcore.bridge.<gateway_hash>` when queried by remote bridges.

**CoreNet destination naming convention** (from `meshcore-lxmf-encoding.md`):
- `meshcore.node.<hash>` — individual node destinations
- `meshcore.bridge.<hash>` — this bridge's gateway and manifest endpoint
- `meshcore.channel.<hash>` — group channel destinations (out of scope for v3)

The `<hash>` in each case is the 16-byte LXMF destination hash, hex-encoded. The bridge registers `meshcore.bridge.<own_hash>` as its primary Reticulum destination on boot.

**Manifest query wire protocol** (now specified in CoreNet encoding table):

Remote bridges query this bridge's manifest using `ControlType::NodeDiscoverReq`:
```
Inbound LXMF direct to meshcore.bridge.<gateway_hash>:
  f[FB]: {mc_type: 0x80, region_tag: "UK-WAL"}

Response sequence (one or more LXMF direct to requester):
  NodeDiscoverResp:
    f[FB]: {mc_type: 0x90, nodes: [
      {pubkey: bytes(32), name: str, coords: [lat_udeg, lon_udeg], last_seen: ts},
      ...
    ]}
```

MCU 2 filters the manifest by `region_tag` before responding. If `region_tag` is absent in the request, return all entries with scope `global` only. Paginate if the result exceeds a single LXMF message payload; send multiple `NodeDiscoverResp` messages with sequential page indices in `f[FB]`.

The contact list sequence (`ContactStart` / `Contact` / `ContactEnd`) is used when a remote bridge requests a full contact sync via `CMD_GET_CONTACTS`. Each `Contact` entry uses msgpack dict format: `{mc_type: 0x03, pubkey[32], display_name, flags, path_len, lat_udeg, lon_udeg, last_seen, region_tag}` — this supersedes the 106-byte flat binary struct as the on-wire manifest format for inter-bridge communication. The 106-byte struct remains the MCU 1 ↔ MCU 2 UART format; MCU 2 translates to/from msgpack when serving remote bridges.

**On-disk format:** 106 bytes per entry (type 0x05 struct), flat array on SPI flash. Configurable cap of 512 entries (~54 KB) to bound query response time.

**Manifest signing:** MCU 2 signs the serialised manifest response with its Reticulum identity private key. Remote bridges verify against the bridge's public key from the Reticulum announce.

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

1. **LXMF destination derivation — resolved and closed.** The CoreNet Python reference implementation does not derive per-node LXMF hashes from MeshCore Ed25519 keys. The bridge is the LXMF endpoint; DMs are addressed to peer router identity hashes, not per-node destinations. The HKDF derivation and salt constant are removed from the spec. No per-node hash derivation is needed.

2. **UART interface integration point in microReticulum.** Since we own RTNode-2400, implement as a new `InterfaceImpl` alongside the existing SX1280 interface. Must support `propagated` LXMF transport. Confirm microReticulum 0.2.4's LXMF layer supports propagated delivery before starting Phase 3.

3. **HKDF salt — removed.** Closed by resolution of Q1. No salt constant is needed.

4. **Peer router registry UART protocol.** MCU 2 learns peer router identity hashes from Reticulum announces on the SX1280 backbone. MCU 1 needs to know which peer router names are reachable and what their identity hashes are, so it can validate `@callsign@router-name` addresses before sending type 0x01 frames. Define a new UART frame type (e.g., type 0x07 peer announce / type 0x08 peer roster response) so MCU 2 can push peer registry updates to MCU 1 and MCU 1 can request the current peer list on boot.

5. **Manifest query protocol — resolved.** `ControlType::NodeDiscoverReq` (`mc_type: 0x80`) / `NodeDiscoverResp` (`mc_type: 0x90`) is the wire format per the encoding table. MCU 2 must implement msgpack — see Q14.

6. **`contact_type = ADV_TYPE_REPEATER` side effects.** The gateway contact is registered as `ADV_TYPE_REPEATER` (2). Characterise routing behaviour and propose `ADV_TYPE_BRIDGE` upstream to MeshCore and CoreNet jointly. Confirm before Phase 2 field testing.

7. **Zero-hop inbound delivery mechanism.** §5.2 uses `CMD_SEND_TXT_MSG` to deliver inbound messages. The no-reflood guarantee depends on local nodes being registered with direct-path (`out_path_len = 0xFF`) in the companion contacts list. Confirm this is sufficient to prevent re-flooding by any repeater that receives the transmission.

8. **6-byte vs 8-byte pubkey prefix.** The companion protocol `RESP_CODE_CONTACT_MSG_RECV_V3` provides 6-byte sender prefix; the CoreNet encoding table `f[FB]` uses `sender_prefix[8]` (8 bytes). MCU 1 uses 6-byte prefix for companion protocol lookups; MCU 2 uses 8-byte for LXMF metadata. Document the handling at the UART boundary — truncate to 6 when parsing for local delivery, pad to 8 when constructing LXMF metadata.

9. **Inbound delivery to unknown callsign.** If an LXMF message arrives with a `target_callsign` that has no entry in the SPI flash manifest, the message cannot be delivered. Define whether a retry hold queue is needed or silent drop with logged error is acceptable.

10. **Region scope enforcement.** The propagation scope byte in §5.4 must be enforced in MCU 2's LXMF outbound pipeline. Specify the enforcement point: MCU 2 must not forward outbound messages for nodes with `scope = local`, and must filter manifest responses by router_name for `scope = region`.

11. **`who` and `channels` query handling.** The CoreNet spec §7 mandates that the bridge respond to `who` and `channels` DM commands. MCU 2 must parse these commands from DMs addressed to the gateway contact (messages that don't match a `@callsign@region` prefix per spec §9.1). `who` returns the local roster from the SPI flash manifest formatted per spec §7.1. `channels` is out of scope for v3 — return `[CoreNet] channels query not supported in v3`.

12. **`corenet-ctl` control channel.** CoreNet spec §7.2 requires a signed router-online announcement on boot. MCU 2 must post to `corenet-ctl` on startup. Channel secret for `corenet-ctl` is per-federation — needs a configuration parameter. Out of scope for v3 Phase 1 but should be in Phase 6.

13. **Channel bridging and publication state.** CoreNet spec §8 channel bridging, `::corenet publish::` / `::corenet unpublish::`, loop prevention (seen-tag LRU), and bridge activation notices are all v0.1 MUST-level requirements for channel bridging. Out of scope for v3 but required for v4. Flag upgrade path that does not break v3 nodes.

14. **Identity conflict transparency.** CoreNet spec §10 requires incumbent/latecomer conflict detection and signed reports to `corenet-ctl`. MCU 2 must detect when a peer announce arrives with the same router_name but a different identity_hash. Out of scope for v3 but should be in Phase 6.

15. **msgpack dependency on MCU 2.** The CoreNet encoding uses `f[FB]` msgpack dicts for all metadata. MCU 2 must encode and decode msgpack. Confirm a suitable embedded msgpack library is available for the ESP32-S3 Arduino/IDF toolchain and add it to the RTNode-2400 dependency list before Phase 3.

16. **SPI flash failure mode.** If the external SPI flash is unreadable at boot, MCU 2 must continue as a functional RTNode-2400 boundary node and notify MCU 1 via a type 0xFF error subcode. Define detection and fallback behaviour.

17. **Power budget and boot sequencing.** The 3V3 shared rail in §1.3 requires current budget verification. Characterise peak current for ESP32-S3 + SX1280 + SPI flash at boot, confirm the rail can supply both MCUs simultaneously, and validate that the 30-second MCU 1 retry interval covers MCU 2 boot completion.
