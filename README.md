# The Thinking Home - Home Assistant Configuration

Welcome to my Home Assistant configuration repository. This collection contains the foundational automations, scripts, and templates that make my smart home a "Thinking Home"—one that is robust, predictable, and requires minimal manual intervention.

My philosophy is one of good stewardship over the technology we invite into our homes. Each configuration here is designed to be a reliable, set-and-forget solution that solves a practical problem with elegance and simplicity. Feel free to explore, adapt, and use these configurations in your own Home Assistant setup.

## Foundational Configurations

### 🏡 Automations

Automations are the workhorses of Home Assistant, performing tasks in the background to make life easier and the home more responsive.

* [Network Reboot Handler](https://github.com/TheThinkingHome/Automations/blob/main/network_reboot_handler.yaml)
   * A fault-tolerant automation that gracefully manages network-dependent integrations during a scheduled router reboot, preventing a cascade of errors and ensuring a stable recovery. [Instructions](https://xeazy.com/taming-the-reboot/)
* [Two-Way Switch Sync](https://github.com/TheThinkingHome/Automations/blob/main/two-way_switch_sync.yaml)
   * A simple and robust automation to synchronize the state of two switches. It intelligently avoids infinite loops by ignoring changes made by other automations. [Instructions](https://xeazy.com/there-can-be-only-one/)
* [Window_Cover_Safety](https://github.com/TheThinkingHome/Automations/tree/main)
   * Prevents motorized blinds from closing on open windows to avoid damage from wind or obstruction. Instructions
* [Random_Voice_Notifications](https://github.com/TheThinkingHome/Automations/blob/main/random_notifications.yaml)
   * Monitors a door sensor and uses a persistent loop to send randomized voice alerts if an obstruction is detected, stopping automatically when the condition is resolved. [Instructions](https://xeazy.com/taming-the-reboot/)
* [Do Not Cook the Fish](https://github.com/TheThinkingHome/Automations/blob/main/do-not-cook-the-fish.yaml)
   * A worked example of the gated trigger problem and its fix. It positions dining and living room shades from outdoor lux, TV state, and open windows, re-checking the whole picture on every trigger so the shades never get stuck in the wrong place. Built to keep the afternoon sun off a west-facing fishbowl. [Instructions](https://xeazy.com/why-your-home-assistant-automation-triggers-but-does-not-run/)
* [Light Control - Office Occupancy](https://github.com/TheThinkingHome/Automations/blob/main/light_control_office_occupancy.yaml)
   * The clean teaching version of the wasp-in-a-box occupancy pattern. A door closing while motion is active captures the occupant and holds the light on, and a physical switch press sets a manual override that motion will not fight. Uses the three-condition context filter to tell a human press from an automation. [Instructions](https://xeazy.com/REPLACE-WITH-WASP-IN-A-BOX-SLUG/)
* [Light Control - Master Shower Occupancy](https://github.com/TheThinkingHome/Automations/blob/main/light_control_master-shower_occupancy.yaml)
   * The same wasp-in-a-box occupancy logic applied to a wet room, with an optional exhaust fan tied to the door. Motion, door state, and a lux reading drive the light, and a closed door keeps it on while someone is inside even after motion stops. [Instructions](https://xeazy.com/REPLACE-WITH-WASP-IN-A-BOX-SLUG/)
* [Litter Box Obstruction](https://github.com/TheThinkingHome/Automations/blob/main/main_bath_litter_box_obstruction.yaml)
   * Watches a bathroom door that blocks the cats from their litter box when it stays shut. After 30 minutes closed it flags an obstruction, sends a notification, and plays a randomized voice nag every few minutes until someone opens the door. [Instructions](https://xeazy.com/REPLACE-WITH-LITTER-BOX-SLUG/)

### 📜 Scripts

Scripts are reusable sequences of actions that can be called from anywhere in Home Assistant, helping to keep automations clean and centralize complex logic.

* [Notify All](https://github.com/TheThinkingHome/Automations/blob/main/notify_all.yaml)
   * A centralized notification script that simplifies sending rich notifications. It handles multiple users and devices (Android & iOS) and intelligently manages photo attachments from both local paths and external URLs. [Instructions](https://xeazy.com/one-script-to-notify-them-all/)

### 🧩 Template Sensors & Entities

Template entities allow for the creation of custom sensors and other entities based on the state of existing ones. They are perfect for combining multiple data points into a single, meaningful state.

* [Guarded Cover Template](https://github.com/TheThinkingHome/Automations/blob/main/covers.yaml)
   * A template for creating a "Guarded Cover" proxy entity. This pattern centralizes complex conditions (like checking for an open window) into a single "guard" sensor, simplifying your automations by separating the action from the condition. [Instructions](https://xeazy.com/the-gatekeeper/)
* [Bedtime Detected Sensor](https://github.com/TheThinkingHome/Automations/blob/main/templat_sensors.yaml)
   * A sophisticated template sensor that uses a weighted confidence score to reliably infer a complex state like "Bedtime." It evaluates multiple conditions, each with a configurable weight, making it resilient to minor deviations in routine. [Instructions](https://xeazy.com/weighted-confidence-for-complex-states/)

### 📐 Blueprints

Blueprints are reusable templates you import once and use with different inputs, so the same tested logic can run in many places without copy-paste.

* [Recently Active](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/recently_active.md)
   * Turns a fired-once event into a state you can ask about later. It builds a binary sensor that stays on while a source is on, and for a set number of seconds after it turns off. Point it at any on/off entity: a contact sensor, a switch, an input_boolean. [Instructions](https://xeazy.com/how-to-use-a-home-assistant-blueprint-template-sensor-the-recently-active-sensor/)

Thank you for visiting. I hope you find these configurations helpful in your own smart home journey.

## Resources

* The Book: Learn more about the methodologies and philosophies behind these automations in "The Thinking Home: A Practical Guide to Planning and Building a Reliable and Private Smart Home".
* The Website: Join our vibrant community, find additional insights, and explore more resources at [XeazY.com](https://xeazy.com/). You can also read and sign The Thinking Home Manifesto.
