# The Thinking Home - Home Assistant Configuration

Welcome to my Home Assistant configuration repository. This collection contains the foundational automations, scripts, and templates that make my smart home in Loja, Ecuador, a "Thinking Home," one that is dependable, predictable, and requires minimal manual intervention.

My philosophy is one of good stewardship over the technology we invite into our homes. Each configuration here is designed to be a reliable, set-and-forget solution that solves a practical problem with elegance and simplicity. Feel free to explore, adapt, and use these configurations in your own Home Assistant setup.

## 📐 Blueprints

Blueprints are reusable templates you import once and use with different inputs, so the same tested logic can run in many places without copy-paste. They are grouped by the kind of entity they build. Each one is capable and built to do a real job well, with settings worth understanding, so the linked instructions are worth reading.

### Automation Blueprints

Watch for events and run actions in response.

* [Linked Entities Pro](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/linked_entities_pro.md)
   * Keeps two or more entities that control the same thing in sync, so the next press never lands wrong. The wall switch and the smart bulb it controls, the Zigbee switch and the Tasmota module wired to a pair of table lamps, an `input_boolean` standing in for a physical control: any number of entities, any mix of integrations, in one linked group, so a change on any of them propagates to the others. On Home Assistant restart, or when a bridge sensor reconnects, a designated authority entity resolves drift the same way every time. [Instructions](https://xeazy.com/linked-entities-pro/)
* [Sensor Watchdog](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sensor_watchdog.md)
   * Some smart devices fail silently: an Aqara FP2 locks up at its last reading, the entity sits there pretending everything is fine, and the only fix is a power cycle. Sensor Watchdog automates that with a smart plug. Point it at one or more monitored entities, a recovery switch, and a timer helper, and it cycles the plug when a monitored entity goes unavailable, stops reporting fresh values, or on a preventive reboot schedule. Pair multiple entities from a single device for better coverage: an FP2's presence and illuminance entities cover complementary parts of the day, and reports from either count as a heartbeat. [Instructions](https://xeazy.com/sensor-watchdog-blueprint/)
* [Sentinel Notify](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/automation/sentinel_notify.md)
   * A Battery Sentinel or Entity Sentinel sensor knows what is wrong, but it does not tell you, that is by design, each is a signal, not a notifier. Sentinel Notify is the companion built specifically to be their mouthpiece. Point it at one or more Sentinel sensors and it turns their output into a notification: the low batteries by name, level, and area, or the entities that have gone quiet, sent to any number of mobile devices. It notifies when the monitored set changes rather than on every cycle, so a battery that stays low does not nag you over and over. A separate alert fires if a source Sentinel itself stops working, so a broken monitor reports itself instead of failing silently. [Instructions](https://xeazy.com/battery-entity-sentinel-blueprints/)

### Template Blueprints

Build derived sensors and helpers from your existing entities.

* [Battery Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/battery_sentinel.md)
   * A house full of battery devices means a steady trickle of dying batteries, and the ones you forget are the ones that fail when you need them. Battery Sentinel watches the battery devices you choose and reports, as a sensor, which are running low against your chosen threshold. Its state is the count of devices with low batteries, and its attributes list each low device by name, level, and area. It is the sister of Entity Sentinel: this one answers "what is running low," the other answers "what stopped reporting." Build either or both, and pair them with Sentinel Notify to turn their signals into alerts. [Instructions](https://xeazy.com/battery-entity-sentinel-blueprints/)
* [Entity Sentinel](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/entity_sentinel.md)
   * Some devices do not go offline cleanly. They lock up at their last reading, and the entity sits there showing a healthy value while the device behind it has gone quiet. A scheduled whole-house scan that only looks for unavailable never sees it. Entity Sentinel watches the specific entities you choose and reports, as a sensor, any that have stopped reporting, both the ones marked unavailable and the ones frozen at a stale value. Its state is the count of stale or unavailable entities, and its attributes list exactly which entities are quiet, so a dashboard, an automation, or its notification companion can read the same signal. It is the sister of Battery Sentinel: this one answers "what stopped reporting," the other answers "what is running low." Pair either with Sentinel Notify to turn the signal into alerts. [Instructions](https://xeazy.com/battery-entity-sentinel-blueprints/)
* [Recently Active](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/recently_active.md)
   * Turns a fired-once event into a state you can ask about later. It builds a binary sensor that stays on while a source is on, and for a set number of seconds after it turns off. Point it at any on/off entity: a contact sensor, a motion sensor, a switch, an `input_boolean`. The usual job is keeping a motion-controlled light on, or suppressing a notification for a short window right after something happens, such as skipping a camera's "a person is at the front door" alert just after the door was opened. [Instructions](https://xeazy.com/how-to-use-a-home-assistant-blueprint-template-sensor-the-recently-active-sensor/)
* [Sensor Failover](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/sensor_failover.md)
   * A sensor going unavailable takes down whatever depends on it: an automation loses its reading and does the wrong thing, or simply stops working. Sensor Failover is a wrapper that reports your sensor's value while it is working, and the moment it goes unavailable, falls back to a backup sensor instead of going unavailable itself. You give it a primary and a list of fallbacks, point your automation at the wrapper, and the reading keeps coming even when the primary drops out. [Instructions](https://xeazy.com/sensor-failover-blueprint/)
* [System Stability](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/system_stability.md)
   * After every Home Assistant restart there is a window where things are settling: integrations stumble out of bed, Zigbee devices republish their state, presence sensors blink wildly trying to remember what a human looks like. System Stability builds a binary sensor that reads off during shutdown and through the settling window after each restart, then on once the system is fully stable. Add `state == 'on'` as a condition to any automation with a misfire risk during the boot window, and the misfire absorbs cleanly. One entity, one rule, no per-automation handlers to write. [Instructions](https://xeazy.com/system-stability-blueprint/)
* [Weighted Confidence](https://github.com/TheThinkingHome/Automations/blob/main/blueprints/template/weighted_confidence.md)
   * Some states are not one sensor's job. Whether the house is empty, whether a room is really in use, whether everyone has actually settled in for the night: each is a handful of weaker signals that mostly agree. Weighted Confidence takes a list of those signals, gives each a weight, and turns a binary sensor on when the agreeing weights cross a threshold. Signals that go unavailable drop out of the sum instead of dragging the score down, any signal can be made a hard gate that forces the answer off on its own, and the attributes show the running score and exactly which signals are helping and which are holding it back. It pairs with Recently Active: feed in a "door shut for ten minutes" sensor as one of the weighted signals. [Instructions](https://xeazy.com/weighted-confidence-a-home-assistant-weighted-sensor-now-a-blueprint/)

## Article Companion Code

The automations, scripts, and template entities below are the full code behind the articles on those subjects. They are learning tools, not products: code you can read, understand, and adapt to your own system, rather than import and deploy as-is. Each one links to the article that explains the thinking behind it.

### 🏡 Automations

* [Network Reboot Handler](https://github.com/TheThinkingHome/Automations/blob/main/network_reboot_handler.yaml)
   * An attempt at handling network-dependent integrations during a scheduled router reboot, so the reboot does not touch off a cascade of errors. It is unfinished and does not yet work reliably, and the approach may be flawed at the concept level. The idea is a good one and worth getting right, so the code is left here in the open in case someone wants to take it further. No write-up, since the approach is still unproven.
* [Two-Way Switch Sync](https://github.com/TheThinkingHome/Automations/blob/main/two-way_switch_sync.yaml)
   * A simple, reliable automation to synchronize the state of two switches. It intelligently avoids infinite loops by ignoring changes made by other automations. [Instructions](https://xeazy.com/there-can-be-only-one/)
* [Window_Cover_Safety](https://github.com/TheThinkingHome/Automations/blob/main/window_covers_safety.yaml)
   * Prevents motorized blinds from closing on open windows to avoid damage from wind or obstruction. [Instructions](https://xeazy.com/protecting-your-smart-home-from-itself/)
* [Random_Voice_Notifications](https://github.com/TheThinkingHome/Automations/blob/main/random_notifications.yaml)
   * Monitors a door sensor and uses a persistent loop to send randomized voice alerts if an obstruction is detected, stopping automatically when the condition is resolved. [Instructions](https://xeazy.com/home-assistant-automation-and-why-randomness-beats-generative-ai/)
* [Do Not Cook the Fish](https://github.com/TheThinkingHome/Automations/blob/main/do-not-cook-the-fish.yaml)
   * A worked example of the gated trigger problem and its fix. It positions dining and living room shades from outdoor lux, TV state, and open windows, re-checking the whole picture on every trigger so the shades never get stuck in the wrong place. Built to keep the afternoon sun off a west-facing fishbowl. [Instructions](https://xeazy.com/do-not-cook-the-fish/)
* [Light Control - Office Occupancy](https://github.com/TheThinkingHome/Automations/blob/main/light_control_office_occupancy.yaml)
   * The clean teaching version of the wasp-in-a-box occupancy pattern. A door closing while motion is active captures the occupant and holds the light on, and a physical switch press sets a manual override that motion will not fight. Uses the three-condition context filter to tell a human press from an automation. [Instructions](https://xeazy.com/home-assistant-occupancy-detection-the-wasp-in-a-box-pattern/)
* [Light Control - Master Shower Occupancy](https://github.com/TheThinkingHome/Automations/blob/main/light_control_master-shower_occupancy.yaml)
   * The same wasp-in-a-box occupancy logic applied to a wet room, with an optional exhaust fan tied to the door. Motion, door state, and a lux reading drive the light, and a closed door keeps it on while someone is inside even after motion stops. [Instructions](https://xeazy.com/advanced-home-assistant-automations/)
* [Litter Box Obstruction](https://github.com/TheThinkingHome/Automations/blob/main/main_bath_litter_box_obstruction.yaml)
   * Watches a bathroom door that blocks the cats from their litter box when it stays shut. After 30 minutes closed it flags an obstruction, sends a notification, and plays a randomized voice nag every few minutes until someone opens the door. [Instructions](https://xeazy.com/home-assistant-automation-and-why-randomness-beats-generative-ai/)

### 📜 Scripts

* [Notify All](https://github.com/TheThinkingHome/Automations/blob/main/notify_all.yaml)
   * A centralized notification script that simplifies sending rich notifications. It handles multiple users and devices (Android & iOS) and intelligently manages photo attachments from both local paths and external URLs. [Instructions](https://xeazy.com/one-script-to-notify-them-all/)

### 🧩 Template Sensors & Entities

* [Guarded Cover Template](https://github.com/TheThinkingHome/Automations/blob/main/covers.yaml)
   * A template for creating a "Guarded Cover" proxy entity. This pattern centralizes complex conditions (like checking for an open window) into a single "guard" sensor, simplifying your automations by separating the action from the condition. [Instructions](https://xeazy.com/the-gatekeeper/)
* [Bedtime Detected Sensor](https://github.com/TheThinkingHome/Automations/blob/main/template_sensors.yaml)
   * A sophisticated template sensor that uses a weighted confidence score to reliably infer a complex state like "Bedtime." It evaluates multiple conditions, each with a configurable weight, making it resilient to minor deviations in routine. [Instructions](https://xeazy.com/weighted-confidence-for-complex-states/)

Thank you for visiting. I hope you find these configurations helpful in your own smart home journey.

## Resources

* The Book: Learn more about the methodologies and philosophies behind these automations in "The Thinking Home: A Practical Guide to Planning and Building a Reliable and Private Smart Home".
* The Website: Join our vibrant community, find additional insights, and explore more resources at [XeazY.com](https://xeazy.com/). You can also read and sign The Thinking Home Manifesto.

## License

Copyright (C) 2026 James Lander, The Thinking Home (https://xeazy.com)

Licensed under the GNU General Public License v3.0 or later (GPL-3.0-or-later). You may use, modify, and redistribute these configurations under those terms, with no warranty. See the LICENSE file for the full text. If you adapt or redistribute any of them, keep the copyright and license notices intact.
