# Hardware upgrade notes: MCU swap to TI MSPM0G3507-Q1 (VQFN-32), S32K144 removed

Supersedes [`S32K144-MCU-swap.md`](S32K144-MCU-swap.md), which replaced the STM32G431 with an
NXP S32K144 (LQFP48) and dropped the SLB9672 TPM in favour of the S32K144's on-chip CSEc.
This pass replaces `U1` again, with a **Texas Instruments MSPM0G3507-Q1 in the 32-pin VQFN
(RHB) package**, orderable part number **`M0G3507QRHBRQ1`**.

Local datasheet copy: [`datasheets/mspm0g3507-q1.pdf`](datasheets/mspm0g3507-q1.pdf) —
*MSPM0G350x-Q1 Automotive Mixed-Signal Microcontrollers With CAN-FD Interface*,
TI literature number **SLASF88C**, October 2023, revised September 2025.
All pin numbers, IOMUX function codes, electrical limits and mechanical dimensions below are
cited to that document; nothing here is inferred from vendor convention or community sources.

Unlike the S32K144 pass, this one is **not** a symbol-only guess: SLASF88C carries the complete
per-package pin table (§6.2, Table 6-2) and signal descriptions (§6.3, Table 6-3) in the data
sheet itself, so every pin assignment below is traceable to a specific silicon pad on a
specific package. That closes the single largest open item carried by the previous two passes.

---

## 1. Why MSPM0G3507-Q1 — and where it is a regression

Texas Instruments (USA, HQ Dallas TX). Honest scorecard against the S32K144 it replaces and
against the STM32F302K8U6 the board was originally laid out for:

| | STM32F302K8U6 (original) | S32K144 (superseded) | **MSPM0G3507-Q1 (this pass)** |
|---|---|---|---|
| Core | Cortex-M4 + FPU @ 72 MHz | Cortex-M4F @ 80/112 MHz | **Cortex-M0+, no FPU, @ 80 MHz** |
| Flash / SRAM | 64 KB / 16 KB | 512 KB / 64 KB | **128 KB (ECC) / 32 KB (parity)** |
| CAN | bxCAN, no FD | 3× FlexCAN, 1 FD | 1× CAN 2.0A/B + **CAN-FD** |
| ADC | 12-bit | 2× 12-bit | **2× 12-bit 4 Msps simultaneous**, 11 ch on this package |
| Other analog | — | — | **1× 12-bit DAC, 2× zero-drift chopper OPA, 3× COMP, GPAMP, internal VREF** |
| Crypto | — | **CSEc (SHE-class HSM)** | AES-128/256, TRNG, CRC — **no HSM** |
| Auto qual | — | ISO 26262 ASIL-B capable | **AEC-Q100 Grade 1**, ISO 26262 ASIL-B, TÜV certified |
| Smallest package | UFQFPN-32, 5 × 5 mm | **LQFP-48, 9 × 9 mm** | **VQFN-32, 5 × 5 mm** |

**Two regressions must be stated plainly, because both are one-way doors:**

1. **The FPU is gone.** Cortex-M0+ has no floating-point unit and no DSP instructions
   (SLASF88C §1 "Core", §8.1). The original STM32F302 *did* have one. `Src/LS_*.c` runs PID
   and curve generation; any `float`/`double` math in that code becomes soft-float library
   calls. Partial mitigation is on-die: SLASF88C §1 lists a **math accelerator (MATHACL)
   supporting DIV, SQRT, MAC and TRIG** (§8 block diagram, `MATHACL` on the CPU-only PD1 bus),
   which is the intended substitute — but taking advantage of it means **reworking the control
   loop in fixed-point**, not recompiling. Budget for that explicitly.
2. **The hardware key store is gone — but secure boot is not.** See §6 for the full,
   corrected treatment. In short: the MSPM0G3507 *does* have a ROM-anchored secure boot with
   **asymmetric ECDSA P-256** image authentication, which is something CSEc (symmetric-only
   SHE) could never do. What it genuinely loses versus CSEc is the **protected key store**, the
   **hardware monotonic counter** (so no hardware rollback protection), and **hardware
   AES-CMAC**.

Against that, the wins that motivate the swap are real: it is the only candidate so far that
**fits the existing 5 × 5 mm board footprint envelope** (the S32K144's smallest package was
9 × 9 mm, an unresolved open item in the previous pass), it brings native CAN-FD, it is
AEC-Q100 Grade 1, and its analog integration (dual simultaneous 4 Msps ADC, chopper OPAs,
comparators with 8-bit reference DACs) is directly useful for motor current sensing.

## 2. Package selection: 32-pin VQFN (RHB), 5 × 5 mm

Requirement is **24 GPIO**: the 21 signals `U1` already carried, plus `HFCLK_IN` for the
existing 8 MHz oscillator, plus `VREF+` and `VREF-`. Audited against Table 6-2's per-package
availability columns:

