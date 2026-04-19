# MeshCore–Reticulum Bridge Node: Technical Specification v2

**Status:** Draft for community review  
**Supersedes:** meshcore_bridge_spec.md (v1)  
**Companion document:** LXMF as an Interoperability Plane for MeshCore Wide-Area Bridging (Botterell, KD6O)

---

## Overview

This specification describes a dual-radio bridge node connecting a local 868 MHz MeshCore mesh to a 2.4 GHz Reticulum backbone. The backbone carries inter-region LXMF messages between bridge nodes. It does not carry MeshCore advertisements, position data, or any traffic that causes flood amplification on the local RF medium.

**The governing constraint:** the local 868 MHz mesh behaves identically whether or not a bridge is present. The bridge is invisible to local nodes except as a single gateway contact. The mesh works when the bridge is down.

---

## 1. Hardware

### 1.1 Bill of Materials

| Component | Part | Role |
|---|---|---|
| MCU 1 (Access) | Seeed XIAO nRF52840 + Wio SX1262 | 868 MHz MeshCore last-mile |
| MCU 2 (Backbone) | Seeed XIAO nRF52840 + Mikroe SX1280 | 2.4 GHz Reticulum backbone |
| Storage | 8 MB SPI Flash (W25Q64, on Wio board) | Store-and-forward buffer |
| Interconnect | UART + RTS/CTS hardware flow control | MCU-to-MCU link |

### 1.2 Rationale for 2.4 GHz Backbone

Sub-1 GHz options in the EU are constrained:
- 869.525 MHz (Meshtastic default, 250 kHz BW): occupied
- 869.618 MHz (MC Narrow default, 62.5 kHz BW): occupied
- 869.4–869.65 MHz sub-band: 10% duty cycle but largely filled by above
- 865.2 MHz: clean but 1% duty cycle, 25 mW EIRP

The SX1280 at 2.4 GHz avoids all of these constraints. It provides up to 1.6 Mbps at LoRa mode (versus ~5 kbps for 868 MHz LoRa), handles LXMF store-and-forward at meaningful throughput, and needs no EU licence. The tradeoff is range: expect 500–800 m urban, 2–3 km line-of-sight from elevated nodes. For a backbone between infrastructure nodes with rooftop or elevated placement, this is acceptable.

**2.4 GHz channel plan (SX1280, EU):**

SX1280 LoRa coexists poorly with Wi-Fi on channels 1 (2412 MHz), 6 (2437 MHz), and 11 (2462 MHz), each 20–40 MHz wide. Use the following centre frequencies which fall in gaps between common Wi-Fi channels:

| Backbone channel | Centre frequency | Clear of Wi-Fi ch. |
|---|---|---|
| B1 (primary) | 2425 MHz | Between ch.1 and ch.6 |
| B2 (alternate) | 2450 MHz | Between ch.6 and ch.11 |
| B3 (fallback) | 2479 MHz | Above ch.11 |

Use B1 by default. Configure B2 or B3 if a site survey shows sustained Wi-Fi interference. All three use 800 kHz LoRa bandwidth, SF7, giving approximately 150 kbps throughput — adequate for LXMF messaging backbone traffic.

### 1.3 Interconnect Wiring

```
MCU 1 (Access)              MCU 2 (Backbone)
──────────────              ────────────────
TX   D6  ──────────────►    RX   D7
RX   D7  ◄──────────────    TX   D6
CTS  D8  ◄──────────────    RTS  D9
RTS  D9  ──────────────►    CTS  D8
3V3      ────────────────   3V3   (if powered from MCU1)
GND      ────────────────   GND
```

**Flow control:** MCU 1 pulls CTS HIGH when:
- SX1262 is actively transmitting (TX_DONE interrupt not yet received), or
- Local receive queue depth exceeds 12 packets (75% of 16-packet buffer)

MCU 2 halts UART output immediately on CTS HIGH. Resumes on CTS LOW. MCU 2 must buffer at least 2 packets in RAM to absorb CTS latency — do not assume instantaneous halt.

**Baud rate:** 115200. Bottleneck is 868 MHz airtime (~5 kbps effective), not UART.

---

## 2. Packet Framing (UART)

MeshCore lacks KISS framing. The following length-prefixed frame format is used on the UART link between MCU 1 and MCU 2. It is not MeshCore's wire format — it is a transport envelope for inter-MCU communication only.

