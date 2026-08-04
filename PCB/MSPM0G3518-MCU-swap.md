# Hardware upgrade notes: MCU is TI MSPM0G3518-Q1 (VQFN-32, RHB)

**Current design.** `U1` is a **Texas Instruments MSPM0G3518-Q1** in the 32-pin VQFN (RHB)
package, orderable part number **`M0G3518QRHBRQ1`**.

Supersedes [`MSPM0G3507-MCU-swap.md`](MSPM0G3507-MCU-swap.md) (same MSPM0 platform and package,
smaller die), which superseded [`S32K144-MCU-swap.md`](S32K144-MCU-swap.md) and
[`RS485-CANFD-TPM-upgrade.md`](RS485-CANFD-TPM-upgrade.md).

Sources, both local copies, both cited by section throughout:

- **SLASFA6B** — *MSPM0G351x-Q1 Automotive Mixed-Signal Microcontrollers With CAN-FD Interface*,
  November 2024, revised October 2025:
  [`../../Open-Secure-ESC/docs/datasheets/mspm0g3519-q1.pdf`](../../Open-Secure-ESC/docs/datasheets/mspm0g3519-q1.pdf).
  One data sheet covers **both** MSPM0G3519-Q1 and MSPM0G3518-Q1; the filename names only the
  larger part.
- **SLAAE29A** — *Cybersecurity Enablers in MSPM0 MCUs*, January 2023, revised December 2025:
  [`../../Open-Secure-ESC/docs/datasheets/slaae29a.pdf`](../../Open-Secure-ESC/docs/datasheets/slaae29a.pdf).
  The MSPM0G3518 sits in that document's **`M0Gx5 1x`** column of Table 1-2.

---

## 1. Why the MSPM0G351x die

The MSPM0G3507 pass had two acknowledged weaknesses: a security stack materially weaker than the
S32K144 CSEc it displaced, and only 128 KB/32 KB of memory against the S32K144's 512 KB/64 KB.
Moving to the **MSPM0G351x** die closes both while keeping the identical VQFN-32 5 × 5 mm package
and, as shown in §3, **the identical pinout**.

| | S32K144 (superseded) | MSPM0G3507 (superseded) | **MSPM0G3518-Q1 (this pass)** |
|---|---|---|---|
| Core | Cortex-M4F @ 80/112 MHz | Cortex-M0+ @ 80 MHz | Cortex-M0+ @ 80 MHz, MPU |
| Main flash | 512 KB (ECC) | 128 KB (ECC) | **256 KB (ECC), dual-bank 2 × 128 KB with address swap for OTA** |
| Data flash | 4 KB FlexRAM (EEPROM emu.) | — | **16 KB dedicated data flash bank, ECC** |
| SRAM | 64 KB | 32 KB (parity) | **128 KB** (64 KB ECC/parity + 64 KB) |
| CAN | 3 × FlexCAN, 1 FD | 1 × CAN-FD | 1 × CAN-FD on this package |
| ADC | 2 × 12-bit | 2 × 12-bit 4 Msps, 11 ch | **2 × 12-bit 4 Msps, 16 ch** |
| Serial | 3 LPUART/3 LPSPI/1 LPI2C | 4/2/2 | **5 UART / 2 SPI / 3 I²C** on this package |
| Crypto engine | CSEc, AES-128 | AES basic (no CMAC/GCM) | **AESADV 128/256: GCM/GMAC, CCM/CBC-MAC, CMAC, CBC, CTR** |
| Key storage | SHE key slots | **none** | **KEYSTORE, up to 4 AES keys** |
| Secure boot | HW AES-CMAC | BIM (SW ECDSA) | **CSC + HW AES-CMAC *and* SW ECDSA P-256** |
| Rollback protection | SHE counters | **none** | **hardware monotonic counter** |
| Memory firewalls | — | **none** | **flash write / read-execute / IP-protection, SRAM W^X** |
| Package | LQFP-48, 9 × 9 mm | VQFN-32, 5 × 5 mm | **VQFN-32, 5 × 5 mm, wettable flank** |

