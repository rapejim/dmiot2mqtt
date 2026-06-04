# dmiot2mqtt

This project allows controlling various Dream Maker fans through MQTT. The implementation is rather simple and just bridges between MQTT and the Dream Maker JSON protocol. The fan must be disconnected from the cloud for this.

Supported fans:

* Dream Maker Smart Fan DM-FAN01
* Dream Maker Smart Fan DM-FAN02-W (battery-powered version)

Warning: This is just a proof-of-concept which has been created within a few hours. Don't expect anything fancy: neither in functionality nor in style! :smile:

## How it works

dmiot2mqtt will provide the same interface (with a subset of features) that the official Dream Maker cloud server offers. This way the fans will "think" that they are talking to their usual Chinese control servers. No software modifications on the fan are required.

For this to work you need to have control over your local DNS. You need to alter the DNS record for `cloud1.dm-maker.com` to return the IP address of the machine running dmiot2mqtt.

## Prerequisites

This is what you need in order to get started:

* Control over your local DNS (e.g. through your router or through Adguard/Pi Hole)
* Python 3.12 (or later) with amqtt 0.11.0b1 (or use docker image for run a docker container)
* A MQTT broker
* Ideally an always-on machine which will run dmiot2mqtt (e.g. a Raspberry Pi) with a static IP adress for DNS redirect.
* Optional (but recommended): Any Linux machine with a Bluetooth 4.0+ dongle for provisioning the fan without the Dream Maker app (one-time only)

## How to get this running?