```
Byte offset   Length   Field
───────────   ──────   ─────
0             1        Magic 0xAA
1             1        Magic 0x55
2             1        Packet type (see below)
3–4           2        Payload length, big-endian uint16 (max 512)
5–N           N        Payload
N+1–N+2       2        CRC-16/CCITT over bytes 2..N inclusive
```

**Packet types:**

| Type byte | Direction | Meaning |
|---|---|---|
| 0x01 | MCU1 → MCU2 | MeshCore packet for backbone delivery |
| 0x02 | MCU2 → MCU1 | LXMF payload for local delivery |
| 0x03 | MCU1 → MCU2 | Gateway contact registration request |
| 0x04 | MCU2 → MCU1 | Manifest entry (response to 0x05) |
| 0x05 | MCU1 → MCU2 | Manifest query |
| 0x06 | Both | Delivery receipt / ACK |
| 0xFF | Both | Error frame (payload = 1-byte error code) |

**CRC:** CRC-16/CCITT (polynomial 0x1021, init 0xFFFF). Standard implementation available in Arduino `CRC16.h` and Zephyr `sys/crc.h`.

**Framing receiver state machine (both MCUs):**

```
WAIT_MAGIC1 → (0xAA) → WAIT_MAGIC2 → (0x55) → READ_TYPE →
READ_LEN_H → READ_LEN_L → READ_PAYLOAD[0..LEN-1] → CHECK_CRC →
  CRC OK:   dispatch(type, payload)
  CRC FAIL: emit error frame 0x01, return to WAIT_MAGIC1
```

On any unexpected byte in WAIT_MAGIC1/WAIT_MAGIC2, return to WAIT_MAGIC1. Do not implement timeouts for framing — use CRC failure detection instead.

---

## 3. Addressing and Identity

### 3.1 MeshCore Node Identity

MeshCore nodes are identified by 32-byte Ed25519 public keys. The first byte of the public key is used as a short display identifier in the MeshCore UI.

### 3.2 Reticulum / LXMF Identity for the Bridge

On first boot, MCU 2 generates a Curve25519 keypair using the nRF52840's hardware RNG:

```c
uint8_t bridge_private_key[32];  // Curve25519 scalar
uint8_t bridge_public_key[32];   // Curve25519 point
nrf_crypto_rng_vector_generate(bridge_private_key, 32);
curve25519_donna(bridge_public_key, bridge_private_key, basepoint);
```

Store both in non-volatile memory (nRF52840 internal flash, not the SPI flash). Never regenerate unless explicitly reset by the operator.

The LXMF destination hash for the bridge is the first 16 bytes of SHA-256(bridge_public_key). This is the address MCU 1 registers as the gateway contact's LXMF hash and advertises to the local mesh.

### 3.3 Mapping MeshCore Node Keys to LXMF Destinations

Each MeshCore node reachable through this bridge needs a stable LXMF destination so remote bridges can address it directly. Derive this deterministically from the MeshCore Ed25519 public key using HKDF-SHA256 as follows:

```
Inputs:
  IKM  = meshcore_ed25519_public_key   (32 raw bytes, little-endian as stored by MeshCore)
  Salt = SHA-256("meshcore-lxmf-bridge-v1")   (32 bytes, computed once at build time)
  Info = "lxmf-destination-v1"                (UTF-8, no null terminator)
  L    = 32 bytes

Output:
  lxmf_dest_preimage = HKDF-SHA256(IKM, Salt, Info, L)

LXMF destination hash:
  lxmf_dest_hash = first 16 bytes of SHA-256(lxmf_dest_preimage)
```

**Computed at build time, for reference:**
```
Salt = SHA-256("meshcore-lxmf-bridge-v1")
     = 3f8a2c... (implementor must compute and hardcode)
```

This derivation is:
- Deterministic: same MeshCore key always produces same LXMF destination
- One-way: LXMF destination does not reveal the MeshCore key
- Collision-resistant: 128-bit output space
- Verifiable: any bridge can re-derive and verify the mapping

**Reverse mapping** (LXMF destination → MeshCore display name) is handled by the contact manifest (Section 5). A node reachable but absent from any manifest is unresolvable by display name — the bridge should present it as `[unknown:HASH_PREFIX_4]` using the first 4 hex chars of the LXMF destination hash. This is the expected state for new nodes before first manifest synchronisation.

---

## 4. MCU 1 — Access Firmware (868 MHz / MeshCore)

### 4.1 Responsibilities

- Interface with local MeshCore mesh as a companion node
- Register and maintain a single gateway contact on the local mesh
- Receive outbound packets from local nodes addressed to the gateway and forward to MCU 2
- Receive inbound LXMF payloads from MCU 2 and deliver to addressed local nodes with hop count 0
- Manage CTS flow control