Memory figures are from SLASFA6B Table 8-5 (Memory Organization), which lists the MSPM0G3518
and MSPM0G3519 columns separately: for the '3518, Flash Bank 0 = 128 KB and Bank 1 = 128 KB
(256 KB main total), Data Flash Bank = 16 KB, SRAM Bank 0 = 64 KB (ECC/parity) and Bank 1 =
64 KB. §8.9 confirms *"a dual bank of non-volatile flash memory (up to 256kB/512kB total) and a
separate data flash bank (16kB)"*, with *"bank address swap for in-system, over-the-air (OTA)
firmware updates"*.

**The one item that does not improve is the CPU core.** This is still a Cortex-M0+ with no FPU,
where the S32K144 was a Cortex-M4F and the board's *original* STM32F302K8U6 also had an FPU. Any
`float`/`double` math in `Src/LS_*.c` becomes soft-float. The mitigation is the on-die
**MATHACL** accelerator, which on this die covers **SINCOS, ATAN2, SQRT, DIV and MPY32**
(SLASFA6B §8.23) — a better fit for a PID/curve-generation servo loop than the '3507's
DIV/SQRT/MAC/TRIG set — but using it still means **reworking the control loop in fixed-point**,
not recompiling. Budget for that.

## 2. Security stack

This is the section that reverses the previous two passes' conclusions. The MSPM0G3507 lacked a
Customer Secure Code stage, and without CSC it could have no key store, no firewalls and no
monotonic counter. The MSPM0G351x has CSC, and everything gated behind it follows.

SLASFA6B **§8.18 Security** lists, verbatim: debug security; device identify; *"AES-128/256
accelerator with support for GCM/GMAC, CCM/CBC-MAC, CBC, CTR"*; *"flexible firewalls for
protecting code and data"* (flash write-erase protection, flash read-execute protection, flash
IP protection, SRAM write-execute mutual exclusion); *"secure boot"*; *"secure firmware
update"*; *"secure key storage for up to four AES keys"*; *"customer secure code"*;
*"hardware monotonic counter"*; TRNG; and CRC-16/32 with custom polynomial.

### 2.1 Against the S32K144 CSEc that was dropped two passes ago

| | S32K144 CSEc (SHE-class HSM) | **MSPM0G3518** |
|---|---|---|
| Symmetric engine | AES-128, ECB/CBC/CMAC | **AES-128/256 (AESADV): CBC, CFB, OFB, CTR/ICM, CBC-MAC, CMAC, CCM, GCM** |
| Protected key store | Yes, SHE key slots | **Yes — KEYSTORE, up to 4 keys (128-bit ×4 or 256-bit ×2, plus a session key)** |
| Secure boot | Hardware AES-CMAC over image | **Yes — CSC, with hardware AES-CMAC *and* software ECDSA P-256 / SHA2-256** |
| Asymmetric verification | **No** (SHE is symmetric-only) | **Yes — ECDSA P-256** |
| Rollback protection | SHE counters | **Yes — hardware monotonic counter** |
| Memory firewalls / IP protection | Not in SHE scope | **Yes — flash W / RX / IP-protection, SRAM W^X** |
| Field firmware update | Not in SHE scope | **Yes — dual-bank address swap** |
| TRNG | Yes | **Yes, with power-on and continuous health tests, dedicated local LDO** |
| Key slot count | 17 general + 5 special | 4 (+ session key) — **CSEc wins here** |
| Standardisation | SHE 1.1, `M1`–`M5` | Vendor-specific; **PSA-L1 and ISO 21434 targeted**, EVITA-Light |

Net assessment: the MSPM0G3518 **matches or exceeds CSEc on every axis except raw key-slot
count and formal SHE standardisation**, and adds asymmetric verification, IP protection and
OTA-capable dual-bank update that CSEc never had. The security regression flagged in the two
previous documents is closed.

### 2.2 How KEYSTORE actually protects keys

Worth understanding before firmware planning, because it constrains the provisioning flow
(SLAAE29A §4.5, Figure 4-1):

- **Only the CSC can write keys into KEYSTORE**, and only before it calls `INITDONE`.
- After `INITDONE`, the application can tell the AES engine *which* key slot to use, but **can
  never read or write any stored key**. The key travels from KEYSTORE to the AES engine over a
  **private data bus** that is not visible to CPU, DMA or debugger.
