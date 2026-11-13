# Bexie BE10KW-1PH: Modbus register reference

Last updated November 2026.

**Source map:** the Bexie follows the generic "APD" hybrid protocol, [Modbus Protocol V1.2](https://sunvec.es/wp-content/uploads/2023/05/Modbus-Protocol-V1.2.pdf). That document is the base reference. This page records what was confirmed on a live unit and where the Bexie differs.

**How to read this page**

- All registers are holding registers. "Dec" is the decimal address of the first register.
- "Observed" values are readings from one BE10KW-1PH in November 2026. Yours will differ.
- Anything not in the generic document is marked *undocumented*.
- U32 and S32 values use two registers.
- Reading 0x1000 itself returns "illegal address". Start at 0x1010.
- **Never read or publish 0x3060 (Wi-Fi SSID) or 0x3070 (Wi-Fi password).**

## Device information (0x1A00 block, read-only)

| Reg | Dec | Meaning | Observed |
|---|---|---|---|
| 0x1A00 (8 regs) | 6656 | Model name, ASCII | "BE10KW-1PH" |
| 0x1A10 (8) | 6672 | Serial number, ASCII | (not published) |
| 0x1A18 | 6680 | *undocumented* | 1049 (0x0419) |
| 0x1A19-0x1A2F | 6681 | Master and slave firmware strings, ASCII. Field boundaries are not certain. | See firmware notes below |
| 0x1A30-0x1A3A | 6704 | *undocumented*, look like PV/DC ratings | 6000, 3600, 700, 400, 700, 5800, 700, 5400, 2000, 3800, 990 |
| 0x1A3B | 6715 | MPPT number | 3 |
| 0x1A3D-0x1A43 | 6717 | *undocumented* | 15000, 3000, 2200, 2200, 10000, 6600, 6600 |
| 0x1A44 | 6724 | Nominal voltage, 0.1 V | 2300 |
| 0x1A45 | 6725 | Nominal frequency, 0.01 Hz | 5000 |
| 0x1A46 / 0x1A4E | 6726 / 6734 | Nominal active power, W (low / high word) | 10000 / 0 |
| 0x1A47 | 6727 | *undocumented*, possibly max apparent power | 11000 |
| 0x1A48 | 6728 | Grid phase number | 1 |
| 0x1A49-0x1A4D | 6729 | *undocumented* | 4170, 4770, 800, 988, 980 |
| 0x1A58 | 6744 | *undocumented* | 652 |
| 0x1A5A | 6746 | Production type. Generic doc: 1 = AC couple without smart load, 4 = AC couple with smart load. | 3. This is not where the AC coupled mode setting is stored. |
| 0x1A60 (10) | 6752 | EMS firmware, ASCII | "01120010004-06260119" |

### Firmware strings

The 0x1A19-0x1A2F block reads as two strings: master, then slave/DCDC.

| When | Master | Slave/DCDC |
|---|---|---|
| Before 12 Nov 2026 | 01100110000-05260123 | 01120010001-06260119 |
| After remote updates on 12 Nov 2026 | 01120110000-06260917 | 01140110001-07260917 |

The inverter dropped offline for a few minutes around each update, and the work mode was rewritten after the master update. Polling this block is a simple way to know when the manufacturer has pushed new firmware.

## Real-time data (read-only)

| Reg | Dec | Meaning |
|---|---|---|
| 0x1010-0x1013 | 4112 | MPPT1 V (0.1 V), A (0.01 A), W (U32, 0.1 W) |
| 0x1014-0x1017 | 4116 | MPPT2, same layout |
| 0x1018-0x101B | 4120 | MPPT3, same layout |
| 0x101C | 4124 | Inner temperature, C |
| 0x101D | 4125 | Inverter mode: 0 Initial, 1 Standby, 3 On-grid, 4 Off-grid, 5 Fault, 9 Shutdown |
| 0x101E / 0x101F / 0x1020 | 4126-4128 | Error code 1 (inverter) / 2 (grid and PV) / 3 (logger, meter, parallel). Bit tables are in the appendix of the generic document. |
| 0x1021 (U32) | 4129 | Total PV energy, kWh |
| 0x1023 (U32) | 4131 | Total generation time, h |
| 0x1027 (U32) | 4135 | Today PV energy, Wh |
| 0x1037 (S32) | 4151 | Grid total active power, 0.1 W |
| 0x1048 (U32) | 4168 | PV total input power, 0.1 W |
| 0x1300 / 0x1302 / 0x1304 (S32) | 4864 | Grid power per phase, 0.1 W (positive = import) |
| 0x1306 / 0x1308 (U32) | 4870 | Total import / export, 10 Wh |
| 0x130A (U32) | 4874 | Load power L1, 0.1 W |
| 0x1310 (U32) | 4880 | Total load energy, 10 Wh |
| 0x131A | 4890 | Grid voltage L1, 0.1 V |
| 0x131D (S32) | 4893 | Grid current L1, 0.01 A |
| 0x1323 | 4899 | Load voltage L1 |
| 0x1326 (U32) | 4902 | Load current L1 |
| 0x1332 / 0x1334 / 0x1336 (U32) | 4914 | Today import / export / load, 10 Wh |
| 0x1338 | 4920 | Grid frequency, 0.01 Hz |
| 0x1350-0x1363 | 4944 | Backup port V / A / W / Hz and energy |

## Battery (0x2000 block, read-only)

| Reg | Dec | Meaning | Observed |
|---|---|---|---|
| 0x2000 | 8192 | SOC % | Matches the Bexie app |
| 0x2001 | 8193 | "Battery temperature" in the generic doc. **Not usable on this model.** | Read 0, then 65506 (-30) after a firmware update, while the app showed 6 to 8 C |
| 0x2002 | 8194 | *undocumented*, likely BMS status | 192 or 193. The app's "BMS Status" shows 192. |
| 0x2006 | 8198 | Battery voltage, 0.1 V | Matches the app |
| 0x2007 (S32) | 8199 | Battery current, 0.01 A | 1000 = 10 A, matches the app |
| 0x2009 (S32) | 8201 | Battery power, 0.1 W | Matches the app |
| 0x200B (U32) | 8203 | Today charge, 10 Wh | |
| 0x200D (U32) | 8205 | Total charge, 10 Wh | Matches the app |
| 0x200F (U32) | 8207 | Today discharge, 10 Wh | |
| 0x2011 (U32) | 8209 | Total discharge, 10 Wh | |
| 0x2013 | 8211 | Error message 4: battery and backup fault bits (below) | 0 |
| 0x2016 | 8214 | *undocumented*, possibly SOH % | 100 |
| 0x201B | 8219 | *undocumented* | 4 to 9, varies |

**0x2013 bits:**

| Bit | Meaning | Bit | Meaning |
|---|---|---|---|
| 0 | Battery absent | 8 | Inverter-BMS communication error |
| 1 | Over voltage | 9 | Internal communication E-M |
| 2 | Under voltage | 10 | Internal communication M-D |
| 3 | Discharge over current | 11 | DCDC abnormal |
| 4 | Over temperature | 12 | Backup DC bias |
| 5 | Under temperature | 13 | Backup short circuit |
| 6 | Neutral/live reversed | 14 | Backup overload |
| 7 | Backup voltage abnormal | | |

## Hybrid settings (0x2100 block, read/write)

| Reg | Dec | Meaning | Observed |
|---|---|---|---|
| 0x2100 | 8448 | Work mode: 0 Self used, 1 Feed-in, 3 Back-up | Written by automation |
| 0x2101-0x210F | 8449 | 3 time slots: once/everyday, charge start/end, discharge start/end (hour in high byte, minute in low byte) | All 0 |
| 0x2110 | 8464 | Battery type (generic doc: 0-5, 1 lead-acid, 2-5 lithium brands) | **31**, Bexie's own lithium type |
| 0x2111 | 8465 | BMS RS485 address | 1 |
| 0x2112 | 8466 | Lead-acid Ah | 260 |
| 0x2113 | 8467 | Lead-acid stop discharge voltage, 0.1 V | 460 |
| 0x2114 | 8468 | Lead-acid stop charge voltage, 0.1 V | 532. **This acts as the charge ceiling when capacity mode (0x2124) is 1.** |
| 0x2115 | 8469 | Grid charge enable | Written by automation |
| 0x2116 | 8470 | Max grid charge power, W | 10000 |
| 0x2117 | 8471 | Grid charge end SOC % | Written by automation |
| 0x2118 | 8472 | Max charge power (all sources), W | 10000 |
| 0x2119 | 8473 | Charge end SOC % | 100 |
| 0x211A | 8474 | Max discharge power, W | 10000 |
| 0x211B | 8475 | Discharge end SOC % (floor) | 15 |
| 0x211C-0x211F | 8476 | Off-grid enable, output voltage, frequency, start-up SOC | 1, 2300, 5000, 25 |
| 0x2120 / 0x2121 | 8480 / 8481 | Max discharge / charge current, 0.01 A | 21000 / 21000 |
| 0x2122 | 8482 | GEN port | 0 |
| 0x2123 | 8483 | Lithium activation | 0 |
| 0x2124 | 8484 | **Capacity mode: 0 = SOC %, 1 = battery voltage** | 0. Switching the battery type to lead-acid and back to lithium leaves this at 1, so check it afterwards. |
| 0x2125 | 8485 | Max input power from grid, W | 10000 |
| 0x2126-0x212A | 8486 | Generator settings | 10000, 10000, 0, 80, 100 |
| 0x212B-0x212D | 8491 | Smart load: min PV W, on SOC, off SOC | 500, 100, 80 |
| 0x212E | 8494 | Always on with grid | 0 |
| 0x212F | 8495 | Off-grid start-up battery voltage, 0.1 V | 400 |
| 0x2130 / 0x2131 | 8496 | Smart load on / off voltage | 600 / 400 |
| 0x2132 / 0x2133 | 8498 | Backup min / max output voltage, 0.1 V | 1760 / 2640 |
| 0x2134 / 0x2135 | 8500 | Generator start / end battery voltage | 480 / 640 |
| 0x2138 / 0x2139 | 8504 | Battery SOC of inverter on / off | 80 / 100 |
| 0x213A / 0x213B | 8506 | Battery voltage of inverter on / off | 400 / 640 |
| 0x213C | 8508 | AC couple frequency high, 0.01 Hz | 5200 |
| 0x2141 | 8513 | Support normal load | 1 |
| 0x2142 | 8514 | *undocumented* | 1 |
| 0x2143 | 8515 | Parallel mode | 0 |
| 0x2144 / 0x2145 | 8516 | Force charge start / end SOC | 10 / 15 |
| 0x2146 / 0x2147 | 8518 | Force charge start / end voltage, 0.1 V | 400 / 420 |
| 0x2148 | 8520 | Battery voltage of grid charge end, 0.1 V | 640 |
| 0x2149 | 8521 | Feed-in grid | 1 |
| 0x214A / 0x214B / 0x214F | 8522 | *undocumented* | 5 / 400 / 5 |
| 0x2150 | 8528 | Max grid forced charge power, W | 400 |
| 0x2168-0x217F | 8552 | Time slot 1-3: charge W (2 regs), charge end SOC, charge end V, discharge W, discharge end SOC, discharge end V | Each 10000 W / 100% / 64.0 V / 10000 W / 10% / 40.0 V |

Registers written successfully on this system: 0x2100, 0x2115, 0x2117, 0x2118, 0x211B, 0x211F and 0x2124. Each was read back to confirm.

## Time, communication and meter (0x3000 block)

| Reg | Dec | Meaning | Observed |
|---|---|---|---|
| 0x3000-0x3003 | 12288 | Inverter clock: year, month/day, hour/minute, second | Local time |
| 0x3005 | 12293 | Power derating % by Modbus | 0 |
| 0x303E | 12350 | Modbus address | Not read |
| 0x304C | 12364 | RS485 baud rate | Not read |
| 0x3060 / 0x3070 | | Wi-Fi SSID / password | **Do not read or publish** |
| 0x30B0 | 12464 | Meter Modbus address | 1 |
| 0x30B1 | 12465 | Meter type (generic doc lists 0-10) | 15, Bexie-specific (system has an Eastron SDM120CT) |
| 0x30B2 | 12466 | Power flow direction | 0 |
| 0x30B3 | 12467 | Power limit function: 0 off, 1 external, 2 PVI CT, 3 digital meter | 3 |
| 0x30B4 | 12468 | CT ratio | 1 |
| 0x30B5 | 12469 | Meter location: 0 grid, 1 load | 0 |
| 0x30B6-0x30B8 | 12470 | *undocumented* | 1, 1, 1 |
| 0x30B9 (U32) | 12473 | Max feed-in (export) power, W | 10000 |

## Grid code and control

| Reg | Dec | Meaning | Observed |
|---|---|---|---|
| 0x5000-0x5084 | 20480 | Grid protection trip limits and times, volt-watt, Q(U), FRT | Not read. **Never write.** |
| 0x5101 | 20737 | Regulation code (generic doc table 0-72) | 24 |
| 0x5104 | 20740 | Derating watt %, of nominal power | 110 |
| 0x6001 | 24577 | Inverter control: 0 on, 1 shut down | 0. **Never write.** |

## Open questions

- What 0x211F (off-grid start-up SOC) does when a Gecko gateway is fitted: no backup below that level, or the restart level after a flat battery.
- Where the off-grid discharge cut-off is set.
- Where the AC coupled mode setting is stored.
- The meaning of the undocumented registers above.

If you know any of these, please open an issue.