### 4.2 Gateway Contact Registration

On boot, after MeshCore companion initialisation, register one contact:

```c
struct GatewayContact {
    char     display_name[32];    // "[GW-{REGION}]", e.g. "[GW-UK1]"
    uint8_t  contact_type;        // REPEATER (or BRIDGE if added upstream)
    uint8_t  lxmf_hash[16];       // Bridge LXMF destination hash from MCU 2
    uint8_t  hop_count;           // 1 — bridge is one hop from local nodes
    int8_t   rssi;                // Measured UART link RSSI proxy: use 0 (unknown)
    uint8_t  snr;                 // Use 0 (unknown)
};
```

**Do not:**
- Set hop count to 0 (that means local direct)
- Set link quality to maximum (dishonest, causes routing pathologies)
- Register more than one gateway contact regardless of how many remote nodes are reachable through it
- Re-register on every boot if the contact already exists in the local node database

Re-register only if the LXMF hash changes (i.e., MCU 2 was reset and generated a new identity).

### 4.3 Inbound Delivery (Backbone → Local)

```
Receive type 0x02 frame from MCU 2
Verify CRC
Extract: destination_meshcore_pubkey[32], meshcore_payload[N]
Find local node matching destination_meshcore_pubkey
If found:
    Deliver meshcore_payload directly to node via SX1262
    Set MeshCore hop limit field to 0 in delivered packet
    Send type 0x06 ACK to MCU 2 with delivery status
If not found:
    Send type 0xFF error 0x02 (node not local) to MCU 2
```

Hop limit 0 means the packet is not re-flooded by any MeshCore repeater that hears it. This is mandatory.

### 4.4 Outbound Handling (Local → Backbone)

```
Receive MeshCore packet addressed to gateway contact
Extract: source_pubkey[32], destination_pubkey[32], payload[N]
Check destination_pubkey is NOT a local node (would be a routing error)
Build type 0x01 frame:
    payload = source_pubkey[32] || destination_pubkey[32] || meshcore_payload[N]
Send to MCU 2 via UART (respecting CTS)
```

---

## 5. MCU 2 — Backbone Firmware (2.4 GHz / Reticulum)

### 5.1 Responsibilities

- Run Reticulum stack on SX1280 via RadioLib
- Maintain LXMF identity and destination mappings
- Route outbound LXMF messages onto backbone
- Receive inbound LXMF messages and queue for local delivery via MCU 1
- Publish and respond to contact manifests
- Manage SPI bus arbitration between SX1280 and flash
- Fast-path transit traffic in RAM

### 5.2 SPI Bus Arbitration

SX1280 and W25Q64 flash share the SPI bus. Both use separate chip-select lines. The SX1280 DIO1 interrupt pre-empts flash operations:

```c
volatile bool flash_op_suspended = false;

// SX1280 DIO1 ISR — highest priority
void sx1280_dio1_isr(void) {
    if (flash_op_in_progress()) {
        w25q64_suspend_write();    // Issue 0x75 suspend command
        flash_op_suspended = true;
    }
    sx1280_service_irq();          // Read packet or handle TX done
    if (flash_op_suspended) {
        w25q64_resume_write();     // Issue 0x7A resume command
        flash_op_suspended = false;
    }
}
```

The W25Q64 supports Program/Erase Suspend (command 0x75) and Resume (0x7A). Maximum suspend latency is 20 µs — acceptable for LoRa packet handling.

### 5.3 Flash Layout and Store-and-Forward Buffer

The W25Q64 has 8 MB = 2048 × 4 KB sectors. MeshCore packets are typically 64–256 bytes. Allocating one 4 KB sector per packet wastes 94–98% of each sector.

Instead, use a packed record format within sectors:

```
Flash layout:
  Sector 0 (4 KB):    Metadata region
    Bytes 0–1:        Magic 0xBE 0xEF (sanity check)
    Bytes 2–3:        Format version (0x0001)
    Bytes 4–7:        Absolute write offset (uint32, byte-addressed within data region)
    Bytes 8–11:       Absolute read offset (uint32, byte-addressed within data region)
    Bytes 12–15:      Packet count (uint32)
    Bytes 16–19:      Total bytes written lifetime (uint32, for endurance tracking)
    Bytes 20–4095:    Reserved / operator config

  Sectors 1–2047 (8 MB - 4 KB = 8,384 KB): Data region
    Packed variable-length records:
      Bytes 0–1:      Record length (uint16, length of payload field only, max 512)
      Bytes 2–3:      Record flags (bit 0: valid, bit 1: delivered)
      Bytes 4–N:      Payload (source[32] || dest[32] || meshcore_data[N-64])
      Bytes N+1–N+2:  CRC-16/CCITT over payload
```

