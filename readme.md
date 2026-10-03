# Custom ZMK Wireless Mouse: Redragon Cobra Mod 🖱️🐍

> **Hardware & Firmware Engineering Portfolio**

<p align="center">
  <img src="img/complete_mouse.jpg" alt="Finished Mouse with RGB" width="800"/>
</p>

A comprehensive hardware and firmware project focused on creating a custom wireless mouse, built upon the Zephyr/ZMK ecosystem. A core idea behind this project was to create a versatile tool that seamlessly transitions between my work and home environments.

The project involved **designing a custom PCB**, specifically tailored to fit perfectly within the ergonomic shell of a **Redragon Cobra**. A key goal was to reuse as many components from the original mouse as possible—such as the switches and the wheel mechanism—transforming a traditional wired mouse into a premium, high-performance wireless tool.

Focused on energy efficiency, multiple productivity layers (Office/Gaming), and RGB Underglow lighting, the development overcame several electrical and logical challenges, especially in integrating high-consumption components (LEDs) with a _low-power_ sensor architecture.

## 📸 Project Gallery (Build Log)

<div align="center">
  <img src="img/pcb_editor_kicad.png" alt="KiCad PCB Editor" width="400"/>
  <img src="img/pcb_3d_kicad.png" alt="PCB 3D Render in KiCad" width="400"/>
</div>
<br/>
<div align="center">
  <img src="img/PCBs_compared_to_old.jpg" alt="Comparison with the Old PCB" width="400"/>
  <img src="img/PCB_with_soldered_components.jpg" alt="Soldered PCB" width="400"/>
</div>

## 🛠️ Hardware Specifications & Custom PCB

The hardware architecture was custom-developed to reuse the Redragon Cobra shell and its main functional components (like the original clicks and side switches). The integration includes:

