# Hardware upgrade notes: isolated RS-485, isolated CAN-FD, TPM (SLB9672), MCU swap to STM32G431

Engineering analysis and part recommendations for the LibreServo v2.3.1 board (`PCB/LibreServo-v2.3.1.sch`, MCU `U1`). This is a decision record and BOM proposal, not yet reflected in the EAGLE `.sch`/`.brd` files — see "Status / next steps" at the bottom for why, and what's needed to finish the layout.

**Decision: MCU is being swapped to STM32G431** (native FDCAN) — see §5. This is the bigger of the two alternatives originally laid out, chosen over adding an external MCP2518FD controller to the existing STM32F302K8U6.

Local datasheet copies for the three parts driving this change are in [`datasheets/`](datasheets/):
- [`ADM2582E-ADM2587E.pdf`](datasheets/ADM2582E-ADM2587E.pdf) — isolated RS-485 transceiver (§2)
- [`ADM3055E-ADM3057E.pdf`](datasheets/ADM3055E-ADM3057E.pdf) — isolated CAN-FD transceiver (§3)
- [`Infineon-SLB9672-TPM2.0-SPI-FW16.xx-datasheet.pdf`](datasheets/Infineon-SLB9672-TPM2.0-SPI-FW16.xx-datasheet.pdf) — TPM (§4)

## 1. MCU CAN-FD capability — verified: no (on the current `U1`, STM32F302K8U6)

`U1` is currently an **STM32F302K8U6** (STM32F302x6/x8 subfamily, Cat.2, UFQFPN32, 64 KB Flash). This subfamily includes ST's classic **bxCAN** peripheral (CAN 2.0B, up to 1 Mbit/s, 8-byte frames only) — it does **not** implement **FDCAN** (the flexible-data-rate peripheral with BRS and up to 64-byte frames). FDCAN was introduced later, starting with the STM32F7/G0/G4/H7/L5 families ([AN5348, "Introduction to FDCAN peripherals for STM32 MCUs"](https://www.st.com/resource/en/application_note/an5348-introduction-to-fdcan-peripherals-for-stm32-mcus-stmicroelectronics.pdf) lists which families have it — F3 is not one of them).

Practically, this was moot either way for the *current* board: the schematic doesn't route `U1`'s CAN pins anywhere and there's no CAN transceiver in the BOM — CAN is unused silicon on this design today. This is exactly why §5 resolves the gap by swapping to a family that has FDCAN natively, rather than bolting a CAN-FD controller onto silicon that never had CAN wired up in the first place.

## 2. Isolated RS-485 transceiver (replaces `U5`, SIT3485EUA)

Current part: `U5`, SIT3485 half-duplex RS-485, MSOP-8, no isolation — the daisy-chain bus (`U$8`/`U$9`, JST-PH-4) shares digital GND directly with every other servo on the chain today.

