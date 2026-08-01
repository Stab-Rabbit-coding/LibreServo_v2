# Hardware upgrade notes: isolated RS-485, isolated CAN-FD, TPM (SLB9672)

Engineering analysis and part recommendations for the LibreServo v2.3.1 board (`PCB/LibreServo-v2.3.1.sch`, MCU `U1`). This is a decision record and BOM proposal, not yet reflected in the EAGLE `.sch`/`.brd` files — see "Status / next steps" at the bottom for why, and what's needed to finish the layout.

## 1. MCU CAN-FD capability — verified: no

`U1` is an **STM32F302K8U6** (STM32F302x6/x8 subfamily, Cat.2, UFQFPN32, 64 KB Flash). This subfamily includes ST's classic **bxCAN** peripheral (CAN 2.0B, up to 1 Mbit/s, 8-byte frames only) — it does **not** implement **FDCAN** (the flexible-data-rate peripheral with BRS and up to 64-byte frames). FDCAN was introduced later, starting with the STM32F7/G0/G4/H7/L5 families ([AN5348, "Introduction to FDCAN peripherals for STM32 MCUs"](https://www.st.com/resource/en/application_note/an5348-introduction-to-fdcan-peripherals-for-stm32-mcus-stmicroelectronics.pdf) lists which families have it — F3 is not one of them).

Practically, this is moot either way for this board: the current schematic doesn't route `U1`'s CAN pins anywhere and there's no CAN transceiver in the BOM — CAN is unused silicon on this design today. So "add CAN-FD" here means building a new interface from nothing, not upgrading an existing one, and it requires solving the FDCAN-controller problem below regardless of transceiver choice.

## 2. Isolated RS-485 transceiver (replaces `U5`, SIT3485EUA)

Current part: `U5`, SIT3485 half-duplex RS-485, MSOP-8, no isolation — the daisy-chain bus (`U$8`/`U$9`, JST-PH-4) shares digital GND directly with every other servo on the chain today.

