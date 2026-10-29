# DAW Controller Project – Summary

Two-channel DAW/MIDI controller board with motorized faders, key switches, a rotary encoder, per-key RGB LEDs and an optional OLED display. An STM32G0B1KET6 on each channel board talks over **UART** to a larger STM32 "motherboard", which handles the connection to the DAW.

> **Legend:** ✅ = current selection · ⏳ = open decision · other entries = alternatives kept for reference

---

## 0. Current selection at a glance

| Function | Selection | Status |
|---|---|---|
| Key switches | Cherry MX Blue RGB | ✅ |
| Shift register | 2× 74HC165, cascaded (16 inputs) | ✅ |
| Per-key LED | ~~Lite-On LTST-A683CEGBW~~ (no longer available) → **SK6812MINI-E** (proposed replacement) | ✅ proposed |
| Motorized fader | ALPS RSA0N11M9A0J | ✅ |
| Motor driver | TB6612FNG | ✅ (fits the fader, see section 4) |
| Rotary encoder | – | ⏳ open |
| Channel controller | **STM32G0B1KET6** (LQFP32, 512 KB flash) | ✅ |
| Dev board | NUCLEO-G0B1RE (same die, 64 pins) + STLINK-V3MINIE for own boards | ✅ |
| Motherboard / DAW link | STM32L5 (e.g. STM32L552), USB MIDI to the DAW | ✅ proposed |
| Board-to-board link | UART over RS-485 bus, 4–8 channel boards | ✅ proposed |

---

## 1. System architecture

```
DAW ──USB MIDI──► Motherboard (STM32L552) ──UART/RS-485 bus──► Channel boards 1…8 (STM32G0B1KET6 each)
                                                     ├─ 2× motor fader + TB6612
                                                     ├─ 2× 74HC165: 8× Cherry MX, touch, encoder switch, address
                                                     ├─ SK6812MINI-E LEDs
                                                     ├─ rotary encoder
                                                     └─ OLED (optional)
```

- The **channel board** does everything time-critical locally: fader position control, key scanning/debouncing, encoder counting, LEDs.
- The **motherboard** handles USB MIDI to the DAW and only exchanges high-level data with the channel boards (fader setpoints/positions, key events, encoder deltas, LED colors, display content).
- This keeps the control loops independent of USB timing, and more channel boards can be added later.

### Motherboard: STM32L552 (proposed)
- Cortex-M33 with FPU, up to 110 MHz, 512 KB flash, 256 KB SRAM
- **USB 2.0 full-speed, crystal-less** → USB MIDI without an external crystal
- Several USARTs/UARTs (one per channel board possible), 3 SPIs, 4 I2Cs
- Up to 22 capacitive sensing channels (could also do touch sensing centrally if needed)
- Packages from 48 to 144 pins
- Alternatives: STM32H5 series, STM32G4, STM32F4 ("Black Pill" for prototyping)

### Link between channel boards and motherboard: UART over RS-485 ✅

**Requirement:** at least 4, preferably 8 channel boards (8–16 faders).

Point-to-point UART is ruled out at this size: the L552 has 3 USARTs, 2 UARTs and 1 LPUART, so only about 5 boards could be connected. **RS-485** keeps the UART on the firmware side but puts all boards on one shared bus.

```
L552 ─[XCVR]══╦══════════╦══════════╦═══ … ═══╦══[120 Ω]
           [120 Ω]    [XCVR]     [XCVR]     [XCVR]
                     Board 1    Board 2 …  Board 8
```

**Wiring**
- One twisted pair (A/B) plus GND, run as a **daisy chain** from board to board (no star, short stubs)
- **120 Ω termination** at both physical ends of the bus
- **Fail-safe biasing** (pull-up on A, pull-down on B, approx. 390–680 Ω) at the master, or transceivers with built-in fail-safe
- Transceiver per board: **THVD1450** (up to 50 Mbps, 3.3 V) or **MAX3485** (10 Mbps, 3.3 V)
- Use the STM32 USART's **hardware driver-enable (DE)** output to switch the transceiver automatically
- Connector suggestion: RJ45 with CAT5 cable (twisted pairs; spare pairs can carry GND/power) or a keyed IDC ribbon cable
- **Board address:** 3–4 solder jumpers or a DIP switch per board (8 boards → 3 bits), read through the second 74HC165 (no extra MCU pins)

