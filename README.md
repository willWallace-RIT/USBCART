# USBCART
Potential idea for usb cartridge in the absence of hardware
# Hardware-Locked Base + Revertible Overlay Controller

A dual-tier storage control system designed for embedded hardware, game flash cards, and secure kiosks. It enforces an unwriteable, compressed base storage layer using a hardware Write-Protect line controlled by a supervisor microcontroller (e.g., RP2040 or CPLD), paired with a dynamic Copy-on-Write (CoW) overlay for saves, updates, and runtime patches.

A dedicated hardware call (tactile switch trigger) instantly wipes the overlay partition, reverting the device to its factory baseline in milliseconds without rewrite cycles on the primary image.

---

## Architecture Diagram

```
+--------------------------------------------------------------------------+
|                          SYSTEM CONTROL HOST                             |
|              (Main Application Processor / SBC / Console)                |
+--------------------------------------------------------------------------+
|                                           |
| High-Speed Interface                      | Data Bus / USB / SDIO
| (SPI / SDIO / PCIe)                       |
v                                           v
+-------------------------------+         +--------------------------------+
|     TIER 1: IMMUTABLE BASE    |         |    TIER 2: MUTABLE OVERLAY     |
|   (Hardware Write-Protected)  |         |     (High-Endurance Flash)     |
|                               |         |                                |
|  [ SPI NOR Flash / eMMC ROM ] |         |  [ NAND / NVMe / Scratch NV ]  |
|  - Compressed Baseline Image  |         |  - Runtime Deltas & Patches    |
|  - Read-Only at Hardware Pin  |         |  - User Saves & Config States  |
+-------------------------------+         +--------------------------------+
^                                           ^
|                                           |
+-------------------+     +-----------------+
|     |
v     v
+------------------------------------+
|      MICROCONTROLLER / CPLD        |
|     (e.g., RP2040 / Supervisor)    |
|                                    |
|  - Monitors Hardware Reset Button  |
|  - Controls WP# Pin State        |
|  - Instant Overlay Sector Wipe     |
+------------------------------------+
^
|
[ HW REVERT TRIGGER ]
(Physical Tactile Switch)
```
---

## Circuit Schematic (ASCII Netlist)

```
VCC_3V3
|
+---+
| R1|  10k Ohm (Pull-up)
+---+
|
[SW1: HW_REVERT_BTN]    |
(Tactile)          +---------------------------- U1: GPIO_0
|--/ --+---------+
|
GND
================================================================================
U1: RP2040 MCU
+----------------------------+
| GPIO_0 <--- HW_REVERT_BTN  |
|                            |
| GPIO_1 ---> FLASH_WP_CTRL  |--- (To U2: WP# Pin)
|                            |
| GPIO_2 ---> OVERLAY_RESET  |--- (To U3: Clear Cmd)
+----------------------------+
================================================================================
U2: TIER 1 BASE (SPI Flash / eMMC)
+----------------------------+
Host SPI_CS   --->| CS#                        |
Host SPI_MOSI --->| DI (IO0)                   |
Host SPI_MISO <---| DO (IO1)                   |
Host SPI_CLK  --->| CLK                        |
|                            |
U1: FLASH_WP_CTRL --->| WP#  (Active Low Lock)     |
+----------------------------+
================================================================================
U3: TIER 2 OVERLAY (Mutable Flash/NVMe)
+----------------------------+
Host Data Bus ===>| DATA / CTRL INTERFACE      |
|                            |
U1: OVERLAY_RESET --->| RESET / SECTOR_CLEAR_LINE  |
+----------------------------+
```
---

## Component Pinout & Signal Mapping

| Component | Pin | Signal Name | Target Connection | Functional Description |
| :--- | :--- | :--- | :--- | :--- |
| **SW1** | Leg 1 | `HW_REVERT` | `U1: GPIO_0` | Pulls `GPIO_0` LOW when pressed. |
| **SW1** | Leg 2 | `GND` | Ground Plane | System ground reference. |
| **U1 (MCU)** | `GPIO_0` | `INPUT_REVERT` | `SW1: Leg 1` | Pulled HIGH via 10k resistor; triggers hardware reset on falling edge. |
| **U1 (MCU)** | `GPIO_1` | `FLASH_WP_CTRL`| `U2: Pin 3 (WP#)` | Drives LOW for hardware write-protect, HIGH during authorized updates. |
| **U1 (MCU)** | `GPIO_2` | `OVERLAY_RESET`| `U3: Sector Clear` | Pulse output to zero Superblock/Allocation Table on Tier 2. |
| **U2 (Base)**| `WP#` | `WRITE_PROTECT`| `U1: GPIO_1` | Hardware lock pin. While LOW, flash write/erase commands are ignored by silicon. |
| **U3 (Overlay)**| `DATA` | `HOST_BUS` | Host Processor | Standard storage interface (SDIO/SPI/NVMe) bound to local filesystem overlay. |

---

## Operating States

1. **State 0: Enforced Immutability (Default)**
   * `U1:GPIO_1` is driven **LOW**.
   * Tier 1 (Base Storage) hardware write-protection is locked via `WP#`.
   * Host mounts Tier 1 as Read-Only and maps Tier 2 as an OverlayFS scratchpad.