- KEYSTORE contents are **wiped on every `BOOTRST`** and the store unlocks for writing; they
  survive `SYSRST` and lower-order resets. So keys are re-deposited from protected flash by the
  CSC on each cold boot — they are not persistent in the key store itself.
- The design explicitly thwarts partial-key-modification attacks (SLASFA6B §8.21).

### 2.3 Provisioning is a one-way door — decide before production

Same warning as the previous pass, and it still applies: locking NONMAIN (static write
protection on the CSC and key region, plus password-protected or disabled mass erase and factory
reset) **permanently** fixes the configuration on that unit. Per SLAAE29A §2.6 / §3.2 this must
be planned before the first production run, not retrofitted. Bench prototypes should be brought
up with the security policies open and locked down only once the flow is proven.

AES throughput for firmware budgeting, AESADV at 80 MHz (SLAAE29A Table 5-3): 128-bit key
**76 cycles / 0.95 µs** per block encrypt, 256-bit key **81 cycles / 1.01 µs**. That is roughly
2–3× faster than the basic AES block on the MSPM0G3507 (Table 5-2: 168 cycles / 2.10 µs).

## 3. Pinout: unchanged from the MSPM0G3507 pass

**The netlist did not move.** SLASFA6B Figure 6-6 (32-pin RHB) has pad-for-pad the same map as
the MSPM0G3507's RHB: pads 1–32 are `PA0`, `PA1`, `NRST`, `VDD`, `VSS`, `PA2`…`PA27`, `VCORE`,
plus the thermal pad. Every function this design uses is available on the same physical pad.

SLASFA6B's Table 6-2 is easier to audit than the '3507's: it has an explicit **`RHB PIN`**
column, so each row below is a direct lookup rather than a cross-reference.

What *did* change is the **IOMUX `PINCM.PF` function codes** — a firmware concern only, but they
must be right in the pin-mux configuration:

| Net | Pin | Pad | Function | PF on '3507 | **PF on '3518** |
|---|---|---:|---|---:|---:|
| `CLK-8MHZ` | `PA6` | 10 | `HFCLK_IN` | 7 | **6** |
| `PWM-4` | `PA4` | 8 | `TIMA0_C3` | 5 | **8** |
| `PWM-2` | `PA22` | 26 | `TIMA0_C1` | 5 | **7** |
| `PWM-3` | `PA24` | 28 | `TIMA0_C3N` | 4 | **5** |
| `PWM-1` | `PA25` | 29 | `TIMA0_C1N` | 6 | 6 |
| `CAN0_TX` | `PA12` | 16 | `CAN0_TX` | 5 | **12** |
| `CAN0_RX` | `PA13` | 17 | `CAN0_RX` | 6 | **12** |
| `UART0_TX` / `UART0_RX` | `PA10` / `PA11` | 14 / 15 | `UART0_TX` / `UART0_RX` | 2 | 2 |
| `SPI_CLK` / `SPI_OUT` / `SPI_IN` | `PA17` / `PA18` / `PA16` | 21 / 22 / 20 | `SPI1_SCK` / `SPI1_PICO` / `SPI1_POCI` | 3 | 3 |
| `SWDIO` / `SWCLK` | `PA19` / `PA20` | 23 / 24 | `SWDIO` / `SWCLK` | 2 | 2 |

### 3.1 Full assignment