| Package | Body | GPIO (Table 5-1) | Spares | Verdict |
|---|---|---|---|---|
| 28 VSSOP (DGS28) | 7.1 × 3 mm | 24 | **0** | Fits, but every pin committed; no thermal pad; `PA7`/`PA8`/`PA12`/`PA13` not bonded, forcing CAN onto `PA26`/`PA27` and consuming `PA2`/`ROSC` and `PA5`/`HFXIN` as plain GPIO |
| **32 VQFN (RHB)** | **5 × 5 mm** | **28** | **4** | **Selected** |
| 32 VSSOP (DGS32) | 8.1 × 4.9 mm | 28 | 4 | Same pin count, larger area, no thermal pad |
| 48 VQFN / 48 LQFP | 7 × 7 / 9 × 9 mm | 44 | 20 | Oversized for this board |

**Decision: `RHB`, the 32-pin VQFN.** It is not the smallest by raw area — DGS28 is 21.3 mm²
against RHB's 25.0 mm² — but DGS28 lands at exactly 24 of 24 GPIO with zero margin, which is
the same zero-spare condition that both previous upgrade passes flagged as a recurring problem.
RHB also has an **exposed thermal pad**, so the symbol's existing `EP`-to-`GND` connection
survives, and the 5 × 5 mm body matches the UFQFPN-32 the board outline was drawn around.
32- and 48-pin QFN are additionally available with **wettable flanks** (SLASF88C §1
"Automotive qualification"), which is useful for the AOI/inspection requirements that usually
accompany an AEC-Q100 build.

### 2.1 The existing `QFN32` footprint is NOT reusable

Worth stating explicitly, because both parts are "32-pin, 5 × 5 mm, 0.5 mm pitch QFN" and the
substitution looks safe:

| | Propio `QFN32` (in use for STM32F301/2) | TI `RHB0032E` (SLASF88C §12, drawing 4223442/B) |
|---|---|---|
| Pad 1 location | bottom row, left end | **left column, top** |
| Numbering | bottom → right → top → left | **left → bottom → right → top** |
| Pad size | 0.7 × 0.3 mm | **0.6 × 0.25 mm** |
| Row-to-row span | 5.5 mm | **4.8 mm** |
| Thermal pad | 3.45 × 3.45 mm | 3.45 × 3.45 mm (pad 33) |

The pad numbering differs by a 90° rotation and the land span is 0.7 mm larger. Only the
thermal pad happens to match. **A new footprint is required** — do not bind the deviceset to
`QFN32`. TI's recommended land pattern is fully dimensioned in §12 and is reproduced above.

## 3. `U1` pin assignment

