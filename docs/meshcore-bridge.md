# MeshCore–Reticulum Bridge Node: Technical Specification v4

**Status:** Draft for community review  
**Supersedes:** meshcore_bridge_spec_v2.md  
**Companion document:** [LXMF as an Interoperability Plane for MeshCore Wide-Area Bridging](https://github.com/artbotterell/CoreNet) (Botterell, KD6O)  
**Upstream firmware base:** [RTNode-2400](https://github.com/GrayHatGuy/RTNode-2400) (GrayHatGuy / jrl290), microReticulum (attermann), RNode Firmware (markqvist)  
**WIP hardware reference:** [awesome-meshcore open hardware](https://github.com/samuk/awesome-meshcore/blob/main/open_hardware.md)  
**Author** Primarily a word guessing machine, some input from samuk
**License:** GPL-3.0 (inherited)

---

## Overview

This specification describes a dual-MCU MeshCore–Reticulum bridge node consisting of:

- **MCU 1 (MeshCore side):** Seeed XIAO nRF52840 + Wio SX1262, running MeshCore firmware with a bridge companion layer
- **MCU 2 (Reticulum side):** Seeed XIAO ESP32-S3 + Wio SX1280, running [RTNode-2400](https://github.com/GrayHatGuy/RTNode-2400) (microReticulum boundary node)
- **Interconnect:** UART with RTS/CTS hardware flow control

[RTNode-2400](https://github.com/GrayHatGuy/RTNode-2400) provides a working, tested microReticulum boundary node implementation on ESP32-S3 with empirically measured RAM usage of ~29% (94 KB of 327 KB) in boundary mode. This eliminates the need to implement Reticulum from scratch and is the primary architectural change from v2. The CoreNet project ([github.com/artbotterell/CoreNet](https://github.com/artbotterell/CoreNet)) defines the LXMF interoperability plane this bridge implements. WIP carrier hardware is tracked at [awesome-meshcore open hardware](https://github.com/samuk/awesome-meshcore/blob/main/open_hardware.md).

The bridge logic is split across both MCUs according to ownership. On MCU 2 we own RTNode-2400 and develop it directly: the UART interface, LXMF identity mapping, manifest storage, SPI flash driver, and any expansions to the path/announce tables are all first-party RTNode-2400 features. On MCU 1 we do not modify MeshCore; the bridge companion layer operates exclusively within the interface that MeshCore's companion protocol exposes, without touching the core routing stack.

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

### 2.2 MCU 1 — MeshCore Repeater Firmware + Bridge Layer

MCU 1 runs upstream MeshCore **repeater** firmware which we do not and cannot modify. The bridge layer runs alongside it, communicating via the serial interface that the repeater firmware exposes. It does not touch the core routing stack, flooding logic, or radio driver. Any behaviour not achievable through the repeater firmware's serial interface cannot be implemented on MCU 1.

The bridge layer responsibilities:
- On boot: register one gateway contact on the local MeshCore mesh via the repeater serial interface
- Intercept outbound MeshCore packets addressed to the gateway contact and forward via UART to MCU 2
- Receive inbound LXMF payloads from MCU 2 via UART and deliver to addressed local nodes with hop count 0
- Maintain a local manifest of reachable MeshCore nodes for MCU 2 to serve remotely
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
| 0x05 | 90 bytes | Manifest entry (fixed, see §5.4) |
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

**Derivation (performed once per node, at manifest entry time):**

```
Inputs:
  IKM  = meshcore_ed25519_pubkey        (32 bytes, raw little-endian)
  Salt = /* SEE CONSTANT BELOW */       (32 bytes)
  Info = "lxmf-destination-v1"          (UTF-8, no null terminator)
  L    = 16 bytes output

Result:
  lxmf_dest_hash = HKDF-SHA256(IKM, Salt, Info, 16)   (16 bytes, stored directly)
```

HKDF-SHA256 with L=16 produces the destination hash directly. The ESP32-S3 hardware SHA accelerator makes this fast, but it is still called only once per node, not per packet.

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

This constant must be computed, verified by at least two independent implementations, published in the project repository, and treated as immutable thereafter. Any change to the salt invalidates all previously derived LXMF destination hashes and breaks interoperability with deployed bridges.

**Why HKDF rather than direct Ed25519→Curve25519 conversion:**  
Direct conversion is mathematically possible but requires explicit access to the Ed25519 scalar, which is not exposed by the nRF52840 Mbed TLS / nrf_crypto API. HKDF from the public key is implementable with standard primitives on both MCUs and produces a stable, one-way mapping.

**Reverse mapping** (LXMF destination hash → MeshCore pubkey and display name) is stored as a flat lookup table in the SPI flash manifest. On MCU 2, inbound LXMF delivery scans this table by lxmf_dest_hash. A node not present in the manifest cannot be delivered to — see open question 8.

---

## 5. MCU 1 Bridge Layer

### 5.1 Gateway Contact Registration

After receiving the LXMF hash from MCU 2 (type 0x04), MCU 1 registers one contact on the local MeshCore mesh via the repeater firmware serial interface:

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

MCU 2 stores manifest entries received via type 0x05 frames on the external SPI flash (see §6.4). It serves these as an LXMF resource at destination aspect `meshcore.bridge.manifest` when queried by remote bridges.

**Manifest entry format on disk:** identical to the type 0x05 payload (90 bytes per entry), stored as a flat array. The 8 MB SPI flash supports far more than the minimum 64-entry (5,760 byte) working set — the practical limit is the local mesh size, not storage. The implementation should nevertheless enforce a configurable cap (default 512 entries, 46,080 bytes) to bound manifest query response time.

**Manifest signing:** MCU 2 signs the serialised manifest with its Reticulum identity private key before serving. Remote bridges verify the signature against the bridge's known public key (obtainable from the Reticulum announce).

### 6.4 External SPI Flash — Storage Layout

The carrier board provides 8 MB of SPI flash accessible to the ESP32-S3 via SPI. This storage is entirely separate from the ESP32-S3's internal flash. RTNode-2400 uses internal flash for its NVS identity store and announce cache; the SPI flash is unused by the base RTNode-2400 firmware and is available exclusively for bridge data. No changes to RTNode-2400's internal partition table are required.

**SPI bus:** The carrier board provides separate CS pins for the SPI flash and the Wio SX1280 shield. CS pin assignment is confirmed resolved — verify against the carrier schematic before firmware bring-up and document the pin assignments in the project repository.

**Proposed SPI flash layout (LittleFS, 8 MB):**

| Region | Size | Contents |
|---|---|---|
| Manifest store | 1 MB | Contact manifest + LXMF→pubkey lookup table (90 bytes × up to 11,650 entries) |
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

1. **MeshCore repeater firmware serial interface boundary.** We cannot modify MeshCore on MCU 1. The bridge layer is constrained to what the repeater firmware's serial interface exposes. Three specific capabilities need confirmation before implementation can proceed: (a) can a gateway contact be registered and advertised programmatically via the serial interface without user interaction; (b) can inbound packets be injected with an explicit hop count of 0; (c) can outbound packets addressed to a specific contact be intercepted before radio transmission. The repeater firmware's serial interface is less fully documented than the companion protocol — if any of these are not available, the architecture requires renegotiation with upstream MeshCore. This is the highest-priority open question.

2. **UART interface integration point in microReticulum.** Since we own RTNode-2400, the integration point is a design decision, not a feasibility question. The cleanest approach is implementing the UART interface as a new `InterfaceImpl` alongside the existing SX1280 interface within microReticulum's abstraction layer. This design should be documented and reviewed before starting Phase 3 implementation.

3. **HKDF salt publication.** The salt value must be computed, published, and frozen as a protocol constant before any two independent bridge implementations attempt to interoperate. This is a coordination task, not a technical one, but it blocks interoperability.

4. **Manifest query protocol.** The spec states manifests are served as LXMF resources at `meshcore.bridge.manifest` on request. The request packet format, response format for multi-entry manifests (stream of 90-byte records vs length-prefixed list), and pagination for large manifests are not yet defined. This must be specified before Phase 5.

5. **`contact_type = REPEATER` side effects.** The gateway contact is registered as type REPEATER (the closest available type) pending a BRIDGE type being added to MeshCore upstream. MeshCore's routing logic behaviour toward a REPEATER-typed contact needs to be characterised — specifically whether it affects path selection, flooding scope, or message TTL handling in unintended ways. This should be resolved before Phase 2 field testing.

6. **Hop count 0 injection via repeater serial interface.** §5.2 specifies that inbound packets are delivered with hop limit set to 0. Confirm that the repeater firmware serial interface permits construction and injection of a packet with an explicit hop count field. If it does not, the no-reflood guarantee cannot be implemented on MCU 1 without upstream MeshCore changes.

7. **Inbound delivery to unknown local node.** If an LXMF message arrives on MCU 2 addressed to a MeshCore node that has no manifest entry, the UART interface returns error 0x02 and the message is silently dropped. Define whether a best-effort retry hold queue should be implemented on MCU 2 (hold for N minutes pending a manifest entry for that destination), or whether silent drop is acceptable.

8. **SPI flash failure mode.** The manifest store and inbound delivery queue both reside on the external SPI flash. If the device is unreadable at boot, MCU 2 must continue as a functional RTNode-2400 boundary node and notify MCU 1 to suppress manifest pushes. Define the detection and fallback behaviour, and add a type 0xFF error subcode for this condition.

9. **Power budget and boot sequencing.** The 3V3 shared rail in §1.3 is noted as requiring a current budget check but no figures are given. The ESP32-S3 with SX1280 and SPI flash initialisation at boot draws significantly more than steady-state. Characterise peak current, confirm the rail can supply both MCUs simultaneously, and document the expected MCU 2 boot time so the MCU 1 retry interval (currently 30 s) can be validated as sufficient.