Analog functions are non-IOMUX (set `PINCM.PF` = 0). Spares are terminated per Table 6-20
(*"PAx, PBx, PCx — Open; set corresponding pin functions to GPIO and configure unused pins to
output low or input with internal pullup/pulldown resistor enabled"*).

| Pad | Pin | Net | Function (PF) | IO type | Notes |
|---:|---|---|---|---|---|
| 1 | `PA0` | `LED_RED` | GPIO [1] | **ODIO, 5 V-tol** | 20 mA sink — §3.2 |
| 2 | `PA1` | — | *(spare)* | ODIO, 5 V-tol | |
| 3 | `NRST` | `NRST` | reset | Reset | **must be pulled high** — §4 |
| 4 | `VDD` | `+3V3` | power | Power | |
| 5 | `VSS` | `GND` | power | Power | |
| 6 | `PA2` | — | *(spare)* | SDIO | also `ROSC`; SYSOSC FCL option preserved |
| 7 | `PA3` | `LED_GREEN` | GPIO [1] | SDIO | |
| 8 | `PA4` | `PWM-4` | `TIMA0_C3` [8] | SDIO | → `M4.INA` |
| 9 | `PA5` | `EN-Q` | GPIO [1] | SDIO | driver enable, `R11`/`R12` |
| 10 | `PA6` | `CLK-8MHZ` | `HFCLK_IN` [6] | SDIO | from oscillator `U$100` |
| 11 | `PA7` | `LED_BLUE` | GPIO [1] | SDIO | |
| 12 | `PA8` | — | *(spare)* | SDIO | **also `HFCLK_IN` [10]** — second clock-input option |
| 13 | `PA9` | `SEL_SERIAL` | GPIO [1] | HSIO | RS-485 `DE`+`RE` turnaround |
| 14 | `PA10` | `UART0_TX` | `UART0_TX` [2] | **HDIO** | default BSL `BSLTX` |
| 15 | `PA11` | `UART0_RX` | `UART0_RX` [2] | **HDIO** | default BSL `BSLRX` |
| 16 | `PA12` | `CAN0_TX` | `CAN0_TX` [12] | HSIO | → `U6.TXD` (ADM3055E) |
| 17 | `PA13` | `CAN0_RX` | `CAN0_RX` [12] | HSIO | → `U6.RXD` |
| 18 | `PA14` | — | *(spare)* | HSIO | also `A0_12`, `COMP0_IN2+` |
| 19 | `PA15` | `V_CORRIENTE` | `A1_0` (analog) | SDIO | **ADC1**; also `DAC_OUT`, `COMP0/1_IN3+` |
| 20 | `PA16` | `SPI_IN` | `SPI1_POCI` [3] | SDIO | |
| 21 | `PA17` | `SPI_CLK` | `SPI1_SCK` [3] | SDIO (wake) | |
| 22 | `PA18` | `SPI_OUT` | `SPI1_PICO` [3] | SDIO (wake) | also `BSL_invoke` |
| 23 | `PA19` | `SWDIO` | `SWDIO` [2] | SDIO | |
| 24 | `PA20` | `SWCLK` | `SWCLK` [2] | SDIO | |
| 25 | `PA21` | `GND` | `VREF-` (analog) | SDIO | |
| 26 | `PA22` | `PWM-2` | `TIMA0_C1` [7] | SDIO | → `M5.INA` |
| 27 | `PA23` | `VDDA-FILTRADO` | `VREF+` (analog) | SDIO | |
| 28 | `PA24` | `PWM-3` | `TIMA0_C3N` [5] | SDIO | → `M4.INB` |
| 29 | `PA25` | `PWM-1` | `TIMA0_C1N` [6] | SDIO | → `M5.INB` |
| 30 | `PA26` | `V_TEMP` | `A0_1` (analog) | SDIO | ADC0; alt `CAN0_TX` [10] |
| 31 | `PA27` | `V_BAT` | `A0_0` (analog) | SDIO | ADC0; alt `CAN0_RX` [10] |
| 32 | `VCORE` | `VCORE` | core LDO | Power | **0.47 µF tank cap only** — §4 |
| 33 | thermal pad | `GND` | — | — | symbol pin `PAD` |

**Spares: `PA1`, `PA2`, `PA8`, `PA14` (4).**

Motor PWM still lands on two matched complementary `TIMA0` pairs with deadband support —
`PWM-2`/`PWM-1` on `TIMA0_C1`/`C1N`, `PWM-4`/`PWM-3` on `TIMA0_C3`/`C3N` — so the H-bridge can
be driven as complementary PWM or as PWM+direction, as a firmware choice.

### 3.2 Rationale carried over unchanged

The reasoning behind each choice is in [`MSPM0G3507-MCU-swap.md`](MSPM0G3507-MCU-swap.md) and is
still valid because the IO structures are the same on this die (Table 6-1; abs-max Table 7-1:
SDIO 6 mA, HSIO 6 mA, **HDIO 20 mA, ODIO 20 mA sink**):

- **§3.1** `LED_RED` on `PA0` — the RGB is common-anode, `R6` = 187 Ω makes red the
  highest-current channel at roughly 6–8 mA, over the 6 mA standard-drive limit; `PA0` is
  open-drain, 20 mA sink, low-side NMOS only, which is exactly right for pulling a cathode down.
- **§3.2** `HFCLK_IN`, not a crystal — `U$100` is an oscillator module (square wave). Note the
  '3518 additionally offers `HFCLK_IN` on `PA8` [PF10], so unlike the '3507 there is now a
  second external-clock pin available among the spares.
- **§3.3** RS-485 UART on the BSL default pins, with the caveat that the BSL ROM will not drive
  `SEL_SERIAL`, so BSL replies cannot reach the bus without a fixture asserting `DE`.
- **§3.4** ADC split — `V_CORRIENTE` alone on ADC1 so it can be sampled simultaneously with an
  ADC0 channel.
- **§3.5** `VDDA-FILTRADO` → `VREF+`, `VREF-` → `GND`, with `C4` (1 µF) as the required VREF
  decoupling. SLASFA6B §8.15 confirms the same external-reference support and that a decoupling
  capacitor on `VREF+`/`VREF-` is required for proper operation.

## 4. Support components — unchanged

SLASFA6B §7.3 and §9.1 specify exactly the same values as the '3507, so **no BOM change**:

| Ref | Value | Net | Requirement |
|---|---|---|---|
| `R21` | 47 kΩ 0402 | `NRST` → `+3V3` | §9.1: *"NRST **must** be pulled to VDD for the device to start"*; Table 6-20 repeats it. 47 kΩ is TI's recommended value. |
| `C40` | 10 nF 0402 | `NRST` → `GND` | TI-recommended `NRST` filter (§9.1). |
| `C39` | 470 nF 0402 | `VCORE` → `GND` | §7.3 C<sub>VCORE</sub> = 470 nF; §9.1: *"Do not connect other circuits to the VCORE pin."* |

§7.3 note (1) adds a requirement worth passing to whoever does layout: *"Connect C<sub>VDD</sub>
and C<sub>VCORE</sub> between VDD/VSS and VCORE/VSS respectively, as close to the device pins as
possible. A low-ESR capacitor with at least the specified value and tolerance of ±20% or better
is required."*

## 5. What changed in the schematic this pass

Only the part identity and two signal names. `PCB/LibreServo-v2.3.1.sch`:

- Symbol and deviceset `MSPM0G3507` → `MSPM0G3518`; `U1` value → `M0G3518QRHBRQ1`.
- Pin names `PA12/CAN_TX` → `PA12/CAN0_TX` and `PA13/CAN_RX` → `PA13/CAN0_RX`. SLASFA6B names
  the signals `CAN0_TX`/`CAN0_RX` because the die has two CAN instances on larger packages.
- Nets `CAN_TX`/`CAN_RX` → `CAN0_TX`/`CAN0_RX` to follow. This restores the names the nets had
  before the MSPM0G3507 pass, so relative to the S32K144 baseline they are unchanged.
- Deviceset description and the two on-sheet annotation blocks updated.

Nothing else moved: no wire, junction, instance or support component was touched.

### Verification performed

Machine-checked against the edited file:

- **XML well-formed**; no residual `MSPM0G3507` or `S32K144` references.
- **All 29 connected `U1` pins land on the intended net**, verified against the RHB pad map;
  all 33 symbol pins are unique and resolve to a real pad; the 4 unconnected pins are exactly
  the intended spares.
- **Zero shared connection points between different nets** across the whole sheet — every wire
  endpoint, junction and rotation-resolved pin coordinate was collision-checked.
- **No unresolved references:** every `pinref` resolves to an existing instance and symbol pin.

## 6. Still open

**Footprint and layout:**

- [ ] **Footprint for `RHB0032T`** — deviceset is still symbol-only. Note the '3518/'3519
      datasheet gives package drawing **`RHB0032T` (4224744/A)**, a different revision from the
      '3507's `RHB0032E` (4223442/B); dimensions are equivalent within rounding. TI's
      recommended land pattern (SLASFA6B §12): **32 pads 0.62 × 0.25 mm on 0.5 mm pitch,
      4.78 mm row-to-row span, 3.45 × 3.45 mm exposed thermal pad (pad 33)**, body 4.85–5.15 mm
      square, 1 mm max height.
- [ ] **The existing Propio `QFN32` footprint is still NOT reusable** — its pad 1 is on the
      bottom row where TI's is on the left column (90° apart), and its span is 5.5 mm against
      TI's 4.78 mm. Only the 3.45 mm thermal pad matches.
- [ ] `.brd` placement for `U1` — the element still references package `QFN32` at
      `x=4.78 y=8.4 rot=R270`. Body size is unchanged at 5 × 5 mm so the envelope fits.
- [ ] **`C1` bulk decoupling is below spec** — §7.3 lists C<sub>VDD</sub> = 10 µF and §9.1
      recommends 10 µF + 0.1 µF within a few millimetres. `C1` is **4.7 µF** in an 0402.
      Raising it to 10 µF in 0402 is severely derated at 3.3 V; this is a real case-size vs.
      capacitance decision, deliberately not changed silently.

**Firmware:**

- [ ] Fixed-point rework of the PID/curve-generation loop (§1), using MATHACL.
- [ ] Full port to the MSPM0 SDK / SysConfig; `Test_LibreServo_v2.ioc` and
      `STM32F302K8UX_FLASH.ld` are dead artefacts.
- [ ] Update the pin-mux configuration to the **'3518 PF codes** in §3, not the '3507's.
- [ ] Decide the secure-boot and KEYSTORE provisioning flow **before** the first production run
      (§2.3) — NONMAIN lockdown is permanent per unit.
- [ ] Confirm what else is live on the shared `SPI_CLK`/`SPI_IN`/`SPI_OUT` bus (`U$4`, `U$6`)
      before enabling hardware SPI1 on it.

**Board-level, carried over and still unresolved:**

- [ ] Footprints/devicesets/3D models and `.brd` placement for `ADM2587E` (`U5`) and
      `ADM3055E` (`U6`).
- [ ] **5 V rail for `U6`'s isoPower `VCC`** (`+5V_ISO_CANFD`) — nothing on the board makes 5 V.
- [ ] CAN-FD bus connector strategy; `CANH`/`CANL` are unterminated labelled nets.
- [ ] `ADM2587E` `GND2` copper-separation caveat; isolation creepage/clearance against the
      board outline for the two SOIC-20 isolated parts.
- [ ] Whether the shared `Vmot`/`GND` on the daisy-chain connector undermines RS-485 isolation.

**Documentation:**

- [ ] **Rewrite the README security claims.** They still describe a "post-quantum Trusted
      Platform Module". That was never accurate for the SLB9672, and the part is three
      revisions gone. The accurate claim now is AES-256 with hardware CMAC/GCM, a protected
      key store, CSC secure boot with ECDSA P-256 and AES-CMAC, memory firewalls, and hardware
      rollback protection — PSA-L1 and ISO 21434 **targeted, not yet certified**.
- [ ] `PCB/LibreServo-v2.3_BOM.txt` is **three MCU generations stale** — still lists `U1` as
      `STM32F301K8U6`/`QFN32`, and has none of `ADM2587E`, `ADM3055E`, `C39`, `C40`, `R21`.
      Needs full regeneration from the schematic.
- [ ] This repository has no `CLAUDE.md`/`AGENTS.md`, `TODO.md`, `REFERENCES.md` or
      `PROJECT_INDEX.md`. Citations here follow the in-repo convention (inline, with local
      document path and TI literature number) rather than REF-IDs.

---

## Attribution

Schematic edits, pin assignment, package selection and this document produced by
**Claude Opus 5** (Anthropic), working from the local TI documents SLASFA6B and SLAAE29A.
Device selection (MSPM0G3518-Q1 over MSPM0G3507-Q1, and the VQFN-32 package) was made by the
human maintainer, who also corrected the secure-boot assessment in
[`MSPM0G3507-MCU-swap.md`](MSPM0G3507-MCU-swap.md) §6.
