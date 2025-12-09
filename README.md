# Luqilo Zehnder Controller Blueprint

<div align="center">
  <a href="https://www.tindie.com/stores/luqilo/?ref=offsite_badges&utm_source=sellers_Luqilo&utm_medium=badges&utm_campaign=badge_small"><img src="https://d2ss6ovg47m0r5.cloudfront.net/badges/tindie-smalls.png" alt="I sell on Tindie" width="200" height="55"></a>
  <a href="https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FLuukgr%2Fzehnder-controller-luqilo%2Fblob%2Fmain%2Fluqilo-zehnder-blueprint.yaml"><img src="https://my.home-assistant.io/badges/blueprint_import.svg" alt="Import Blueprint"></a>
</div>

This blueprint automates fan speed control for Zehnder ventilation systems based on CO2 levels, bathroom humidity, and cooktop usage.

Most of the blueprint configuration is straightforward, but you will need to create a threshold sensor from your humidity sensor to detect showering activity. This process is explained in detail below.

## Requirements

### Hardware
- [Luqilo Zehnder Controller PCB](https://www.tindie.com/products/luqilo/esphome-zehnder-comfoair-e300e400/)

Optional:
- Zehnder Wired 0-10V CO2 sensor
- Bathroom humidity sensor ([example](https://www.sonoff.nl/a-80532623/zigbee-producten/sonoff-zigbee-snzb-02d-temperatuur-en-luchtvochtigheidssensor/#description))
- Cooktop power sensor ([example](https://www.google.com/search?client=ubuntu-sn&channel=fs&q=zigbee+power+sensor#oshopproduct=gid:14869331581711953652,mid:576462827113040157,oid:13833271765102381775,iid:493071279013356128,rds:Q1NTX1BDXzE0ODY5MzMxNTgxNzExOTUzNjUyfFBDXzE0ODY5MzMxNTgxNzExOTUzNjUyfFBST0RfQ1NTX1BDXzE0ODY5MzMxNTgxNzExOTUzNjUyfFBST0RfUENfMTQ4NjkzMzE1ODE3MTE5NTM2NTI%3D,pvt:hg,pvo:3&oshop=apv&pvs=0))

### ESPHome Configuration

You'll need to flash the Luqilo Zehnder Controller PCB with ESPHome firmware. The `zehnder-controller-luqilo.yaml` file in this repository contains the complete ESPHome configuration for:
- Zehnder ComfoAir E300/E400 communication
- Fan speed control (0-10V output)
- CO2 sensor integration (0-10V input)
- Override switch for manual control
- Home Assistant integration via ESPHome API

Flash this YAML configuration to your ESP32 device on the Luqilo PCB to enable all the entities required by this blueprint.

### Setting Up Bathroom Humidity Threshold

The blueprint uses a binary sensor that detects when bathroom humidity is significantly higher than normal (indicating shower/bath usage). Follow these steps to create the required sensors:

#### Step 1: Create Filtered Humidity Sensor
Add this to your `configuration.yaml` under the `sensor:` section:

```yaml
sensor:
  - platform: filter
    name: "Filtered Humidity Bathroom"
    entity_id: sensor.YOUR_BATHROOM_HUMIDITY_SENSOR
    filters:
      - filter: range
        upper_bound: 85
      - filter: time_simple_moving_average
        window_size: "06:00"
        precision: 1
```

**Replace** `sensor.YOUR_BATHROOM_HUMIDITY_SENSOR` with your actual bathroom humidity sensor entity ID.

This creates a 6-hour moving average of bathroom humidity, which represents the "normal" humidity level.

#### Step 2: Create Humidity Difference Template Sensor
Add this to your `configuration.yaml` under the `template:` section:

```yaml
template:
  - sensor:
      - name: "Humidity Difference"
        unique_id: humidity_difference_bathroom
        unit_of_measurement: '%'
        state: >
          {{ states('sensor.YOUR_BATHROOM_HUMIDITY_SENSOR') | float(0) - states('sensor.filtered_humidity_bathroom') | float(0) }}
```

**Replace** `sensor.YOUR_BATHROOM_HUMIDITY_SENSOR` with your actual bathroom humidity sensor entity ID.

This calculates the difference between current and average humidity.

#### Step 3: Create Filtered Humidity Difference Sensor
Add this to your `configuration.yaml` under the `sensor:` section:

```yaml
sensor:
  - platform: filter
    name: "Filtered Humidity Bathroom Difference"
    entity_id: sensor.humidity_difference
    filters:
      - filter: lowpass
        time_constant: 5
        precision: 2
```

This smooths out the humidity difference to avoid rapid fluctuations.

#### Step 4: Create Threshold Binary Sensor
Go to **Settings** → **Devices & Services** → **Helpers** → **Create Helper** → **Threshold**

Configure as follows:
- **Name**: Threshold Bathroom Humidity
- **Input sensor**: `sensor.filtered_humidity_bathroom_difference`
- **Upper threshold**: `3` (or adjust based on your needs)
- **Hysteresis**: `0.5`

This creates `binary_sensor.threshold_bathroom_humidity` which turns ON when humidity rises significantly above normal.

#### Step 5: Restart Home Assistant
After adding the sensors to `configuration.yaml`, restart Home Assistant for the changes to take effect.

## Blueprint Usage

### Installation
1. Copy `luqilo-zehnder-blueprint.yaml` to your Home Assistant `blueprints/automation` folder
2. Reload automations in Home Assistant

### Configuration
Create a new automation using this blueprint and configure:

- **Zehnder Fan**: Your Zehnder fan entity (default: `fan.zehnder_fan_speed`)
- **Bathroom Humidity**: The threshold binary sensor created above (default: `binary_sensor.threshold_bathroom_humidity`)
- **Cooktop Power**: Your cooktop power sensor 
- **CO2 Sensor**: Your CO2 concentration sensor (default: `sensor.co2_concentration`)
- **Override Switch**: Switch to disable automation (default: `switch.override_automations`)

### Parameters
- **PPM for 100% fan**: CO2 level for maximum fan speed (default: 1500 ppm)
- **PPM for minimal fan speed**: CO2 level for minimum fan speed (default: 425 ppm)
- **Minimal fan speed**: Minimum fan percentage (default: 5%)
- **Bathroom/Cooktop Fan speed**: Fan speed when humidity/cooking detected (default: 100%)
- **Schedule start time**: Automation active time start (default: 06:00)
- **Schedule stop time**: Automation active time stop (default: 23:30)

### How It Works

#### Daytime (between schedule start and stop):
- **High humidity or cooking detected**: Fan runs at bathroom/cooktop speed (default 100%)
- **CO2 below minimum**: Fan runs at minimum speed
- **CO2 above maximum**: Fan runs at bathroom/cooktop speed
- **CO2 between min/max**: Fan speed scales linearly based on CO2 level

#### Nighttime (outside schedule hours):
- **High humidity or cooking detected**: Fan runs at 35%
- **CO2 below minimum**: Fan runs at minimum speed
- **CO2 above maximum**: Fan runs at 35%
- **CO2 between min/max**: No change (maintains current speed)

#### Override Mode:
When the override switch is turned ON, the automation stops running completely, allowing manual fan control.

### Trigger Frequency
The automation checks all conditions every 30 seconds and adjusts fan speed accordingly.

## License
This blueprint is provided as-is for use with Luqilo Zehnder Controller PCB systems.