Every row is cited to SLASF88C Table 6-2 (column *48 LQFP, VQFN* → *32 VQFN* pin number) and
its bracketed `PINCM.PF` IOMUX function code. Pins marked *(spare)* are unconnected, terminated
per Table 6-4 ("PAx and PBx — Open; set corresponding pin functions to GPIO and configure
unused pins to output low or input with the internal pullup or pulldown resistor enabled").

| Pad | Pin | Net | Function (PF) | IO type | Notes |
|---:|---|---|---|---|---|
| 1 | `PA0` | `LED_RED` | GPIO [PF1] | **5 V-tol. open drain** | see §3.1 |
| 2 | `PA1` | — | *(spare)* | 5 V-tol. open drain | |
| 3 | `NRST` | `NRST` | reset | Reset | **must be pulled high** — §4 |
| 4 | `VDD` | `+3V3` | power | Power | |
| 5 | `VSS` | `GND` | power | Power | |
| 6 | `PA2` | — | *(spare)* | Standard | also `ROSC`; SYSOSC FCL option preserved |
| 7 | `PA3` | `LED_GREEN` | GPIO [PF1] | Standard | |
| 8 | `PA4` | `PWM-4` | `TIMA0_C3` [PF5] | Standard | → `M4.INA` |
| 9 | `PA5` | `EN-Q` | GPIO [PF1] | Standard | driver enable, `R11`/`R12` |
| 10 | `PA6` | `CLK-8MHZ` | **`HFCLK_IN`** [PF7] | Standard | from oscillator `U$100` — §3.2 |
| 11 | `PA7` | `LED_BLUE` | GPIO [PF1] | Standard | |
| 12 | `PA8` | — | *(spare)* | Standard | |
| 13 | `PA9` | `SEL_SERIAL` | GPIO [PF1] | High-Speed | RS-485 `DE`+`RE` turnaround |
| 14 | `PA10` | `UART0_TX` | `UART0_TX` [PF2] | High-Drive | default BSL `UART_TX` — §3.3 |
| 15 | `PA11` | `UART0_RX` | `UART0_RX` [PF2] | High-Drive | default BSL `UART_RX` |
| 16 | `PA12` | `CAN_TX` | `CAN_TX` [PF5] | High-Speed | → `U6.TXD` (ADM3055E) |
| 17 | `PA13` | `CAN_RX` | `CAN_RX` [PF6] | High-Speed | → `U6.RXD` |
| 18 | `PA14` | — | *(spare)* | High-Speed | also `A0_12` |
| 19 | `PA15` | `V_CORRIENTE` | `A1_0` (analog, PF0) | Standard | **ADC1** — §3.4 |
| 20 | `PA16` | `SPI_IN` | `SPI1_POCI` [PF3] | Standard | |
| 21 | `PA17` | `SPI_CLK` | `SPI1_SCK` [PF3] | Standard | |
| 22 | `PA18` | `SPI_OUT` | `SPI1_PICO` [PF3] | Standard | |
| 23 | `PA19` | `SWDIO` | `SWDIO` [PF2] | High-Speed | dedicated |
| 24 | `PA20` | `SWCLK` | `SWCLK` [PF2] | Standard | dedicated |
| 25 | `PA21` | `GND` | **`VREF-`** (analog) | Standard | §3.5 |
| 26 | `PA22` | `PWM-2` | `TIMA0_C1` [PF5] | Standard | → `M5.INA` |
| 27 | `PA23` | `VDDA-FILTRADO` | **`VREF+`** (analog) | Standard | §3.5 |
| 28 | `PA24` | `PWM-3` | `TIMA0_C3N` [PF4] | Standard | → `M4.INB` |
| 29 | `PA25` | `PWM-1` | `TIMA0_C1N` [PF6] | Standard | → `M5.INB` |
| 30 | `PA26` | `V_TEMP` | `A0_1` (analog, PF0) | Standard | ADC0 |
| 31 | `PA27` | `V_BAT` | `A0_0` (analog, PF0) | Standard | ADC0 |
| 32 | `VCORE` | `VCORE` | core LDO | Power | **0.47 µF tank cap only** — §4 |
| 33 | thermal pad | `GND` | — | — | symbol pin `PAD` |

**Spares: `PA1`, `PA2`, `PA8`, `PA14` (4).** For the first time since the STM32F302, `U1` is
not fully pinned out.

### 3.1 `LED_RED` is deliberately on an open-drain pin

The RGB indicator is **common-anode** (`RGB.CAT_RED/GREEN/BLUE` are the cathodes, fed through
`R6`/`R7`/`R8`), so the MCU **sinks** LED current. `R6` = 187 Ω is much smaller than `R7`
(270 Ω) and `R8` (309 Ω), making red the highest-current channel: on a 3.3 V rail with a
typical red Vf of 1.8–2.2 V that is roughly **6–8 mA**, which straddles or exceeds the **6 mA**
absolute-maximum limit for a Standard-drive pin (Table 7-1, "Current of SDIO pin").

`PA0` and `PA1` are the package's two **5 V-tolerant open-drain** pins, rated **20 mA sink**
(Table 7-1, "Current of ODIO pin"). Their structure — low-side NMOS only, no high-side PMOS
(§9.1) — is exactly right for pulling an LED cathode down, and needs no pull-up because the
LED and its resistor to `+3V3` *are* the load. `LED_RED` therefore lands on `PA0`.
Green and blue draw roughly 1–2.5 mA and stay on Standard-drive pins.

*Firmware note:* `PA0`/`PA1` are also the default BSL I²C pins (Table 6-2). If the BSL is
entered, the red LED may flicker. Harmless — the pin is open-drain either way.

### 3.2 Clock: external square wave, not a crystal

`CLK-8MHZ` is driven by oscillator module `U$100` (`OSC-2X1.6`), i.e. a digital square wave,
not a crystal. The correct destination is therefore **`HFCLK_IN` on `PA6` [PF7]**, the digital
external-clock input, not the `HFXIN`/`HFXOUT` crystal pair. SLASF88C §1 lists "External clock
input" alongside the 4–48 MHz crystal option. The S32K144 symbol's `XTAL` pin had no function
and is deleted, along with its `NC_MARKER` (`U$15`).

Consequence: `PA6` is the only `HFCLK_IN`-capable pin, so committing it forecloses using a
crystal later without re-pinning. `PA5`/`HFXIN` is left as a plain spare-adjacent Standard pin.

### 3.3 RS-485 UART sits on the default BSL pins, deliberately

`UART0_TX`/`UART0_RX` on `PA10`/`PA11` are marked "(Default BSL UART_TX)" / "(Default BSL
UART_RX)" in Table 6-2. Per §8.33 the BSL is **automatically invoked when the reset vector and
stack pointer are unprogrammed**, so a virgin device is reachable over the RS-485 pins with no
BSL-invoke strap and no debug probe. Nothing else competes for these two pins.