1. Pick a machine which will host `dmiot2mqtt`. Without docker, it's recommended to create a new Python venv and run `pip3 install -r requirements.txt` inside.
2. Create a DNS record which points `cloud1.dm-maker.com` to the IP of machine running `dmiot2mqtt`.
3. Adjust the config file `dmiot2mqtt.ini` according to your needs.
4. Run `dmiot2mqtt` service. Without docker, through `python3 dmiot2mqtt.py -c /path/to/dmiot2mqtt.ini`.
5. If your fan is still unprovisioned, use the `provision.py` script from this repository to push your WiFi information to the fan over bluetooth (it's recommended without docker).
6. Wait until your fan connects to the service.
7. Without Docker, you can create a systemd unit file for `dmiot2mqtt.py` according to your needs.

NOTE: If you need debug logs, add `-l DEBUG` to de run command:
- With Docker: Uncomment `command: ["dmiot2mqtt.py", "-c", "/config/dmiot2mqtt.ini", "-l", "DEBUG"]` line of docker-compose.
- Without Docker: run `python3 dmiot2mqtt.py -c /path/to/dmiot2mqtt.ini -l DEBUG`

## Home Assistant integration

Once you have your fan talking to dmiot2mqtt, the fan device (and entities) are registered auto with discovey into Home Assistant.
If you want use manual config, this is an example, just adjust the MQTT topic names according to your fan's `<DEVICE_ID>` and de model of device replacing `<DEVICE_KEY>` for your model (view both on log when fan connect to dmiot2mqtt `[dmoit2mqtt] INFO     Auth handshake for Device Key 'DM-FAN02-W' with Device ID '1a2b3c4d5e6f1a2b3c4d5e6f'`):

```yaml
mqtt:
  - fan:
      device:
        name: "DreamMaker Fan"
        manufacturer: "DreamMaker"
        model: "<DEVICE_KEY>"
        identifiers: "<DEVICE_ID>"
      name: "DreamMaker Fan"
      unique_id: "<DEVICE_ID>_fan"
      command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
      command_template: '{ "power": {{ value }} }'
      oscillation_command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
      oscillation_command_template: '{ "roll_enable": {{ value }}}'
      oscillation_state_topic: "dmiot2mqtt/<DEVICE_ID>"
      oscillation_value_template: "{{ value_json.roll_enable }}"
      state_topic: "dmiot2mqtt/<DEVICE_ID>"
      state_value_template: "{{ value_json.power }}"
      payload_on: "1"
      payload_off: "0"
      payload_oscillation_on: "1"
      payload_oscillation_off: "0"
      percentage_command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
      percentage_command_template: '{ "speed": {{value}} }'
      percentage_state_topic: "dmiot2mqtt/<DEVICE_ID>"
      percentage_value_template: "{{ value_json.speed }}"
      json_attributes_topic: "dmiot2mqtt/<DEVICE_ID>"
      preset_modes:
        - direct
        - natural
        - smart
      preset_mode_command_template: '{% if value == "natural" %}{"mode":1}{% elif value == "smart" %}{"mode":2}{% else %}{ "mode":0}{% endif %}'
      preset_mode_command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
      preset_mode_state_topic: "dmiot2mqtt/<DEVICE_ID>"
      preset_mode_value_template: "{% if value_json.mode == 1 %}natural{% elif value_json.mode == 2 %}smart{% else %}direct{% endif %}"
  - light:
      device:
        name: "DreamMaker Fan"
        manufacturer: "DreamMaker"
        model: "<DEVICE_KEY>"
        identifiers: "<DEVICE_ID>"
      name: "Lights"
      unique_id: <DEVICE_ID>_lights
      entity_category: config
      state_topic: "dmiot2mqtt/<DEVICE_ID>"
      command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
      state_value_template: '{% if value_json.light == 1 %}{ "light": 1 }{% elif value_json.mode == 0 %}{ "light": 0 }{% endif %}'
      payload_on: '{ "light": 1 }'
      payload_off: '{ "light": 0 }'
  - lock:
      device:
        name: "DreamMaker Fan"
        manufacturer: "DreamMaker"
        model: "<DEVICE_KEY>"
        identifiers: "<DEVICE_ID>"
      name: "Child Lock",
      icon: "mdi:lock-open-outline",
      unique_id: "<DEVICE_ID>_child_lock",
      entity_category: "config",
      state_topic: "dmiot2mqtt/<DEVICE_ID>",
      command_topic: "dmiot2mqtt/<DEVICE_ID>/command",
      value_template: "{{ value_json.child_lock }}",
      payload_lock: '{ "child_lock": 1 }',
      payload_unlock: '{ "child_lock": 0 }',
      state_locked: "1",
      state_unlocked: "0"
  - number:
      device:
        name: "DreamMaker Fan"
        manufacturer: "DreamMaker"
        model: "<DEVICE_KEY>"
        identifiers: "<DEVICE_ID>"
      name: "Countdown"
      icon: "mdi:fan-clock"
      unique_id: <DEVICE_ID>_countdown
      state_topic: "dmiot2mqtt/<DEVICE_ID>"
      command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
      command_template: '{ "power_delay": {{ value }} }'
      value_template: "{{ value_json.power_delay }}"
      min: 0
      max: 480
      step: 5
      unit_of_measurement: minutes
  - select:
      device:
        name: "DreamMaker Fan"
        manufacturer: "DreamMaker"
        model: "<DEVICE_KEY>"
        identifiers: "<DEVICE_ID>"
      name: " Roll Angle"
      icon: "mdi:arrow-oscillating"
      unique_id: <DEVICE_ID>_roll_angle
      entity_category: config
      state_topic: "dmiot2mqtt/<DEVICE_ID>"
      command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
      command_template: '{ "roll_angle": {{ value }} }'
      value_template: "{{ value_json.roll_angle }}"
      options:
        - "30"
        - "60"
        - "90"
        - "120"
        - "140"
  - sensor:
      - device:
          identifiers: "<DEVICE_ID>"
        name: "Temperature"
        unique_id: <DEVICE_ID>_temperature
        device_class: temperature
        state_topic: "dmiot2mqtt/<DEVICE_ID>"
        suggested_display_precision: 1
        unit_of_measurement: "°C"
        value_template: "{{ value_json.temperature }}"
      - device:
          identifiers: "<DEVICE_ID>"
        name: "Humidity"
        unique_id: <DEVICE_ID>_humidity
        device_class: humidity
        state_topic: "dmiot2mqtt/<DEVICE_ID>"
        unit_of_measurement: "%"
        value_template: "{{ value_json.humidity }}"
  - switch:
      - device:
          name: "DreamMaker Fan"
          manufacturer: "DreamMaker"
          model: "<DEVICE_KEY>"
          identifiers: "<DEVICE_ID>"
        name: "Sounds"
        icon: "mdi:volume-high"
        unique_id: <DEVICE_ID>_sounds
        entity_category: config
        state_topic: "dmiot2mqtt/<DEVICE_ID>"
        command_topic: "dmiot2mqtt/<DEVICE_ID>/command"
        value_template: "{{ value_json.sound }}"
        payload_on: '{ "sound": 1 }'
        payload_off: '{ "sound": 0 }'
        state_on: "1"
        state_off: "0"
```
