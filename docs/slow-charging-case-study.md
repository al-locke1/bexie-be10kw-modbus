# Case study: battery charging collapsed to a trickle

**System:** Bexie BE10KW-1PH with 20 kWh of Bexie BXB 5KLV-PRO low-voltage lithium batteries.
**When:** 10 to 16 November 2026.
**Status:** resolved. All three causes fixed, and the full 185 A charge rate confirmed on 15 and 16 November. Low rates on cold nights are the battery's own temperature limit.

## Short version

What looked like one fault was three separate ones, overlapping in time:

1. **The inverter was in AC coupled mode.** This caused the original slow charging. Bexie found and corrected it remotely.
2. **Capacity mode was left on "battery voltage" after a test.** While fault-finding, Bexie switched the battery type to lead-acid and then back to lithium. Switching back did not return capacity mode (register 0x2124) to SOC, so the inverter stopped charging at the lead-acid stop charge voltage (0x2114, 53.2 V). Writing 0x2124 = 0 restored charging within about 90 seconds.
3. **A firmware bug limited the charge current.** The firmware installed on 12 November was wrongly limiting battery charge current. This was confirmed by the manufacturer and fixed by a firmware update on 13 November.

A fourth effect confused the picture without being a fault: **the battery limits its own charge current when cold.** The first night after the fixes was near freezing and the battery charged at only about 40 A. On the next two, milder nights it charged at the full 185 A.

The second and third faults were introduced while the first was being investigated, which is why the symptoms looked like one continuous problem that kept getting worse.

## Timeline

| Date | What happened |
|---|---|
| Up to 9 Nov | Normal. Battery charged at up to 185 A (about 10 kW), daily maximum 56.8 V. |
| 10 Nov | Overnight charge current fell to 32 A, then on following charges 12 A, about 9 A, then 0 to 4 A. Daily maximum voltage dropped to 53.9 V. No fault codes. |
| 11 Nov | Bexie found the system in AC coupled mode and corrected it remotely. They also changed the meter setting. |
| 12 Nov | Bexie ran a lead-acid test: battery type set to lead-acid to check charging without BMS communication, then set back to lithium. They also pushed master and slave firmware updates. |
| 13 Nov, 00:30 to 01:31 | Overnight grid charge stalled at 29% state of charge with about 35 W going in, holding at 53.1 to 53.2 V. |
| 13 Nov, 01:32 | 0x2124 written from 1 to 0. Charging resumed within about 90 seconds, but only at about 41 A. |
| 13 Nov, 10:12 to 10:23 | Bexie pushed an EMS firmware update to fix the charge current bug, and updated the datalogger firmware to address connection dropouts. |
| 13 Nov, 11:30 | Inverter online, no errors, all settings as expected, battery at 100%. |
| 14 Nov, 00:30 to 05:30 | First overnight charge on the fixed firmware. Battery charged from 67% to 74% at 35 to 42 A (1.9 to 2.3 kW) with the battery at 1 to 3 C. The BMS set its under-temperature bit (0x2013 bit 5) at 01:00. No settings changed. |
| 15 Nov | Overnight charge from 47% to 81% in 75 minutes. About 130 A (7 kW) for the first 30 minutes at 8 to 10 C, then 185 A (10.1 kW). |
| 16 Nov | Overnight charge from 15% to 98% in about 2 hours 10 minutes. 185 A sustained for over an hour, battery at 10 to 19 C. Fault closed. |

## Fault 1: AC coupled mode

- **Symptom:** charge current falling day by day from 10 November, with no fault codes. The battery fault bits (0x2013) and error codes (0x101E to 0x1020) were all zero.
- **Cause:** the inverter was in AC coupled mode. This was a configuration issue on the manufacturer's side, since corrected.
- **Fix:** corrected remotely by Bexie on 11 November.
- **Not yet known:** which register holds this setting. It is not production type (0x1A5A), which read 3 before and after. This could not be diagnosed or fixed from the Modbus side.

## Fault 2: capacity mode left on voltage

- **Symptom:** grid charging stalled at a low state of charge, with the battery sitting at exactly 53.1 to 53.2 V.
- **How it was found:** reading the whole 0x2100 settings block showed two values side by side:
  - 0x2124, capacity mode = 1 (battery voltage, instead of 0 = SOC %)
  - 0x2114, lead-acid stop charge voltage = 53.2 V
- **Cause:** the lead-acid test put the inverter into voltage-based battery management. Setting the battery type back to lithium (0x2110 = 31) did not switch capacity mode back to SOC. The inverter then treated 53.2 V as full, which for this lithium pack is nowhere near full.
- **Fix:** write 0x2124 (decimal 8484) = 0, then read it back.
  - Before: stalled at 29%, 35 W, 53.1 to 53.2 V.
  - About 90 seconds after: 2.24 kW / 41 A, with voltage rising past the old ceiling.
  - To undo, write 8484 = 1.
- Bexie were told about the change and accepted it. They said nothing else from the test needed setting back by hand.

## Fault 3: firmware limiting the charge current

- **Symptom:** after fault 2 was fixed, the battery charged at only about 41 A, against 185 A before the faults. The BMS was reporting a charge limit of 38.4 A.
- **First reading:** the battery was at about 4 C, so this looked like a normal cold-temperature limit and not a fault.
- **Cause:** the manufacturer then confirmed that the firmware installed on 12 November was wrongly limiting the battery charge current.
- **Fix:** EMS firmware update pushed by Bexie on 13 November.

| Firmware | 9 Nov to 12 Nov | After 12 Nov updates | After 13 Nov fix |
|---|---|---|---|
| Master | 01100110000-05260123 | 01120110000-06260917 | unchanged |
| Slave/DCDC | 01120010001-06260119 | 01140110001-07260917 | unchanged |
| EMS (0x1A60) | 01120010004-06260119 | unchanged | 01140310004-07261008 |