**Recommendation: Analog Devices [ADM2587E](https://www.analog.com/ADM2587E/datasheet)** (USA). Single-chip signal *and* power isolation (iCoupler + isoPower — no external isolated DC/DC needed), half/full-duplex configurable, ±15 kV ESD, 2.5 kV rms isolation, single 3.3 V or 5 V supply on the logic side, 20-lead wide-body SOIC.

Why this over a signal-only isolator (e.g. TI [ISO1176](https://www.ti.com/product/ISO1176)): board space on this design is very tight, and a signal-only part still needs a separate isolated bus-side supply (push-pull driver + transformer, or a small isolated DC/DC module) — more parts, more area, more failure modes, for a part that's smaller/cheaper only in isolation. If SOIC-20 footprint area turns out to be the binding constraint, ISO1176 + an isolated supply is the fallback.

**Note on what "isolated" actually buys here**: the JST-PH-4 daisy-chain connector carries `Vmot`/`GND` (shared power distribution across the chain) alongside A/B. Isolating the transceiver splits the MCU/digital-logic ground from the RS-485 bus-side reference, which cuts ground-loop coupling and protects against bus miswiring/surges — but it does **not** isolate one servo's power domain from the next, since `Vmot`/`GND` still run straight through the connector by design. True node-to-node galvanic isolation would need the power distribution reworked too; that's a materially bigger change and is out of scope unless you want it.

## 3. Isolated CAN-FD transceiver (new)

**Recommendation: Analog Devices [ADM3055E](https://www.analog.com/media/en/technical-documentation/data-sheets/adm3055e-adm3057e.pdf)** (USA). Same iCoupler/isoPower approach as the ADM2587E above — 5 kV rms signal+power isolation, single 5 V supply, ISO 11898-2:2016 compliant, rated to 5 Mbps CAN FD (up to 12 Mbps demonstrated), 20-lead increased-creepage SOIC.

Picking the same vendor/technology family as the RS-485 part (§2) is deliberate: both isolate power internally, so neither needs its own isolated DC/DC design, and the two parts share a design/verification approach (same iCoupler isolation barrier characterization, same qualification data).

Fallback if SOIC-20 area is a problem: TI [ISO1042](https://www.ti.com/lit/ds/symlink/iso1042.pdf) (signal-only isolation, needs an external isolated supply — same tradeoff as ISO1176 above).

## 4. TPM: Infineon SLB9672

[SLB9672](https://www.infineon.com/assets/row/public/documents/30/49/infineon-slb9672-tpm20-spi-fw16.xx-ds-rev1-3-2024-11-18-datasheet-en.pdf) (Germany/EU) — TPM 2.0, SPI up to 33 MHz, PG-UQFN-32 package, internal power management (no explicit standby control needed).

Connects to `U1` on **SPI1** (`PA5`/`PA6`/`PA7` = SCK/MISO/MOSI), which is currently completely unused in this design (`Test_LibreServo_v2.ioc` only configures `ADC1`, `TIM1/15/16/17`, and `USART1` — no SPI). Needs one GPIO for `~CS` and ideally one for an interrupt/ready line if the firmware wants event-driven reads rather than polling.

**Pin-budget flag**: `U1` is a 32-pin UFQFPN part with a fair number of pins already spoken for (motor gate drivers, ADC current sense, encoder, USART1, RGB LED, oscillator). Adding SLB9672 (SPI + CS [+ IRQ]) is workable since SPI1 is free today, but if §5's Alternative 1 (external CAN-FD controller, also SPI) is chosen, both devices should share the SPI1 bus with independent `~CS` lines rather than each wanting a dedicated bus — do a full pin audit against the schematic before finalizing, this MCU doesn't have much headroom left.

## 5. No on-chip FDCAN controller → two alternatives

Since §1 confirmed `U1` has no FDCAN peripheral, an isolated CAN-FD *transceiver* alone isn't sufficient — something has to run the CAN-FD protocol/arbitration. Two options, both US/EU-sourced:

### Alternative 1 — Keep the MCU, add an external CAN-FD controller

**[Microchip MCP2518FD](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/External-CAN-FD-Controller-with-SPI-Interface-DS20006027B.pdf)** — Microchip Technology Inc., Chandler, Arizona, **USA**. External CAN FD controller, SPI interface (SCK/SDI/SDO/`nCS`, plus `INT`), sits between `U1`'s SPI1 and the ADM3055E transceiver from §3. Arbitration up to 1 Mbps, data phase up to 8 Mbps.

| | Pros | Cons |
|---|---|---|
| **Design impact** | Smallest change: keeps the proven, already-qualified STM32F302K8U6 and existing firmware/motor-control code untouched; no MCU footprint or pinout rework | Extra SPI-bus contention/latency vs. a hardware peripheral; needs a driver stack (register-level MCP2518FD driver) added to firmware; CPU has to service SPI transactions for every frame instead of DMA-into-a-peripheral-FIFO |
| **Sourcing / "reach"** | Microchip is a large US IDM with broad, redundant stocking at Digi-Key, Mouser, Arrow, Newark/Farnell — good availability and long industrial-lifecycle commitment; standard EAR99-class part, no unusual export licensing | Single extra BOM line/cost; if SPI1 is shared with the SLB9672 TPM (§4), bus arbitration and timing budget need care |
| **REACH/RoHS** | Microchip publishes REACH SVHC and RoHS declarations for this part as standard | — |

### Alternative 2 — Replace the MCU with one that has native FDCAN

**STMicroelectronics STM32G431** series (e.g. STM32G431K6/K8, UFQFPN32) — HQ Plan-les-Ouates, Switzerland, fabs in France/Italy, **EU**. Cortex-M4 @ up to 170 MHz, native FDCAN peripheral (hardware bit-timing, message RAM, no CPU-mediated SPI framing), and is ST's official successor line to F3 — see [AN5094, "Migrating between STM32F334/303 and STM32G431/G474/G491"](https://www.st.com/resource/en/application_note/an5094-migrating-between-stm32f334303-and-stm32g431g474g491-mcus-stmicroelectronics.pdf).

| | Pros | Cons |
|---|---|---|
| **Design impact** | Hardware FDCAN = precise bit timing, higher achievable throughput, no CPU cycles spent bit-banging SPI framing; frees SPI1 for the TPM alone; more GPIO/peripheral headroom than the F302K8 for future features | Full MCU swap: new schematic symbol/footprint, PCB relayout, and a firmware port (clock tree, HAL/peripheral register differences, re-mapping ADC/TIM/USART pins for existing PID/curve-generation/current-sense code) — the biggest-impact item in this whole upgrade |
| **Sourcing / "reach"** | ST has deep, redundant EU + global distribution (Digi-Key, Mouser, Farnell/element14, RS Components, Arrow); G4 is a current mainstream family with a long support runway, not a legacy part being phased out | Bigger BOM/unit-cost delta than Alternative 1; verification/requalification effort (schematic, layout, firmware, EMC) is proportionally larger |
| **REACH/RoHS** | ST publishes REACH SVHC and RoHS declarations for this family as standard | — |

**On "UK-sourced"**: there isn't a UK fabless/IDM vendor making a comparable CAN-FD controller or MCU — UK-based semiconductor firms in this space are mostly IP licensors (e.g. Arm) rather than chip suppliers. If UK sourcing specifically matters (e.g. procurement policy), the practical route is buying either of the above through a UK distributor (Farnell/element14 or RS Components, both UK-headquartered) rather than a UK-origin die.

**Recommendation**: Alternative 1 (MCP2518FD) is the lower-risk, lower-effort path and is what I'd default to given how much else is already changing on this board (isolated RS-485 + isolated CAN-FD + TPM all at once). Alternative 2 is the better long-term architecture if a board respin was already on the table for other reasons.

## Status / next steps

This document captures the part selection and the reasoning; it does **not** modify `PCB/LibreServo-v2.3.1.sch` or `.brd`. Those are EAGLE 7.6 XML files, and the new parts (ADM2587E/ADM3055E's SOIC-20 wide-body isolation packages, SLB9672's PG-UQFN-32, MCP2518FD) don't have existing library symbols/footprints/3D models in this project's `Propio` library. Hand-editing that XML to fabricate new library parts isn't something I can do reliably or verify visually without EAGLE itself — a wrong pad stack or symbol pin mapping would look "done" in a diff but be wrong on the board. Recommend doing the actual schematic/footprint placement in EAGLE, using this document as the part list and pin-mapping reference; happy to iterate on the design decisions further if useful.

Open items for whoever does the layout pass:
- Full pin audit of `U1` before committing to SPI1 sharing between TPM and (if Alternative 1) MCP2518FD.
- Confirm isolation-barrier creepage/clearance fits the existing board outline for two SOIC-20 parts.
- Decide whether the isolated-power caveat in §2 (shared `Vmot`/GND on the daisy-chain connector) needs to be addressed, or is acceptable as-is.