**Caveat, so nobody plans around a capability that isn't there:** the BSL ROM drives only
`BSLRX`/`BSLTX`. It does *not* drive `SEL_SERIAL`, which gates the ADM2587E's `DE`/`RE`. With
`SEL_SERIAL` undriven and pulled low by `R20`, the transceiver sits in receive-only mode, so
the BSL's replies never reach the bus — programming over the daisy chain would need a fixture
that asserts `DE` externally. The upside is that this arrangement is also **safe**: a blank
board cannot babble onto a live RS-485 bus.

### 3.4 ADC channel split

`V_CORRIENTE` is the time-critical measurement for the control loop, so it is placed on
**ADC1** (`A1_0`, `PA15`) while `V_TEMP` and `V_BAT` share **ADC0** (`A0_1`, `A0_0`). The two
converters are simultaneous-sampling (SLASF88C §1), so current can be captured in lockstep
with a bus/temperature sample rather than serialised behind it.

`PA15` additionally exposes `OPA1_IN2+`, `COMP0_IN3+` and `COMP1_IN3+`. That leaves the option
open to route the ACS711 current signal through a chopper OPA gain stage, or into a comparator
with its 8-bit reference DAC for a **hardware overcurrent trip**, with no board change beyond
firmware — worth considering, since `EN-Q` is already a hard driver-disable.

### 3.5 `VDDA-FILTRADO` now feeds `VREF+`

The MSPM0G3507 has **no separate `VDDA` pin** — a single `VDD` supplies the die. The board's
existing filtered analog rail (`B1` ferrite + `C4` 1 µF + `C5` 100 nF) would otherwise have
nowhere to land, and the ADC would lose its clean reference.

It is therefore wired to **`VREF+` (`PA23`)**, with **`VREF-` (`PA21`)** tied to `GND`. This is
exactly TI's characterised external-reference condition: SLASF88C §7.12.3 note (2) states that
all external reference specifications are measured with *"V<sub>R+</sub> = VREF+ = VDD and
V<sub>R−</sub> = VSS = 0 V, external 1 µF cap on VREF+ pin"*. It also preserves the
**ratiometric** behaviour the existing `R4`/`R5` battery divider and current-sense scaling were
designed around, which switching to the internal 1.4 V / 2.5 V reference would break.

**The required decoupling already exists:** §7.15.2 specifies C<sub>VREF</sub> = 0.7–1.15 µF
(1 µF typical, 0805 or smaller ceramic, ±20 % acceptable) from `VREF+` to `VREF-`/`GND`.
`C4` is already a 1 µF 0402 on that rail. No new part needed — but `C4` is now a
**functional requirement**, not just supply filtering, and must be placed tight to pad 27.

*Firmware note:* §7.12.3 note (3) — external and internal reference modes are mutually
exclusive. The internal VREF buffer must stay disabled, or it will fight the external rail.

## 4. New support components (all mandatory)

Three parts are added. None were needed by the S32K144 or the STM32 parts before it, and two
of them will stop the board booting if omitted.

| Ref | Value | Net | Requirement |
|---|---|---|---|
| `R21` | 47 kΩ 0402 | `NRST` → `+3V3` | **`NRST` must be pulled to VDD or the device cannot start** (Table 6-4; §9.1: *"NRST **must** be pulled to VDD for the device to start"*). 47 kΩ is TI's recommended value. |
| `C40` | 10 nF 0402 | `NRST` → `GND` | TI-recommended `NRST` filter (§9.1), keeping the pin controllable by a debug probe. |
| `C39` | 470 nF 0402 | `VCORE` → `GND` | §9.1: *"A 0.47 µF tank capacitor is required for the VCORE pin and must be placed close to the device with minimum distance to the device ground. **Do not connect other circuits to the VCORE pin.**"* |

This is a behavioural change worth calling out: the board has **never** had a reset pull-up.
`PG10-NRST` on the STM32G431 and `RESET_b` on the S32K144 were both deliberately left
no-connect, relying on internal pull-ups, and that was a documented design decision. It is no
longer valid — on this device an undriven `NRST` means a dead board. Both `NC_MARKER`s
(`U$15`, `U$16`) that terminated `XTAL` and `RESET_b` are deleted.

`VCORE` (pad 32) is the internal 1.35 V core-LDO output (§7.3: V<sub>CORE</sub> = 1.35 V).
It is **not** a supply input — it must never be tied to `+3V3`.

## 5. What changed in the schematic

`PCB/LibreServo-v2.3.1.sch` (EAGLE 7.6 XML) was edited directly:

- **Library:** symbol and deviceset `S32K144` replaced by `MSPM0G3507`. The symbol keeps every
  retained pin at the **same x/y coordinate** it occupied on the S32K144 symbol, so no existing
  wire had to move; pins were renamed onto real MSPM0 port/function names. Four spare pins were
  added at previously-unused symbol positions, all verified to sit on empty sheet coordinates.
- **`U1`** retargeted to `MSPM0G3507`, value `M0G3507QRHBRQ1`.
- **Pins removed:** `VDD_2` and `VSS_2` (the MSPM0 has one `VDD` and one `VSS` plus the thermal
  pad, not two of each) — their stub wires and now-redundant junctions were removed, leaving
  `C1`/`C2`/`C3`/`SUPPLY24`/`+3V14` connected exactly as before. `XTAL` removed (§3.2).
