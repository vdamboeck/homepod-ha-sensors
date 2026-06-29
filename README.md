# HomePod HA Sensors

Expose HomePod mini temperature and humidity sensors to Home Assistant — without a local server, without custom integrations, and without polling the Apple API.

## The Problem

The Apple TV integration in Home Assistant does **not** forward HomePod sensor data (temperature, humidity). HomePods expose this data only within the Apple Home ecosystem, with no direct path into HA.

## The Solution

This project bridges that gap using a combination of HA automations, the HomeKit Bridge integration, and an Apple Home Shortcut — entirely within your local network.

```
Home Assistant
  Time pattern (every N min)
    └── input_boolean [TRIGGER]
              │
         HomeKit Bridge
              │
         Apple Home
              └── Automation: when TRIGGER turns on
                    └── Shortcut: read HomePod sensor(s)
                          └── POST /api/webhook/... (LAN only)
                                │
              ┌─────────────────┘
         Webhook Automation
              └── input_number (temp + humidity per HomePod)
                    └── Template Sensor
                          (device_class, unit, long-term statistics)
```

**Works with any number of HomePods.** A single HomePod is exactly the same setup — just configure one slot in the webhook blueprint.

## Prerequisites

- Home Assistant 2024.8 or newer
- [HomeKit Bridge](https://www.home-assistant.io/integrations/homekit/) integration set up
- Apple Home app with at least one HomePod mini acting as a Home Hub

## Setup Overview

1. Create helpers (`ha_config/helpers.yaml`)
2. Add template sensors (`ha_config/template_sensors.yaml`)
3. Import and configure the **Trigger** blueprint
4. Import and configure the **Webhook Receiver** blueprint
5. Set up the Apple Home automation + Shortcut (see `docs/03_apple_home_setup.md`)

Full step-by-step instructions: [`docs/02_homeassistant_setup.md`](docs/02_homeassistant_setup.md)

## Import Blueprints

### Trigger Blueprint
Polls your HomePod(s) on a configurable interval by toggling a Boolean Helper that is exposed via HomeKit Bridge.

[![Import Trigger Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?url=https%3A%2F%2Fraw.githubusercontent.com%2Fvdamboeck%2Fhomepod-ha-sensors%2Fmain%2Fblueprints%2Fhomepod_trigger.yaml)

### Webhook Receiver Blueprint
Receives a single JSON payload with data from all your HomePods and writes each value into the corresponding numeric helper.

[![Import Webhook Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?url=https%3A%2F%2Fraw.githubusercontent.com%2Fvdamboeck%2Fhomepod-ha-sensors%2Fmain%2Fblueprints%2Fhomepod_webhook_receiver.yaml)

## Documentation

- [`docs/01_setup_overview.md`](docs/01_setup_overview.md) — Architecture and data flow (German)
- [`docs/02_homeassistant_setup.md`](docs/02_homeassistant_setup.md) — Step-by-step HA setup (German)
- [`docs/03_apple_home_setup.md`](docs/03_apple_home_setup.md) — Apple Home Shortcut setup (German)

## License

Licensed under MIT. See [LICENSE](LICENSE).
