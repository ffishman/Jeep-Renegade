# Jeep Renegade — Diesel Signalset (Fork)

Fork of [OBDb/Jeep-Renegade](https://github.com/OBDb/Jeep-Renegade) focused on the **2.0 MultiJet II diesel** variant.

## Vehicle

- **Model:** 2019 Jeep Renegade Trailhawk (BU/520, facelift)
- **Engine:** 2.0 MultiJet II (VM Motori), 170 hp
- **Transmission:** ZF 9HP48 9-speed automatic
- **Market:** Brazil (Goiana plant, WMI 988)
- **CAN:** 29-bit 500 kbps (ISO 15765-4), protocol 7

## ECU Map

| Address | Header | Module | Manufacturer | Notes |
|:-------:|--------|--------|-------------|-------|
| 0x10 | DA10 | ECM | Bosch EDC17 (0281031204) | Extended session required (`din: "03"`), ~5s warmup |
| 0x18 | DA18 | TCM | ZF ES11-1029 | Instant response, no session needed |
| 0x40 | DA40 | BCM | Magneti Marelli BC521K | Battery voltage |
| -- | DB33 | Broadcast | OBD-II Mode 01 | DPF temperatures (PID 7C) |

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
| Temperature | `raw * 0.02 - 40` C | 18DE, 1900, 1935, 193F, 3808, 3915, 3917 |
| Signed offset | `(raw - 32768) / N` | 189B (N=100), 18E2 (N=10), 195A (N=10) |
| VGT position | `raw * 100 / 65535` % | 189F, 18A0 |
| Fuel corrections | `int16(raw) / 100` mg/str | 3800-3803 |
| Soot | `raw * 1000 / 65535` % | 18E4 |
| Lambda | `raw / 1000` | 18ED |
| M-Prop | `raw * 3 / 1000` % | 18D0 |
| Wheel speed | `raw / 16` km/h | 2B1B-2B1E |
| Battery (BCM) | `raw / 10` V | 1004 |
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

## Variants and Compatibility

This signalset was **tested and validated on a 2019 diesel 2.0 MultiJet II (Brazil)** only. Behavior on gasoline/flex variants has not been tested by the author.

### What we know

- All Renegade variants use **29-bit CAN (protocol 7)** with ECM at DA10 and TCM at DA18.
- OBDb test cases from other contributors show that MY2015 and MY2020 diesel vehicles also respond to DA10 DIDs (partially). MY2018, MY2021 and MY2024 have no DA10 test data.
- The OBDb pipeline aggregates all variants into the make-level repo ([OBDb/Jeep](https://github.com/OBDb/Jeep)). Unsupported commands fail silently (NRC or NO DATA).

### Engine options by market

| Market | Gasoline / Flex | Diesel |
|--------|----------------|--------|
| US | 2.4L Tigershark, 1.3L Turbo | Not available |
| Europe | 1.0L FireFly, 1.3L Turbo Multiair | 1.6L MultiJet II, 2.0L MultiJet II |
| Brazil | 1.8L E.torQ Flex, 1.3L Turbo Flex | 2.0L MultiJet II |

### Related vehicles

Other vehicles share the 2.0 MultiJet II + ZF 9HP48 powertrain and are expected to support the same diesel DIDs: Jeep Compass, Fiat Toro, Ram 1500 (Brazil/Europe markets).

## PRs to Upstream

| PR | Status | Description |
|----|:------:|-------------|
| [#106](https://github.com/OBDb/Jeep-Renegade/pull/106) | Merged | Initial diesel signalset (75 commands) |
| [#115](https://github.com/OBDb/Jeep-Renegade/pull/115) | Merged | Add `din: "03"` for Extended Diagnostic Session |
| [#116](https://github.com/OBDb/Jeep-Renegade/pull/116) | Closed | Wheel speed div=16 (superseded by #121) |
| [#121](https://github.com/OBDb/Jeep-Renegade/pull/121) | Open | Cleanup: fix formulas, remove duplicates (54 -> 46 cmds) |

## Sources

- **OBDb project:** [github.com/OBDb](https://github.com/OBDb) — schema, signalset format, CI/CD
- **OBDb schema (signals.json):** [github.com/OBDb/.schemas](https://github.com/OBDb/.schemas) — valid units, filterObject, formatter
- **Jeep Renegade specs:** [Wikipedia](https://en.wikipedia.org/wiki/Jeep_Renegade), [autoevolution](https://www.autoevolution.com/cars/jeep-renegade-2018.html)
- **2019 facelift press release:** [Stellantis Media](https://www.media.stellantis.com/em-en/jeep/press/new-2019-jeep-renegade)
- **ECU identification:** AlfaOBD diagnostic software (device_id 15463, variant MARELLI8F3_CAN)
- **Formula validation:** Live CAN data collected via OBDLink MX+ (STN2255), cross-checked against AlfaOBD and MultiEcuScan parameter databases
- **Adapter behavior (warmup, ATCRA):** Empirical testing documented in [Clutch Discord](https://discord.gg/clutch) contributions