- **Pins added:** `VCORE`, `NRST`, `VREF+`, `VREF-` and the four spares.
- **Nets added:** `VCORE`, `NRST`, plus the `GND` and `+3V3` segments for `C39`/`C40`/`R21`.
- **Nets deleted:** `N$10`, `N$11` (the two no-connect nets).
- **Nets renamed** to TI peripheral naming — connectivity unchanged, labels follow the net:

  | Before | After |
  |---|---|
  | `LPUART1_TX` / `LPUART1_RX` | `UART0_TX` / `UART0_RX` |
  | `CAN0_TX` / `CAN0_RX` | `CAN_TX` / `CAN_RX` |
  | `SWD_DIO` / `SWD_CLK` | `SWDIO` / `SWCLK` |

- **Sheet annotation** updated (MCU banner and the security note that previously described CSEc).

### Verification performed

Machine-checked against the edited file, not eyeballed:

- **XML well-formed** (`xml.etree` parse).
- **All 29 connected `U1` pins land on the intended net**, and every one of the 33 symbol pins
  resolves to a real package pad; the 4 unconnected pins are exactly the intended spares.
- **Zero shared connection points between different nets** across the entire sheet — every wire
  endpoint, junction and pin coordinate on the whole schematic was resolved (including instance
  rotation) and checked for collisions. This is the check that catches a new wire accidentally
  landing on a foreign net; the new `NRST` trunk in particular crosses the two pre-existing
  long diagonal wires (`GND` and `VDDA-FILTRADO`) without a junction, which is correct.
- **No dangling references:** every `pinref` resolves to an existing instance and symbol pin.

## 6. Security: corrected assessment

An earlier revision of this document claimed the MSPM0G3507 had "no ROM-enforced secure boot".
**That was wrong.** The authoritative source is TI application note **SLAAE29A**, *Cybersecurity
Enablers in MSPM0 MCUs* (January 2023, revised December 2025), local copy at
`../../Open-Secure-ESC/docs/datasheets/slaae29a.pdf`. The MSPM0G3507 sits in that document's
**`M0G3x0x/M0G150x`** column of Table 1-2, "MSPM0 MCU Platform Security Enablers".

### 6.1 What this device actually has

| Capability | MSPM0G3507 | Source |
|---|---|---|
| Immutable ROM root of trust (BCR) | **Yes** | SLAAE29A §2.3 |
| Single point of entry to main flash at boot | **Yes** | Table 1-2 |
| CRC-32 verified main flash region | **Yes** | Table 1-2 |
| **Asymmetric image auth (ECDSA P-256 + SHA2-256, software)** | **Yes** | Table 1-2 |
| Secure boot solution | **BIM** (Boot Image Manager, MCUboot-based) | §3.3 |
| Permanently lockable main flash | **Yes** | Table 1-2 |
| SRAM write-execute mutual exclusion (W^X) | **Yes** | Table 1-2, §4.6 |
| Password-authenticated debug / BSL / mass erase / factory reset | **Yes** | Table 1-2 |
| Complete hardware disable of SWD | **Yes** | Table 1-2 |
| TRNG with power-on and continuous self-test | **Yes** | Table 1-2, §5.2 |
| AES accelerator (basic) | **Yes** — ECB, CBC, OFB, CFB, CTR, CBC-MAC | Table 5-1 |
| 96-bit unique device identifier | **Yes** | Table 1-2, §2.1 |
| EVITA capability | **EVITA-Light** | Table 1-2 |
| Customer Secure Code (CSC) / INITDONE | No | Table 1-2 |
| **Protected key store (KEYSTORE)** | **No** — requires CSC | Table 1-2, §4.5 |
| Flash firewalls (write / read-execute / IP protection) | No — require CSC | Table 1-2 |
| **Hardware monotonic counter** (rollback protection) | **No** | Table 1-2, §4.7 |
| Hardware AES-CMAC / CCM / GCM (AESADV) | No | Table 5-1 |
| Passwords stored hashed (SHA2-256) | No — stored in NONMAIN | Table 1-2 |

AES throughput, for firmware budgeting (Table 5-2, at 80 MHz): 128-bit key 168 cycles
(2.10 µs) per block encrypt or decrypt; 256-bit key 234 cycles (2.93 µs).

### 6.2 Secure boot via BIM

Because this die has no CSC, the applicable secure boot solution is **BIM**, supplied in the
MSPM0 SDK (SLAAE29A §3.3). It is MCUboot-derived: two application image slots (primary and
secondary) at fixed flash addresses, images signed with MCUboot's `imgtool`, and at each boot
the BIM verifies the candidate slot with **SHA-256 + ECDSA**; on failure it erases that slot
and falls back to the other, failing closed if neither verifies (§3.3.1, Figure 3-13).

