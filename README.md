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