Records are written sequentially. Write offset advances by (4 + payload_len + 2) per record, aligned to 2 bytes. When write offset reaches end of data region, wrap to start of data region (circular).

**Sector erase before write:** Before writing into a new sector, erase it (W25Q64 sector erase command 0x20, ~150 ms). Do not erase sectors ahead of the read pointer — check that write pointer sector ≠ read pointer sector before erasing.

**Flash endurance calculation:**
- W25Q64: 100,000 erase cycles per sector
- 2047 data sectors × 100,000 cycles = 204,700,000 total sector erases
- At 1 packet/second average, 256 bytes/packet: sector fills in ~16 seconds, erased ~5,400 times/day
- Endurance: 204,700,000 / 5,400 = ~37,900 days ≈ **103 years**

At 10× sustained load (10 packets/second): ~10 years. Flash endurance is not a concern at realistic messaging traffic volumes. It would become a concern if used for high-rate telemetry — do not use this bridge for telemetry bridging.

**Metadata sector write frequency:** Metadata (offsets, count) is updated on every packet write and read. The metadata sector will exhaust at:
- 100,000 cycles / (writes_per_second × 2) = 100,000 / 2 = 50,000 seconds at 1 pkt/s ≈ **14 hours**

This is unacceptable. Mitigate by:
1. Buffering metadata updates in RAM and flushing to flash every 60 seconds or on clean shutdown
2. Using a wear-levelled metadata region across 8 sectors (sector 0–7), rotating on each flush

Implement option 1 minimum. Option 2 for production.

### 5.4 Fast-Path Transit

Transit packets are those whose LXMF destination does not match the local bridge or any locally reachable MeshCore node. They should be re-transmitted on the backbone immediately without touching flash:

```c
void handle_inbound_lxmf(LXMFMessage *msg) {
    if (is_local_destination(msg->dest_hash)) {
        // Queue for local delivery via MCU 1
        flash_fifo_write(msg);
    } else {
        // Transit: re-transmit on backbone from RAM
        reticulum_send(msg);  // Non-blocking, uses TX queue in RAM
    }
}
```

### 5.5 Contact Manifest

The bridge publishes a signed manifest of locally reachable MeshCore nodes at Reticulum resource path `meshcore.bridge.manifest`.

**Manifest entry (binary, 89 bytes):**

```
Offset   Length   Field
──────   ──────   ─────
0        16       LXMF destination hash (derived per Section 3.3)
16       32       MeshCore Ed25519 public key (raw bytes)
48       16       Display name (UTF-8, null-padded)
64       4        Last seen (Unix timestamp, uint32 big-endian)
68       8        Region tag (ASCII, null-padded, e.g. "UK-WAL\0\0")
76       1        Opt-in flags:
                    bit 0: position shared
                    bit 1: telemetry shared
77       4        Latitude (int32, microdegrees, 0 if not opted in)
81       4        Longitude (int32, microdegrees, 0 if not opted in)
85       4        Position quantisation radius in metres (uint32)
                    Minimum 3000 (3 km) for publicly shared manifests
```

**Position quantisation:** If bit 0 of opt-in flags is set, snap lat/lon to nearest grid point at the declared quantisation radius before including in the manifest. A node at 51.481583°N, 3.179090°W with 3000 m quantisation would appear as approximately 51.468°N, 3.158°W — enough to indicate region, not enough to locate an individual.

**Manifest publication:** Serve on request only. Never push. Update the local manifest when:
- A new local MeshCore node is heard on the 868 MHz mesh
- An existing node's last-seen timestamp advances by more than 10 minutes
- A node has not been heard for 24 hours (remove from manifest)

**Manifest signing:** Sign the full serialised manifest with the bridge's Curve25519 private key (using Reticulum's standard signing primitive) before serving. Remote bridges verify the signature against the bridge's known public key.

**Unknown node handling:** A node reachable through this bridge but absent from the manifest (e.g., newly joined, not yet heard) is represented to remote bridges as `[unknown:XXXX]` where XXXX is the first 4 hex characters of its derived LXMF destination hash. This is the expected state; it resolves at the next manifest refresh.

---

## 6. Protocol Boundary Table