**Protocol (master/slave polling)**
- Motherboard is the **master** and polls each board in turn; only the addressed board answers
- 8N1, **2 Mbaud** (1 Mbaud also works)
- Packet: `| 0xA5 | Address | Type | Length | Data… | CRC-8 |`
- Poll request carries outgoing data (fader setpoints, LED colors); the answer carries incoming data (fader positions while touched, touch state, key events, encoder deltas)
- Timing: poll approx. 4 bytes + answer approx. 12 bytes + bus turnaround
  - at 1 Mbaud: approx. 170 µs per board → **8 boards ≈ 1.4 ms per cycle**
  - at 2 Mbaud: approx. 90 µs per board → **8 boards ≈ 0.7 ms per cycle**
- Bandwidth need per board approx. 5 kB/s → plenty of headroom
- OLED: send text/commands, never pixel data; the G0 renders locally
- On both MCUs: USART + DMA, RX in circular mode with idle-line interrupt

**Alternative kept for reference: CAN FD**
- STM32G0B1 has FDCAN, the STM32L552 has an FDCAN controller too (the G071 has none)
- Addressing, priorities, arbitration and CRC in hardware; boards send events without polling
- Transceiver per board (e.g. TCAN1042, MCP2562), twisted pair, 120 Ω at both ends
- Scales to dozens of boards; more initial setup than UART
- The selected **STM32G0B1KET6** has two FDCAN controllers, so this option stays open without changing the MCU (pins for FDCAN TX/RX must be kept free or reachable in the 32-pin layout)

---

## 2. Key switches: Cherry MX Blue RGB ✅

- MX variant **optimized for SMD LEDs** mounted directly on the PCB: transparent housing with light-scattering lens, **opening in the base** for the LED (LED not included)
- **Blue = clicky:** tactile bump plus audible click; actuation force approx. 50–60 cN (sources differ, check the Cherry datasheet)
- Life > 50 million actuations; contact bounce ≤ 5 ms
- Order codes: MX1A-E1N\* / MX2A-E1N\* (B = blue)
- ⚠️ **Studio note:** Blue switches are loud. If the controller is used near open microphones during recording, consider MX RGB Brown (tactile, quiet) or MX Silent variants

---

## 3. Per-key LEDs

### ✅ Proposed replacement: SK6812MINI-E
The de-facto standard for per-key RGB under MX switches in keyboards. Reverse mount through a PCB cutout under the switch's LED window.

| Parameter | SK6812MINI-E |
|---|---|
| Manufacturer | OPSCO Optoelectronics |
| Size | 3.2 × 2.8 × 1.78 mm |
| Mounting | Reverse mount: LED body sits in a PCB cutout, soldered via long side pads ("-E" = legs, easy to hand-solder) |
| Pins | 1 VDD, 2 DOUT, 3 GND, 4 DIN |
| Supply | 3.7–5.5 V |
| Data rate | 800 kbps, WS2812-compatible protocol |
| Viewing angle | 120° |
| Moisture sensitivity | MSL 3 |
| Sources | JLCPCB (C5149201, for PCBA), keyboard-part shops (e.g. splitkb), Adafruit #4960 (10-pack) |

**Differences from the LTST-A683CEGBW to watch out for:**
- **Different pinout** → new footprint and symbol; use the cutout dimensions from the SK6812MINI-E datasheet
- Datasheet: <https://cdn-shop.adafruit.com/product-files/4960/4960_SK6812MINI-E_REV02_EN.pdf>
- Reset time and input-high threshold: check the datasheet. SK6812 parts typically need a longer reset pause than WS2812B and a high level relative to VDD → **use a level shifter with a 3.3 V MCU** (see below)
- Color order: SK6812 is usually GRB; verify with a test

### Circuit around the LEDs (applies to SK6812MINI-E as well)
- **100 nF ceramic capacitor per LED** (X7R/X5R, ≥ 10 V, 0402/0603), close to VDD/GND
- 5 V supply, budget the LED current per LED from the datasheet plus margin
- Bulk capacitor 10–100 µF at the LED supply input
- 100–470 Ω (typ. 330 Ω) series resistor in the data line
- **Level shifter** from 3.3 V MCU to 5 V LED logic: **74AHCT1G125** or **SN74LV1T34**, powered from 5 V
- Drive the data line with **timer + DMA** (or SPI + DMA), never bit-banged