- **Microcontroller:** SuperMini nRF52 (nice!nano alternative based on the nRF52840)
- **Optical Sensor:** PixArt PMW3610 (Since the original was a wired sensor, it was replaced with this Low-Power architecture ideal for wireless)
- **Scroll / Wheel:** TTC Gold 13mm Rotary Encoder (The original encoder was damaged and an exact replacement wasn't available, so it was swapped for this equivalent TTC Gold)
- **Lighting:** WS2812 LED Strip (9 LEDs for underglow)
- **Battery:** 1200mAh 3.7V LiPo (Model 503048, 5x30x48mm)
- **LED Power Control:** Two-stage power-gating circuit (level shifter): an **A19T P-channel MOSFET**(salvaged from a damaged SuperMini) switches the raw battery line (B+) to the LED strip VCC, and its gate is driven by an **SS8050 NPN transistor** (label Y1, salvaged from the original Redragon Cobra PCB) acting as a level shifter. The NPN base is driven by the MCU's `EXT_POWER` pin (GPIO 1.11) through a ~1.4k base resistor (also salvaged from the damaged SuperMini). This guarantees full enhancement and full cut-off of the MOSFET across the entire battery range (3.0V–4.2V).

## ✨ Features and Functionalities

The firmware was designed to act as a hybrid device (Mouse + Keyboard), allowing the execution of complex macros and direct shortcuts on the board.

### Layer System

- **Gaming Layer (Default):** Standard mouse commands (LCLK, RCLK, MCLK, MB4, MB5) with high-sensitivity scroll.
- **Office Layer:** Transforms the mouse into a productivity tool focused on **Simulink** environments. Features built-in macros (`Ctrl + .`, `C`, `E`, `N`, `T`, `E`, `R`) injected in 30ms for quick alignments, compiling, and block organization.
- **Utility Layer (Bluetooth Modifier):** Holding the "Top 3" button transforms the main clicks into a Bluetooth remote control (Force Slot 0, Force Slot 1, Format Memory).

### Physical Combos

- **TOP_1 + TOP_2:** Clean Soft Off (cuts `EXT_POWER` first, then enters deep sleep).
- **TOP_1 + TOP_3:** Toggle RGB On/Off.
- **TOP_2 + TOP_3:** Next RGB effect.
- **SIDE_1 + SIDE_2:** System Reset (Hardware sys_reset).
- **Bottom Button:** Directly triggers System Reset.

> Combos were deliberately moved off the main click buttons (LMB/RMB/MMB): any key position participating in a combo is held for up to `timeout-ms` on every press, which added perceptible latency to middle-click drags.

## ⚡ Engineering Notes & Hardware Solutions

During development, several electrical integration and mechanical design bottlenecks were resolved. If you intend to replicate this project, pay attention to the following design decisions and topology rules:

### 1. PCB Modeling and Component Alignment

Without schematics for the original Redragon Cobra PCB, the new board had to be modeled from scratch. The PCB outline and component placements were reverse-engineered using top-down photos of the original board and a physical ruler for scale.

### 2. 3D Modeling for Fitment Check

To ensure the SuperMini, LiPo battery, and all reused components would fit within the limited internal space of the original shell, a full 3D model of the assembly was created. This step was crucial to guarantee clearance for the new components without modifying the external plastics.

### 3. Wired to Wireless Conversion Challenges

Because the original Redragon Cobra was a wired mouse, several structural challenges were addressed:

- **Sensor Swap:** The original power-hungry wired sensor had to be replaced by the low-power PixArt PMW3610.
- **Charging Port Routing:** To utilize the original cable exit hole at the front of the mouse for charging, a custom-made male-to-female USB-C cable was fabricated. It routes power from the front of the mouse internally back to the SuperMini microcontroller.

### 4. Power Separation (Brownout Prevention)

The 9-LED WS2812 strip consumes up to ~180mA, which exceeds the chronic regulation capacity of the board's LDO, causing a _brownout_ (VCC drop to ~2.6V) if powered by the 3.3V pin.

- **Solution:** The board's VCC NPN Level Shifter + P-MOSFET)
  By default, WS2812 LEDs continuously drain quiescent current even when turned off (displaying black color) via software. To properly maximize battery life during sleep states, a hardware switch was implemented.

**The design constraint:** the LED strip is powered directly from the raw battery rail (B+), which swings between ~3.0V and 4.2V, while the MCU GPIOs only swing between 0V and 3.3V. Driving the gate of a P-channel MOSFET directly from a 3.3V GPIO is therefore unreliable: with a full battery (4.1V–4.2V), a logic-high GPIO produces a VGS of only about **-0.8V**, which is dangerously close to the low threshold voltage (VGS_th) of the A19T. A proper high-side switch needs the gate to be pulled all the way up to the source voltage (B+) to guarantee cut-off, and all the way down to GND to guarantee full enhancement — something a 3.3V GPIO cannot do on its own.

- **Solution (Hardware — two-stage level shifter):** The gate drive is handled by an **SS8050 NPN transistor** (label Y1, salvaged from the original Redragon Cobra PCB) wired as an open-collector level shifter:
  - **A19T P-MOSFET:** Source -> B+, Drain -> LED VCC, with a **10k pull-up resistor between Gate and Source** (keeps the MOSFET fully off by default).
  - **SS8050 NPN:** Collector -> A19T Gate, Emitter -> GND, Base -> `EXT_POWER` GPIO **1.11** through a **~1.4k base resistor** (salvaged from a damaged SuperMini).
- **How it works:** when `EXT_POWER` is low, the NPN is cut off and the 10k resistor pulls the gate up to B+ (VGS = 0V), fully turning the MOSFET off regardless of battery voltage. When `EXT_POWER` goes high (3.3V), the NPN saturates and pulls the gate to GND, producing VGS = -B+ (up to -4.2V), fully enhancing the MOSFET with minimal RDS(on). Both states are now referenced to the correct rails, so the circuit switches cleanly across the whole battery range with no leakage and no heating.

### 6. Boot Noise (LEDs flashing electronic junk)

When powering up the board, the nRF52 has floating pins that injected noise into the WS2812 strip's data channel.

- **Solution (`.overlay`):** Insertion of the `bias-pull-down;` property on the `pinctrl` MOSI pin to anchor the signal at 0V during boot, coupled with the initialization delay `init-delay-ms = <350>;` to wait for the strip's capacitors.

### 7. Physical Surface Conditioning

**Known Limitation:** The PixArt PMW3610 has extremely weak optical emission to save battery. It **does not work properly on wool or felt mousepads**, as the three-dimensional fibers blur the lens. It requires the use of traditional microfiber (cloth/fabric) mousepads or rigid surfaces.

### 8. PCB Rev1 Limitations & Rev2 Updates

**Known Limitation (Rev1):** Revision 1 (rev1) of the custom PCB does not natively route the Battery (B+) directly to the LED's VCC, nor does it include footprints for the power-gating circuit. These modifications (A19T P-MOSFET, SS8050 NPN, and resistors) had to be manually bodged (hardwired) onto the physical board.

**Rev2 Architecture Overhaul:** To eliminate the bulky discrete transistor assembly, the **Revision 2 (rev2)** of the PCB replaces the entire BJT+MOSFET circuit with a single **TPS22917DBV Load Switch** (SOT-23-6). 

- **How it works:** The raw battery line (B+) connects directly to `VIN`, and the MCU's `EXT_POWER` pin (GPIO 1.11) drives the `ON` pin directly. Since the TPS22917 features an internal smart level-shifter, it seamlessly interfaces a 3.3V GPIO with the fluctuating 3.0V–4.2V battery rail, ensuring a clean cut-off with sub-microampere leakage current.
- **Quick Discharge:** The `QOD` pin is tied directly to `VOUT` to leverage the internal 150Ω discharge resistor, instantly draining the LED strip's capacitors during sleep transitions.

> [!WARNING]
> **Rev2 Disclaimer:** While the Rev2 layout integrates these native traces, capacitors (1μ F input / 0.1μ F output), and the SOT-23-6 footprint, **this board revision has not been physically manufactured or tested yet**. If you plan to order it, please review the schematics and Gerber files carefully.


---

_Status: v1.0 (Full logical functionality validated in Hardware)._
