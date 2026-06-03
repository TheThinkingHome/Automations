# Weighted Confidence

A Home Assistant template blueprint that infers a fuzzy state from many signals at once, each carrying its own weight, and turns on when enough of them agree.

Some states are not one sensor's job. "Is the house at bedtime" is not a single switch, it is the door having been shut a while, the TV off, a phone on its charger, nobody moving about. Any one of those can be wrong on a given night without changing the answer. Weighted Confidence takes a list of signals like that, gives each a weight, and turns on when the agreeing weight crosses a line you set. It pairs naturally with the Recently Active blueprint: anything time-based, such as "the door has been shut for ten minutes," becomes one of the signals you feed in.

Full write-up and worked examples: https://xeazy.com/REPLACE-WITH-ARTICLE-SLUG/
Questions and discussion: https://xeazy.com/logbook

## What you get

- A `binary_sensor` that turns on when the weighted share of agreeing signals reaches your threshold.
- Each signal carries its own weight and its own idea of what counts as agreement.
- Signals that go unavailable drop out of the sum instead of dragging the score down.
- A `required` flag for any signal that should act as a hard gate.
- Attributes that show the running score and name exactly which signals are helping and which are holding it back.

## How the scoring works

Each signal carries a weight and a state that counts as agreement. The sensor looks at every signal that is currently available, adds up the weight of the ones that agree, and divides by the total weight of all the available ones. When that share reaches your threshold, it turns on.

The word available is doing real work there. If a signal goes unavailable or unknown, it drops out of both the top and the bottom of the sum, so a dead sensor lowers the bar to clear rather than counting as a vote against. The signals still reporting decide it between themselves.

One signal can be promoted to a hard gate with the `required` flag. If a required signal is available and does not agree, the sensor is off, whatever weight the rest pile up in favour. Mark a settled door required and an open door keeps the answer off no matter what the bedroom says.

## Requirements

Home Assistant 2026.5.4 or newer. That is the version it is verified on.

## Step 1: Import the blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fweighted_confidence.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:
   ```
   https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/weighted_confidence.yaml
   ```
4. Confirm and import.

Importing only registers the blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/<author>/weighted_confidence.yaml`, typically `TheThinkingHome/weighted_confidence.yaml`. Open that folder and note the exact `<author>` subfolder name Home Assistant assigned, because you need it in the next step.

## Step 2: The catch with template blueprints

Automation blueprints get a friendly "Create automation" button right on the Blueprints page. Template blueprints do not. You will not find Weighted Confidence anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`:

```yaml
use_blueprint:
  path: TheThinkingHome/weighted_confidence.yaml
  input:
    unique_id: house_at_bedtime
    sensor_name: House At Bedtime
    threshold_percent: 80
    signals:
      - entity: binary_sensor.front_door_recently_used
        state: "off"
        weight: 4
        name: Front door settled
        required: true
      - entity: binary_sensor.living_room_tv
        state: "off"
        weight: 2
        name: TV off
      - entity: binary_sensor.phone_charging
        weight: 3
        name: Phone charging
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. The inputs are explained near the end.

That block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that scales once you make more of these, is a package.

## Step 3: What a package is, and how to use one

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering a sensor here and an automation there across your main files, you put everything for one feature into one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In your `configuration.yaml`, add the two lines below. If you already have a `homeassistant:` section, just add the `packages:` line underneath it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then create a folder named `packages` next to your `configuration.yaml`. Every YAML file you drop in there is now a package.

Now make a package file, for example `packages/house_at_bedtime.yaml`, and put your `use_blueprint` block inside a `template:` list:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/weighted_confidence.yaml
      input:
        unique_id: house_at_bedtime
        sensor_name: House At Bedtime
        threshold_percent: 80
        signals:
          - entity: binary_sensor.front_door_recently_used
            state: "off"
            weight: 4
            name: Front door settled
            required: true
          - entity: binary_sensor.living_room_tv
            state: "off"
            weight: 2
            name: TV off
          - entity: binary_sensor.phone_charging
            weight: 3
            name: Phone charging
```

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Step 4: Load it

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks or editing your signal list only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a `binary_sensor` named after your `sensor_name`. Watch it in Developer Tools, States, where its attributes show you the score and which signals are pulling their weight.

## The inputs

