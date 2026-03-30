# Jeep Renegade — OBD-II Signal Set

[![OBDb v3](https://img.shields.io/badge/OBDb-v3-blue)](https://github.com/OBDb)
[![PRs Merged](https://img.shields.io/badge/PRs-106%20%2B%20115%20merged-green)](#contributing)
[![Commands](https://img.shields.io/badge/commands-54-orange)](#signal-set)
[![ECUs](https://img.shields.io/badge/ECUs-4%20active-purple)](#ecu-map)

Signal set for **Jeep Renegade** vehicles (2014+), focused on the **2019 2.0 MultiJet II Diesel (Trailhawk)** with ZF 9HP48 transmission. Part of the [OBDb](https://github.com/OBDb) open vehicle data project.

## Vehicle Coverage

| Generation | Years | Engine | Status |
|---|---|---|---|
| First Gen | 2014-2018 | 1.4L/2.4L gas, 1.6L/2.0L diesel | Partial (shared platform) |
| **First Gen Facelift** | **2019-2023** | **2.0 MultiJet II diesel** | **Primary target** |
| Continued (LatAm/EU) | 2024+ | 1.3L Turbo-Flex | Untested |

### Primary Test Vehicle

- **2019 Trailhawk 2.0 MultiJet II (VM Motori) + ZF 9HP48**
- CAN 29-bit 500kbps only (protocol 7)
- Platform: SCCS (shared with Fiat 500X, Jeep Compass, Fiat Tipo)
- 152,000+ km, DPF original, Brazil market

## Signal Set

**54 commands** across 4 ECUs covering engine, transmission, DPF, fuel, and emissions:

### ECU Map

| ECU | Header | Manufacturer | Module | Commands | Session |
|:---:|--------|-------------|--------|:--------:|---------|
| ECM | DA10 | Bosch | EDC17 diesel | 42 | Extended (warmup ~5s) |
| TCM | DA18 | ZF | 9HP48 relay via PCM | 8 | Default |
| BCM | DA40 | Magneti Marelli | BC521K body control | 1 | Default |
| M01 | DB33 | — | Mode 01 broadcast | 3 | None |

### Signal Categories

| Category | Signals | Examples |
|---|:---:|---|
| Engine | 15 | RPM, coolant temp, boost, VGT, throttle, oil pump |
| DPF/Aftertreatment | 18 | Soot %, regen count/progress, inlet/outlet temps, diff pressure, NOx |
| Fuel/Injection | 8 | Rail pressure, fuel corrections (4 cyl), fuel temp, M-prop |
| Transmission | 6 | ATF temp, wheel speeds (4 wheels), gear |
| Emissions | 5 | Lambda, DeSOx, NOx upstream/downstream |
| Electrical | 2 | Battery voltage (ECM + BCM) |
| EEPROM | 2 | Odometer, key-on counter |

### Key Formulas (FCA Diesel Pattern)

| Type | Formula | DIDs |
|---|---|---|
| Temperatures | `(A*256+B)*0.02-40` C | 1003, 1935, 3914, 3915, 18DE |
| Signed offset | `(raw-32768)/N` | 189B, 18E2, 195A, 388A |
| Soot loading | `(A*256+B)*1000/65535` % | 18E4 |
| Wheel speed | `(A*256+B)/16` km/h | 2B1B-2B1E |
| Rail pressure | `(A*256+B)/20` bar | 1946, 1947 |
| RPM | `(A*256+B)/4` | 1000 |

## Discovery

341 responsive PIDs discovered across 7 ECUs (ECM 272, TCM 42, ABS 18, EPS 17, IPC 21, SCR 34, BCM 1). Formulas validated through:

- AlfaOBD cross-reference (13 formula corrections)
- 424 km highway trip (214,000+ data points, MX+ adapter)
- Regen event captured live (soot 100% to 18%, DPF 681C)

### Additional ECUs Identified (not in signalset yet)

| ECU | Header | Manufacturer | PIDs | Status |
|:---:|--------|-------------|:----:|--------|
| ABS/ESC | DA1A | Continental (ATE) | 18 | Wave 5 planned |
| EPS | DA1F | FGA | 17 | Wave 6 planned |
| IPC | DA60 | Magneti Marelli | 21 | Wave 7 planned |
| SCR | DAC7 | Continental | 34 | Wave pending |

## Adapter Notes

This vehicle requires **CAN 29-bit** (protocol 7). CAN 11-bit does not work.

| Adapter | Connection | Multiframe | Warmup DA10 | Latency |
|---|---|:---:|:---:|:---:|
| OBDLink MX+ (STN2255) | BT Classic | OK | ~5s (1 att) | 51ms |
| Vgate iCar Pro (ELM327 clone) | BLE | Broken | ~5s (11 att) | 76ms |

**Critical sequence for 29-bit CAN:** ATCRA must be sent BEFORE diagnostic session (10 03). Sending ATCRA after session causes 0% response rate. This affects Pelican and any app using SignalPoller pattern.

### ECM Extended Session

The ECM (DA10, Bosch EDC17) requires Extended Diagnostic Session (10 03) before reading UDS DIDs. After any header switch, a warmup period of ~5 seconds is needed before the ECM responds to 10 03. DB33 broadcast resets this warmup on both adapters.

## Contributing

This signal set is actively maintained. Contributions welcome for:

- Additional model years (2015-2018 gas engines, 2024+ Turbo-Flex)
- Formula validation (many DIDs still need AlfaOBD confirmation)
- Other FCA SCCS platform vehicles (Fiat 500X, Compass, Tipo)

### Merged PRs

- **PR #106** — Initial signal set: 75 commands, first diesel FCA on OBDb
- **PR #115** — Extended Session (din:03) + 4 formula corrections for 42 DA10 commands

### Planned Waves

| Wave | Focus | PIDs | Status |
|---|---|:---:|---|
| 0 | Original signalset | 54 | **Merged** |
| 1 | DPF/Aftertreatment | 20 | Commands generated |
| 2 | Engine/Turbo/Fuel | 19 | Commands generated |
| 3 | TCM ZF 9HP48 | 14 | Discovery done |
| 4 | EEPROM/Config | 10 | Discovery done |
| 5-7 | ABS, EPS, IPC | 56 | Discovery done |

## Research Tools

This signal set was developed using a custom diagnostic platform:

- **[vehicle-diag](https://github.com/ffishman/vehicle-diag)** — Multi-vehicle OBD-II diagnostic library (Python)
- **424 PIDs** cataloged in YAML knowledge base with formulas, quality flags, and physical position mapping
- **35 health indicators** with threshold-based scoring (DPF, engine, transmission, electrical)
- **Soot predictor** — Linear model predicts km to next DPF regen
- Full trip data collection with adaptive tuning, quality tracking, and health scores

## License

See upstream [OBDb](https://github.com/OBDb) for licensing.