- **Other changes seen after the fix:** 0x2001 reads 0 again (it had read -30 since 12 November). 0x2002 read 65 once straight after the update, then went back to its usual 192.

### First overnight charge after the fix (14 November)

The battery was already full when the fix went in, so the first test was the overnight charge on 14 November. Predbat asked for a small top-up in two steps.

| Time | What happened | Current | Power | Battery temperature |
|---|---|---|---|---|
| 00:30 to 00:37 | Charge starts | about 42 A | 2.2 kW | 1 C |
| 00:55 to 01:13 | Charge to 69% | about 42 A | 2.3 kW | 1 C |
| 01:15 to 03:00 | Held at 69% while the car charged | trickle | 0.13 kW | 1 C |
| 03:00 to 03:47 | Charge to 74% | steady 35 A | 1.9 kW at 54.0 V | 1 to 3 C |
| 03:50 to 05:40 | Held at 74% | trickle | 0.15 kW | 3 C |

- **Result:** peak 42 A (2.3 kW), 7% gained. The rate was no higher than the 41 A seen under the faulty firmware.
- **Not limited by the inverter settings:** max charge current 210 A, max charge power 10 kW (5 kW while the car charges, still more than double what the battery took), capacity mode 0, firmware on the fixed version, no fault bits or error codes.
- **Conditions were harder than the night before:** the battery was at 1 to 3 C, against about 4 C on 13 November, and it was only asked for a short top-up.
- **Verdict at the time: inconclusive.** The result fitted a cold-temperature charge limit, but did not show that the firmware fix had changed anything.

### Confirmed on the next two nights (15 and 16 November)

| Night | State of charge | Peak rate | Battery temperature |
|---|---|---|---|
| 14 Nov | 67% to 74% | 42 A, 2.3 kW | 1 to 3 C |
| 15 Nov | 47% to 81% in 75 minutes | 185 A, 10.1 kW | 8 to 13 C |
| 16 Nov | 15% to 98% in about 2 hours 10 minutes | 185 A, 10.1 kW | 10 to 19 C |

- 185 A (10 kW) is the inverter's maximum and the same rate as before 10 November.
- No settings or firmware changed between 13 and 16 November, checked every 15 minutes by the monitor. The only thing that changed was the battery temperature.
- So the 14 November result was the cold, not a remaining fault. The 41 A seen on 13 November, with the battery at about 4 C, was probably at least partly the cold as well. The manufacturer says the 12 November firmware also limited charge current. The data cannot separate the two, and it no longer matters.

## Not a fault: the cold-weather charge limit

Observed on this system. These are not manufacturer figures.

| Battery temperature | Charge current seen |
|---|---|
| 1 to 3 C | 35 to 42 A, under-temperature bit set |
| 5 to 8 C | up to 59 A from solar (may have been limited by the solar available) |
| 8 to 10 C | about 130 A |
| 10 C and above | 185 A, the inverter's maximum |

- Charging at 10 kW warms the battery by itself: 10 C to 19 C in about two hours.
- The battery has a heating film (230.4 W per the user manual), but the published documents do not say when it switches on, what powers it or whether it needs enabling. On 14 November the battery sat at 1 C for over two hours. Whether the heater ran is unconfirmed.
- Battery temperature can be read at 0x201B. 0x2001, which the generic protocol document calls battery temperature, is not valid on this model.

## Still open

- How the heating film is triggered, and the BMS charge limit between 0 and 10 C. To ask Bexie.
- Which register holds the AC coupled mode setting.

## What was ruled out along the way

| Setting | Register | Value | Verdict |
|---|---|---|---|
| Max charge current | 0x2121 | 210 A | Not limiting |
| Max charge power | 0x2118 | 10 kW | Not limiting |
| Max grid charge power | 0x2116 | 10 kW | Not limiting |
| Production type | 0x1A5A | 3, unchanged | Not the cause |
| Battery fault bits | 0x2013 | 0 | No faults |

## Lessons

1. **A fault can change shape during fault-finding.** Test settings that are not fully reversed, and firmware pushed as a fix, can each add a new problem that looks like the old one. After any remote work, compare every setting and firmware version with a known-good record.
2. **Changing battery type does not restore everything.** If the battery type is ever switched to lead-acid and back, check that capacity mode (0x2124) has returned to 0.
3. **Track firmware versions.** The firmware strings at 0x1A19 to 0x1A2F and 0x1A60 show exactly when an update lands. Without them, the 12 November update would not have been tied to the low charge rate.
4. **A plausible explanation is not a confirmed one.** The 41 A rate was first put down to a cold battery, then to a firmware bug, and the first test after the fix could not separate the two. It took two milder nights to close it. Keep a finding open until it has been tested under conditions that can tell the causes apart.
5. **Check battery temperature before treating a low charge rate as a fault.** Below about 10 C the BMS limits charge current, and near freezing it drops to roughly a fifth of the full rate.
6. **Read the whole settings block, not just the registers you control.** The second cause was in a register that no integration profile exposes.
7. **Match symptoms to numbers.** A charge that stalls at a specific voltage points to a voltage setting. Search the settings for that value.
8. **Compare against fixed known-good values, not only the previous reading.** The 13 November update was missed by the "changed since last read" alert because Home Assistant restarted during the outage, which reset the comparison. Capacity mode and the firmware versions are now in the known-good list, so a change is caught even across a restart.
9. **Not everything is visible over Modbus.** The first and third faults needed the manufacturer. Share your readings with them: the register evidence made the second fault quick to confirm.
10. **Leave the lead-acid values alone.** The fix was the mode, not the voltage. Do not change 0x2114 or other battery settings without the manufacturer's confirmation.