Provisioning, per §3.3.1, requires all of: the BIM firmware and its authentication keys placed
in MAIN flash with the reset vector at `0x0000.0004` pointing at BIM; those sectors and the
NONMAIN sector set **static write protected**; mass erase and factory reset **password
protected or disabled**; and the MAIN flash memory integrity check enabled over the BIM and key
region. Note §3.3.1(c): disabling factory reset alongside those settings makes the NONMAIN
configuration **permanently locked** — a genuine one-way door on a production unit.

### 6.3 Honest comparison with the S32K144 CSEc it replaces

| | S32K144 CSEc | MSPM0G3507 |
|---|---|---|
| Secure boot | Hardware AES-CMAC over image | ROM-anchored BIM, SHA-256 + ECDSA P-256 |
| **Asymmetric signing / verification** | **No** (SHE is symmetric-only) | **Yes — ECDSA P-256** |
| Protected key store | **Yes** (SHE key slots) | **No** |
| Rollback protection | SHE counters | **No** hardware monotonic counter |
| Hardware CMAC | Yes | No (CBC-MAC only) |
| TRNG | Yes | Yes, with self-test |
| Debug lockdown | Yes | Yes, incl. permanent SWD disable |

So this is **not** a straight downgrade. The MSPM0 *gains* public-key image authentication,
which is the capability the README's "cryptographic authentication and message signing" claim
actually needs and which CSEc could not provide. It *loses* hardware key protection and
rollback protection: signing keys are public keys (fine in flash), but any **private** key or
shared secret the application holds lives in ordinary flash, protected only by static write
protection, debug lock and BSL lockout — not by a key store the CPU cannot read.

**README claims still needing correction:** "post-quantum" remains unsupportable (P-256 is
classical ECC, and there is no PQC support anywhere in this part). "Trusted Platform Module" is
also gone as of the S32K144 pass. "Cryptographic authentication and message signing" is now
defensible in software.

## 7. Isolated transceivers: verified pin maps and the `GND2`/`GNDISO` split

Verified pin maps for both isolated transceivers were supplied from the sibling `Open-Secure-ESC`
project (`symbols/specs/ADM2582E_ADM2587E.json`, `symbols/specs/ADM3055E_ADM3057E.json`). Both
were **re-checked here against the local ADI datasheets** rather than taken on trust:

- ADM2582E/ADM2587E Rev. H, p.8, Table 10 — matches the JSON exactly.
- ADM3055E/ADM3057E Rev. D, p.15, Table 10 — matches the JSON exactly.

The existing hand-authored EAGLE symbols for `U5` and `U6` were checked against those maps and
**every distinct pin name is correct**. The only modelling defect was on the ADM2587E, below.

### 7.1 ADM2587E `GND2` was one pin where the silicon has two grounds

`U5`'s symbol collapsed all four `GND2` pads (11, 14, 16, 20) onto a single schematic pin.
That is not merely cosmetic: Table 10 says of **pin 16** — *"Ground, Bus Side. **Do not connect
this pin to Pin 14 and Pin 11.**"* — while pins 11 and 14 are the isolated DC-DC converter
ground, which Table 10 says to *"connect ... together through one ferrite bead to PCB ground"*.
With one collapsed pin, that separation could not be expressed in the netlist at all.

The ADM3055E gives the same two nodes distinct mnemonics (`GND2` for pads 11/15, `GNDISO` for
pads 18/20), and `U6` was already wired correctly with ferrite `B5` between them. `U5` is now
modelled the same way:

- symbol pin `GND2` → pads **16, 20** (bus side) → net `RS485_ISO_GND`
- new symbol pin `GNDISO` → pads **11, 14** (converter side) → new net `RS485_GNDISO`
- new ferrite **`B7`** bridges the two, mirroring `B5` on the CAN side

### 7.2 isoPower reservoir capacitors re-referenced

Both datasheets are explicit about *which* ground the `VISOOUT` reservoir pair returns to, and
both boards had it wrong — the ripple was returning through the ferrite instead of locally to
the converter ground, which defeats the point of fitting the ferrite:

| Cap pair | Was | Now | Datasheet |
|---|---|---|---|
| `C27` 10 µF + `C28` 100 nF (`U5` VISOOUT) | `RS485_ISO_GND` | **`RS485_GNDISO`** | ADM2587E Table 10 pin 12: *"a reservoir capacitor of 10 µF and a decoupling capacitor of 0.1 µF be fitted between Pin 12 and Pin 11"* |
| `C35` 220 nF + `C36` 10 µF (`U6` VISOOUT) | `CANFD_ISO_GND` | **`CANFD_GNDISO`** | ADM3055E Table 10 pin 19: *"requires 0.22 µF and 10 µF capacitors to GND_ISO"* |

