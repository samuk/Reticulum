
# MeshCore–Reticulum Bridge Node: Technical Specification

**Status:** Draft for community review  
**Supersedes:** WIP bridge hardware spec (open_hardware.md)  
**Companion document:** LXMF as an Interoperability Plane for MeshCore Wide-Area Bridging (Botterell, KD6O)

---

## Overview

This specification describes a dual-radio bridge node that connects a local 868 MHz MeshCore mesh to a 2.4 GHz Reticulum backbone. The backbone carries inter-region LXMF messages between bridge nodes; it does not carry MeshCore advertisements, position data, or any traffic that would cause flood amplification on the local RF medium.

The governing constraint throughout this document: **the local 868 MHz mesh must behave identically whether or not a bridge is present.** The bridge is invisible to local nodes except as a single gateway contact.

---

## 1. Hardware

### 1.1 Bill of Materials

| Component | Part | Role |
|---|---|---|
| MCU 1 (Access) | Seeed XIAO nRF52840 + Wio SX1262 | 868 MHz MeshCore last-mile |
| MCU 2 (Backbone) | Seeed XIAO nRF52840 + Mikroe SX1280 | 2.4 GHz Reticulum backbone |
| Storage | 8 MB SPI Flash (on Wio board) | Store-and-forward buffer |
| Interconnect | UART + RTS/CTS hardware flow control | MCU-to-MCU link |

### 1.2 Rationale for 2.4 GHz Backbone

The 2.4 GHz SX1280 avoids all EU sub-1 GHz regulatory constraints (duty cycle, EIRP limits, spectrum conflicts with Meshtastic at 869.525 MHz and MC Narrow at 869.618 MHz). It provides substantially higher bandwidth than any sub-1 GHz LoRa configuration, suitable for LXMF store-and-forward at meaningful throughput. Range is shorter than 868 MHz but adequate for urban node-to-node backbone links where elevated placement is available.

### 1.3 Interconnect Wiring

```
MCU 1 (Access)          MCU 2 (Backbone)
─────────────           ────────────────
TX  (D6)   ──────────►  RX  (D7)
RX  (D7)  ◄──────────   TX  (D6)
CTS (D8)  ◄──────────   RTS (D9)   [MCU1 asserts CTS high to pause MCU2]
RTS (D9)   ──────────►  CTS (D8)
GND        ──────────   GND
```

**Flow control logic:** MCU 1 asserts CTS HIGH when the 868 MHz radio is actively transmitting or when the local receive queue exceeds 75% capacity. MCU 2 halts UART output immediately on CTS HIGH and resumes on CTS LOW. This prevents serial buffer overflow during backbone-to-local delivery.

**Baud rate:** 115200. This is adequate; the bottleneck is 868 MHz airtime, not the UART link.

---

## 2. Software Architecture

### 2.1 Principles (Non-Negotiable)

These derive from the Botterell RFC and the MeshCore Discussion #1736 / #2093 record:

