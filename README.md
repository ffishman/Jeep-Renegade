# Jeep Renegade — Diesel Signalset (Fork)

Fork of [OBDb/Jeep-Renegade](https://github.com/OBDb/Jeep-Renegade) focused on the **2.0 MultiJet II diesel** variant (Europe/Brazil).

## Vehicle

- **Model:** 2019 Jeep Renegade Trailhawk (BU/520, facelift)
- **Engine:** 2.0 MultiJet II (VM Motori), 170 hp, Euro 5
- **Transmission:** ZF 9HP48 9-speed automatic
- **Market:** Brazil (Goiana plant, WMI 988)
- **CAN:** 29-bit 500 kbps (ISO 15765-4), protocol 7

## ECU Map

| Address | Header | Module | Manufacturer | Notes |
|:-------:|--------|--------|-------------|-------|
| 0x10 | DA10 | ECM | Bosch EDC17 (0281031204) | Extended session required (`din: "03"`), ~5s warmup |
| 0x18 | DA18 | TCM | ZF ES11-1029 | Instant response, no session needed |
| 0x40 | DA40 | BCM | Magneti Marelli BC521K | Battery voltage |
| — | DB33 | Broadcast | OBD-II Mode 01 | DPF temperatures (PID 7C) |

## Signalset Coverage (46 commands)

| Category | Signals | Examples |
|----------|:-------:|---------|
| Emissions / DPF | 19 | Soot level, regen count/progress/temps, diff pressure, flow resistance, NOx catalyst load, lambda |
| Engine | 18 | Swirl, VGT cmd/actual, rail pressure, boost, fuel corrections (4 cyl), oil degradation, temperatures |
| Transmission | 6 | Gear (ECU + TCM), torque, gearbox RPM, ATF temp |
| Movement | 4 | Individual wheel speeds (div=16) |
| Battery | 1 | BCM voltage |

All ECM signals require UDS Extended Diagnostic Session (service 10 03) before ReadDataByIdentifier (service 22).

## OBD Adapters Tested

| Adapter | Chip | Single-frame | Multiframe | Latency | Notes |
|---------|------|:------------:|:----------:|:-------:|-------|
| OBDLink MX+ | STN2255 v5.12.4 | OK | OK (STPX) | ~51ms | Recommended. BT Classic (COM3) |
| Vgate iCar Pro BLE | ELM327 clone | OK | FAIL | ~76ms | Clone does not support 29-bit flow control |

### Known Adapter Issues

- **DB33 broadcast resets ECM warmup.** After any Mode 01 query via DB33, the ECM at DA10 needs ~5 seconds to respond again. Physical addressing (DA10/DA18/DA40) does not cause this issue.
- **ATCRA must be set before `10 03`** on 29-bit CAN. Without the receive address filter, the DiagSessionControl response is missed.

## Formulas

FCA diesel ECUs use specific patterns for data encoding:

| Type | Formula | DIDs |
|------|---------|------|
| Temperature | `raw * 0.02 - 40` C | 1003, 18DE, 1900, 1935, 193F, 3808, 3915, 3917 |
| Signed offset | `(raw - 32768) / N` | 189B (N=100), 18E2 (N=10), 195A (N=10) |
| VGT position | `raw * 100 / 65535` % | 189F, 18A0 |
| Fuel corrections | `int16(raw) / 100` mg/str | 3800-3803 |
| Soot | `raw * 1000 / 65535` % | 18E4 |
| Wheel speed | `raw / 16` km/h | 2B1B-2B1E |
| Battery | `raw / 2000` V (ECU), `raw / 10` V (BCM) | 1955, 1004 |
| Rail pressure | `raw / 20` bar | 1946, 1947 |
| Distance (3-byte) | `(A*65536 + B*256 + C) / 10` km | 3807 |

## Signals Removed from Upstream

These signals exist in [OBDb/Jeep-Renegade](https://github.com/OBDb/Jeep-Renegade) but were removed from this fork with justification:

| DID | Signal | Reason |
|-----|--------|--------|
| 1000 | RPM (ECU) | Duplicates OBD-II PID 0C |
| 1003 | Coolant temp (ECU) | Duplicates OBD-II PID 05 |
| 1955 | Battery voltage (ECU) | Duplicates OBD-II PID 42 + BCM DA40:1004 |
| 1009 | Water in fuel distance | Mislabeled: runtime counter, not distance |
| 190B | Vehicle speed (ECU) | Always returns 0 on 2019 diesel |
| 3914 | Catalyst inlet temp | Returns NO DATA during normal driving |
| 2024 | Tire size | Static config, not telemetry |
| F158 | Model year | Static config, not telemetry |

## Gas vs Diesel

The Jeep Renegade exists in gasoline (US/Brazil) and diesel (Europe/Brazil) variants with different ECU hardware:

- **Gasoline** (2.4L Tigershark / 1.3L Turbo Flex): 11-bit CAN, ECM at 7E0, TCM at 7E1
- **Diesel** (2.0 MultiJet II): 29-bit CAN, ECM at DA10, TCM at DA18

This signalset is diesel-specific. Gasoline vehicles will not respond to DA10 commands. The OBDb pipeline aggregates both variants into the make-level repo ([OBDb/Jeep](https://github.com/OBDb/Jeep)) — incompatible commands fail silently with NO DATA.

## PRs to Upstream

| PR | Status | Description |
|----|:------:|-------------|
| [#106](https://github.com/OBDb/Jeep-Renegade/pull/106) | Merged | Initial diesel signalset (75 commands) |
| [#115](https://github.com/OBDb/Jeep-Renegade/pull/115) | Merged | Add `din: "03"` for Extended Diagnostic Session |
| [#116](https://github.com/OBDb/Jeep-Renegade/pull/116) | Closed | Wheel speed div=16 (superseded by #121) |
| [#121](https://github.com/OBDb/Jeep-Renegade/pull/121) | Open | Cleanup: fix formulas, remove duplicates (54 -> 46 cmds) |
