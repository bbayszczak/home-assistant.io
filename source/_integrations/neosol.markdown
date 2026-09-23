---
title: Profalux Neosol
description: Instructions on how to control Profalux Neosol roller shutters with their 868 MHz USB dongle in Home Assistant.
ha_category:
  - Cover
ha_iot_class: Assumed State
ha_release: 2026.10
ha_codeowners:
  - '@bbayszczak'
ha_config_flow: true
ha_domain: neosol
ha_platforms:
  - cover
  - diagnostics
ha_integration_type: hub
ha_quality_scale: platinum
---

The **Profalux Neosol** {% term integration %} lets you control Profalux Neosol roller shutters from Home Assistant. It talks to the shutters through the 868&nbsp;MHz USB dongle sold for them, plugged directly into the machine that runs Home Assistant.

Everything stays in your home: the dongle transmits over radio to the motors, so no Calyps'HOME box, manufacturer account, or cloud service is involved. Once set up, you can open, close, and stop each paired shutter, put them in scenes and scripts, and automate them alongside the rest of your home. For example, you can close every shutter at sunset, or open the bedroom shutter when your alarm goes off.

This is an independent integration. It is not affiliated with, endorsed by, or supported by Profalux or Stella Advanced Technology.

## Supported devices

The following device is known to be supported by the integration:

- The `MAI-DONGLE868-1A` USB dongle, which identifies itself as `PFX KEELOQ` and reports software revision `Rev10`.

Through that dongle, the integration controls Profalux Neosol roller shutter motors that are already paired with one of its channels.

Other dongle references from the same family may work, but none has been tested.

## Unsupported devices

- The Calyps'HOME box and any shutter reached through it. The integration only talks to the USB dongle.
- Shutters that are not paired with a channel of your dongle. Home Assistant only sees channels the dongle has already transmitted on.

## Prerequisites

1. Plug the dongle into a USB port of the machine that runs Home Assistant.
2. Place the dongle within radio range of the shutters. It transmits at 868&nbsp;MHz, so thick walls and metal shutter boxes reduce the range.
3. Keep the original remote of each shutter at hand. It is what selects the shutter during pairing, and shutters already paired with your dongle are picked up automatically.

{% include integrations/config_flow.md %}

Home Assistant discovers the dongle when you plug it in, and offers to set it up. You can also add it manually, in which case you are asked for the serial port.

{% configuration_basic %}
Serial port:
    description: "The serial port the dongle is plugged into. For example, `/dev/ttyACM0`. Home Assistant stores the stable `/dev/serial/by-id/` path for it, so the dongle keeps working after a reboot even if the port number changes. The port is checked before the setup completes."
{% endconfiguration_basic %}

The stored path follows the dongle rather than the USB port, so you can plug it into another port and Home Assistant reconnects to it on its own. If the path changes anyway, for example on a system that has no `/dev/serial/by-id/` directory, select the new port with the **Reconfigure** option of the integration entry.

## Supported functionality

The integration adds one device per paired channel of the dongle, each with a single cover {% term entity %}, plus a device for the dongle itself.

### Cover

For each paired shutter, you can use Home Assistant to:

- Open the shutter. The motor runs until its end stop.
- Close the shutter. The motor runs until its end stop.
- Stop the shutter where it is.

The shutters report no state. The radio link only goes one way: the dongle transmits and the motors never answer, so Home Assistant cannot know whether a shutter is open or closed. Rather than guess from the last command it sent — a guess that any use of the original remote would invalidate — it reports the state as unknown, and the open, close, and stop buttons stay available at all times.

The position is unknown for the same reason, so the shutters have no position slider.

## Data updates

The shutters themselves send nothing back, so there is nothing to poll on them.

Home Assistant {% term polling polls %} the dongle every 5 minutes for its channel list. This serves two purposes: it confirms the dongle is still reachable, and it picks up shutters you paired after the setup. A shutter paired with a new channel appears as a new device within 5 minutes, without restarting Home Assistant.

If the dongle stops answering, for example because it was unplugged, the shutters become unavailable and Home Assistant reconnects on its own once it is plugged back in.

## Pairing a shutter

Shutters that already use a channel of your dongle appear on their own. To add one, go to **Settings** > **Devices & services**, select the Profalux Neosol integration, and select **Configure**.

The dongle does not pair a shutter by itself: it opens a window of about a minute, and the shutter you drive from its own remote during that window is the one that gets paired. The others are left alone.

Before you start, move the shutter to mid-travel and keep its remote at hand. Once you confirm, you have 60 seconds to perform this sequence on the remote:

1. Press up, and wait for the shutter to reach the top.
2. Press down, and let about four slats appear.
3. Press stop.
4. Press up again, and wait for the top.

Then leave the shutter alone until the window closes. The new shutter appears within a few seconds.

