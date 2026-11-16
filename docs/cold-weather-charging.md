# Cold-weather charging: what the battery accepts

Last updated 16 November 2026.

**These are observations from one system** (a BE10KW-1PH with four BXB 5KLV-PRO packs, 400 Ah in total, installed outdoors in the UK). They are not a manufacturer table, and Bexie have not confirmed the figures.

## Short version

Check the battery temperature before treating a low charge rate as a fault. On this system the battery's own management system (BMS) limits charge current below about 10 C, and near freezing it accepts roughly a fifth of the full rate. No inverter setting changes this.

## Observed charge current against battery temperature

| Battery temperature | Charge current seen | Notes |
|---|---|---|
| 1 to 3 C | 35 to 42 A (1.9 to 2.3 kW) | Overnight grid charge. Under-temperature bit set (0x2013 bit 5). |
| 5 to 8 C | up to 59 A | Midday, from solar. May have been limited by the solar available and not by the BMS. |
| 8 to 10 C | about 130 A (7 kW) | First 30 minutes of an overnight grid charge. |
| 10 C and above | 185 A (10.1 kW) | The inverter's maximum, sustained for over an hour. |

Throughout these readings the inverter settings were the same: max charge current 210 A (0x2121), max charge power 10 kW (0x2118), capacity mode 0 (0x2124), no error codes.

## Three nights compared

| Night | State of charge | Peak rate | Battery temperature |
|---|---|---|---|
| 14 Nov 2026 | 67% to 74% | 42 A, 2.3 kW | 1 to 3 C |
| 15 Nov 2026 | 47% to 81% in 75 minutes | 185 A, 10.1 kW | 8 to 13 C |
| 16 Nov 2026 | 15% to 98% in about 2 hours 10 minutes | 185 A, 10.1 kW | 10 to 19 C |

Charging at 10 kW warms the battery by itself: it rose from 10 C to 19 C in about two hours on 16 November.

## How to see it over Modbus

| What | Register | Notes |
|---|---|---|
| Battery temperature | 0x201B (decimal 8219), likely | Matches the battery temperature shown in Home Assistant. Not in the generic protocol document. |
| Not battery temperature | 0x2001 (decimal 8193) | The generic document calls this battery temperature. On this model it is not. |
| Under-temperature state | 0x2013 (decimal 8211), bit 5 | Set at 1 C and cleared as the battery warmed. This is a protection state, not a hardware fault. |
| Charge current | 0x2007 (decimal 8199) | S32, 0.01 A |

See [registers.md](registers.md) for the full map.

## What the published specification says

From the BXB 5KLV-PRO datasheet and user manual:

- Charge temperature range: -5 C to 55 C. Discharge: -15 C to 55 C.
- Recommended operating temperature: 10 C to 30 C.
- Maximum charge current: 0.6C, 60 A per pack (240 A for four).
- IP65.
- Heating film in pack: yes. The user manual gives a heating power of 230.4 W.

The published documents give no charge current limits by temperature.

## Battery heating: not yet understood

- Neither the datasheet nor the manual says at what temperature the heating film switches on, what powers it, or whether it needs enabling.
- No heating setting has been found in the registers.
- On 14 November the battery sat at 1 C for over two hours, including short charges, then warmed to 4 C during a 35 A charge. That may have been the heater or just the charge current. It is unconfirmed.

If you know how the heating is triggered, please open an issue.

## Practical points

- **Alerts:** a "slow charging" alert based on a fixed power threshold will fire on cold nights. Take battery temperature into account, or expect it.
- **Fault alerts:** the under-temperature bit will raise any alert that treats 0x2013 as a fault. Consider handling bit 5 separately.
- **Charge planning:** a planner that assumes the full charge rate will fall short in a short cheap-rate window when the battery is near freezing. On 14 November this system could add only about 2 kW.
- **Fault-finding:** compare like with like. A low rate on a cold night and a full rate on a mild night can come from the same healthy system. The [case study](slow-charging-case-study.md) shows how this muddied a real fault.

## Sources

- [BXB 5KLV-PRO datasheet](https://infinity-renewables.com/wp-content/uploads/2025/10/Bexie-Pro-5kW-Battery-1.pdf)
- [Bexie low-voltage battery datasheet](https://bexieenergy.com/wp-content/uploads/2026/05/Low-voltage-Battery.pdf)
- [BXB 5KLV-PRO user manual](https://bexieenergy.com/wp-content/uploads/2026/05/User-Manual-BXB_5KLV-PRO.pdf)