- **Signals** (`signals`), required: the list of signals to weigh. Each item is described in its own section below.
- **Threshold** (`threshold_percent`): the share of currently available weight that must agree before the sensor turns on. Default 80.
- **Sensor name** (`sensor_name`): the name for the binary sensor this creates, and what its entity id is built from. Default "Weighted Confidence".
- **Unique ID** (`unique_id`), required: a short id string, distinct for every sensor you build, such as `house_at_bedtime`. It registers the entity so you can rename it, place it in an area, or change its device class from the interface.
- **Device class** (`device_class`): how the sensor reads in the frontend. Default `occupancy`, which shows Detected and Clear. Pick another to suit what the sensor means, or clear it for plain On and Off.

## The signals list, field by field

Each item in `signals` accepts up to five keys. Two are needed, three are optional:

```yaml
- entity: binary_sensor.front_door_recently_used   # required
  weight: 4                                         # required
  state: "off"                                      # optional, default "on"
  name: Front door settled                          # optional, default the entity id
  required: true                                    # optional, default false
```

- **entity**: the entity to read. Anything with a state works, not only binary sensors.
- **weight**: how much this signal counts toward the score. Bigger means more say. Defaults to 1 if you leave it off.
- **state**: the value that counts as agreement. Defaults to `"on"`. It can be a single value, such as `"off"` or `"charging"`, or a list of values, such as `["Sleep", "Wake"]`, in which case any of them counts.
- **name**: a label for the attributes, so the contributing and not_met lists read in plain language instead of entity ids. Defaults to the entity id.
- **required**: set it true to make this signal a hard gate. An available required signal that does not agree forces the sensor off regardless of the score.

## Time-based signals live elsewhere

This blueprint scores states, it does not watch the clock. For a signal like "the door has been shut for ten minutes" or "no movement in the last twenty," build that with the [Recently Active blueprint](recently_active.md) first, then hand its result in here as one more signal. Point Recently Active at the door with a ten minute linger, and remember that its `off` state is the settled one, so the signal you add here reads `state: "off"`. Keeping the timing in its own sensor leaves this one a clean scorer.

## Reading the attributes

The sensor carries six attributes, and they turn tuning from guesswork into reading:

- **confidence**: the current agreeing share, as a number out of 100.
- **score**: the agreeing weight over the available weight, such as `14 / 18`.
- **threshold**: the percentage it is checking against, echoed back.
- **contributing**: the names of the signals currently agreeing.
- **not_met**: the names of available signals that are not agreeing.
- **unavailable**: the entities that have dropped out because they are unavailable or unknown.

When the sensor stubbornly stays off, `not_met` tells you why in one glance, and `confidence` tells you how close you were. When it turns on too easily, `contributing` shows you which light signals are carrying it and want more weight on the firmer ones.

## Example use

A "house at bedtime" sensor is the natural fit. It leans on a Recently Active sensor for the settled door, then weighs that against the everyday signs that the day is over.

First the door feed, a Recently Active sensor with a ten minute linger:

```yaml
template:
  - use_blueprint:
      path: TheThinkingHome/recently_active.yaml
      input:
        source_entity: binary_sensor.front_door
        unique_id: front_door_recently_used
        linger_seconds: 600
        sensor_name: Front Door Recently Used
        device_class: door
```

Then the weighted sensor that reads it:

```yaml
  - use_blueprint:
      path: TheThinkingHome/weighted_confidence.yaml
      input:
        unique_id: house_at_bedtime
        sensor_name: House At Bedtime
        threshold_percent: 80
        signals:
          - entity: binary_sensor.front_door_recently_used
            state: "off"
            weight: 4
            name: Front door settled
            required: true
          - entity: input_select.house_mode
            state: ["Night", "Sleep"]
            weight: 5
            name: House set to night
          - entity: binary_sensor.phone_charging
            weight: 3
            name: Phone charging
          - entity: light.nightstand_lamp
            state: "off"
            weight: 3
            name: Nightstand lamp off
          - entity: binary_sensor.living_room_tv
            state: "off"
            weight: 2
            name: TV off
          - entity: binary_sensor.house_recently_active
            state: "off"
            weight: 1
            name: No recent movement
```

The settled door is required, so an open front door keeps the answer off however sleepy everything else looks. Everything else is weighed, and once the agreeing share clears 80 percent the sensor turns on. Hang your goodnight scene, your away checks, or your overnight automations off that single sensor, and they all share one honest read on whether the house has actually turned in.

## License

Use it, adapt it, share it. A link back to https://xeazy.com is appreciated but not required.