`C29`/`C30` (`U5` VISOIN) and `C37`/`C38` (`U6` VISOIN) were already correct on the bus-side
ground, per ADM2587E Table 10 pin 19 (*"between Pin 19 and Pin 20"*).

## 8. CAN-FD bus connectors

The CAN-FD bus connector decision, open since the `RS485-CANFD-TPM-upgrade.md` pass, is now
resolved and on-sheet. `CANH`/`CANL` were previously labelled but unterminated and went nowhere.

**`U$26` (CAN-FD IN) and `U$27` (CAN-FD OUT)** — a daisy-chain pair mirroring the RS-485
connectors `U$8`/`U$9`, on a new `JST_PH-3_HOLE` deviceset:

| Pin | Net | Rationale |
|---|---|---|
| `P$1` | `CANFD_ISO_GND` | ISO 11898-2 bus reference. An isolated node whose bus side floats has an undefined common-mode voltage relative to other nodes; the CAN_GND conductor is what bounds it. This is why CANopen/DeviceNet cabling carries CAN_GND. |
| `P$2` | `CANL` | kept adjacent to `CANH` for twisted-pair continuity up to the shell |
| `P$3` | `CANH` | |

**Why 3-way and not 4-way.** The obvious move was to reuse the existing `JST_PH-4_HOLE`, which
already has a footprint and is already in the BOM. It was rejected deliberately: the RS-485
daisy-chain connectors are 4-way JST-PH **carrying `+7V` on `P$1`**. A 4-way CAN port would be
physically cross-mateable with them, and plugging an RS-485 cable into a CAN port would drive
`+7V` straight onto the isolated CAN bus and its ground reference. A 3-way shell makes that
mistake impossible by construction, which is worth more than connector-family commonality on a
board whose whole reason for using isolated transceivers is fault tolerance.

**Termination.** `R22`/`R23` (60.4 Ω) and `C41` (4.7 nF) form a **split termination** from
`CANH` to `CANL` with the midpoint (`CAN_SPLIT`) bypassed to `CANFD_ISO_GND` — the standard
arrangement, which damps common-mode as well as differential energy. All three are marked
**`DNP` (do not populate)** in their value fields: CAN is a linear bus and only the two
physical **end** nodes may be terminated. A mid-chain servo with termination fitted will load
the bus and break it. Populate on end nodes only.

Note the RS-485 side has **no** termination provision at all, which is a pre-existing gap — see
§10.

## 9. Firmware impact

The port is again a full rewrite, and larger than the S32K144 estimate implied:

- Toolchain moves to **TI's MSPM0 SDK / SysConfig** with Code Composer Studio, IAR, Keil or
  TI Arm-Clang (§10.3). `Test_LibreServo_v2.ioc` (STM32CubeMX) and `STM32F302K8UX_FLASH.ld`
  are dead artefacts.
- **Peripheral remap:** `TIMA0` (advanced timer, deadband-capable) for motor PWM, `UART0`,
  `SPI1`, `ADC0`/`ADC1`, `CANFD`, IOMUX `PINCMx` register configuration per pin.