### Reference: Lite-On LTST-A683CEGBW (previous selection, no longer available)
| Parameter | Value (released datasheet, 2019) |
|---|---|
| Package | PLCC4, reverse mount, 6.2 × 2.2 × 1.9 mm |
| Pins | 1 VSS, 2 DIN, 3 DOUT, 4 VDD |
| Supply | 4.5–5.5 V, max. 19 mA total |
| Brightness | R 80–160, G 280–560, B 60–120 mcd, 120° |
| Data rate / timing | 800 kbps; T0H 300 ns, T0L 900 ns, T1H 900 ns, T1L 300 ns (±80 ns), reset > 50 µs |
| Input high level | VIH min. 3 V |
| Decoupling | 100 nF per LED (value from the preliminary 2018 datasheet; reference design also shows an unspecified VDD series resistor → 0–10 Ω) |

Other reverse-mount Lite-On parts in the same style exist, but for MX keyboards the SK6812MINI-E is the most widely used and easiest to source.

---

## 4. Motorized faders

### ✅ Fader: ALPS RSA0N11M9A0J
| Parameter | Value |
|---|---|
| Series | Alps Motor N Fader (motor-driven master type) |
| Travel | 100 mm |
| Resistance | 10 kΩ, linear, ±20 % |
| Elements | Single unit (servo track only) |
| Touch sense | Touch sense track available (conductive lever) |
| Lever | 9-T (T-bar) |
| Motor | **rated 10 V DC, max. 800 mA at 10 V** |
| Operating force | 0.8 ± 0.5 N |
| Life | 30,000 cycles |
| Temperature | −10 to +60 °C |

The suffix (A0J) denotes a specific variant; check the exact lever/terminal version in the Alps ordering table before ordering.

**Consequences for the design:**
- Needs a **10 V motor supply** (e.g. 12 V input with a buck converter, or a 10 V rail)
- Worst case 2 × 800 mA = 1.6 A for two faders on the motor rail → size the supply accordingly
- Fader track: ends to 3.3 V/GND, wiper to the ADC, 100 nF at the ADC pin

### ✅ Motor driver: TB6612FNG
Fits the ALPS fader: VM 2.5–13.5 V (covers 10 V), **1.2 A continuous / 3.2 A peak per channel** (above the 800 mA motor current), one chip drives both faders.

| TB6612 pin           | Connection                                                   |
| -------------------- | ------------------------------------------------------------ |
| PWMA / PWMB          | MCU PWM, **≥ 20 kHz** (inaudible)                            |
| AIN1/AIN2, BIN1/BIN2 | Direction GPIOs (10 = up, 01 = down, 11 = brake, 00 = coast) |
| STBY                 | GPIO or 3.3 V                                                |
| VCC                  | 3.3 V                                                        |
| VM                   | 10 V motor supply, with 10–47 µF + 100 nF                    |
| AO1/AO2, BO1/BO2     | Fader motors                                                 |

**Alternatives (kept for reference):**
| Driver | Notes |
|---|---|
| DRV8838 | Single H-bridge, PH/EN interface, up to approx. 11 V, approx. 1.7 A |
| DRV8833 | Dual, 2.7–10.8 V → at its limit with 10 V fader motors |
| DRV8871 | Single, 6.5–45 V, 3.6 A peak, for larger motors |
| L293D / L293DD, L298N | Bipolar, 1.5–3 V drop, slow (audible PWM) → **avoid** |

### Touch sense
- Read the fader's touch track; stop the motor while touched and only send positions to the motherboard while touched (avoids feedback with the DAW)
- The STM32G0 has no touch controller → use a small touch IC per fader (e.g. **AT42QT1011**, digital output)
- The touch outputs are read through the **second 74HC165** at 1 kHz → motor stops within approx. 1 ms of touching, no extra MCU pins

### Position control (software on the STM32G0)
- **P or PD controller** with deadband, minimum PWM (static friction), maximum PWM (speed limit), active braking inside the deadband
- Fixed loop rate **1 kHz**; integer/fixed-point math (the M0+ has no FPU)
- Average ADC readings (DMA)
- Tuning: increase KP until the fader moves briskly, then dampen overshoot with KD

### Cycle time on the STM32G0
| Step | Time |
|---|---|
| ADC, 2 channels (DMA, in hardware) | approx. 1–2 µs |
| Averaging 8–16 samples (DMA) | approx. 15–30 µs, in the background |
| PD, integer, 2 faders | approx. 1–2 µs |
| PD, float (software emulation) | approx. 5–20 µs → avoid |
| PWM/GPIO update, ISR entry | < 1 µs each |
| **Total** | **approx. 5–10 µs → 0.5–2 % CPU at 1 kHz** |

