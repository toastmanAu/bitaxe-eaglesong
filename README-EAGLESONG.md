# BitAxe Eaglesong — CKB ASIC Miner

> Fork of [skot/bitaxe](https://github.com/skot/bitaxe) targeting Nervos CKB (Eaglesong PoW).  
> Uses bare BM2042AA Eaglesong ASIC chips on a custom PCB within the open BitAxe framework.

## The Vision

BitAxe proved you can build an open-source, single-ASIC Bitcoin miner for ~$20.  
This project does the same for CKB — replacing the SHA256 hash board with a BM2042AA  
(Bitmain's Eaglesong ASIC, used in the Antminer K5) on a custom open-source PCB.

**This is not a mod of an existing BitAxe hash board.** It's a new board around a bare chip,  
plugging into the existing ESP32 host board via the standard BitAxe connector.

---

## Architecture

```
┌─────────────────────────────┐
│   BitAxe ESP32 Host Board   │  ← unchanged, open hardware
│   WiFi · Stratum · Display  │
│   Power Management · USB-C  │
└──────────────┬──────────────┘
               │ UART (J7 connector)
┌──────────────┴──────────────┐
│   Eaglesong Hash Board      │  ← NEW — designed by this project
│   BM2042AA bare chip        │  ← sourced from Zeus Mining
│   0.8V–1.0V core power      │
│   25MHz clock · BGA footprint│
└─────────────────────────────┘
```

## Lineage

**[NerdMiner_CKB](https://github.com/toastmanAu/NerdMiner_CKB)** was the precursor —  
it proved Eaglesong can run on an ESP32 and connect to CKB stratum pools.  
That code is the firmware foundation for this project. Same Eaglesong implementation,  
same stratum protocol — just with the hashing offloaded from ESP32 CPU to ASIC silicon.

**Proof of concept → production silicon:**
```
NerdMiner_CKB (ESP32 software Eaglesong) 
    → validated protocol + algorithm
    → BitAxe Eaglesong (BM2042AA ASIC Eaglesong)
```

---

## Bill of Materials (target)

| Component | Source | Notes |
|---|---|---|
| BM2042AA bare chip | Zeus Mining | Eaglesong ASIC, Antminer K5 |
| ESP32-S3 host board | BitAxe Ultra PCB | Unchanged from upstream |
| Custom hash board PCB | OSHPark / JLCPCB | KiCad design (this repo) |
| BGA rework / paste | Standard SMD supply | BGA soldering required |

---

## Firmware Plan

### What stays the same (from ESP-Miner)
- WiFi provisioning
- Display / web UI
- Power management
- Stratum connection management
- Temperature / fan control

### What changes
- `stratum_task.c` — CKB stratum job format (64-bit nonce, full CKB block header)
- New driver: `bm2042.c` — replaces `bm1397.c` / `bm1366.c`
- Eaglesong job construction — adapted from `NerdMiner_CKB/src/mining/eaglesong.cpp`
- Hash validation — software Eaglesong for share verification before submission

### CKB Stratum vs Bitcoin Stratum
| Field | Bitcoin | CKB |
|---|---|---|
| Algorithm | SHA256d | Eaglesong |
| Nonce | 32-bit | 64-bit |
| Header | 80 bytes | ~200 bytes |
| Job method | `mining.notify` | `mining.notify` |

---

## Hardware Design

### Hash Board (Phase 1 of build)
- BM2042AA in BGA package — PCB footprint from chip dimensions
- Core voltage regulator: 0.8V–1.0V (TBD from Zeus Mining datasheet)
- I/O voltage: 1.8V (supplied by host board via J7)
- Clock: 25MHz crystal oscillator
- Connector: J7 pinout compatible with BitAxe host board
- Target size: ~50×30mm (same as BitAxe Ultra hash board)

### Tools needed
- KiCad 8.x (schematic + layout)
- Logic analyser — verify BM2042AA UART comms after first board spin
- BGA reflow setup (hot air + paste or reflow oven)

---

## Repository Structure

```
/                          ← BitAxe upstream hardware reference
README-EAGLESONG.md        ← This file
eaglesong/
  firmware/               ← ESP-Miner fork (CKB stratum + BM2042 driver)
  hardware/               ← KiCad hash board design
  protocol/               ← Logic analyser captures, BM2042 comms notes
  research/               ← BM2042AA datasheet fragments, Zeus Mining notes
```

---

## Status

| Phase | Status |
|---|---|
| Eaglesong on ESP32 (proof of concept) | ✅ Done — NerdMiner_CKB |
| BM2042AA sourcing (Zeus Mining) | 🔜 Pending order |
| Hash board KiCad design | ⏳ Awaiting chip datasheet |
| Firmware: BM2042 driver | ⏳ After board design |
| First board spin | ⏳ After design |
| Logic analyser validation | ⏳ After first spin |

---

## Related Projects

- [toastmanAu/NerdMiner_CKB](https://github.com/toastmanAu/NerdMiner_CKB) — the precursor. ESP32 software Eaglesong miner.
- [toastmanAu/ckb-stratum-proxy](https://github.com/toastmanAu/ckb-stratum-proxy) — local stratum proxy for CKB
- [skot/bitaxe](https://github.com/skot/bitaxe) — upstream BitAxe hardware
- [skot/ESP-Miner](https://github.com/skot/ESP-Miner) — upstream BitAxe firmware

---

*Open hardware. Open source. Open mining.*