- **Motor PWM can now use true complementary pairs with deadband.** `PWM-2`/`PWM-1` are
  `TIMA0_C1`/`C1N` and `PWM-4`/`PWM-3` are `TIMA0_C3`/`C3N` — matched complementary channels on
  the same advanced timer (SLASF88C §1: *"Two 16-bit advanced control timers with deadband
  support and complementary outputs"*). The previous designs had no such pairing. Whether the
  H-bridge is driven as complementary PWM or as PWM+direction is now a firmware choice rather
  than a hardware constraint.
- **The SPI bus can become hardware SPI1.** `SPI_CLK`/`SPI_OUT`/`SPI_IN` are on
  `SPI1_SCK`/`SPI1_PICO`/`SPI1_POCI`. It was bit-banged GPIO on every previous MCU. Still
  shared with connector `U$4` and `POTEN_SERVO` (`U$6`) — the open question from the previous
  passes about what else lives on that bus is **still open** and still gates using it.
- **Fixed-point rework of the control loop** — see §1.

## 10. Still open

Carried forward, plus new items from this pass.

**New, from this pass:**

- [ ] **Footprint/package for `MSPM0G3507` (RHB0032E)** — deviceset is symbol-only. Do not
      reuse `QFN32`; see §2.1 for why, and for TI's dimensioned land pattern.
- [ ] **`.brd` placement for `U1`** — the existing `U1` element still references package
      `QFN32` at `x=4.78 y=8.4 rot=R270`. Body size is unchanged at 5 × 5 mm, so the envelope
      fits, but the pad numbering is rotated and the land geometry differs.
- [ ] **`C1` bulk decoupling is below spec.** §7.3 lists C<sub>VDD</sub> = 10 µF between `VDD`
      and `VSS`, and §9.1 recommends 10 µF + 0.1 µF placed within a few millimetres. `C1` is
      **4.7 µF** in an 0402. Raising it to 10 µF in 0402 is possible but severely
      derated at 3.3 V — this is a real BOM/footprint decision (larger case size vs. accepting
      4.7 µF), deliberately not changed silently here.
- [ ] **Rewrite the README security claims** — see §6.3 for exactly which claims are now
      defensible and which are not.
- [ ] Confirm the FPU loss and the fixed-point rework are acceptable, or reconsider the part.
- [ ] **Footprint for the new `JST_PH-3_HOLE` CAN connectors** (`U$26`/`U$27`) — symbol-only
      deviceset; the land pattern is a one-position derivative of the existing
      `JST_PH-4_HOLE_B`. Silkscreen must clearly distinguish the CAN ports from the RS-485
      ports (see §8).
- [ ] **BIM secure-boot provisioning plan** — key generation and custody, `imgtool` signing in
      the build, and the NONMAIN static-write-protect / factory-reset-disable step, which is
      **permanent** (SLAAE29A §3.3.1c). Needed before anything ships beyond a bench prototype.
- [ ] **No RS-485 termination provision exists** on this board (the CAN side now has a DNP
      split termination; `RS485_P`/`RS485_N` have nothing). Decide whether end-of-chain servos
      need a fitted or switchable 120 Ω across the RS-485 pair.

**Pre-existing schematic-geometry defects found by this pass — NOT fixed, see below:**

- [ ] **14 places where a pin sits on top of a foreign net's wire.** These are drawing defects
      inherited from the `RS485-CANFD-TPM-upgrade.md` pass, in three clusters:
      1. the `GND` wire `(33.02,-44.45)→(33.02,-60.96)` runs straight down `U5`'s **entire
         logic-side pin column**, passing through `VCC`, `RxD`, `RE`, `DE`, `TxD` and `R20.1`;
      2. the `RS485_VISOOUT` and `RS485_VISOIN` rails at `y=-88.9` run through the pin 1 of
         `C25`/`C26` (`+3V3`) and `C27`/`C28` — **bridging the isolation barrier** on the
         drawing;
      3. `C37`/`C38`'s pin-2 risers pass through their own pin 1, shorting those capacitors.

      **These do not corrupt the exported netlist today** — EAGLE's `<nets>` section is the
      netlist and it declares the correct connectivity (verified: zero coincident connection
      points between different nets). They are still serious: they read as shorts to a human,
      EAGLE ERC will flag them, and any GUI edit that makes EAGLE re-derive geometry can
      silently merge these nets. Fixing them means redrawing the `U5` logic side and both
      isoPower capacitor banks — deliberately left alone rather than bundled into an unrelated
      commit.

**Carried over, still unresolved:**

- [ ] Footprints/devicesets/3D models and `.brd` placement for `ADM2587E` (`U5`) and
      `ADM3055E` (`U6`).
- [ ] **5 V rail for `U6`'s isoPower `VCC`** (`+5V_ISO_CANFD`) — nothing on the board produces
      5 V; a regulator must be selected and added.
- [ ] Confirm what else is live on the shared `SPI_CLK`/`SPI_IN`/`SPI_OUT` bus (`U$4`, `U$6`)
      before enabling hardware SPI1 on it.
- [ ] Isolation creepage/clearance for the two SOIC-20 isolated parts against the board outline.
- [ ] At layout, keep `U5` pad 16 off the same copper as pads 11/14 — now expressible in the
      netlist (§7.1), but the physical separation is still a layout obligation.
- [ ] Whether the shared `Vmot`/`GND` on the daisy-chain connector undermines the RS-485
      isolation (see `RS485-CANFD-TPM-upgrade.md` §2).

**Repository hygiene (pre-existing, not introduced here):**

- [ ] `PCB/LibreServo-v2.3_BOM.txt` is **two MCU generations stale** — it still lists `U1` as
      `STM32F301K8U6` / `QFN32` and contains none of `ADM2587E`, `ADM3055E`, or the support
      parts added by the last three passes. It was not updated by this pass either, because a
      partial edit would be more misleading than an obviously-stale file. It needs a full
      regeneration from the schematic.
- [ ] This repository has no `CLAUDE.md`/`AGENTS.md`, `TODO.md`, `REFERENCES.md` or
      `PROJECT_INDEX.md`. Citations in this document therefore follow the existing in-repo
      convention (inline, with the local datasheet path and TI literature number) rather than
      REF-IDs.

---

## Attribution

Schematic edit, pin assignment, package selection and this document produced by
**Claude Opus 5** (Anthropic), working from the local TI datasheet copy
`PCB/datasheets/mspm0g3507-q1.pdf` (SLASF88C). Package choice between 28-VSSOP and 32-VQFN was
decided by the human maintainer. Prior passes are recorded in
[`S32K144-MCU-swap.md`](S32K144-MCU-swap.md) and
[`RS485-CANFD-TPM-upgrade.md`](RS485-CANFD-TPM-upgrade.md).
