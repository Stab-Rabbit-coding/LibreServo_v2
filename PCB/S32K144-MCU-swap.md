# Hardware upgrade notes: MCU swap to NXP S32K144, TPM (SLB9672) dropped in favor of on-chip CSEc

> **SUPERSEDED.** `U1` is no longer an S32K144 — it is now a TI **MSPM0G3507-Q1 (VQFN-32,
> `M0G3507QRHBRQ1`)**. See [`MSPM0G3507-MCU-swap.md`](MSPM0G3507-MCU-swap.md) for the current
> design. Everything below is retained as the decision record for the S32K144 pass only.
>
> Three points below are now resolved or reversed:
>
> - The "no verified pin-to-package-pad table" problem in §3/§4 is **gone** — the MSPM0
>   datasheet (SLASF88C §6.2) carries the full per-package pin table, so the new symbol has
>   real, cited silicon pin numbers rather than net labels.
> - The 48-pin LQFP body-size concern in §1/§4 is **moot** — the MSPM0 replacement is a
>   5 × 5 mm VQFN-32, matching the original UFQFPN-32 envelope.
> - The CSEc capability discussion in §2 is **still relevant but now understates the gap**: the
>   MSPM0G3507 has AES + TRNG but no SHE-class HSM at all. See §1 of the new document.

Follow-up to [`RS485-CANFD-TPM-upgrade.md`](RS485-CANFD-TPM-upgrade.md), which took the board from STM32F302K8U6 to STM32G431 and added the Infineon SLB9672 TPM 2.0 as `U7`. This pass replaces the STM32G431 (`U1`) with an **NXP S32K144** and removes `U7` entirely, using the S32K144's built-in **CSEc** (Cryptographic Services Engine) instead of an external TPM. Local datasheet copy: [`datasheets/S32K1xx.pdf`](datasheets/S32K1xx.pdf).

## 1. Why S32K144

NXP (USA, HQ Austin TX / Eindhoven NL, automotive-grade fabs) — S32K1xx is NXP's automotive general-purpose MCU family, ISO 26262 capable up to ASIL-B, and every device in the family (S32K116 through S32K148) ships with the same on-die CSEc hardware security module, so choosing any S32K14x member gets the security requirement "for free" with no separate TPM part. Key specs for S32K144 specifically (from the local datasheet, `Figure 3` / feature-comparison table):

- Arm Cortex-M4F @ 80 MHz (RUN) / 112 MHz (HSRUN), IEEE-754 FPU
- 512 KB flash (ECC), 64 KB system RAM, 4 KB FlexRAM (EEPROM-emulation capable)
- **3x FlexCAN, 1 with CAN-FD** — native hardware CAN-FD, same category of upgrade the STM32G431 swap was chosen for
- 3x LPUART, 3x LPSPI, 1x LPI2C, 2x 12-bit ADC, up to 89 I/O
- **CSEc (Cryptographic Services Engine)** — SHE (Secure Hardware Extension)-compliant hardware crypto module (§4 below)
- Packages: 48-pin LQFP, 64-pin LQFP, 100-pin LQFP, 100-pin MAPBGA — **no 32-pin QFN option** (32-pin QFN exists only for the smaller S32K116/S32K118 subfamily, which lacks CAN-FD and has far less flash/RAM)

**Decision: 48-pin LQFP**, the smallest package S32K144 comes in. This is a step up from the STM32G431's 32-pin UFQFPN — unavoidable, since no S32K144 package is pin-count-competitive with the 32-pin part it replaces. Whoever does the footprint/layout pass needs to confirm the larger body size still fits the existing board outline.

## 2. TPM removed: `U7` (SLB9672) deleted, CSEc used instead

**What was removed**: part `U7` (SLB9672 TPM 2.0), its decoupling (`C39`–`C42`), its `CS#` pull-up (`R21`), the three local `+3V3` supply-symbol instances and one `GND` instance that only existed to feed it, and the six `NC_MARKER` instances wired to its unused pins. The `SLB9672` symbol and deviceset are deleted from the library section of the schematic — nothing references them anymore. The `TPM_CS#` net (which used the STM32G431's one spare pin, `PA15`) is gone too; that signal simply doesn't exist on the new symbol.

**What replaces it**: NXP's **CSEc**, present on every S32K144 die, needs **zero schematic changes** — no external bus, no chip-select, no decoupling, no pull-up. This is a genuine simplification versus the TPM, which needed a dedicated (well, shared — see the prior doc's caveat) SPI leg, a `CS#` pull-up, and its own decoupling network. CSEc capabilities (per NXP `AN5401`, "Getting Started with CSEc Security Module", and the SHE 1.1 spec it implements):

