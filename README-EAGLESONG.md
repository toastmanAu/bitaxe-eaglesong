# BitAxe Eaglesong — CKB Mining Hardware Project

> Fork of [skot/bitaxe](https://github.com/skot/bitaxe) targeting Nervos CKB (Eaglesong PoW algorithm).

## The Vision

BitAxe proved you can build an open-source, single-ASIC Bitcoin miner for ~$20. This project asks: can we do the same for CKB?

CKB uses **Eaglesong** — a custom sponge construction designed by Nervos, deliberately ASIC-friendly but distinct from SHA256d. Existing BitAxe hash boards (BM1397, BM1366, BM1368) are SHA256-hardwired and cannot run Eaglesong. This project aims to:

1. **Reverse-engineer the BitAxe host↔hashboard protocol** (SPI/UART, logic analyser capture)
2. **Identify a candidate Eaglesong ASIC** — BM2042AA or equivalent
3. **Design a drop-in hash board** using the new ASIC, retaining the BitAxe ESP32 host board
4. **Adapt the BitAxe firmware** (ESP-Miner) to speak Eaglesong stratum and drive the new board

---

## Hardware Targets

### Host board
BitAxe Ultra / Supra — ESP32-S3 based, handles WiFi, stratum, display, power management.
**We keep this as-is.** The ESP32 is not the bottleneck — the hash board is.

### Hash board (current, SHA256)
| Model | ASIC | Algorithm |
|---|---|---|
| BitAxe 100 (Supra) | BM1397 | SHA256d |
| BitAxe 200 (Ultra) | BM1366 | SHA256d |
| BitAxe 400 (Gamma) | BM1370 | SHA256d |

### Target hash board (Eaglesong)
- **Primary candidate**: BM2042AA — Bitmain's Eaglesong ASIC used in Antminer K5 (1.13 TH/s @ 1148W)
- **Alternative**: Any custom ASIC tape-out if BM2042AA is unobtainium (long-term)
- Single-chip target: ~5–50 GH/s at low power (comparable to BM1366 single-chip performance ratio)

---

## Phase 1 — Protocol Reverse Engineering

### What we need to capture
The BM1397/BM1366 communicate with the ESP32 via UART (and some SPI for control). The protocol is partially documented in [ESP-Miner source](https://github.com/skot/ESP-Miner) but underdocumented at the signal level.

**Logic analyser capture plan:**
- Probe points: TX/RX lines between ESP32 and hash board connector
- Capture: full mining session — init sequence, work submission, nonce response
- Decode: identify command/response framing, job format, nonce encoding, difficulty target format
- Compare against ESP-Miner source (`bm1397.c`, `bm1366.c`) to validate decode

### Tools
- Logic analyser (Phill's incoming unit)
- [PulseView](https://sigrok.org/wiki/PulseView) + UART decoder
- ESP-Miner source as reference: `main/tasks/stratum_task.c`, `main/drivers/bm1397.c`

### Probe points on BitAxe Ultra
```
J7 connector (hash board interface):
  Pin 1: VCC (1.8V logic)
  Pin 2: UART TX (ESP32 → ASIC)
  Pin 3: UART RX (ASIC → ESP32)
  Pin 4: RESET
  Pin 5: GND
  (verify against schematic: hardware/esp-miner/esp-miner.kicad_sch)
```

---

## Phase 2 — BM2042AA Research

### What we know
- Used in: Antminer K5 (CKB, ~1.13 TH/s)
- Process node: unknown (likely 8nm or 12nm given K5 vintage)
- Package: BGA (same family as BM1397)
- Eaglesong pipeline: 43-round sponge permutation, 256-bit state

### What we need to find
- [ ] Datasheet / register map (likely NDA — reverse from K5 board capture)
- [ ] Pin mapping (X-ray or careful decapping)
- [ ] Communication protocol (likely same UART framing as BM1397 with different job format)
- [ ] Power domain requirements (core voltage, I/O voltage)
- [ ] Thermal profile (TDP per chip at target frequency)

### Sourcing
- AliExpress: bare BM2042AA chips sometimes available from board repair suppliers
- Antminer K5 hash boards: available secondhand — useful for protocol capture and chip extraction
- Direct from Bitmain: unlikely without volume commitment

---

## Phase 3 — Firmware Adaptation

### ESP-Miner changes needed
- `stratum_task.c`: Eaglesong stratum protocol (CKB uses `eth_getWork`-style, not Bitcoin getblocktemplate)
- New driver: `bm2042.c` (adapted from `bm1397.c` with Eaglesong job format)
- Hash function: port `eaglesong.cpp` from [NerdMiner_CKB](https://github.com/toastmanAu/NerdMiner_CKB) — useful for validation even if ASIC does the real hashing
- Difficulty: CKB uses compact target format, different from Bitcoin

### Stratum differences (CKB vs Bitcoin)
| Field | Bitcoin | CKB |
|---|---|---|
| Job method | `mining.notify` | `mining.notify` (same) |
| Hash algo | SHA256d | Eaglesong |
| Nonce size | 32-bit | 64-bit |
| Header size | 80 bytes | ~200 bytes (CKB block header) |
| Target format | nBits compact | compact (similar) |

---

## Phase 4 — Hardware Design

### New hash board concept
```
[BM2042AA x1]
  ├── UART interface → BitAxe host board (J7 compatible pinout)
  ├── Core power: 0.8V–1.0V (TBD from K5 measurements)
  ├── I/O power: 1.8V (from host board)
  ├── Clock: external 25MHz crystal or from ESP32 GPIO
  └── Thermal: single heatsink + 40mm fan (same as BitAxe Ultra)

Target board size: same as BitAxe Ultra hash board (~50x30mm)
Target power: <30W per chip (lower freq, lower power = open-source mining viability)
```

### KiCad design (Phase 4 only — after protocol + chip confirmed)
- Fork `hardware/esp-miner/` schematic
- Replace ASIC symbol + footprint
- Adjust power rails for BM2042AA requirements
- Keep all other circuitry (USB-C, ESP32, display header) unchanged

---

## Repository Structure

```
/                          ← BitAxe upstream (hardware reference)
README-EAGLESONG.md        ← This file
eaglesong/
  protocol/               ← Logic analyser captures + decode notes
  firmware/               ← ESP-Miner fork with Eaglesong stratum
  hardware/               ← KiCad hash board design (Phase 4)
  research/               ← BM2042AA datasheet fragments, notes
```

---

## Related Projects

- [toastmanAu/NerdMiner_CKB](https://github.com/toastmanAu/NerdMiner_CKB) — ESP32 software Eaglesong miner (no ASIC)
- [toastmanAu/ckb-stratum-proxy](https://github.com/toastmanAu/ckb-stratum-proxy) — local stratum proxy for CKB mining
- [skot/bitaxe](https://github.com/skot/bitaxe) — upstream BitAxe hardware
- [skot/ESP-Miner](https://github.com/skot/ESP-Miner) — upstream BitAxe firmware

---

## Status

| Phase | Status |
|---|---|
| 1. Protocol reverse engineering | 🔜 Waiting on logic analyser |
| 2. BM2042AA research | 🔬 In progress |
| 3. Firmware adaptation | ⏳ After Phase 1 |
| 4. Hardware design | ⏳ After Phase 2 |

---

*This is experimental hardware research. No warranty. Don't connect anything you're not comfortable losing.*