2. **State 1: Factory State Revert Trigger**
   * User depresses `SW1` (Hardware Revert Switch).
   * `U1:GPIO_0` transitions to **LOW**.
   * Microcontroller interrupts host and asserts `U1:GPIO_2` to issue an instant partition block clear on Tier 2.
   * Host restarts system with pristine baseline mounted from untouched Tier 1.

3. **State 2: System Firmware/Baseline Update Mode**
   * Host initiates authenticated update handshake with `U1`.
   * `U1:GPIO_1` is pulled **HIGH**, releasing hardware write-protect on Tier 1.
   * Baseline image is refreshed, `U1:GPIO_1` drops back **LOW**, and Tier 2 is cleared.
## Appendix: System-Writable Milestone & Update Bank (Tier 1.5)

To allow the system itself to download, manage, and flash alternative version milestones or rollback updates without risking the factory base cartridge, the architecture incorporates an intermediate **Tier 1.5: System-Writable Update Bank**. 

Unlike the permanently locked Base Cartridge (Tier 1) and the temporary runtime scratchpad (Tier 2), this bank is flashable by the host system under strict, MCU-gated cryptographic handshake rules. It provides a persistent, updatable fallback environment that is protected from arbitrary or accidental software corruption.

---

## Architecture Diagram (Tier 1.5 Integration)

```
+--------------------------------------------------------------------------+
|                          SYSTEM CONTROL HOST                             |
|              (Main Application Processor / OS / Console)                 |
+--------------------------------------------------------------------------+
|                                           |
| Read / Unprivileged Exec                  | System Update Command Bus
v                                           v
+-------------------------------+         +--------------------------------+
|     TIER 1: BASE CARTRIDGE    |         |    TIER 1.5: UPDATE BANK ROM   |
|   (Permanently Hardware-Locked)|        |    (System-Writable Milestone) |
|                               |         |                                |
|  - Factory Compressed Master  |         |  - Flashed via OS Handshake    |
|  - Unwritable by Software     |         |  - MCU-Gated Write-Protect     |
+-------------------------------+         +--------------------------------+
^                                           ^
| Hard-Grounded / Fixed                     | MCU-Gated Write Control
+-------------------+     +-----------------+
|     |
v     v
+------------------------------------+
|      MICROCONTROLLER / CPLD        |
|     (e.g., RP2040 / Supervisor)    |
|                                    |
|  - Intercepts Update Handshake     |
|  - Temporarily toggles WP_1.5    |
|  - Manages Boot-Select Priority    |
+------------------------------------+
```
---

## Circuit Schematic & Control Interconnect

```
================================================================================
U1: RP2040 MCU
+----------------------------+
| GPIO_0 <--- REVERT_BTN     |
| GPIO_1 <--- BOOT_SEL_BTN   |  (Selects Base vs. Update)
| GPIO_2 <--- UPDATE_REQ_RX  |  (Serial/SPI host update trigger)
|                            |
| GPIO_3 ---> WP_CTRL_1      |---> (To U2: Tier 1 WP# [Hard Tied LOW])
| GPIO_4 ---> WP_CTRL_1.5    |---> (To U3: Tier 1.5 WP# [Gated])
+----------------------------+
================================================================================
U2: TIER 1 BASE CARTRIDGE (Immutable Factory ROM)
+----------------------------+
Host Bus      --->| DATA / SPI INTERFACE       |
U1: WP_CTRL_3   --->| WP#  (Tied LOW / Permanent)|
+----------------------------+
================================================================================
U3: TIER 1.5 UPDATE BANK (System-Writable Milestone ROM)
+----------------------------+
Host Bus      --->| DATA / SPI INTERFACE       |
U1: WP_CTRL_4    --->| WP#  (Gated Software-Lock) |
+----------------------------+
```
---

## Signal Mapping & Access Rights

| Component | Pin | Signal Name | Target Connection | Functional Description |
| :--- | :--- | :--- | :--- | :--- |
| **U1 (MCU)** | `GPIO_2` | `UPDATE_REQ` | Host Processor | Listens for authorized flash requests from the host OS. |
| **U1 (MCU)** | `GPIO_4` | `WP_1.5` | `U3: Pin 3 (WP#)` | Dynamically lifts hardware write-protection on Tier 1.5 only during verified updates. |
| **U3 (Update Bank)**| `WP#` | `GATED_LOCK` | `U1: GPIO_4` | When LOW, update block is read-only. When pulsed HIGH by MCU, accepts new milestone binary. |

---

## Safe In-System Update Workflow

1. **Restricted Default State:**
   * Both Tier 1 (Base) and Tier 1.5 (Update Bank) default to write-protected states (`WP#` held LOW by the supervisor MCU). The system can boot freely from either, but cannot modify code on either.
2. **Authorized Milestone Flashing:**
   * When a user downloads an official update or alternative version milestone, the host OS sends a signed flash package and a control packet to `U1:GPIO_2` (`UPDATE_REQ`).
   * The MCU verifies the cryptographic signature of the update payload.
   * If valid, the MCU drives `U1:GPIO_4` **HIGH**, temporarily releasing the write-protect pin on Tier 1.5 (while Tier 1 remains locked).
   * The host writes the new compressed milestone image directly to Tier 1.5.
3. **Immediate Re-Locking:**
   * Once writing completes and verification checksums pass, the MCU instantly pulls `U1:GPIO_4` back **LOW**, returning the milestone update ROM to a secure read-only state until the next version cycle.


