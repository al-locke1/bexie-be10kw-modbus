# Bexie BE10KW-1PH: Modbus notes for Home Assistant

Working notes from running a Bexie BE10KW-1PH hybrid inverter with Bexie low-voltage batteries under Home Assistant. Bexie publish very little about the Modbus interface, so this repo collects what has been found and tested on a real system.

**Unofficial.** This is not affiliated with or endorsed by Bexie. Everything here comes from one installation and may not match yours.

## What is here

| File | What it covers |
|---|---|
| [docs/registers.md](docs/registers.md) | Register reference: device info, live data, battery, hybrid settings, meter, grid code. Includes values observed on a live unit and registers that are not in the generic protocol document. |
| [docs/home-assistant.md](docs/home-assistant.md) | How the inverter is connected to Home Assistant, how to read and write registers, and what is worth monitoring. |
| [home-assistant/packages/bexie_monitor.yaml](home-assistant/packages/bexie_monitor.yaml) | Read-only Home Assistant package that checks the inverter's settings, firmware and faults every 15 minutes and alerts on any change. |
| [home-assistant/packages/predbat_bexie.yaml](home-assistant/packages/predbat_bexie.yaml) | Home Assistant package that lets Predbat control the Bexie through the three profile controls, with safety nets. |

## Key findings

- The BE10KW-1PH follows the generic "APD" hybrid inverter Modbus protocol (V1.2), with some Bexie-specific values and several registers the generic document does not describe.
- It works with the [ha-solarman](https://github.com/davidrapan/ha-solarman) integration using the community Bexie profile from [netmindz/ha-solarman, branch `bexie_be`](https://github.com/netmindz/ha-solarman/tree/bexie_be). That profile was written for the BE6KW-1PH, so the 10 kW model does not autodetect and the profile has to be selected by hand.
- Register 0x2124 (capacity mode) decides whether the inverter manages the battery by state of charge or by voltage. In voltage mode the charge stops at the lead-acid stop charge voltage in 0x2114, which is far too low for a lithium battery. If the battery type is ever switched to lead-acid and back to lithium, check that capacity mode has returned to 0.
- Register 0x2001 is documented as battery temperature but is not usable on this model.
- Firmware updates pushed remotely by the manufacturer can be seen by watching the firmware strings at 0x1A19 to 0x1A2F.

## Warnings

- **Writing registers is at your own risk.** A wrong value can stop the system working, damage a battery or affect your warranty. Read first, change one thing at a time, and read it back.
- **Never write the grid protection block (0x5000 to 0x5084) or the regulation code.** These are your grid connection settings (G98/G99 in the UK) and are your installer's responsibility.
- **Never write 0x6001** (inverter shut down) unless you know exactly why.
- **Do not publish 0x3060 or 0x3070.** They hold the Wi-Fi name and password.
- Settings are stored in non-volatile memory, so avoid automations that rewrite the same setting many times a day.
- Tell your installer or Bexie about any setting you change. They can also change settings remotely, so monitor for changes.

## System these notes come from

- Bexie BE10KW-1PH single-phase hybrid inverter, UK grid
- Bexie BXB 5KLV-PRO low-voltage lithium batteries (20 kWh total)
- Bexie Gecko gateway as the whole-house changeover switch
- Eastron SDM120CT meter
- Home Assistant with ha-solarman over Modbus TCP, and Predbat for charge planning

## Contributing

Corrections and readings from other Bexie models are welcome. Open an issue, or send a pull request, with the model, firmware strings and what you observed.

## Licence

MIT. See [LICENSE](LICENSE).
