# Zigbee2MQTT – Tuya/Moe's MS-105Z Dimmer Power Restore Fix (Blueprint)

## TL;DR:
The Moe's / Tuya MS-105Z Zigbee dimmer completely ignores its “power_on_behavior” (Off / Previous / On) setting. It always boots in OFF after a mains outage. Zigbee2MQTT + Home Assistant also do not generate any reliable “device online” or “device available” events for this device.

After a long journey involving HA internals, MQTT, Z2M quirks, missing UI features, and helper logic, here is a working, (probably) reliable fix. It restores the dimmer to its previous ON/OFF state and brightness every time power is restored.

It’s packaged as a Home Assistant Blueprint so anyone can drop it in and be done.

## Acknowledgements

Thanks to all the pilgrims who came this way before and left useful information that contributed to this solution. This solution took a LOT of digging:

* Z2M advanced config differences
* MQTT listener not where it used to be
* HA context.user_id tricks
* Tuya’s broken “power_on_behavior”
* Helpers not updating at the right moment
* Race conditions between Z2M and HA
* Detecting real user off-vs-power-loss off
* Debug logging via Activity

If you’re struggling with Tuya dimmers or MS-105Z modules:
this blueprint seems to fix the only real flaw they have.


## Problem

Some Tuya/Moes Zigbee dimmers (notably the **Moes MS-105Z** module) expose a “Power On State” / `power_on_behavior` setting (Off / Previous / On), but in practice:

* They **always** boot in `OFF` after mains power is cut and restored.
* They often do **not** generate useful `unavailable` / `available` state changes.
* They may not emit `device_announce` / `device_online` bridge events in a consistent way.
* Home Assistant + Zigbee2MQTT therefore have no reliable built-in way to:
   * Detect that the device just came back from a power cut.
   * Restore the dimmer to its previous ON/OFF state and brightness.

## This seems overly complicated.  Why do this?

1. I have one circuit with a set of smart bulbs that all properly store and use the default power on state.
2. On this circuit there is one ceiling can with a custom bulb that I cannot replace with a smart bulb, so need to make it smart with a dimmer controller.
3. My house is used by a lot of people who don't know about, or care about, or like automation, so all the switches need to behave in the ordinary way:  when a light is switched on, it turns on.
4. I bought this MS-105Z... which doesn't handle default power on state correctly and I'm too stubborn to just throw it away and go get one from another vendor that works.
5. This was a great end-to-end lesson in the deep magic of HA and Z2M automation and debugging.

## Idea

Instead of fighting the broken device firmware, this blueprint:

1. Tracks the *desired* state of the dimmer in Home Assistant:
    - Whether the light is supposed to be ON or OFF.
    - The last brightness level when it was ON.
2. Listens to the **Zigbee2MQTT state topic** for that device.
3. When Zigbee2MQTT publishes an `OFF` state right after power is restored, the automation checks:
    - Does HA say this light *should* be ON?
4. If yes, it turns the light back ON at the remembered brightness.

This effectively re-implements a working “power restore to previous state” behavior in Home Assistant, even though the dimmer’s own `power_on_behavior` is broken.

## Requirements

* Home Assistant  (tested with HomeAssistant Core 2025.11.2, but should work with older versions)
* Zigbee2MQTT (tested with 2.6.3-1)
* A Tuya/Moes-style Zigbee dimmer (e.g. MS-105Z) that:
   * Publishes its state to `zigbee2mqtt/<friendly name>`
   * But always boots in OFF after a mains outage

## Helpers you need

Create two helpers in Home Assistant:

1. **Desired ON state (input\_boolean)**
   * Go to: Settings → Devices & Services → Helpers → Create Helper → Toggle
   * Example:
      * Name: `<Friendly Name> – Desired ON State`
      * Entity ID: `input_boolean.<friendly_name>_desired_on_state`
2. **Last brightness (input\_number)**
   * Go to: Create Helper → Number
   * Example:
      * Name: `<Friendly Name> – Last brightness`
      * Entity ID: `input_number.<friendly_name>_last_brightness`
      * Min: `1`
      * Max: `254`
      * Step: `1`

## Installing the blueprint

1. Save the blueprint YAML (in the next comment) as:`config/blueprints/automation/tuya_ms105z_power_restore.yaml`
2. Restart Home Assistant or reload automations.
3. Go to: Settings → Automations & Scenes → Blueprints.
4. Use this blueprint to **Create Automation**.

## Filling in the inputs

When creating an automation from this blueprint, set the blueprint properties like this:

* **Target light/dimmer**
    * → Your dimmer light entity (e.g. `light.<friendly_name>`)
* **Zigbee2MQTT state topic**
    * → The Z2M topic for this device
    * → Example: `zigbee2mqtt/<Friendly Name>`
* **Desired ON state helper**
    * → Stored state set the last time changed by HA. → Your input\_boolean (e.g. `input_boolean.<friendly_name>_desired_on_state`)
* **Last brightness helper**
    * → Stored brightness level set last time updated by HA. → Your input\_number (e.g. `input_number.<friendly_name>_last_brightness`)
* **Enable debug logging?**
    * Optional — shows detailed events in Activity Log.

## How it works – logic flow

1. **When the light turns ON (via HA)**

* Helper `<friendly_name>_desired_on_state` is set to **on**
* Helper `<friendly_name>_last_brightness` is updated to current brightness

1. **When the light turns OFF (via HA UI/automation)**

* Helper `<friendly_name>_desired_on_state` is set to **off**
* Only if the OFF came from a **user/automation** (using HA context)
* Prevents false “OFF” caused by power glitches

1. **When power is restored**

* The dimmer powers on and boots in `OFF` state.
* Z2M publishes a state payload with `"state":"OFF"`
* If the HA stored helper state `input_number.<friendly_name>_desired_on_state` says it *should* be ON:
   * HA waits 2 seconds  (you can tune this)
   * Sends `light.<friendly_name>_turn_on brightness=<last brightness>`
   * Light returns exactly as before power loss

## Why this should be reliable

This blueprint does *not* depend on:

* Zigbee availability state changes
* `last_seen`
* Tuya’s broken `power_on_behavior`
* deviceAnnounce events
* Z2M advanced settings

It only relies on one thing Tuya/Moes *always* does:

>When it boots at main power-on, it publishes a normal Z2M state payload to  
`zigbee2mqtt/<Friendly Name>`

By pairing that with stored HA helper state, we restore the prior dimmer state.

## Notes / Tips

* Works even if the dimmer never reports `unavailable`
* Survives HA restarts and Z2M restarts
* Handles multiple sequential power flickers
* Debug logging helps see exactly why/when a restore happened

## Extending the blueprint

If needed, the logic can be extended to:

* Support multiple dimmers
* Restore a *fixed* brightness instead of last brightness
* Notify the user when recovery occurs
* Add offline-time thresholds (restore only after X seconds offline)
* Add dashboard controls for “Force restore” or “Power-failure behavior”

## Summary

If you own a Tuya/Moes MS-105Z (or similar Tuya-based dimmer module), this blueprint gives you the power-restore behavior the hardware should have had in the first place — consistent, reliable, HA-controlled, and robust.

## Blueprint is in next comment.
