
### **Meshcore Bridge System**

This system utilises a higher-speed **2.4GHz Reticulum backbone** to interconnect separate **868MHz MeshCore** islands. It functions as a high-capacity trunk line, allowing local meshes to communicate over long distances without saturating the slower 868MHz frequency.

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

