# Using the Bexie BE10KW-1PH with Home Assistant

## Connection

- The inverter's logger speaks Modbus TCP on port 502 on the local network.
- Integration: [davidrapan/ha-solarman](https://github.com/davidrapan/ha-solarman), installed through HACS.
- Inverter profile: `bexie_hybrid.yaml` from [netmindz/ha-solarman, branch `bexie_be`](https://github.com/netmindz/ha-solarman/tree/bexie_be). Credit for the profile goes to its author.

### Installing the profile

1. Install ha-solarman through HACS as normal.
2. Copy `bexie_hybrid.yaml` into `custom_components/solarman/inverter_definitions/custom/`. Files in the `custom` folder survive integration updates. Files placed alongside the shipped profiles do not.
3. When adding the inverter, select the profile by hand. The profile autodetects the BE6KW-1PH model string only, so a BE10KW-1PH is not recognised automatically.

### Things to know about the profile on a 10 kW unit

- Use a copy from after the 27 August 2026 fix for battery power overflowing above 3276.7 W. A 10 kW inverter exceeds that figure routinely, and older copies report high-power discharge as charging.
- The profile exposes three controls: work mode (0x2100), grid charge (0x2115) and grid charge end SOC (0x2117). That is enough for overnight charging to a target.
- Do not use the battery temperature at 0x2001. It is not valid on this model. 0x201B looks like the real one.
- PV total input power (0x1048) should be read as signed, or clamped. Read as unsigned it produces spikes of about 429 MW at dawn and dusk. See [registers.md](registers.md).

## Reading registers directly

ha-solarman provides an action for reading any holding registers. Addresses are decimal.

```yaml
action: solarman.read_holding_registers
data:
  device: "{{ device_id('sensor.inverter_battery_soc') }}"
  address: 8448   # 0x2100, start of the hybrid settings block
  count: 40
response_variable: result
```

Replace `sensor.inverter_battery_soc` with any entity that belongs to your inverter. Read in blocks of 40 or fewer registers.

## Writing a register

```yaml
action: solarman.write_single_register
data:
  device: "{{ device_id('sensor.inverter_battery_soc') }}"
  address: 8484   # 0x2124 capacity mode
  value: 0        # 0 = SOC, 1 = battery voltage
```

Always read the register back afterwards. See the warnings in the [README](../README.md) before writing anything.

## Example: limit battery charging while a car charges

To keep the total grid draw within the supply limit when the battery and an EV charge at the same time overnight, an automation writes 0x2118 (max charge power, decimal 8472):

- 5000 W while the car is charging and grid charge is on
- 10000 W otherwise

Write only when the value needs to change, because settings are stored in non-volatile memory.

## What this setup writes

Keeping the list of written registers short makes faults much easier to reason about.

- **Written by automations:** work mode (0x2100), grid charge enable (0x2115), grid charge end SOC (0x2117) and max charge power (0x2118). Nothing else.
- **Written by hand, once each, and read back:** discharge end SOC (0x211B), off-grid start-up SOC (0x211F) and capacity mode (0x2124).
- **Never written:** the grid code block (0x5000 onwards), inverter on/off (0x6001), the lead-acid voltages (0x2113, 0x2114) and other battery settings.
- **Never read or published:** 0x3060 and 0x3070, the Wi-Fi name and password.

## If the inverter goes offline while a charge is forced

When the local Modbus link drops, Home Assistant cannot send the command that ends a forced charge. On this system the inverter then stayed in Back-up mode with grid charge on, and kept charging and running the house from the grid after the cheap rate ended.

Any automation that forces a charge mode should alert if the inverter goes offline while forced, and again if it is still offline when the cheap rate ends. [bexie_monitor.yaml](../home-assistant/packages/bexie_monitor.yaml) does this.

The dropouts seen here lined up with remote sessions and firmware updates by the manufacturer. None has been noticed since a datalogger firmware update on 13 November 2026.

## What is worth monitoring

A read-only check every 15 minutes covers most problems seen so far. The working package is [bexie_monitor.yaml](../home-assistant/packages/bexie_monitor.yaml). Read the notes at the top of the file before using it.

**Blocks to read**

| Block | Why |
|---|---|
| 0x2100 to 0x2150 | Every hybrid setting, including capacity mode |
| 0x2000 to 0x201B | Battery state and fault bits |
| 0x1A19 to 0x1A62 | Firmware strings and production type |
| 0x30B0 to 0x30BA | Meter and export limit settings |
| 0x101D to 0x1020 | Inverter mode and error codes |

**Alerts that have proved useful**

- Any setting that changes between reads, including firmware. This catches remote changes by the manufacturer.
- Known-good values drifting, for example the battery type, capacity mode, charge limits, meter settings or firmware versions. Because these are fixed values, a change is caught even if Home Assistant restarts between reads.
- Any fault bit or error code. The under-temperature bit (0x2013 bit 5) will trigger this on cold nights, so you may want to treat it separately.
- The battery not taking spare solar when it is below full.
- Slow grid charging during the cheap-rate window. Allow for battery temperature: a cold battery charges slowly by design. See [cold-weather-charging.md](cold-weather-charging.md).
- The inverter going offline during the night charge.

## Charge planning

This system uses [Predbat](https://springfall2008.github.io/batpred/) with a custom inverter type that calls a Home Assistant script. The script maps Predbat's requests onto the three profile controls:

| Predbat request | Grid charge end SOC | Work mode | Grid charge |
|---|---|---|---|
| Charge to N% | N | Back-up | On |
| Hold | Current SOC | Back-up | On |
| Stop | (unchanged) | Self use | Off |

The working package is [predbat_bexie.yaml](../home-assistant/packages/predbat_bexie.yaml). It also holds the battery while a car charges, keeps the battery from cycling inside the cheap-rate window, and falls back to Self use if Predbat stops updating. It is one household's setup, so treat it as an example to adapt.