- AES-128 (ECB, CBC, CMAC) in hardware
- Up to 17 general-purpose key slots + 5 special-purpose keys (SECRET_KEY, MASTER_ECU_KEY, BOOT_MAC_KEY, BOOT_MAC, RAM_KEY) — keys live in protected flash, never exposed to the CPU/bus
- Hardware **secure boot**: CSE reads the bootloader/application image via its own bus-master interface, computes an AES-CMAC over it using the boot key, and compares against a stored MAC before release — sequential, parallel, or strict modes
- True random number generation, key derivation (per SHE's `M1`–`M5` message protocol)

**This is a real capability trade-off, not a drop-in replacement, and is worth flagging explicitly**: the project README describes the TPM addition as adding "a post-quantum Trusted Platform Module allowing cryptographic authentication and message signing." CSEc is a **symmetric-only** SHE-class HSM:

| | SLB9672 (TPM 2.0) | S32K144 CSEc |
|---|---|---|
| Symmetric crypto (AES, MAC) | Yes | Yes, hardware-accelerated |
| Asymmetric crypto (RSA/ECC) — signing, key exchange | Yes (TPM 2.0 command set) | **No** |
| Post-quantum algorithms | No (TPM 2.0 spec predates NIST PQC standardization; SLB9672 doesn't claim PQC either) | **No** |
| Remote attestation (PCRs, quotes) | Yes, TCG-standardized | No — no PCR/attestation model |
| Software stack | TCG TSS / TPM2 command interface (broad ecosystem support) | NXP CSEc driver (S32 SDK) / SHE `M1`–`M5` protocol (narrower, vendor-specific) |
| Key storage | TPM-internal, TCG hierarchy | CSEc key slots in protected flash |
| Board complexity | External chip, SPI bus, `CS#`, decoupling | None — on-die, zero extra parts |

So: **"message signing"** in the asymmetric sense and **"post-quantum"** are both capabilities the SLB9672 offered that CSEc does not — CSEc can authenticate/MAC firmware and data with AES-CMAC (a real, useful, and arguably *more* appropriate mechanism for this project's actual stated goal of "cryptographic authentication and message signing" between servo controllers on a bus, since symmetric MAC is cheaper and faster on a Cortex-M4 than RSA/ECC would have been on the TPM's host), but it cannot do public-key signatures, remote attestation, or anything PQC. If the project's real goal includes public-key servo-to-servo authentication (e.g. verifying a servo's identity to a third party without sharing a secret) or attestation to an external verifier, CSEc alone doesn't cover that — it would need either software asymmetric crypto running on the Cortex-M4 itself (slower, no hardware acceleration, and the M4 has no PQC acceleration either) or keeping some form of external secure element. Flagging this now since it's a one-way door once the SLB9672 footprint work is abandoned in favor of CSEc-only firmware.

## 3. `U1`: STM32G431 → S32K144, net-for-net

Unlike the STM32F302→STM32G431 pass, this repo does **not** have a verified physical pin-to-package-pad table for the S32K144. The `S32K1xx.pdf` data sheet's own "10 Pinouts" section states: *"For package pinouts and signal descriptions, refer to the Reference Manual"* — and the Reference Manual's actual pin table ships as a separate `S32K1xx_IO_Signal_Description_Input_Multiplexing` spreadsheet, not a document, and neither is present in this repo or reachable in this session. So, consistent with how `STM32G431`/`ADM2587E`/`ADM3055E` are already modeled in this schematic (symbol-only placeholders, no footprint/package), the new `S32K144` deviceset is **also** symbol-only, and its pin *names* are functional/net labels (`LED_RED`, `V_TEMP`, `PWM-1`, ...) rather than claimed physical port letters — except for the handful of signals below where a specific peripheral mapping is well-established (cited).

To avoid re-touching every wire in the schematic (high risk of breakage for a "which pins are free" mapping I can't fully verify anyway), the new symbol keeps every retained pin at the **same x/y position** it had on the STM32G431 symbol, just relabeled. Net connectivity is therefore unchanged for every signal below — only the symbol pin's name (and, for a few nets, the net's own name) changed:

| Function | Net name (before → after) | Notes |
|---|---|---|
| Core power | `+3V3` | `VDD_1`, `VDD_2` unchanged |
| Core ground | `GND` | `VSS_1`, `VSS_2` unchanged |
| Analog supply | `VDDA-FILTRADO` | pin renamed `VDDA/VREF+` → `VDDA` |
| Analog ground | (in `GND`) | pin renamed `VSSA` → `VSS_A` |
| External clock | `CLK-8MHZ` | pin renamed `PF0-OSC_IN` → `EXTAL`. S32K144 SOSC explicitly supports "up to 50 MHz DC external square input clock in external clock mode" (datasheet §1.1) — same external-oscillator-IC topology as before, `XTAL` left NC same as `PF1-OSC_OUT` was |
| Reset | NC (`N$11`) | pin renamed `PG10-NRST` → `RESET_b`, kept NC per the existing no-reset-circuit design decision |
| Bit-banged SPI (encoder/pot bus) | `SPI_IN` / `SPI_OUT` / `SPI_CLK` | pins renamed to match net names directly (was `PA0`/`PA1`/`PA2`) — this was already a GPIO-bit-banged bus, not a hardware SPI peripheral, on the G431 |
| RGB LED | `LED_RED` / `LED_GREEN` / `LED_BLUE` | pins renamed to match net names (was `PA3`/`PA4`/`PA5`) |
| Analog: temp / current / battery | `V_TEMP` / `V_CORRIENTE` / `V_BAT` | pins renamed to match net names (was `PA6`/`PA7`/`PB0`) — **must land on ADC0/ADC1-capable pins**, unverified until the RM pin-mux table is available |
| Motor PWM | `PWM-1`/`PWM-2`/`PWM-3`/`PWM-4` | pins renamed to match net names (was `PA8`/`PB4`/`PA9`/`PB3`) — **must land on FTM-capable pins**, same caveat |
| Motor driver enable | `EN-Q` | pin renamed to match net name (was `PA10`) |
| CAN-FD to `U6` (ADM3055E) | `FDCAN1_RX`/`FDCAN1_TX` → **`CAN0_RX`/`CAN0_TX`** | S32K144 calls its CAN-FD peripheral `FlexCAN0`; pins renamed accordingly (was `PA11`/`PA12`). Publicly documented S32K144 convention maps these to `PTE4`/`PTE5` (NXP community / S32K144EVB schematic), **not independently verified from a local source** — confirm against the RM before layout |
| RS-485 UART to `U5` (ADM2587E) | `USART1_TX`/`USART1_RX` → **`LPUART1_TX`/`LPUART1_RX`** | S32K144 calls the peripheral `LPUART`; pins renamed accordingly (was `PB6`/`PB7`). Publicly documented convention maps these to `PTC7`/`PTC6` — same "not independently verified here" caveat |
| RS-485 direction control | `SEL_SERIAL` | pin renamed to match net name (was `PB5`) |
| Debug | `SWDIO`/`SWCLK` → **`SWD_DIO`/`SWD_CLK`** | renamed for clarity; SWD signal names are vendor-agnostic so this is unaffected by the pin-table gap (was `PA13`/`PA14`) |
| Boot select | *(removed)* | STM32's `PB8-BOOT0` (hard-tied to `GND`) has no S32K144 equivalent — the S32K1xx boot ROM always boots from internal flash, no external boot-select pin exists. The tie-to-`GND` and the pin itself were deleted; `VSS_2`'s own `GND` connection is untouched |
| TPM `CS#` | *(removed)* | Was on the STM32G431's one spare pin (`PA15`). No equivalent needed — see §2 |

**Net effect on pin budget**: STM32G431 (32-pin UFQFPN) was reported as *fully* pinned out (zero spares) with the TPM attached. S32K144 in 48-pin LQFP has strictly more physical pins available than that, even before accounting for the freed `BOOT0`/`TPM_CS#` slots, so there should be headroom once real port/pin assignments are worked out — but that can't be confirmed until the RM pin-mux table is in hand.

## 4. Still missing / open items

Carried over from the prior doc, still true, now also applying to `S32K144`:

- **Footprint/package/deviceset and `.brd` placement** for `S32K144` (LQFP48), plus still-pending ones for `ADM2587E`, `ADM3055E`. None have board-side placement yet.
- **Physical pin-to-pad numbering for `S32K144`** — needs the NXP S32K1xx Reference Manual's IO Signal Description Input Multiplexing spreadsheet (not in this repo). Until then, treat every symbol pin name in this schematic as a *net label*, not a verified silicon pin. Do this before finalizing which GPIOs the four PWM channels and three ADC inputs land on (must be FTM-capable / ADC-capable respectively) and before finalizing `LPUART1`/`FlexCAN0` pin choices.
- **Confirm the 48-pin LQFP body size fits the existing board outline** — this is a bigger package than the 32-pin UFQFPN it replaces.
- Everything already open from the prior doc and still unresolved: RS-485/CAN-FD isolated-transceiver footprints, the missing 5 V rail for `U6`'s isoPower `VCC` (`+5V_ISO_CANFD`), the CAN-FD bus connector decision, and confirming what else lives on the shared bit-banged `SPI_CLK`/`SPI_IN`/`SPI_OUT` bus.
- **Firmware is a full rewrite, not a port**: `Src/main.c`, `Src/LS_*.c`, and `Test_LibreServo_v2.ioc` all target ST's HAL/STM32CubeMX and the F302/G431 register set. S32K144 firmware uses NXP's S32 SDK (S32 Design Studio, GCC-based) — different clocking (SCG vs. RCC), different peripheral drivers (LPUART/LPSPI/LPI2C/FlexCAN/FTM vs. USART/SPI/I2C/bxCAN-FDCAN/TIM), and a CSEc driver replacing whatever TPM2/TSS call sites would have been written against the SLB9672. This is a from-scratch firmware effort, not a `#define` swap.
- **CSEc key provisioning** needs its own decision: SHE key slots are typically provisioned once (factory/production step) via the CSEc command interface or a debugger fixture — very different from a TPM's more standardized provisioning/ownership flow. Needs a plan before this ships to more than a bench prototype.
- **Confirm the asymmetric/PQC capability gap in §2 is acceptable** — this is the one item in this pass that's a genuine scope/requirements question, not just an engineering TODO, and should get an explicit answer before README/marketing claims are updated to match.
