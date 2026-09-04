# Home Assistant Zigbee2MQTT Router Offline Monitor

A reusable Home Assistant solution for monitoring Zigbee2MQTT Routers without
maintaining a static Router list.

It dynamically reads the Zigbee2MQTT device inventory, selects devices whose
actual Zigbee type is `Router`, listens for Zigbee2MQTT availability changes,
and alerts only when a monitored device remains offline for a configurable
confirmation period.

You can also add selected non-Router devices by their Zigbee2MQTT friendly name.

## What this repository provides

- an **MQTT Router inventory sensor** based on
  `<base_topic>/bridge/devices`;
- an **automation blueprint** for offline monitoring;
- support for **optional critical non-Router devices**;
- a built-in **Home Assistant persistent notification**;
- user-configurable **additional notification actions** such as a phone;
- a clean pattern for **multiple Zigbee2MQTT instances**;
- documentation for timing, retained MQTT messages, and troubleshooting.

## Architecture

```text
Zigbee2MQTT
    |
    +-- <base_topic>/bridge/devices  (retained)
    |          |
    |          v
    |   MQTT Router inventory sensor
    |          |
    |          +-- state: Router count
    |          +-- routers: [friendly_name, ...]
    |
    +-- <base_topic>/<friendly_name>/availability  (retained)
               |
               v
          Blueprint automation
               |
               +-- Router in inventory? --------+
               |                                |
               +-- Explicit additional device? -+
                                                |
                                                v
                                      wait N minutes for online
                                                |
                                  +-------------+-------------+
                                  |                           |
                               online                       timeout
                                  |                           |
                                stop                          v
                                                  persistent notification
                                                  + optional user actions
```

## Why the Router list is dynamic

Zigbee2MQTT publishes `bridge/devices` as a retained MQTT message and republishes
it when the device inventory changes. Each device entry includes its Zigbee
`type`, such as `Router` or `EndDevice`.

The included Home Assistant MQTT sensor filters that inventory and stores only:

```yaml
state: <number of routers>
attributes:
  routers:
    - <friendly_name>
    - <friendly_name>
```

The automation therefore does not need a manually maintained list of Router
names. Newly added Routers become eligible automatically when the inventory
updates.

## Quick start

1. Make sure Zigbee2MQTT availability is enabled.
2. Install/configure the Router inventory sensor from
   `packages/zigbee2mqtt_router_inventory.yaml`.
3. Verify the sensor exposes a `routers` attribute.
4. Import the automation blueprint:

```text
https://github.com/iarj/home_assistant_automations/blob/main/blueprints/automation/iarj/zigbee2mqtt_router_offline_monitor.yaml
```

5. Select the inventory sensor, enter the matching Zigbee2MQTT base topic, and
   choose the confirmation timeout.
6. Optionally add exact Zigbee2MQTT friendly names for critical EndDevices or
   other devices and configure phone/email notification actions.

See [Installation](docs/installation.md) for the complete procedure.

## Timing: what "offline for 30 minutes" actually means

The blueprint starts its timer **after Zigbee2MQTT publishes `offline`**.

Zigbee2MQTT's current default availability behavior is:

- active/non-battery devices: 10-minute check-in timeout, then a ping; a failed
  ping marks the device offline;
- passive/battery devices: 1500 minutes (25 hours);
- active timeout jitter: up to 30 seconds by default.

So with default active-device settings and a 30-minute blueprint confirmation:

```text
about 10 minutes until Zigbee2MQTT declares offline
+ 30 minutes confirmation in Home Assistant
= about 40 minutes from loss of communication to alert
```

This is why the confirmation time should not be described as an exact
"time since last Zigbee packet".

More details: [How it works](docs/how-it-works.md) and
[Troubleshooting](docs/troubleshooting.md).

## Multiple Zigbee2MQTT instances

The blueprint intentionally represents **one Zigbee2MQTT instance per automation
instance**. This keeps the configuration generic and avoids assumptions about
how many Zigbee networks a user runs.

If you have two Zigbee2MQTT instances, create two inventory sensors and two
automations from the same blueprint. See
[examples/two_z2m_instances.yaml](examples/two_z2m_instances.yaml).

## Requirements

- Home Assistant 2024.10.0 or newer;
- MQTT integration configured in Home Assistant;
- Zigbee2MQTT using the same MQTT broker;
- Zigbee2MQTT availability enabled for monitored devices.

## Important limitations

- Zigbee2MQTT `friendly_name` values containing `/` are not supported by the
  current single-level MQTT wildcard used by the blueprint.
- A Home Assistant automation reload/restart can restart the confirmation timer
  from a retained offline message.
- The Router inventory is based on Zigbee2MQTT's reported `type`; it does not
  guess Router status from whether a device is mains powered.

See [Troubleshooting and limitations](docs/troubleshooting.md).

## Runtime behavior and privacy

After installation, the solution operates locally between Home Assistant,
Zigbee2MQTT, and the MQTT broker. The automation itself does not require a cloud
service. Additional notification actions may use cloud services if the user
chooses them.

## Documentation references

- Home Assistant automation blueprints:
  https://www.home-assistant.io/docs/automation/using_blueprints/
- Home Assistant blueprint schema:
  https://www.home-assistant.io/docs/blueprint/schema/
- Home Assistant MQTT triggers:
  https://www.home-assistant.io/docs/automation/trigger/#mqtt-trigger
- Zigbee2MQTT MQTT topics and `bridge/devices`:
  https://www.zigbee2mqtt.io/guide/usage/mqtt_topics_and_messages.html
- Zigbee2MQTT device availability:
  https://www.zigbee2mqtt.io/guide/configuration/device-availability.html

## License

MIT. See [LICENSE](LICENSE).
