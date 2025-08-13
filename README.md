# **The Thinking Home \- Home Assistant Configuration**

Welcome to my Home Assistant configuration repository. This collection contains the foundational automations, scripts, and templates that make my smart home a "Thinking Home"—one that is robust, predictable, and requires minimal manual intervention.

My philosophy is one of good stewardship over the technology we invite into our homes. Each configuration here is designed to be a reliable, set-and-forget solution that solves a practical problem with elegance and simplicity. Feel free to explore, adapt, and use these configurations in your own Home Assistant setup.

## **Foundational Configurations**

### **🏡 Automations**

Automations are the workhorses of Home Assistant, performing tasks in the background to make life easier and the home more responsive.

* [**Network Reboot Handler**](https://github.com/TheThinkingHome/Automations/blob/main/network_reboot_handler.yaml)  
  * A fault-tolerant automation that gracefully manages network-dependent integrations during a scheduled router reboot, preventing a cascade of errors and ensuring a stable recovery. [**Instructions**](https://xeazy.com/taming-the-reboot/)
* [**Two-Way Switch Sync**](https://github.com/TheThinkingHome/Automations/blob/main/two-way_switch_sync.yaml)  
  * A simple and robust automation to synchronize the state of two switches. It intelligently avoids infinite loops by ignoring changes made by other automations. [**Instructions**](https://xeazy.com/there-can-be-only-one/)

* [**Random_Voice_Notifications**](https://github.com/TheThinkingHome/Automations/blob/main/random_notifications.yaml)
  * Monitors a door sensor and uses a persistent loop to send randomized voice alerts if an obstruction is detected, stopping automatically when the condition is resolved. [**Instructions**](https://xeazy.com/taming-the-reboot/)

### **📜 Scripts**

Scripts are reusable sequences of actions that can be called from anywhere in Home Assistant, helping to keep automations clean and centralize complex logic.

* [**Notify All**](https://github.com/TheThinkingHome/Automations/blob/main/notify_all.yaml)  
  * A centralized notification script that simplifies sending rich notifications. It handles multiple users and devices (Android & iOS) and intelligently manages photo attachments from both local paths and external URLs. [**Instructions**](https://xeazy.com/one-script-to-notify-them-all/)

### **🧩 Template Sensors & Entities**

Template entities allow for the creation of custom sensors and other entities based on the state of existing ones. They are perfect for combining multiple data points into a single, meaningful state.

* [**Guarded Cover Template**](https://github.com/TheThinkingHome/Automations/blob/main/covers.yaml)  
  * A template for creating a "Guarded Cover" proxy entity. This pattern centralizes complex conditions (like checking for an open window) into a single "guard" sensor, simplifying your automations by separating the action from the condition. [**Instructions**](https://xeazy.com/the-gatekeeper/)
* [**Bedtime Detected Sensor**](https://github.com/TheThinkingHome/Automations/blob/main/templat_sensors.yaml)  
  * A sophisticated template sensor that uses a weighted confidence score to reliably infer a complex state like "Bedtime." It evaluates multiple conditions, each with a configurable weight, making it resilient to minor deviations in routine. [**Instructions**](https://xeazy.com/weighted-confidence-for-complex-states/)

Thank you for visiting. I hope you find these configurations helpful in your own smart home journey.

## **Resources**

* **The Book:** Learn more about the methodologies and philosophies behind these automations in **"The Thinking Home: A Practical Guide to Planning and Building a Reliable and Private Smart Home"**.  
* **The Website:** Join our vibrant community, find additional insights, and explore more resources at [XeazY.com](https://xeazy.com/). You can also read and sign The Thinking Home Manifesto.
