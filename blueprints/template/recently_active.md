# Recently Active

A Home Assistant template blueprint that turns a fired-and-forgotten event into a state you can still ask about a little while later.

Home Assistant triggers fire once, at the instant a state crosses a line. The door opens, the event fires, and it is gone. Ask a moment later whether the door was just opened and there is nothing left to read, because Home Assistant evaluated that crossing once and moved on. Recently Active closes the gap. It builds a binary sensor that reads `on` while your source is on, and keeps reading `on` for a chosen number of seconds after the source turns off. So "the door is open" quietly becomes "the door was open within the last N seconds," which is the question your other automations actually want to ask.

Full write-up and a worked example: https://xeazy.com/how-to-use-a-home-assistant-blueprint-template-sensor-the-recently-active-sensor/
Questions and discussion: https://xeazy.com/logbook/d/36-recently-active-blueprint

## What you get

- A `binary_sensor` that is `on` while the source is `on`, and for a set number of seconds after it turns off.
- Works with any on/off source: a `binary_sensor`, a `switch`, an `input_boolean`, anything that reads on or off.
- If the source goes unavailable or unknown, the sensor returns `off`, so a dead source never sticks the sensor on and never silently suppresses whatever depends on it.

One honest limit, stated up front so it does not surprise you: the off transition is rechecked on Home Assistant's once-a-minute clock tick, so the sensor can stay on up to about a minute past the window you set. The source turning off is caught instantly; only the window closing rides that minute tick. For second-precise timing or long windows, a timer helper is the better tool. This one is for light, short linger.

## Requirements

Home Assistant 2026.5.4 or newer. That is the version it is verified on.

## Step 1: Import the blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Frecently_active.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:
   ```
   https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/recently_active.yaml
   ```
4. Confirm and import.

Importing only registers the blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/<author>/recently_active.yaml`, typically `TheThinkingHome/recently_active.yaml`. Open that folder and note the exact `<author>` subfolder name Home Assistant assigned, because you need it in the next step.

## Step 2: The catch with template blueprints

Automation blueprints get a friendly "Create automation" button right on the Blueprints page. Template blueprints do not. You will not find Recently Active anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`. It is a path, your settings, and that is it:

```yaml
use_blueprint:
  path: TheThinkingHome/recently_active.yaml
  input:
    source_entity: binary_sensor.front_door
    linger_seconds: 60
    sensor_name: Front Door Recently Open
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. The three inputs are explained near the end.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales once you make more of these, is a package.

## Step 3: What a package is, and how to use one

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering a sensor here and an automation there across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/recently_active.yaml`, and put your `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: binary_sensor.front_door
        linger_seconds: 60
        sensor_name: Front Door Recently Open
```

Each entity you build from the blueprint is one more list item in the same file. They can watch different sources with different windows, all in one place:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: binary_sensor.front_door
        linger_seconds: 60
        sensor_name: Front Door Recently Open

  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: binary_sensor.hallway_motion
        linger_seconds: 30
        sensor_name: Hallway Recently Active
```

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Step 4: Load it

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a `binary_sensor` named after your `sensor_name`. Watch it in Developer Tools, States: flip the source on and it goes on, turn the source off and it holds on for your window, then drops to off.

## The inputs

- **Source entity** (`source_entity`): the on/off entity to track. A `binary_sensor`, `switch`, `input_boolean`, or anything that reads on or off.
- **Linger window** (`linger_seconds`): how many seconds the sensor stays on after the source turns off. Default 60.
- **Sensor name** (`sensor_name`): the name for the binary sensor this creates, and what its entity id is built from.

## A small note on display

The sensor is created without a device class, so it shows plain On and Off rather than themed labels such as Open and Closed. The underlying state value is on or off either way, so it reads correctly in templates and conditions regardless.

## Example use

A common one: an LLM analyzes snapshots from a front-door camera to flag unusual activity, but every ordinary entry trips the camera and the model dutifully reports a person walking through the door, which is noise. Point Recently Active at the door contact, and use its `on` state to suppress the analysis for a short window after the door is used. The full worked version is in the article linked at the top.

## License

Use it, adapt it, share it. A link back to https://xeazy.com is appreciated but not required.
