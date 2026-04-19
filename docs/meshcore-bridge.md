# MeshCore–Reticulum Bridge Node: Technical Specification v3

**Status:** Draft for community review  
**Supersedes:** meshcore_bridge_spec_v2.md  
**Companion document:** [LXMF as an Interoperability Plane for MeshCore Wide-Area Bridging (Botterell, KD6O)](https://github.com/artbotterell/CoreNet) 
**Upstream firmware base:** RTNode-2400 (GrayHatGuy / jrl290), microReticulum (attermann), RNode Firmware (markqvist)  
**License:** GPL-3.0 (inherited)

---

## Overview

This specification describes a dual-MCU MeshCore–Reticulum bridge node consisting of:

- **MCU 1 (MeshCore side):** Seeed XIAO nRF52840 + Wio SX1262, running MeshCore firmware with a bridge companion layer
- **MCU 2 (Reticulum side):** Seeed XIAO ESP32-S3 + Wio SX1280, running RTNode-2400 (microReticulum boundary node)
- **Interconnect:** UART with RTS/CTS hardware flow control

RTNode-2400 provides a working, tested microReticulum boundary node implementation on ESP32-S3 with empirically measured RAM usage of ~29% (94 KB of 327 KB) in boundary mode. This eliminates the need to implement Reticulum from scratch and is the primary architectural change from v2.

The MeshCore bridge logic — gateway contact registration, LXMF identity mapping, hop-count-zero ingress delivery, and contact manifest — is implemented as a companion layer on MCU 1, sitting between the MeshCore radio stack and the UART link to MCU 2.

**The governing constraint throughout:** the local 868 MHz mesh behaves identically whether or not the bridge is present. The bridge is invisible to local nodes except as a single gateway contact. The mesh works when the bridge is down.

---

## 1. Hardware
[WIP hardware](https://github.com/samuk/awesome-meshcore/blob/main/open_hardware.md)

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

### 2.1 MCU 2 — RTNode-2400 (No Modifications Required)

MCU 2 runs RTNode-2400 `seeed_xiao_esp32s3_boundary` build without modification. It operates as a standard Reticulum boundary transport node:

- SX1280 LoRa interface in `MODE_ACCESS_POINT`: sole backbone interface; blocks backbone announces from propagating back to RF
- Path table capped at 48 entries; hash list at 32; announce queue at 4
- Backbone announces cached to flash, served on demand when local nodes request paths
- Default route forwarding: unroutable packets forwarded to backbone over SX1280

The only addition required on MCU 2 is **UART serial input/output** handling to forward LXMF-encapsulated MeshCore packets between the UART link and the Reticulum stack. This is a new serial interface type, not a modification to existing interfaces. It can be implemented as a thin shim that reads framed packets from UART and injects them into the microReticulum packet pipeline, and vice versa.

**MCU 2 memory budget (empirical, from RTNode-2400 README):**

```
XIAO ESP32-S3 boundary build:
  RAM:   28.8% used  (94,492 bytes of 327,680)
  Flash: 27.0% used  (1,768,825 bytes of 6,553,600)
```

Headroom is comfortable. The UART shim adds negligible RAM overhead.

### 2.2 MCU 1 — MeshCore + Bridge Companion Layer

MCU 1 runs MeshCore firmware  with an additional bridge companion layer that intercepts relevant packets and manages the UART link to MCU 2. This layer does not modify MeshCore's core routing behaviour.

The bridge companion layer responsibilities:
- On boot: register one gateway contact on the local MeshCore mesh
- Intercept outbound MeshCore packets addressed to the gateway contact and forward via UART
- Receive inbound LXMF payloads from MCU 2 via UART and deliver to addressed local nodes with hop count 0
- Maintain a local manifest of reachable MeshCore nodes for MCU 2 to serve remotely
- Manage CTS flow control

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
N+1–N+2       2        CRC-16/CCITT over bytes 2..N inclusive
                       (polynomial 0x1021, init 0xFFFF)
```

If LEN > 512, emit error frame type 0xFF with error code 0x03 and return to WAIT_MAGIC1 immediately. Do not wait for the declared number of bytes — a corrupted length field will otherwise stall the receiver indefinitely.

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

Each MeshCore node reachable through this bridge needs a stable LXMF destination. Derive deterministically from the MeshCore Ed25519 public key using HKDF-SHA256:

```
Inputs:
  IKM  = meshcore_ed25519_pubkey        (32 bytes, raw little-endian)
  Salt = 0x3f2a1b8e...                  (32 bytes — see note below)
  Info = "lxmf-destination-v1"          (UTF-8, no null terminator)
  L    = 32 bytes output

Intermediate:
  lxmf_preimage = HKDF-SHA256(IKM, Salt, Info, 32)

LXMF destination hash:
  lxmf_dest_hash = SHA-256(lxmf_preimage)[0:16]   (first 16 bytes)
```

**Salt value — computed once, hardcoded in both MCUs:**

```
Salt = SHA-256("meshcore-lxmf-bridge-v1")

To compute:
  python3 -c "import hashlib; print(hashlib.sha256(b'meshcore-lxmf-bridge-v1').hexdigest())"
```

This value must be computed by the implementor, verified independently, and hardcoded in both MCU 1 and MCU 2 firmware as a 32-byte constant. It must be identical across all bridge implementations for interoperability. A reference value should be published in the project repository before first release and treated as a protocol constant thereafter.

**Why HKDF rather than direct Ed25519→Curve25519 conversion:**  
Direct conversion is mathematically possible but requires explicit access to the Ed25519 scalar, which is not exposed by the nRF52840 Mbed TLS / nrf_crypto API. HKDF from the public key is implementable with standard primitives on both MCUs and produces a stable, one-way mapping.

**Reverse mapping** (LXMF destination → MeshCore display name) is handled by the contact manifest (Section 6). A node present in the bridge's coverage area but not yet in any manifest is displayed as `[unknown:XXXX]` where XXXX is the first 4 hex characters of the LXMF destination hash.

---

## 5. MCU 1 Bridge Companion Layer

### 5.1 Gateway Contact Registration

After receiving the LXMF hash from MCU 2 (type 0x04), MCU 1 registers one contact on the local MeshCore mesh:

```c
struct GatewayContact {
    char    display_name[32];   // "[GW-{REGION}]" e.g. "[GW-UK1]"
    uint8_t contact_type;       // REPEATER (or BRIDGE if added upstream)
    uint8_t lxmf_hash[16];      // From MCU2 type 0x04 response
    uint8_t hop_count;          // 1 — bridge is one hop from local nodes
};
```

**Do not:**
- Set hop_count to 0 (reserved for direct local contacts)
- Force link quality to maximum (dishonest, causes routing pathologies)
- Register more than one gateway contact regardless of how many remote nodes are reachable
- Re-register on every boot if the contact and lxmf_hash are unchanged in persistent storage

Re-register only if the lxmf_hash changes, which happens only if MCU 2 is factory-reset. Re-advertise the gateway contact every 4 hours to match MeshCore repeater advert interval.

### 5.2 Inbound Delivery (Backbone → Local, MCU2 → MCU1)

```
Receive type 0x02 frame from MCU 2
Verify CRC
Extract from payload:
  dest_meshcore_pubkey[32]
  meshcore_payload[N]
Find local MeshCore node matching dest_meshcore_pubkey
If found:
  Deliver meshcore_payload to node via SX1262
  Set MeshCore hop limit field to 0 in delivered packet
  Send type 0x06 ACK to MCU 2
If not found:
  Send type 0xFF error 0x02 to MCU 2
```

Hop limit 0 is mandatory. It prevents the packet being re-flooded by any MeshCore repeater that receives it.

### 5.3 Outbound Handling (Local → Backbone, MCU1 → MCU2)

```
Receive MeshCore packet addressed to gateway contact
Extract:
  source_pubkey[32]
  destination_pubkey[32]
  meshcore_payload[N]
Verify destination_pubkey is NOT a locally reachable node (routing error if so)
Build type 0x01 payload:
  source_pubkey[32] || destination_pubkey[32] || meshcore_payload[N]
Send to MCU 2 via UART (honour CTS)
```

### 5.4 Manifest Push

When MCU 1 hears a MeshCore node on the local 868 MHz mesh (new node or updated last_seen), it pushes a type 0x05 manifest entry to MCU 2:

```
Type 0x05 payload (90 bytes):
  Byte 0:      Version = 0x01
  Bytes 1–32:  MeshCore Ed25519 public key (raw)
  Bytes 33–48: LXMF destination hash (derived per Section 4.2)
  Bytes 49–64: Display name (UTF-8, null-padded to 16 bytes)
  Bytes 65–68: Last seen Unix timestamp (uint32 big-endian)
  Bytes 69–76: Region tag (ASCII, null-padded, e.g. "UK-WAL\0\0")
  Byte  77:    Opt-in flags (bit 0: position shared)
  Bytes 78–81: Latitude (int32 microdegrees, 0 if not opted in)
  Bytes 82–85: Longitude (int32 microdegrees, 0 if not opted in)
  Bytes 86–89: Position quantisation radius in metres (uint32, min 3000)
```

MCU 2 stores these entries and serves them as the contact manifest when queried by remote bridges. MCU 1 sends a manifest entry:
- When a new node is first heard
- When a known node's last_seen advances by more than 10 minutes
- On bridge boot (replay all known local nodes to resync MCU 2)

MCU 1 removes a node from its local manifest and sends an update with last_seen = 0 when a node has not been heard for 24 hours.

---

## 6. MCU 2 UART Shim (RTNode-2400 Addition)

RTNode-2400 runs unmodified except for one addition: a UART serial interface shim that bridges the UART link to the microReticulum packet pipeline.

### 6.1 What the Shim Does

The shim is a new interface type alongside the existing SX1280 LoRa and TCP interfaces. It:

1. Reads framed packets from UART (type 0x01 from MCU 1)
2. Unpacks source/destination MeshCore pubkeys and payload
3. Derives LXMF destination hash for the destination pubkey (per Section 4.2)
4. Constructs an LXMF message addressed to the derived destination
5. Injects into the microReticulum send pipeline (same path as any outbound LXMF message)

Inbound (backbone → local):
1. microReticulum delivers an LXMF message addressed to a locally-registered MeshCore node destination
2. Shim extracts destination MeshCore pubkey (reverse-mapped from LXMF hash via manifest)
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

MCU 2 stores manifest entries received via type 0x05 frames in flash (SPIFFS/LittleFS partition, separate from RTNode-2400's existing log partition). It serves these as an LXMF resource at destination aspect `meshcore.bridge.manifest` when queried by remote bridges.

**Manifest entry format on disk:** identical to the type 0x05 payload (90 bytes per entry), stored as a flat array. Maximum 64 entries (5,760 bytes) — adequate for a local mesh of typical size.

**Manifest signing:** MCU 2 signs the serialised manifest with its Reticulum identity private key before serving. Remote bridges verify the signature against the bridge's known public key (obtainable from the Reticulum announce).

### 6.4 Flash Partition Consideration

RTNode-2400 on XIAO ESP32-S3 uses `default_16MB.csv` partition table. Flash usage in boundary build is 27% (1.77 MB of 6.55 MB). Approximately 4.8 MB remains. LittleFS manifest storage of 64 × 90 bytes = 5,760 bytes is negligible. No partition changes required.

---

## 7. RTNode-2400 Interface Modes — Unchanged

RTNode-2400's existing interface mode logic already implements the flood mitigation the bridge requires:

| Interface | Mode | Effect |
|---|---|---|
| SX1280 LoRa (MCU2) | `MODE_ACCESS_POINT` | Sole backbone interface; blocks backbone announces from re-propagating to RF |
| UART shim (new) | `MODE_ACCESS_POINT` | Blocks backbone announces from crossing to MeshCore side |

The UART shim must use `MODE_ACCESS_POINT` so that backbone Reticulum announces do not propagate to MCU 1 and from there onto the 868 MHz MeshCore mesh. This is the Reticulum-layer enforcement of the "no advert propagation" principle; MCU 1's hop-count-zero ingress is the MeshCore-layer enforcement.

---

## 8. Protocol Boundary Table

| Traffic type | 868 MHz RF | MCU1 bridge layer | UART | MCU2 UART shim | SX1280 backbone |
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
| MCU 2 power loss | Unaffected | Gateway contact goes stale after MeshCore 4h timeout |
| SX1280 backbone RF link lost | Unaffected | MCU 2 buffers outbound; retries with exponential backoff (RTNode-2400 built-in); no backbone delivery until RF restored |
| UART CRC errors >5% over 60s | Unaffected | MCU 1 logs, backs off drip rate; MCU 2 continues normally |
| Gateway contact stale | Local nodes stop routing to bridge | MCU 1 re-advertises every 4h |
| MCU 1 power loss | Local mesh unaffected | MCU 2 continues as RTNode; no MeshCore traffic |
| Manifest stale (MCU 2 reboots) | Unaffected | MCU 1 replays all known nodes via type 0x05 on next boot |

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
- MCU 2 UART shim receives, derives LXMF destination, sends via Reticulum
- Verify delivery at a remote Reticulum node

### Phase 4 — Inbound path (backbone → local)
- MCU 2 receives LXMF message for local MeshCore node
- UART shim sends type 0x02 to MCU 1
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

1. **MeshCore companion protocol packet interception.** The companion protocol used by MCU 1 to interface with the MeshCore stack is not formally versioned. The bridge companion layer intercepts packets at this layer. What is the stability guarantee across MeshCore firmware updates? Version negotiation or a stability commitment is needed before this can be deployed at scale.

2. **UART shim integration point in RTNode-2400.** The shim needs to inject packets into the microReticulum send pipeline on MCU 2. The cleanest integration point is as a new `InterfaceImpl` alongside the existing SX1280 interface. Confirm this is feasible within microReticulum 0.2.4's interface abstraction before starting Phase 3.

3. **HKDF salt publication.** The salt value must be computed, published, and frozen as a protocol constant before any two independent bridge implementations attempt to interoperate. This is a coordination task, not a technical one, but it blocks interoperability.

4. **Manifest query protocol.** The spec states manifests are served as LXMF resources at `meshcore.bridge.manifest` on request. The request packet format, response format for multi-entry manifests (stream of 90-byte records vs length-prefixed list), and pagination for large manifests are not yet defined. This must be specified before Phase 5.
