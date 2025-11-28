# Tuya/Moes MS-105Z Zigbee Dimmer Power Restore Fix (Home Assistant + Zigbee2MQTT Blueprint)

**A reliable Home Assistant blueprint that restores ON/OFF state and brightness after a mains power outage for Tuya/Moes MS-105Z Zigbee dimmers (and similar Tuya-based modules) using Zigbee2MQTT.**

## Overview

Many Tuya/Moes Zigbee dimmer modules — especially the **Moes MS-105Z** — expose a “Power On State” or `power_on_behavior` setting (Off / Previous / On).  
**However, these devices ignore that setting and *always boot in OFF* after a power loss**, regardless of configuration.

Additionally:

- They often **do not send** `device_announce`, `device_online`, or `availability` events.  
- Zigbee2MQTT may not report `unavailable → available`.  
- Home Assistant does not see any reliable state transition.  
- The device always reports a “fresh boot” state of `OFF`, even if it was ON before mains power cut.

This makes normal HA automation impossible.

This project provides a **robust Home Assistant blueprint** that:

- Tracks the *desired* ON/OFF state  
- Tracks the last brightness value  
- Listens for Zigbee2MQTT state messages after boot  
- Detects when the dimmer reports `"state": "OFF"` during boot  
- Restores ON + correct brightness **if and only if** the user intended it to be ON

This effectively reimplements a correct **“restore previous state after power loss”** feature.

---

# How It Works

The solution uses two Home Assistant helpers:

| Helper Type        | Purpose                                 | Example Entity ID                                   |
|--------------------|-----------------------------------------|-----------------------------------------------------|
| `input_boolean`    | Whether the dimmer *should* be ON       | `input_boolean.<friendly_name>_desired_on_state`   |
| `input_number`     | Last brightness level (1–254)           | `input_number.<friendly_name>_last_brightness`     |

### Logic Flow

1. **User turns the light ON**  
   - Update brightness helper  
   - Set “desired on state” helper → `on`

2. **User turns the light OFF**  
   - Only update helper if the OFF came from a **user or automation** (HA context)  
   - Prevents unwanted updates during power glitches

3. **Power is restored**  
   - Dimmer boots and Zigbee2MQTT publishes a state payload to:  
     ```
     zigbee2mqtt/<friendly name>
     ```
   - If payload shows `"state": "OFF"` **and** “desired on state” is `on`:  
     - HA restores the light  
     - With the stored brightness

4. **If user turned the light OFF before power cut**  
   - Helper is `off` → automation does **nothing**

This gives you correct, predictable restore behavior.

---

# 📦 Installation

### 1. Create Helpers (once)

#### 🔹 Desired ON State (`input_boolean`)
Home Assistant → Settings → Devices & Services → Helpers → **Create Helper → Toggle**

Example:
- Name: `<Friendly Name> – Desired ON State`  
- Entity ID: `input_boolean.<friendly_name>_desired_on_state`

#### 🔹 Last Brightness (`input_number`)
Home Assistant → Helpers → **Create Helper → Number**

Config:
- Min: `1`
- Max: `254`
- Step: `1`

Entity ID:  
`input_number.<friendly_name>_last_brightness`

---

### 2. Install the Blueprint

Place this file at:
`/config/blueprints/automation/tuya_ms105z_fixg/tuya_ms105z_power_restore.yaml`


Then reload blueprints:

Home Assistant →  
**Settings → Automations & Scenes → Blueprints → Reload Blueprints**

---

# The Blueprint

> Put this in `tuya_ms105z_power_restore.yaml`  
> (Use the version from `blueprint.yaml` in this repository)

[️Click here to view the full blueprint YAML](./tuya_ms105z_power_restore.yaml)

---

# ⚙️ Creating an Automation from the Blueprint

Go to:

**Settings → Automations & Scenes → Create Automation → From Blueprint**

Fill in:

| Blueprint Input             | What to select                                | Example                                 |
|-----------------------------|-----------------------------------------------|------------------------------------------|
| **light_entity**           | Your dimmer device's light entity             | `light.<friendly_name>`                |
| **mqtt_topic**             | Zigbee2MQTT topic for device state            | `zigbee2mqtt/<Friendly Name>`          |
| **desired_state_boolean**  | Boolean helper created above                  | `input_boolean.<friendly_name>_desired_on_state` |
| **last_brightness_number** | Number helper created above                   | `input_number.<friendly_name>_last_brightness`   |
| **enable_debug**           | (Optional) log debug to Activity              | `true` or `false`                        |

---

# 🧪 Testing

1. Turn the dimmer **ON**, set brightness to ~125.  
2. Confirm helpers update in the dashboard.  
3. Kill mains power to the dimmer.  
4. Restore power.  
5. Observe that Zigbee2MQTT publishes `{"state":"OFF", ...}`.  
6. Observe blueprint checks helpers → restores ON + brightness.

### Expected behavior:
- If dimmer was ON → it comes back ON  
- If dimmer was OFF → it stays OFF  
- Brightness restored exactly as last used  
- No false triggers during power glitches  
- Debug output appears in HA's Activity Log (if enabled)

---

# Troubleshooting

### Automation didn’t trigger?

Check:
- You used the **correct Zigbee2MQTT topic**  
- The blueprint was reloaded after installation  
- Helpers have correct entity names  
- `enable_debug = true` to see debug events in Activity

### Debug shows `"desired_state":"off"` when it should be `"on"`?

You likely turned the light off via hardware toggle or physical wiring →  
Track logic only updates OFF if `context.user_id` exists (HA UI or service call).

### Still boots OFF?

On some networks, the Z2M state payload arrives before HA’s light state settles.  
Try increasing the delay in restore step:

```yaml
- delay: "00:00:02"