---

## 5. Input readout: 2× 74HC165 ✅

8-bit parallel-in/serial-out shift registers, cascaded to **16 inputs**, 2–6 V, read via hardware SPI.
The second chip saves MCU pins on the 32-pin STM32G0B1KET6: all slow digital inputs go through the shift registers.
Alternatives kept for reference: CD4021B, 74HC597, I/O expanders MCP23017/MCP23S17/TCA9555 (built-in pull-ups, interrupt output).

### Input assignment
| Chip | Inputs | Use |
|---|---|---|
| 74HC165 #1 | A–H | 8× Cherry MX keys |
| 74HC165 #2 | A–B | Touch sense fader 1/2 (AT42QT1011 outputs) |
| 74HC165 #2 | C | Encoder push switch |
| 74HC165 #2 | D–F (G) | Board address jumpers (3–4 bits) |
| 74HC165 #2 | (G)–H | Spare |

### Wiring
- Key inputs and switch inputs: to GND, **10 kΩ pull-up** each (resistor arrays possible) → pressed/closed = 0
- Touch IC outputs go directly to the 165 inputs (push-pull outputs need no pull-up; check the IC's output type)
- Unused inputs tied to VCC or GND
- Shared on both chips: SH/LD → one GPIO, CLK → SPI SCK, CLK INH → GND
- Cascade: **QH of chip #2 → SER of chip #1**, **QH of chip #1 → SPI MISO**, SER of chip #2 → GND
- 100 nF decoupling per chip, powered from 3.3 V
- SPI: master, receive only, mode 0; read **2 × 8 bits** (or one 16-bit frame); on the G0 set FRXTH for 8-bit frames and use 8-bit access to `DR`

### Timing and handling
- One read of both chips: approx. **3–5 µs**; scan at **1 kHz** in the same timer ISR as the fader loop
- **No pin interrupts** on the switches; poll instead
- **Eager debouncing:** report the first edge, then 5 ms lockout per key → latency < 1 ms
- ISR pushes events into a **lock-free ring buffer**; the main loop forwards them over UART to the motherboard

---

## 6. Rotary encoder ⏳ (open decision)

Prefer **no detents** for a smooth, pot-like feel.

| Option | Type | Resolution | Notes |
|---|---|---|---|
| Bourns PEC11R | Mechanical | 24 PPR → 96 steps/turn | Cheap; "N" = no switch, "S" = switch; check detent option in the ordering table; needs debouncing |
| Bourns ENA1J-B28-L00128L | Optical | 128 PPR → 512 steps/turn | No detents, no bounce, no wear; best feel |
| AS5600 / AS5048A | Magnetic | 4096–16384 positions/turn | Contactless; requires own shaft/bearing/magnet mechanics; I²C/SPI |

On the G0: timer **encoder mode** (hardware counting), read the counter difference in the 1 ms tick, send deltas to the motherboard. Acceleration can be applied on either side.

---

## 7. Channel controller: STM32G0B1KET6 ✅

| Parameter | STM32G0B1KET6 |
|---|---|
| Core | Arm Cortex-M0+, 64 MHz, no FPU |
| Memory | 512 KB flash, 144 KB SRAM |
| Package | **LQFP32**, 7 × 7 mm, 0.8 mm pitch (hand-solderable) |
| Usable I/Os | approx. 30 (check the pinout table; "up to 94" applies to the 100-pin version) |
| Peripherals used | ADC, timers (PWM, encoder mode, LED DMA, tick), SPI, I2C, USART with hardware DE, 12-channel DMA |
| Reserves | USB FS (crystal-less) and **2× FDCAN** → CAN FD option stays open |
| Part-number code | K = 32 pins, E = 512 KB, T = LQFP, 6 = −40 to +85 °C |

⚠️ Order the **standard pinout without "N" suffix** (STM32G0B1KET6, not KET6N). The N variants have an alternative pinout with a separate I/O supply pin (VDDIO2), which costs a usable pin.

### Pin budget (21 of approx. 30 I/Os)
| Function | Peripheral | Pins |
|---|---|---|
| Fader positions (2×) | ADC1 + DMA | 2 |
| Motor PWM (2×) | TIM3 CH1/CH2, 20 kHz | 2 |
| Motor direction | GPIO (AIN1/2, BIN1/2) | 4 |
| TB6612 STBY | **tied to 3.3 V** | 0 |
| 2× 74HC165 (keys, touch, encoder switch, address) | SPI1 (SCK, MISO) + GPIO SH/LD | 3 |
| SK6812MINI-E data | TIM16 or TIM1 channel + DMA (via level shifter) | 1 |
| Rotary encoder A/B | TIM2 encoder mode | 2 |
| OLED (SSD1306 128×64) | I2C1 + DMA | 2 |
| **Link to motherboard** | **USART1 + DMA (TX/RX) + hardware DE** → RS-485 transceiver | **3** |
| 1 kHz tick | TIM6 | 0 |
| Debug | SWD (SWDIO, SWCLK) | 2 |
| **Total** | | **21** |

About 9 pins remain for CubeMX to resolve alternate-function conflicts and for later additions (e.g. FDCAN TX/RX if switching to CAN FD).

**Before PCB layout:** enter the full peripheral setup in **CubeMX with the STM32G0B1KET6 selected** and confirm that every function maps to a pin. In the 32-pin package there are fewer alternative pin positions per peripheral. Fallback if it doesn't fit: **STM32G0B1CET6** (LQFP48, same die, same firmware).

### Development hardware
- **Dev board: NUCLEO-G0B1RE** (STM32G0B1RET6, same die in LQFP64, 512 KB / 144 KB, built-in ST-Link). There is no dev board with the 32-pin version.
  - Create the CubeMX project for the KET6 first, then use exactly the same port pins (e.g. PA0, PB3) on the Nucleo → firmware moves to the own board unchanged, only the target chip setting differs
- **Programmer for the own channel boards: STLINK-V3MINIE**; put a small SWD header on each board (SWDIO, SWCLK, NRST, GND, 3.3 V)
- Cheap alternative board: WeAct Studio STM32G0B1 Core Board (STM32G0B1CBT6, LQFP48, no debugger → needs an external ST-Link)

### Alternatives kept for reference
| MCU | Notes |
|---|---|
| STM32G0B1CET6 / CBT6 | LQFP48, same die, more pins; fallback if the 32-pin mapping doesn't fit |
| STM32G071RBT6 | LQFP64, up to 60 I/Os, 128 KB flash / 36 KB RAM, 7 DMA channels, matching NUCLEO-G071RB; **no CAN, no USB** → closes the CAN FD option |

### Design rules
- Don't share the 165's SPI with the OLED
- LEDs via timer + DMA (approx. 48 bytes RAM per LED)
- UART RX with DMA in circular mode + idle-line interrupt, so incoming packets are handled without per-byte interrupts
- **Interrupt priorities (M0+, 4 levels):** UART DMA/idle → 1 kHz timer (faders + keys, approx. 10–20 µs) → main loop (packet processing, LEDs, display)
- Expected CPU load: **< 10 %**

---

## 8. Side notes
- **LM555:** classic timer IC (astable, monostable, bistable); CMOS versions (TLC555, LMC555, ICM7555) run at 3.3 V. Not needed here, since the MCU handles timing, PWM and debouncing.

## 9. Suggested development order
0. CubeMX pin mapping for the STM32G0B1KET6 (confirms the 32-pin package fits)
1. Fader control on the NUCLEO-G0B1RE with a fixed setpoint
2. Controller tuning (KP, then KD), touch sense
3. Keys and inputs (2× 74HC165) and encoder
4. LEDs (SK6812MINI-E via timer + DMA, level shifter)
5. UART protocol between G0 and motherboard
6. USB MIDI on the motherboard (STM32L552)
7. OLED display

## 10. Open points
- ⏳ Rotary encoder selection
- CubeMX check: all peripherals mappable on the STM32G0B1KET6 (LQFP32) without conflicts; otherwise fall back to LQFP48
- Verify ALPS suffix "A0J" (lever/terminal variant) in the Alps ordering table
- SK6812MINI-E: confirm reset time, VIH and color order in the datasheet; design new footprint and cutout
- RS-485 connector and cable type (RJ45/CAT5 or IDC ribbon)
- Power distribution for 8 boards: 16 faders × up to 800 mA = up to 12.8 A at 10 V in the worst case (all faders moving at once, e.g. on project recall). Either a central supply sized for realistic peaks plus current limiting, or one supply/regulator per board; add the 5 V LED current of 64 LEDs
- Consider quieter MX RGB variant if the controller is used near microphones
