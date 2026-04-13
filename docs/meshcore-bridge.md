


[WIP bridge hardware](https://github.com/samuk/awesome-meshcore/blob/main/open_hardware.md)

### **Meshcore Bridge System**

This system utilises a higher-speed **2.4GHz Reticulum backbone** to interconnect separate **868MHz MeshCore** islands. It functions as a high-capacity trunk line, allowing local meshes to communicate over multiple 2.4Ghz hops without saturating the slower 868MHz frequency.

---

### **1. Hardware Configuration**
* **Dual-MCU Node:** Two Seeed XIAO nRF52840 microcontrollers linked via **UART** (115200 baud).
    * **Unit 1 (Access):** XIAO + Wio SX1262 (868MHz) running **MeshCore**. Handles local "last-mile" user traffic.
    * **Unit 2 (Backbone):** XIAO + Mikroe SX1280 (2.4GHz) running **Reticulum**. Handles high-speed inter-bridge backbone traffic.
* **Flow Control:** Hardware **RTS/CTS** on pins D6/D7 is mandatory. This prevents the fast 2.4GHz side from overflowing the 868MHz side's serial buffer during bursts.
* **Storage Tiers:**
    * **256kB RAM:** Used for immediate packet processing and "Fast-Path" backbone transit.
    * **8MB SPI Flash:** Operates as a **Circular Drip Buffer**. It stores high-speed backbone data and "drips" it into the local mesh at a sustainable rate.

---

### **2. Software & Routing Logic**
* **Source-Route Spoofing:** The bridge firmware injects "spoofed" advertisements into the 868MHz mesh. This tricks MeshCore nodes into seeing the backbone as a low-cost, high-quality "Express Hop" to distant islands.
* **Encapsulation:** Device 2 wraps raw MeshCore blobs into Reticulum frames. Reticulum manages the dynamic, self-healing pathfinding across the 2.4GHz backbone.
* **The "Fast Path":** Transit traffic (data passing through the 2.4GHz node to another bridge) is kept in RAM and re-transmitted immediately, bypassing the SPI Flash to minimize backbone latency.

---

### **3. Implementation Requirements**
* **Serial Framing:** Since MeshCore is KISS-less, Device 1 uses a **5ms silence-timeout** state machine to identify packet boundaries on the UART line.
* **SPI Arbitration:** Both units use interrupt-driven SPI management. The **SX1280 (Radio) is given priority** over the 8MB Flash to ensure no backbone packets are dropped during write cycles.
* **MTU Filtering:** The bridge acts as a protocol filter, stripping Reticulum signatures and overhead before sending data to the 868MHz unit to preserve local airtime.
### **Technical Engineering Specification: Mesh-of-Meshes Bridge**

To build this system, you must develop two separate firmware packages for the XIAO nRF52840s that coordinate via a synchronised serial link. The objective is to make the 2.4GHz backhaul appear as a single, transparent, high-speed cable to the 868MHz network.


### **3.1. Hardware Interconnect & Flow Control**
The link between the two microcontrollers is the primary bottleneck. You must implement hardware flow control to manage the speed discrepancy.
* **Wiring:** Cross-connect D6 (TX) to D7 (RX). You must connect the **RTS (Ready to Send)** and **CTS (Clear to Send)** pins to manage the data flow.
* **Logic:** Unit 1 (868MHz) must pull the CTS line HIGH whenever the 868MHz radio is transmitting or the local mesh is congested. Unit 2 (2.4GHz) must then immediately halt its UART "drip" from the flash memory to prevent serial buffer overflow.

---

### **3.2. Unit 1 (868MHz): MeshCore Access Point**
This unit runs a modified MeshCore stack to handle local "last-mile" traffic.
* **Packet Framing:** Since MeshCore lacks standard KISS framing, implement a **Silence-Timeout** state machine. If the UART line is quiet for >5ms, treat the accumulated bytes as a complete packet and push to the SX1262.
* **Beacon Injection:** Develop a function to take "Distant Node IDs" received from the 2.4GHz backbone and wrap them into local **Node Announcements**. Manually set the "hop count" to 1 and "link quality" to maximum to ensure the local mesh prefers the bridge for all inter-island traffic.

---

### **3.3. Unit 2 (2.4GHz): Reticulum Backbone Node**
This unit manages the Reticulum protocol, the SX1280 radio, and the external storage.
* **SPI Bus Arbitration:** The radio and the 8MB Flash share the same SPI bus. You must use **SPI Transactions**. The SX1280 must have **Interrupt Priority**; if a packet arrives on 2.4GHz while writing to Flash, the MCU must pause the Flash operation to clear the radio buffer.
* **Circular Drip Buffer:** Treat the 8MB Flash as a **FIFO (First-In, First-Out)** queue. 
    * **Write:** Incoming packets from Reticulum meant for the local island are written to the next available Flash sector.
    * **Read (The Drip):** In the main loop, if the 868MHz unit’s CTS line is LOW, read one packet from Flash and send it over UART.
* **Fast-Path Switching:** Use the 256kB internal RAM to inspect headers. "Transit" packets (destined for other bridges) must be re-transmitted on 2.4GHz immediately from RAM, bypassing the SPI Flash entirely to maintain backbone speed.

---

### **3.4. Software Requirements & Milestones**
* **Framework:** Arduino or Zephyr RTOS (preferred for superior interrupt handling on nRF52).
* **Critical Libraries:** RadioLib (for low-level SX1280/SX1262 control) and a custom RNode implementation for the Reticulum stack.
* **Addressing Map:** Implement a lookup table in Unit 2 mapping 1-byte MeshCore IDs to 160-bit Reticulum Destination Hashes (e.g., *IDs 10–20 = South_Bridge_Hash*).

---

### **3.5. Summary of Engineering Tasks**

| Component | Task | Critical Engineering Detail |
| :--- | :--- | :--- |
| **UART** | Packet Framing | 5ms silence-timeout to detect packet boundaries. |
| **Storage** | Flash Lifespan | Use sector-aligned writes to the 8MB Flash. |
| **Routing** | Path Spoofing | Force maximum Link Quality metrics in injected MeshCore adverts. |
| **Bus** | SPI Priority | Radio interrupts must always pre-empt Flash storage tasks. |

