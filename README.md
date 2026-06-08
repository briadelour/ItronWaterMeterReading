# Integrating an Itron Water Meter (SCMplus, 912MHz) with Home Assistant via RTL-SDR

## Overview

This guide documents how to receive consumption data from an **Itron transmitter on a Badger Water Meter** broadcasting on **912.6MHz** using an RTL-SDR dongle and the `rtl_433` add-on in Home Assistant. It builds directly on the setup described in the [345MHz Honeywell Security / RTL-SDR README](https://github.com/briadelour/HoneywellSecuritySensors), which covers the base hardware, software installation, Mosquitto broker configuration, and MQTT integration setup. Start there if you haven't already.

The water meter transmitter uses the **SCMplus (Standard Consumption Message Plus)** protocol (rtl_433 protocol 154) and broadcasts a raw consumption reading that can be mapped to your billing unit with a simple multiplier.

---

## Background

Most residential water meters from utilities use Automatic Meter Reading (AMR) transmitters that broadcast consumption data wirelessly so the utility can drive by and collect readings without physically accessing the meter box. These transmitters are one-way and unencrypted, which means an RTL-SDR dongle can receive them.

Using MQTT Explorer to browse the rtl_433 data stream, you may find **multiple Itron SCMplus transmitters** — one for your meter and potentially one for a neighbor's if they are within range. Look inside your meter box for the 8-digit transmitter ID printed on the device to identify which is yours.

> **Note**: The raw reading from the transmitter is a 5-digit number. Cross-referencing against a water bill revealed that the utility appends a trailing zero, so the actual gallon value is the raw reading multiplied by 10. Your reading may differ — always verify against your bill before using the value for billing estimates.

---

## Hardware Requirements

Same SDR dongle as described in the [base README](https://github.com/briadelour/HoneywellSecuritySensors):

- **SDR Dongle**: [Nooelec RTL-SDR v5](https://www.amazon.com/dp/B01GDN1T4S) (or compatible RTL2832U-based dongle)
- **Physical meter access**: You'll need to open your meter box to find the 8-digit transmitter ID on the Itron device attached to your Badger meter

---

## Software Requirements

Same stack as the [base README](https://github.com/briadelour/HoneywellSecuritySensors) — no additional add-ons required:

- **Mosquitto broker**
- **rtl_433** add-on
- **MQTT Integration** in Home Assistant
- **MQTT Explorer** (strongly recommended for identifying your transmitter ID and confirming the topic path)

---

## Configuration

### 1. rtl_433 Config — Add 912MHz with Frequency Hopping

Edit your `rtl_433.conf` (or `rtl_433.config.template`) to add the water meter frequency alongside your existing ones. The `hop_interval` controls how many seconds the radio spends on each frequency before switching.

```conf
output mqtt://[IP_of_HAOS]:1883,user=mqtt,pass=mqtt,retain=true
report_meta time:iso:usec:tz

# Frequency hopping — add 912.6M for the Itron water meter
frequency 344.975M  # Honeywell security sensors
frequency 433.92M   # Temperature and weather sensors
frequency 344.975M  # Honeywell (repeated to weight dwell time)
frequency 912.6M    # Itron water meter (SCMplus)
hop_interval 60     # Seconds per frequency

# Protocol filter for SCMplus water meter
protocol 154  # Standard Consumption Message Plus (SCMplus)

# Keep your existing protocol filters as needed, e.g.:
# protocol 70  # Honeywell Door/Window Sensor
```

**Notes:**
- `344.975M` appears twice to give the Honeywell sensors proportionally more listening time during each hop cycle, since those sensors report time-sensitive open/close events.
- `hop_interval 60` means the radio listens on each frequency for 60 seconds before cycling. Water meter transmitters broadcast infrequently, so a longer interval is fine for that frequency.
- `protocol 154` limits processing to SCMplus messages. See the [rtl_433 documentation](https://triq.org/rtl_433/) for the full protocol list.

---

### 2. Identify Your Transmitter with MQTT Explorer

1. With rtl_433 running and hopping to 912.6MHz, open **MQTT Explorer** and connect to your Mosquitto broker.
2. Browse to `rtl_433/[your-device-id]/devices/SCMplus/`.
3. You will likely see **two or more 8-digit device IDs** — one yours, one or more from neighbors within RF range.
4. Open your meter box and locate the Itron transmitter. The **8-digit ID is printed on the device label**.
5. Match that number to the topic in MQTT Explorer. **Disable or ignore** any topics corresponding to neighbors' meter IDs by simply not creating sensors for them.

#### Example MQTT Topic Structure

```
rtl_433/
  └── 17069798-rtl433/
      └── devices/
          └── SCMplus/
              ├── xxxxxxxx/          ← your meter's 8-digit ID
              │   ├── time
              │   ├── id
              │   ├── Consumption    ← raw 5-digit reading
              │   └── ...
              └── yyyyyyyy/          ← neighbor's meter (ignore)
```

---

### 3. mqtt.yaml — Add the Water Meter Sensor

Add the following sensor definition to your `mqtt.yaml`. Replace `xxxxxxxx` with your actual 8-digit transmitter ID.

```yaml
sensor:
  - state_topic: "rtl_433/17069798-rtl433/devices/SCMplus/xxxxxxxx/Consumption"
    name: "Consumption"
    unique_id: "water_meter_xxxxxxxx"
    value_template: "{{ (value | float * 10) | round(0) }}"
    unit_of_measurement: "gal"
    device_class: water
    state_class: total_increasing
    icon: mdi:water
    qos: 0
```

**Why multiply by 10?**  
The raw `Consumption` value from the transmitter is a 5-digit number. Comparing against a water bill showed the utility records consumption with a trailing zero (i.e., in units of 10 gallons). Multiplying the raw value by 10 aligns the Home Assistant sensor with billing records. Always verify this against your own bill — the multiplier may differ depending on your utility.

---

### 4. configuration.yaml — Add Utility Meters

Add the following utility meter entries to `configuration.yaml` for daily and monthly water tracking:

```yaml
utility_meter:
  water_daily_usage:
    source: sensor.water_meter_water_meter_consumption
    cycle: daily
    name: Water Daily Usage
    unique_id: water_daily_usage_meter

  water_monthly_usage:
    source: sensor.water_meter_water_meter_consumption
    cycle: monthly
    name: Water Monthly Usage
    unique_id: water_monthly_usage_meter
```

---

### 5. Energy Dashboard Integration

<img width="557" height="786" alt="image" src="https://github.com/user-attachments/assets/9348a881-4b0a-4864-b858-96527f8e8bf6" />

1. Go to **Energy** Dashboard in Home Assistant and Edit.
2. Under the **Water** section, add the **Water Daily Usage** utility meter sensor (unit: gallons).
3. Set a **static price** matching your water bill rate — for example, `0.01015 USD/gal`.

This enables water consumption and cost tracking in the Home Assistant Energy Dashboard alongside electricity and gas.

---

## Dashboard Card Example

<img width="534" height="281" alt="image" src="https://github.com/user-attachments/assets/cea78324-c159-4759-b063-56b5a014685a" />

Add this card to a Lovelace dashboard to display current meter readings:

```yaml
type: grid
cards:
  - type: heading
    heading: Itron Standard Consumption Message (SCM) Devices
    heading_style: title
  - type: vertical-stack
    cards:
      - type: entities
        entities:
          - entity: sensor.water_meter_water_meter_consumption
            name: Last Meter Reading
          - entity: sensor.scmplus_xxxxxxxx_timestamp
            name: Timestamp
        title: Itron Water Meter Reader (912MHz)
```

Replace `xxxxxxxx` in the timestamp entity with your actual transmitter ID.

---

## Troubleshooting

### No SCMplus Data Appearing

1. **Check hop timing** — The radio only listens on 912.6MHz for `hop_interval` seconds per cycle. Wait at least one full hop cycle after startup. Water meter transmitters also broadcast infrequently (every few minutes).
2. **Verify protocol 154 is enabled** — Check your rtl_433 config. If you have a broad `protocol` filter, ensure SCMplus is not excluded.
3. **Antenna placement** — 912MHz is attenuated more by walls and distance than 345/433MHz. Try moving the SDR dongle closer to an exterior wall or window facing the meter.
4. **Check rtl_433 logs** — Look for `SCMplus` entries in the add-on log output to confirm signals are being received before trying to map them in MQTT.

### Two Transmitters Visible

This is expected in a neighborhood setting. Simply identify yours by the physical ID on the meter device and only configure sensors for that ID. You do not need to do anything to "block" the neighbor's transmitter — just don't create a sensor for it.

### Reading Doesn't Match My Bill

The `* 10` multiplier in the `value_template` was derived by comparing the raw 5-digit transmitter value to a billed reading. If your reading still doesn't match, try `* 100` or check whether your utility bills in CCF (hundred cubic feet) rather than gallons, which would require a different conversion entirely.

---

## Related Documentation

- **[Base RTL-SDR / Honeywell 345MHz README](https://github.com/briadelour/HoneywellSecuritySensors)** — Hardware setup, Mosquitto broker configuration, MQTT integration, and general rtl_433 add-on installation
- [rtl_433 Documentation](https://triq.org/rtl_433/)
- [Home Assistant MQTT Integration](https://www.home-assistant.io/integrations/mqtt)
- [Home Assistant Utility Meter Integration](https://www.home-assistant.io/integrations/utility_meter/)
- [Home Assistant Energy Dashboard](https://www.home-assistant.io/docs/energy/)
- [RTL-SDR Blog](https://www.rtl-sdr.com/)

---

## License

This documentation is provided as-is for educational purposes. Use at your own risk.

---

**Last Updated**: June 2026
