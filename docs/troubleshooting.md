# Troubleshooting and limitations

## No offline events are received

Check Zigbee2MQTT device availability first. The feature is disabled by default
unless enabled globally or per device.

Expected MQTT topic:

```text
<base_topic>/<friendly_name>/availability
```

Expected payloads are JSON states containing `online` or `offline`.

## The inventory sensor is missing

Check that:

1. Home Assistant's MQTT integration is configured and connected;
2. the `state_topic` and `json_attributes_topic` use the correct Zigbee2MQTT
   base topic;
3. Home Assistant loaded the YAML/package containing the MQTT sensor;
4. Zigbee2MQTT is publishing `<base_topic>/bridge/devices`.

`bridge/devices` is retained, so a correctly configured sensor normally receives
the latest inventory immediately after it subscribes.

## The Router count is correct but a device is not monitored

The automation compares exact Zigbee2MQTT `friendly_name` strings. For an
additional monitored device, spelling and case must match Zigbee2MQTT exactly.

Routers are not inferred from power source or Home Assistant device class. The
Router role comes from Zigbee2MQTT's `bridge/devices` inventory.

## Why does a 30-minute alert take around 40 minutes?

The blueprint confirmation time starts when Zigbee2MQTT publishes `offline`, not
when the last radio packet was received.

With Zigbee2MQTT defaults, an active device has an availability timeout of 10
minutes. If it does not check in, Zigbee2MQTT pings it and marks it offline if
the ping fails. The default maximum timeout jitter is 30 seconds.

Therefore a default configuration with a 30-minute blueprint confirmation time
can alert roughly 40 minutes after the device stopped communicating:

```text
~10 min Zigbee2MQTT active-device timeout
+30 min blueprint confirmation
=~40 min total
```

This is an approximation, not a guaranteed RF-loss timestamp.

Passive/battery devices use a much longer Zigbee2MQTT default timeout (1500
minutes / 25 hours). This matters if you add a battery EndDevice to the
blueprint's additional-device list.

## What happens after an automation reload or Home Assistant restart?

Zigbee2MQTT availability messages are retained. After Home Assistant subscribes
again, an already-offline retained availability message can start a new
confirmation wait.

This means the confirmation timer is not a durable historical timer across an
automation reload/restart; it starts from the offline message observed by that
automation run.

## Friendly names containing `/`

This blueprint intentionally subscribes to:

```text
<base_topic>/+/availability
```

Zigbee2MQTT permits `/` inside `friendly_name`, which creates deeper MQTT topic
levels. Those names do not match the single-level `+` wildcard used by this
blueprint.

Version 1 therefore supports friendly names that do not contain `/`. Renaming a
device to remove `/`, or extending the blueprint to handle arbitrary topic
levels, is required for that edge case.

## Stale retained availability messages

A broker can retain an old availability topic from a device that was renamed,
removed, or moved between Zigbee2MQTT instances. The Router inventory filter
helps prevent unrelated retained topics from becoming alerts: an offline message
is acted on only when its friendly name is in that instance's current Router
inventory or the user explicitly added it to the extra-device list.

## Database size

The inventory sensor stores only a Router count and a compact list of Router
friendly names. It intentionally does not mirror the full `bridge/devices` JSON
payload into Home Assistant attributes.
