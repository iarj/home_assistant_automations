# How it works

The solution has two independent parts: a Router inventory sensor and an
automation blueprint.

## 1. Router inventory

Zigbee2MQTT publishes a retained device inventory on:

```text
<base_topic>/bridge/devices
```

Each item contains, among other fields, `friendly_name` and the Zigbee device
`type`. The MQTT sensor filters that retained JSON array for items whose type is
exactly `Router`.

The sensor exposes:

- **state**: number of Routers currently present in the Zigbee2MQTT inventory;
- **attribute `routers`**: the corresponding Zigbee2MQTT `friendly_name` values.

Example conceptual state:

```yaml
state: 4
attributes:
  routers:
    - Living room plug
    - Hallway repeater
    - Kitchen light
    - Office plug
```

The complete `bridge/devices` payload is intentionally not copied into Home
Assistant attributes. Only the compact Router-name list is stored.

Because `bridge/devices` is retained and Zigbee2MQTT republishes it when device
inventory changes, newly joined Routers are picked up without maintaining a
static Router list in the automation.

## 2. Availability monitoring

When Zigbee2MQTT device availability is enabled, device state is published to:

```text
<base_topic>/<friendly_name>/availability
```

with JSON containing either `online` or `offline`.

The blueprint subscribes to:

```text
<base_topic>/+/availability
```

When an `offline` message arrives, it extracts the Zigbee2MQTT friendly name and
checks whether that name is:

1. present in the inventory sensor's `routers` attribute; or
2. present in the optional additional-device list configured by the user.

Unmonitored EndDevices are ignored immediately.

For a monitored device, the blueprint waits for the exact same availability
topic to publish `online`. If that happens before the confirmation timeout, the
run ends without an alert. If the timeout expires first, Home Assistant creates
a persistent notification and then executes any additional notification actions
configured by the user.

## Why `mode: parallel`

Several Zigbee devices can become unavailable at the same time. Each device must
have an independent confirmation timer. `parallel` mode lets every offline
device maintain its own wait without blocking other devices.

## No Zigbee polling added by this automation

The Home Assistant side only listens to MQTT messages already produced by
Zigbee2MQTT. It does not run a network map, scan the Zigbee network, or poll each
Router itself.