Pairing is additive: the shutter keeps answering its original remote, and any channel it was already paired with.

### If the pairing did not take

Home Assistant cannot tell a successful pairing from a failed one, because the motors never answer. A channel counts as used as soon as the window is opened, so a shutter appears either way. If the new shutter does not obey its controls, delete its device from the integration page and start again. Deleting it also keeps it from coming back on the next refresh.

## Neosol automation examples

Here are a few ideas to get you started.

{% include docs/paste_yaml_tip.md %}

### Close every shutter at sunset

- **Trigger**: Sun: after sunset
- **Action**: Close cover

{% details "YAML example for closing the shutters at sunset" %}

{% example %}
automation: |
  alias: "Close the shutters at sunset"
  triggers:
    - trigger: sun
      event: sunset
  actions:
    - action: cover.close_cover
      target:
        entity_id: cover.shutter_0
{% endexample %}

{% enddetails %}

### Close the shutters when it gets too hot

- **Trigger**: Numeric state of a temperature sensor, above a threshold
- **Action**: Close cover

{% details "YAML example for closing the shutters on a hot day" %}

{% example %}
automation: |
  alias: "Close the living room shutter when it gets too hot"
  triggers:
    - trigger: numeric_state
      entity_id: sensor.living_room_temperature
      above: 26
  actions:
    - action: cover.close_cover
      target:
        entity_id: cover.shutter_0
{% endexample %}

{% enddetails %}

## Known limitations

- **No feedback from the shutters.** The dongle only transmits. A successful command means a radio frame was sent, never that a shutter moved. If a shutter is out of range or its motor is unpowered, Home Assistant still reports the command as done.
- **No state.** A shutter is always reported as unknown, so you cannot base an automation or a condition on whether it is open or closed. Commands can be sent at any time, which is what matters in practice.
- **No position control.** The motors cannot be sent to a given position over this protocol, so the shutters have no position slider and report no percentage.
- **The favorite position is not exposed.** Neosol motors can store a favorite position, but the integration does not offer it.
- **A pairing cannot be confirmed.** The dongle counts the attempt as a transmission whether or not the shutter accepted it, so a failed attempt still produces a shutter, and uses up one of the fifty channels. Delete it and try again.
- **Pairing still needs the original remote.** Home Assistant opens the window, but the sequence that selects the shutter is performed on its own remote.
- **One dongle per installation.** The integration takes a single configuration entry, and a dongle only reaches the shutters paired with its own channels.
- **Commands are sent one at a time.** The dongle has a single serial link, so closing ten shutters at once sends ten frames in sequence rather than simultaneously.

## Troubleshooting

### Can't set up the dongle

{% details "Symptom: \"Failed to connect\"" %}

#### Description

Home Assistant could not open the serial port, or the dongle did not answer.

#### Resolution

To resolve this issue, try the following steps:

1. Make sure the dongle is plugged in, and that its LED behaves as usual.
2. Confirm the correct serial port was selected.
3. Make sure no other program is using the port. The dongle accepts a single connection at a time, so close any tool you used to configure it.
4. Unplug the dongle, plug it back in, and try again.

{% enddetails %}

{% details "Symptom: \"The device on this serial port did not identify itself as a Neosol dongle\"" %}

#### Description

Something answered on that serial port, but it is not a compatible dongle. The USB vendor ID of the dongle belongs to Silicon Labs and is shared by many unrelated serial adapters, so Home Assistant confirms the device by asking it to identify itself.

#### Resolution

Select the serial port that belongs to the dongle. If you have several serial devices, unplug the dongle, note which port disappears, and plug it back in.

{% enddetails %}

### A shutter is missing

Home Assistant only creates a device for channels the dongle has already transmitted on, which in practice means the paired ones. If a shutter is missing, it is not paired with any channel of this dongle. Pair it with the shutter's own remote, then wait up to 5 minutes for it to appear.

### A shutter does not move

The shutters never report back, so Home Assistant cannot tell a command that arrived from one that did not. If a shutter ignores Home Assistant but still obeys its own remote:

1. Move the dongle closer to the shutter, or away from metal enclosures and other 868&nbsp;MHz transmitters.
2. Check that the shutter is powered.
3. Confirm the shutter is still paired with the dongle channel, by testing the same channel with the dongle's own tooling.

### The shutters are unavailable

The dongle stopped answering. Check that it is still plugged in, and that no other program took over the serial port. Home Assistant reconnects on its own once the dongle answers again.

## Removing the integration

This integration follows standard integration removal. No extra steps are required.

{% include integrations/remove_device_service.md %}

You can also delete a single shutter from the integration page, which is how you get rid of one whose pairing did not take. Home Assistant remembers the deletion, so the shutter does not reappear; setting the integration up again brings it back.

Removing the integration does not unpair the shutters from the dongle. They keep working with their own remotes, and setting the integration up again brings them all back.
