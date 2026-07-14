# Eufy RoboVac 30C - ESPHome / Home Assistant Integration

Local control of the Eufy RoboVac 30C with no cloud, no app, and no Eufy account. Based on the original work by [sid-the-squid](https://github.com/sid-the-squid/eufy_robovac_30c), updated for current ESPHome versions.

## How it works

The RoboVac 30C contains two chips: the main vacuum CPU, and a **TYWE1S Wi-Fi module** (ESP8266-based) that handles connectivity. These two chips communicate over a serial UART connection using Tuya's protocol.

This project replaces the firmware on the TYWE1S module with ESPHome, which has a built-in Tuya component that speaks the same protocol. The result is a vacuum that talks directly to Home Assistant over your local network, no Eufy cloud involved.

```
Vacuum CPU  ←→  UART  ←→  TYWE1S (running ESPHome)  ←→  Wi-Fi  ←→  Home Assistant
```

## Requirements

### Hardware
- Eufy RoboVac 30C
- USB-to-serial adapter (CP2102 or CH340 based, ~$5) - I used a [MicroFTX](https://jim.sh/ftx/)
- External 3.3V power supply capable of at least 1A (the TYWE1S pulls significant current spikes when Wi-Fi transmits. Insufficient current causes brownouts)
- Soldering equipment

### Software
- [ESPHome](https://esphome.io) 2026.x or later
- Home Assistant 2026.2 or later

> **Note for Synology NAS users:** HA images from 2026.3 onwards use zstd compression which requires Docker 20.10+. Synology ships an older Docker version. Either stay on `ghcr.io/home-assistant/home-assistant:2026.2`, migrate HA to another machine, or update Docker using [telnetdoogie/synology-docker](https://github.com/telnetdoogie/synology-docker).

## Initial Flash (one-time, requires physical access)

### 1. Open the vacuum

Remove the screws from the bottom of the vacuum and locate the TYWE1S Wi-Fi module on the PCB.

### 2. Identify the UART pins

The TYWE1S has the following pins you need:
- **TX** - connect to your USB-serial adapter's RX
- **RX** - connect to your USB-serial adapter's TX  
- **GND** - connect to GND on both adapter and external power supply
- **3.3V** - connect to your external power supply (do NOT use the adapter's 3.3V, it likely can't supply enough current)
- **IO0** - pull to GND to enter flash mode

> Note: TX and RX are crossed - the module's TX goes to the adapter's RX and vice versa.

### 3. Prepare the secrets.yaml file

Create a `secrets.yaml` file in the same directory as the YAML config:

```yaml
wifi:
  ssid: "YourNetworkName"
  password: "YourPassword"
  ap:
    ssid: "Eufy Fallback"
    password: "yourfallbackpassword"
```

### 4. Install ESPHome

On Linux (Arch):
```bash
sudo pacman -S python-pipx
pipx install esphome
pipx ensurepath
source ~/.bashrc
```

On other Linux distros:
```bash
pip install esphome
# or if externally managed:
pipx install esphome
```

> **Linux permission note:** You may need to grant access to the serial port:
> ```bash
> sudo chmod 666 /dev/ttyUSB0
> ```
> Or add yourself to the dialout group (requires logout/login):
> ```bash
> sudo usermod -a -G dialout $USER
> ```

### 5. Flash the firmware

Connect IO0 to GND, then apply power to the module. Then run:

```bash
esphome run eufy_30c.yaml
```

When prompted, select the serial port option. After flashing, disconnect IO0 from GND and power cycle.

### 6. Set Wi-Fi credentials (if not using secrets.yaml)

After the first boot the module will broadcast a fallback hotspot called **"Eufy Fallback"**. Connect to it with your phone and enter your Wi-Fi credentials in the captive portal that appears.

> If the hotspot doesn't appear, check that IO0 is no longer connected to GND and power cycle the module.

## All future updates

Once the module is on your Wi-Fi, all future firmware updates are over the air:

```bash
esphome run eufy_30c.yaml --device 192.168.x.x
```

Use the device's IP address if mDNS (`eufy-30c.local`) doesn't resolve on your network.

## Adding to Home Assistant

Go to **Settings → Devices & Services** and the device should be auto-discovered via mDNS. If not, add it manually using its IP address on port 6053.

## Entities

Once connected you will have the following entities in Home Assistant:

| Entity | Type | Description |
|--------|------|-------------|
| Eufy 30c Battery | Sensor | Battery percentage |
| Eufy 30c Robot Status | Text Sensor | Human readable status (Cleaning / Idle / Sleeping / Charging / Fully Charged / Error) |
| Eufy 30c Clean | Switch | Start/stop cleaning |
| Eufy 30c Go Home | Switch | Send to dock |
| Eufy 30c Locate | Switch | Play locate sound |
| Eufy 30c Cleaning Mode | Select | Auto / Single Room / Spot / Edge / Idle |
| Eufy 30c Suction Power | Select | Standard / BoostIQ / Max |
| Eufy 30c Direction | Select | Forward / Back / Left / Right |
| Eufy 30c Move Forward/Back/Left/Right | Button | Momentary directional movement |

## Tuya Datapoint Reference

```
Datapoint 1:   switch  - unknown
Datapoint 2:   switch  - start cleaning
Datapoint 3:   enum    - directional move (0=fwd, 1=back, 2=left, 3=right)
Datapoint 5:   enum    - clean mode (0=auto, 1=single room, 2=spot, 3=edge, 4=idle)
Datapoint 15:  enum    - status (0=cleaning, 1=idle, 2=sleeping, 3=charging, 4=charged)
Datapoint 101: switch  - go home
Datapoint 102: enum    - suction (0=standard, 1=boostIQ, 2=max)
Datapoint 103: switch  - locate
Datapoint 104: int     - battery percentage
Datapoint 106: enum    - error (0=none, 3=cliff/edge, 5=left wheel stuck, 6=left brush stuck)
```

## ESPHome API Changes (for those updating from older configs)

The original YAML from sid-the-squid was written for an older ESPHome version. The following breaking changes affect newer versions:

- `platform: ESP8266` inside the `esphome:` block must be moved to a separate `esp8266:` block
- `set_datapoint_value` was renamed to `set_raw_datapoint_value`, then removed entirely
- `set_raw_datapoint_value` now requires a vector rather than an int — use `tuya.datapoint_value.set` action or the native `select` platform instead
- The `tuya:` component needs an explicit `id:` field to be referenced in automations
- An `ota:` block with `platform: esphome` is required for OTA updates
- Text sensor lambdas must have a default return path or the compiler will error

## Credits

- Original configuration: [sid-the-squid](https://github.com/sid-the-squid/eufy_robovac_30c)
- Datapoint research: [dhumpf.de](https://www.dhumpf.de/?tag=robovac-30c) and [mitchellrj/eufy_robovac](https://github.com/mitchellrj/eufy_robovac)

## References
- [HA Container Update Fails due to outdated docker on Synology](https://community.home-assistant.io/t/2026-3-container-update-fails/993151/)
- [Module Datasheet](https://developer.tuya.com/en/docs/iot/wifie1smodule?id=K9605thnvg3e7)
- [MicroFTX](https://jim.sh/ftx/)
- [Robovac 30C Disassembly](https://www.youtube.com/watch?v=seTS44ZnDiY)
- [Using a Raspberry Pi to flash the ESP instead of a serial converter](https://github.com/tasmota/docs-8.1/blob/master/Flash-Sonoff-using-Raspberry-Pi.md)