1. **No advertisement propagation across the bridge.** Remote node adverts are never injected into the local 868 MHz mesh. The LAX↔SEA incident (Discussion #1736) is treated as settled precedent.
2. **One gateway contact per bridge on the local mesh.** The bridge registers a single contact with a display name (e.g., `[GW-UK1]`) and a custom-var field containing its LXMF destination hash. Local nodes address the bridge as a relay; the bridge handles the LXMF leg.
3. **Zero hop-count ingress.** Messages arriving from the backbone are delivered directly to the addressed local node. They are not re-flooded.
4. **Pull-based remote discovery.** Remote contacts are available via manifest query, not pushed onto the local mesh.
5. **The mesh works when the bridge is down.** No local routing dependency on bridge availability.

### 2.2 MCU 1 — Access Firmware (868 MHz / MeshCore)

MCU 1 runs a MeshCore companion firmware modified with the following additions:

#### Packet Framing

MeshCore lacks KISS framing. Implement a length-prefixed framing layer on the UART link between MCU 1 and MCU 2:

```
[ 0xAA ] [ 0x55 ] [ LEN_H ] [ LEN_L ] [ PAYLOAD ... ] [ CRC16 ]
```

- Magic bytes `0xAA 0x55` for sync
- 2-byte big-endian payload length
- CRC16 over payload for integrity

This replaces the silence-timeout approach, which is fragile under variable radio latency and congested channel conditions.

#### Gateway Contact Registration

On boot, MCU 1 registers a single bridge contact on the local mesh:

```c
struct BridgeContact {
    char display_name[32];   // e.g. "[GW-UK1]"
    uint8_t contact_type;    // REPEATER or future BRIDGE type
    uint8_t lxmf_hash[32];  // LXMF destination hash from MCU 2
    uint8_t hop_count;       // Always 1 — bridge is one hop away
    // No position, no telemetry, no link quality spoofing
};
```

**Critically:** hop count is set to 1 because the bridge *is* one hop away. Link quality is reported honestly from measured UART reliability, not forced to maximum. No path spoofing. Local nodes that want to reach a remote node address the gateway contact; they are not deceived into treating the backbone as a free local hop.

#### Inbound Message Handling (Backbone → Local)

1. Receive framed packet from MCU 2 via UART
2. Verify CRC16
3. Strip LXMF envelope — extract MeshCore payload and destination node ID
4. Deliver directly to addressed local node via 868 MHz radio
5. Set MeshCore hop limit to 0 — message is not re-flooded
6. Send LXMF delivery receipt back to MCU 2

#### Outbound Message Handling (Local → Backbone)

1. Receive MeshCore packet addressed to gateway contact
2. Wrap in length-prefixed frame
3. Pass to MCU 2 via UART
4. MCU 2 handles LXMF encapsulation and backbone routing

### 2.3 MCU 2 — Backbone Firmware (2.4 GHz / Reticulum)

MCU 2 runs a minimal Reticulum stack (RNode-compatible, via RadioLib SX1280 driver) with LXMF message handling and store-and-forward buffering.

#### LXMF Identity and Addressing

Each bridge node has a stable Reticulum identity (Curve25519 keypair) stored in non-volatile memory on first boot. The LXMF destination hash derived from this identity is the bridge's address on the backbone — this is what MCU 1 registers as the gateway contact's `lxmf_hash`.

Remote node addressing uses LXMF destinations derived from MeshCore Ed25519 public keys via HKDF-SHA256:

```
lxmf_dest = HKDF(
    salt    = "meshcore-lxmf-v1",
    ikm     = meshcore_ed25519_pubkey,
    info    = "destination",
    len     = 32
)
```

This mapping is deterministic and verifiable. It replaces the static 1-byte-ID lookup table, which breaks above 256 nodes and does not survive topology changes.

#### SPI Bus Arbitration

SX1280 and 8 MB Flash share the SPI bus. Use SPI transactions with interrupt-driven priority:

```c
// ISR: SX1280 DIO1 interrupt
void sx1280_isr() {
    spi_transaction_begin(PRIORITY_HIGH);
    // Suspend any in-progress flash write — save sector pointer
    flash_suspend();
    sx1280_read_packet(rx_buf);
    flash_resume();
    spi_transaction_end();
    xQueueSendFromISR(rx_queue, rx_buf, NULL);
}
```

Flash writes use sector-aligned 4 KB blocks to maximise write endurance. With 8 MB flash and 4 KB sectors: 2048 sectors. At maximum bridging load (assume 1 packet/second average), flash lifespan exceeds 10 years before sector exhaustion — acceptable without wear levelling, but implement a simple round-robin sector pointer regardless.

#### Store-and-Forward Buffer (Circular FIFO)

The 8 MB flash operates as a FIFO queue for messages destined for the local 868 MHz mesh:

```
Flash layout:
  Bytes 0–3:       Write pointer (sector index)
  Bytes 4–7:       Read pointer (sector index)
  Bytes 8–11:      Packet count
  Sector 1–2047:   Packet slots (4 KB each, length-prefixed)
```

**Write:** Incoming LXMF message for local delivery → write to flash at write pointer → advance write pointer.  
**Read (drip):** If CTS LOW and packet count > 0 → read one packet from flash at read pointer → send to MCU 1 via UART → advance read pointer.  
**Fast path:** Transit packets (destined for another bridge, not the local mesh) are re-transmitted directly from RAM on the 2.4 GHz backbone. They never touch flash. This keeps backbone latency low for multi-hop transit.

#### Contact Manifest Service

MCU 2 publishes an LXMF resource at `meshcore.bridge` containing a signed manifest of locally reachable nodes:

```
Manifest entry (per local node):
  lxmf_dest    [32 bytes]   Derived destination hash
  display_name [32 bytes]   MeshCore display name
  last_seen    [4 bytes]    Unix timestamp
  region_tag   [8 bytes]    e.g. "UK-WAL"
  opt_in_flags [1 byte]     Bit 0: position shared
  lat, lon     [8 bytes]    Quantised to ~3 km if position opt-in set
```

Manifests are served on request only — never pushed. Remote bridges query the manifest to populate their local contact lists. This is the discovery mechanism; it does not involve the 868 MHz RF medium at any point.

---

## 3. Protocol Boundary Summary

| Traffic type | 868 MHz mesh | UART | 2.4 GHz backbone |
|---|---|---|---|
| Local MeshCore DM | ✓ | → MCU2 if remote dest | LXMF delivery |
| Remote DM inbound | Delivered to dest only | ← MCU2 | LXMF receipt |
| Node advertisements | Local only | ✗ never | ✗ never |
| Position data | Local only | ✗ unless opt-in | Manifest only, quantised |
| Transit backbone traffic | ✗ never | ✗ never | RAM fast-path |
| Manifest queries | ✗ never | ✗ | Served by MCU2 |
| Bridge gateway contact | Single advert | Boot registration | ✗ |

---

## 4. Implementation Path

### Phase 1 — Hardware bring-up
- UART link with length-prefixed framing and CRC16
- RTS/CTS flow control verified under load
- SPI arbitration between SX1280 and flash

### Phase 2 — Local mesh integration
- Gateway contact registration on 868 MHz mesh
- Honest hop count and link quality reporting
- Inbound zero-hop delivery to local nodes

### Phase 3 — Backbone
- Reticulum stack on SX1280 via RadioLib
- LXMF identity generation and persistence
- HKDF destination mapping for MeshCore nodes
- Flash FIFO store-and-forward with drip delivery
- RAM fast-path for transit traffic

### Phase 4 — Manifest and discovery
- Contact manifest publication on backbone
- Remote contact list population in MCU 1
- Position opt-in and quantisation

### Phase 5 — Hardening
- Seen-message cache (loop prevention)
- Rate limiting at RF egress
- Operator dashboard: queue depth, delivery stats, manifest audit
- Failure behaviour: graceful degradation when backbone unreachable

---

## 5. What This Spec Does Not Include

The following were in the original spec and are deliberately removed:

- **Source-route spoofing / beacon injection with forced link quality.** This causes the local mesh to prefer the bridge for all traffic regardless of actual path quality, producing routing pathologies and replicating the flood amplification problem at the routing layer. Honest metrics only.
- **Static 1-byte ID → Reticulum hash lookup table.** Replaced by deterministic HKDF derivation from MeshCore public keys. Scales to any network size and survives topology changes.
- **Silence-timeout packet framing.** Replaced by length-prefixed framing with magic bytes and CRC16. Reliable under variable latency.

---

## 6. Open Questions

1. **SX1280 2.4 GHz range in practice.** Urban elevated node-to-node links need characterisation. What spreading factor and bandwidth settings give adequate range without sacrificing too much throughput?
2. **HKDF mapping reversibility.** Remote bridges need to map LXMF destinations back to MeshCore display names. The manifest handles this, but what if a node is reachable but not in any manifest?
3. **MeshCore companion protocol stability.** The companion protocol is not formally specified. What version guarantees exist for the packet format MCU 1 intercepts?
4. **Flash endurance under sustained load.** The 10-year estimate assumes ~1 packet/second average. What is a realistic worst-case load for a well-connected bridge node?
5. **2.4 GHz coexistence with Wi-Fi.** SX1280 LoRa at 2.4 GHz shares spectrum with Wi-Fi and Bluetooth. What channel plan minimises interference in dense urban deployments?








| **Bus** | SPI Priority | Radio interrupts must always pre-empt Flash storage tasks. |