**Recommendation: Analog Devices [ADM2587E](https://www.analog.com/ADM2587E/datasheet)** (USA, local copy: [`datasheets/ADM2582E-ADM2587E.pdf`](datasheets/ADM2582E-ADM2587E.pdf)). Single-chip signal *and* power isolation (iCoupler + isoPower — no external isolated DC/DC needed), half/full-duplex configurable, ±15 kV ESD, 2.5 kV rms isolation, single 3.3 V or 5 V supply on the logic side, 20-lead wide-body SOIC.

Why this over a signal-only isolator (e.g. TI [ISO1176](https://www.ti.com/product/ISO1176)): board space on this design is very tight, and a signal-only part still needs a separate isolated bus-side supply (push-pull driver + transformer, or a small isolated DC/DC module) — more parts, more area, more failure modes, for a part that's smaller/cheaper only in isolation. If SOIC-20 footprint area turns out to be the binding constraint, ISO1176 + an isolated supply is the fallback.

**Note on what "isolated" actually buys here**: the JST-PH-4 daisy-chain connector carries `Vmot`/`GND` (shared power distribution across the chain) alongside A/B. Isolating the transceiver splits the MCU/digital-logic ground from the RS-485 bus-side reference, which cuts ground-loop coupling and protects against bus miswiring/surges — but it does **not** isolate one servo's power domain from the next, since `Vmot`/`GND` still run straight through the connector by design. True node-to-node galvanic isolation would need the power distribution reworked too; that's a materially bigger change and is out of scope unless you want it.

## 3. Isolated CAN-FD transceiver (new)

**Recommendation: Analog Devices [ADM3055E](https://www.analog.com/media/en/technical-documentation/data-sheets/adm3055e-adm3057e.pdf)** (USA, local copy: [`datasheets/ADM3055E-ADM3057E.pdf`](datasheets/ADM3055E-ADM3057E.pdf)). Same iCoupler/isoPower approach as the ADM2587E above — 5 kV rms signal+power isolation, single 5 V supply, ISO 11898-2:2016 compliant, rated to 5 Mbps CAN FD (up to 12 Mbps demonstrated), 20-lead increased-creepage SOIC.

Picking the same vendor/technology family as the RS-485 part (§2) is deliberate: both isolate power internally, so neither needs its own isolated DC/DC design, and the two parts share a design/verification approach (same iCoupler isolation barrier characterization, same qualification data).

Fallback if SOIC-20 area is a problem: TI [ISO1042](https://www.ti.com/lit/ds/symlink/iso1042.pdf) (signal-only isolation, needs an external isolated supply — same tradeoff as ISO1176 above).

## 4. TPM: Infineon SLB9672

[SLB9672](https://www.infineon.com/assets/row/public/documents/30/49/infineon-slb9672-tpm20-spi-fw16.xx-ds-rev1-3-2024-11-18-datasheet-en.pdf) (Germany/EU, local copy: [`datasheets/Infineon-SLB9672-TPM2.0-SPI-FW16.xx-datasheet.pdf`](datasheets/Infineon-SLB9672-TPM2.0-SPI-FW16.xx-datasheet.pdf)) — TPM 2.0, SPI up to 33 MHz, PG-UQFN-32 package, internal power management (no explicit standby control needed).

Connects to `U1` on **SPI1**. On the new STM32G431 (§5), SPI1 is dedicated to the TPM alone — FDCAN is a separate hardware peripheral with its own TX/RX pins, so there's no bus-sharing between the TPM and the CAN-FD interface as there would have been with the external-controller alternative. Needs one GPIO for `~CS` and ideally one for an interrupt/ready line if the firmware wants event-driven reads rather than polling.

**Pin-budget flag**: still worth a full pin audit once the STM32G431 package/pinout is finalized in EAGLE (motor gate drivers, ADC current sense, encoder, USART1, RGB LED, oscillator, FDCAN TX/RX, and now SPI1 + TPM CS/IRQ all need a home) — the G431 has more GPIO headroom than the F302K8 it's replacing, but it's not unlimited, especially if a similarly small package (e.g. K6/K8, UFQFPN32) is kept for footprint/size parity.

## 5. MCU swap: STM32G431 for native FDCAN

Since §1 confirmed the current `U1` (STM32F302K8U6) has no FDCAN peripheral, an isolated CAN-FD *transceiver* alone isn't sufficient — something has to run the CAN-FD protocol/arbitration. Two options were evaluated; **STM32G431 is the one being taken forward**.

**Decision: [STMicroelectronics STM32G431](https://www.st.com/resource/en/datasheet/stm32g431c6.pdf)** series (e.g. STM32G431K6/K8, UFQFPN32) — HQ Plan-les-Ouates, Switzerland, fabs in France/Italy, **EU**. Cortex-M4 @ up to 170 MHz, native FDCAN peripheral (hardware bit-timing, message RAM, no CPU-mediated SPI framing), and is ST's official successor line to F3 — see [AN5094, "Migrating between STM32F334/303 and STM32G431/G474/G491"](https://www.st.com/resource/en/application_note/an5094-migrating-between-stm32f334303-and-stm32g431g474g491-mcus-stmicroelectronics.pdf).

| | Pros | Cons |
|---|---|---|
| **Design impact** | Hardware FDCAN = precise bit timing, higher achievable throughput, no CPU cycles spent bit-banging SPI framing; frees SPI1 for the TPM alone; more GPIO/peripheral headroom than the F302K8 for future features | Full MCU swap: new schematic symbol/footprint, PCB relayout, and a firmware port (clock tree, HAL/peripheral register differences, re-mapping ADC/TIM/USART pins for existing PID/curve-generation/current-sense code) — the biggest-impact item in this whole upgrade |
| **Sourcing / "reach"** | ST has deep, redundant EU + global distribution (Digi-Key, Mouser, Farnell/element14, RS Components, Arrow); G4 is a current mainstream family with a long support runway, not a legacy part being phased out | Bigger BOM/unit-cost delta than the external-controller alternative; verification/requalification effort (schematic, layout, firmware, EMC) is proportionally larger |
| **REACH/RoHS** | ST publishes REACH SVHC and RoHS declarations for this family as standard | — |

**Alternative considered, not chosen — external CAN-FD controller on the existing MCU**: [Microchip MCP2518FD](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/External-CAN-FD-Controller-with-SPI-Interface-DS20006027B.pdf) (Microchip Technology Inc., Chandler, Arizona, **USA**) — SPI CAN-FD controller, would have sat between `U1`'s SPI1 and the ADM3055E transceiver from §3, keeping the STM32F302K8U6 in place. Pros were smallest change/lowest risk (no MCU footprint or firmware-port rework) and equally solid US sourcing (Digi-Key/Mouser/Arrow/Newark, EAR99, standard REACH/RoHS declarations). Cons were SPI-bus contention/latency vs. a hardware peripheral, a bus shared with the TPM, and CPU overhead servicing SPI transactions per frame instead of DMA-into-a-peripheral-FIFO — the STM32G431's native FDCAN avoids all of that, which is why it was picked instead.

**On "UK-sourced"**: there isn't a UK fabless/IDM vendor making a comparable CAN-FD controller or MCU — UK-based semiconductor firms in this space are mostly IP licensors (e.g. Arm) rather than chip suppliers. If UK sourcing specifically matters (e.g. procurement policy), the practical route is buying the STM32G431 through a UK distributor (Farnell/element14 or RS Components, both UK-headquartered) rather than a UK-origin die.

## Status / next steps

This document captures the part selection and the reasoning; it does **not** modify `PCB/LibreServo-v2.3.1.sch` or `.brd`. Those are EAGLE 7.6 XML files, and the new parts (STM32G431 MCU, ADM2587E/ADM3055E's SOIC-20 wide-body isolation packages, SLB9672's PG-UQFN-32) don't have existing library symbols/footprints/3D models in this project's `Propio` library. Hand-editing that XML to fabricate new library parts isn't something I can do reliably or verify visually without EAGLE itself — a wrong pad stack or symbol pin mapping would look "done" in a diff but be wrong on the board. Recommend doing the actual schematic/footprint placement in EAGLE, using this document as the part list and pin-mapping reference; happy to iterate on the design decisions further if useful.

The MCU swap also means a firmware port, not just a layout change: `Src/main.c`, `Src/LS_*.c`, and `Test_LibreServo_v2.ioc` all target the STM32F302K8U6's HAL/register set today (ADC1, TIM1/15/16/17, USART1 pin mappings) and will need re-mapping onto the STM32G431's pinout, plus new FDCAN and SPI1 (TPM) init code.

Open items for whoever does the layout/firmware pass:
- Pick the exact STM32G431 part/package (K6 vs K8 flash size, UFQFPN32 vs a larger package if the extra FDCAN/SPI/TPM pins don't fit UFQFPN32) and re-map existing peripherals (ADC current sense, TIM1 motor PWM, TIM15/16/17, USART1, encoder read, RGB LED) onto its pinout.
- Full pin audit once that package is chosen, to confirm FDCAN TX/RX + SPI1 (TPM) + everything existing all fit.
- Confirm isolation-barrier creepage/clearance fits the existing board outline for two SOIC-20 parts (ADM2587E, ADM3055E).
- Decide whether the isolated-power caveat in §2 (shared `Vmot`/GND on the daisy-chain connector) needs to be addressed, or is acceptable as-is.