| Traffic type | 868 MHz RF | UART MCU1↔MCU2 | 2.4 GHz RF |
|---|---|---|---|
| Local DM (both nodes local) | ✓ direct | ✗ | ✗ |
| Local→Remote DM | Received by MCU1 | → MCU2 as type 0x01 | LXMF outbound |
| Remote→Local DM | Delivered to dest, hop=0 | ← MCU2 as type 0x02 | LXMF inbound |
| Node advertisements | Local only | ✗ never | ✗ never |
| Position data | Local only | ✗ unless opt-in | Manifest only, quantised |
| Transit backbone traffic | ✗ never | ✗ never | RAM fast-path |
| Manifest queries | ✗ never | type 0x05 for local data | Served by MCU2 |
| Bridge gateway contact | Single advert on boot | type 0x03 registration | ✗ |
| Telemetry | Local only | ✗ | ✗ |

---

## 7. Failure Modes and Degradation

| Failure | Local mesh behaviour | Bridge behaviour |
|---|---|---|
| MCU 2 power loss | Unaffected | Gateway contact goes stale after MeshCore timeout |
| 2.4 GHz backbone unreachable | Unaffected | Flash buffer fills; drip resumes when backbone recovers |
| Flash full | Unaffected | Oldest undelivered packets overwritten (circular FIFO) |
| UART CRC errors > 5% over 60s | MCU1 logs error | MCU2 backs off to 50% drip rate |
| Gateway contact stale (>4h no advert refresh) | Local nodes stop routing to gateway | Expected — bridge should re-advert on 4h interval |

The bridge should re-transmit its gateway contact advertisement every 4 hours, matching MeshCore repeater advert interval.

---

## 8. Implementation Path

### Phase 1 — Hardware and framing
- UART link with length-prefixed framing, CRC-16/CCITT
- RTS/CTS flow control verified under sustained load
- SPI bus arbitration under interrupt (SX1280 pre-empts flash)
- Both MCUs boot and exchange type 0x06 heartbeat frames

### Phase 2 — Local mesh integration
- MCU 2 generates and persists Curve25519 identity
- MCU 1 registers single gateway contact on 868 MHz mesh
- Inbound delivery with hop count 0 verified on local mesh
- Outbound packets from local nodes reach MCU 2

### Phase 3 — Flash buffer
- Flash FIFO with packed record format
- Metadata RAM buffering with 60s flush
- Drip delivery respecting CTS
- Fast-path transit in RAM verified end-to-end

### Phase 4 — Backbone
- Reticulum stack on SX1280 via RadioLib
- LXMF message send and receive
- HKDF destination mapping for local MeshCore nodes
- End-to-end DM from local node to remote node via two bridges

### Phase 5 — Manifest and discovery
- Contact manifest publication and signing
- Remote manifest fetch and contact list population in MCU 1
- Position opt-in flag handling and quantisation
- Unknown node display name fallback

### Phase 6 — Hardening
- Seen-message cache (64-entry LRU, keyed on LXMF message hash)
- Rate limiting: max 10 inbound deliveries per local node per minute
- Operator LED indicators: backbone link, local mesh activity, buffer depth
- Metadata wear-levelling across 8 sectors (Phase 6, not Phase 1)

---

## 9. Open Questions

These are genuinely open — not deferred design decisions.

1. **MeshCore companion protocol stability.** The companion protocol (USB/BLE/serial) packet format is not formally versioned. What guarantees exist that MCU 1's packet interception remains valid across MeshCore firmware updates? A formal stability commitment or version negotiation handshake is needed.

2. **Reticulum on nRF52840 RAM budget.** The nRF52840 has 256 KB RAM. A minimal Reticulum stack with LXMF, one active link, and the flash FIFO drip logic needs characterisation. Has anyone measured Reticulum RAM usage on this MCU? If it exceeds ~180 KB there is no headroom for the TX queue.

3. **SX1280 LoRa and 2.4 GHz Wi-Fi coexistence in practice.** The channel plan in Section 1.2 is based on Wi-Fi channel centre frequencies and typical 20 MHz channel width. Real deployments use 40 MHz and 80 MHz channels (Wi-Fi 5/6). A site survey tool or empirical interference measurement is needed before finalising the channel plan.

4. **Multi-bridge mesh topology.** This spec describes a single bridge node. When multiple bridges exist in a region, do their backbone Reticulum instances form a mesh automatically via Reticulum's transport routing, or does each bridge need explicit peering configuration? The answer affects the Phase 4 implementation significantly.
