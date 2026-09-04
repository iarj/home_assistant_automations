# Installation

## Requirements

- Home Assistant with the MQTT integration configured.
- Zigbee2MQTT connected to the same MQTT broker.
- Zigbee2MQTT device availability enabled.
- Home Assistant 2024.10.0 or newer for the blueprint syntax used here.

## Step 1 - Enable Zigbee2MQTT availability

Zigbee2MQTT availability must be enabled for the devices you want to monitor.
A global configuration is typically:

```yaml
availability:
  enabled: true
```

Zigbee2MQTT currently defaults this feature to disabled. Changing this setting
requires a Zigbee2MQTT restart.

You can also enable or tune availability per device; consult the Zigbee2MQTT
Device Availability documentation before changing timeouts on a production
network.

## Step 2 - Create the Router inventory sensor

Copy `packages/zigbee2mqtt_router_inventory.yaml` into your Home Assistant YAML
configuration structure.

The example assumes the default Zigbee2MQTT base topic:

```text
zigbee2mqtt
```

If your base topic is different, replace it in both `state_topic` and
`json_attributes_topic`. If you create more than one inventory sensor, also give
each sensor a unique `name` and `unique_id`.

After loading the MQTT sensor, verify that it exists in Home Assistant and has:

- a numeric state equal to the current Router count;
- a `routers` attribute containing Zigbee2MQTT friendly names.

## Step 3 - Import the blueprint

Import:

```text
https://github.com/iarj/home_assistant_automations/blob/main/blueprints/automation/iarj/zigbee2mqtt_router_offline_monitor.yaml
```

Then create an automation from the blueprint.

Configure:

- **Zigbee2MQTT base topic**: e.g. `zigbee2mqtt`;
- **Router inventory sensor**: the sensor from Step 2;
- **Network label**: any human-readable label for notifications;
- **Additional monitored devices**: optional exact Zigbee2MQTT friendly names;
- **Offline confirmation time**: default 30 minutes;
- **Additional notification actions**: optional mobile, email, TTS, etc.

A Home Assistant persistent notification is always generated when the timeout
expires. Additional actions are optional.

### Mobile notification example

Inside **Additional notification actions**, add the normal notification action
for your own phone. The blueprint makes `alert_message` available to templates,
so the message can be:

```yaml
- action: notify.mobile_app_your_phone
  data:
    title: Zigbee offline alert
    message: "{{ alert_message }}"
```

Do not copy the example service name literally; select the notify action that
exists in your Home Assistant instance.

## Multiple Zigbee2MQTT instances

Use one inventory sensor and one blueprint automation per Zigbee2MQTT instance.

For example:

```text
Instance A: base topic zigbee2mqtt
  -> sensor.zigbee2mqtt_main_router_inventory
  -> blueprint automation A

Instance B: base topic z2m_secondary
  -> sensor.zigbee2mqtt_secondary_router_inventory
  -> blueprint automation B
```

See `examples/two_z2m_instances.yaml` for the sensor definitions.
