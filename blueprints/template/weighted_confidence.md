# Weighted Confidence (Beta)

A Home Assistant template blueprint that infers a fuzzy state from many signals at once, each carrying its own weight, and turns on when enough of them agree.

Some states are not one sensor's job. "Is the house at bedtime" is not a single switch, it is the door having been shut a while, the TV off, a phone on its charger, the room dark, nobody moving about. Any one of those can be wrong on a given night without changing the answer. Weighted Confidence takes a list of signals like that, gives each a weight, and turns on when the agreeing weight crosses a line you set. It pairs naturally with the Recently Active blueprint: anything time-based, such as "the door has been shut for ten minutes," becomes one of the signals you feed in.

This one is a beta. It is tested and it works, but the way you describe a signal may still change between beta releases, so treat a config you build today as something you might revisit when a new version lands.

Full write-up and worked examples: https://xeazy.com/REPLACE-WITH-ARTICLE-SLUG/
Questions and discussion: https://xeazy.com/logbook

## What you get

- A `binary_sensor` that turns on when the weighted share of agreeing signals reaches your threshold.
- Each signal carries its own weight and its own idea of what counts as agreement, by text match or by number.
- A per-signal say in what an unavailable reading means: ignore it, treat it as agreeing, or treat it as disagreeing.
- A `required` flag for any signal that should act as a hard gate.
- Attributes that show the running score and name exactly which signals are helping and which are holding it back.

## How the scoring works

Each signal carries a weight and a rule for what counts as agreement. The sensor looks at every signal that is currently available, adds up the weight of the ones that agree, and divides by the total weight of all the available ones. When that share reaches your threshold, it turns on.

The word available is doing real work there. By default, if a signal goes unavailable or unknown it drops out of both the top and the bottom of the sum, so a dead sensor lowers the bar to clear rather than counting as a vote against. You can override that per signal, telling one to count as agreeing or as disagreeing while it is out. More on that below.

One signal can be promoted to a hard gate with the `required` flag. If a required signal is available and does not agree, the sensor is off, whatever weight the rest pile up in favour. Mark a settled door required and an open door keeps the answer off no matter what the bedroom says.

## Requirements

Home Assistant 2026.5.4 or newer. That is the version it is verified on.

## Step 1: Import the blueprint

The quick way, one click:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fweighted_confidence_beta.yaml)

Or by hand:

1. Go to Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste this URL and click Preview:
   ```
   https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/weighted_confidence_beta.yaml
   ```
4. Confirm and import.

Importing only registers the blueprint. It does not create a sensor yet. After import, the file lands at `config/blueprints/template/<author>/weighted_confidence_beta.yaml`, typically `TheThinkingHome/weighted_confidence_beta.yaml`. Open that folder and note the exact `<author>` subfolder name Home Assistant assigned, because you need it in the next step.

## Step 2: The catch with template blueprints

Automation blueprints get a friendly "Create automation" button right on the Blueprints page. Template blueprints do not. You will not find Weighted Confidence anywhere in the Create Helper list, and that is expected, not a fault. Template blueprints become real entities through a short piece of YAML called `use_blueprint`:

```yaml
use_blueprint:
  path: TheThinkingHome/weighted_confidence_beta.yaml
  input:
    unique_id: house_at_bedtime
    sensor_name: House At Bedtime
    threshold_percent: 80
    signals:
      - entity: binary_sensor.front_door_recently_used
        state: "off"
        weight: 4
        required: true
        name: Front door settled
      - entity: binary_sensor.living_room_tv
        state: "off"
        weight: 2
        unavailable: agree
        name: TV off
      - entity: binary_sensor.phone_charging
        weight: 3
        name: Phone charging
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder from Step 1 plus the filename. Every parameter is laid out in the reference below.

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
      path: TheThinkingHome/weighted_confidence_beta.yaml
      input:
        unique_id: house_at_bedtime
        sensor_name: House At Bedtime
        threshold_percent: 80
        signals:
          - entity: binary_sensor.front_door_recently_used
            state: "off"
            weight: 4
            required: true
            name: Front door settled
          - entity: binary_sensor.phone_charging
            weight: 3
            name: Phone charging
```

If you already keep template entities in your `configuration.yaml` under a `template:` key, you can drop the `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Step 4: Load it

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks or editing your signal list only needs a quick reload through Developer Tools, YAML, Reload Template Entities.

When it comes up you will have a `binary_sensor` named after your `sensor_name`, with attributes that show the score and which signals are pulling their weight.

## Parameters

Two levels: the inputs you pass to the blueprint, and the fields on each signal inside the `signals` list.

### Blueprint inputs

| Input | Required | Default | Accepted values | What it does |
|---|---|---|---|---|
| `signals` | Yes | none | a list of signal items (fields in the next table) | The signals to weigh. |
| `threshold_percent` | No | `80` | a whole number from 1 to 100 | The share of currently available weight that must agree before the sensor turns on. |
| `sensor_name` | No | `Weighted Confidence` | any text | The friendly name, and what the entity id is built from. |
| `unique_id` | Yes | none | any short text id, distinct per sensor | Registers the entity so you can rename it, place it in an area, or change its device class from the interface. |
| `device_class` | No | `occupancy` | any binary_sensor device class (`occupancy`, `motion`, `door`, and so on), or left blank | How the sensor reads in the frontend. `occupancy` shows Detected and Clear; blank shows plain On and Off. |

### Signal fields

Each item in the `signals` list takes these. A signal needs only an `entity` and usually a `weight`; the rest shape how it is judged.

```yaml
- entity: binary_sensor.front_door_recently_used   # required
  weight: 4                                          # almost always set
  operator: equals                                   # optional, default equals
  state: "off"                                        # used by equals / not_equals
  unavailable: drop                                   # optional, default drop
  required: true                                      # optional, default false
  name: Front door settled                            # optional, default the entity id
```

| Field | Required | Default | Accepted values | What it does |
|---|---|---|---|---|
| `entity` | Yes | none | any entity id | The entity this signal reads. Anything with a state, not only binary sensors. |
| `weight` | No | `1` | any number, including `0` | How much the signal counts toward the score. `0` makes a pure gate that adds nothing but can still be `required`. |
| `operator` | No | `equals` | `equals`, `not_equals`, `above`, `below`, `between` | How agreement is judged. The first two compare text, the last three read the state as a number. |
| `state` | Used by `equals` and `not_equals` | `"on"` | a single value, or a list of values | The value or values that count as agreement under `equals`, or that disqualify under `not_equals`. |
| `value` | Used by `above` and `below` | none | a number | The number the entity's value is compared against. |
| `low` | Used by `between` | none | a number | The inclusive lower bound. |
| `high` | Used by `between` | none | a number | The inclusive upper bound. |
| `unavailable` | No | `drop` | `drop`, `agree`, `disagree` | What to do when the entity is unavailable or unknown. `drop` leaves it out, `agree` counts its weight as agreeing, `disagree` counts its weight against. |
| `required` | No | `false` | `true`, `false` | Makes the signal a hard gate. An available required signal that does not agree forces the sensor off, whatever the score. |
| `name` | No | the entity id | any text | The label shown in the `contributing` and `not_met` attributes, so they read in plain language. |

### Text and numbers

`equals` and `not_equals` compare the state as text, which suits on and off, charging, Home, and the like. The three numeric operators read the state as a number, which is how you bring in lux, battery percentage, temperature, or the hour of the day. A "room is dark" signal is `operator: below` with `value: 5` against a lux sensor. A "battery healthy" signal is `operator: above` with `value: 20`. A reading that is not a number simply does not agree.

### When a signal is unavailable

A signal that drops out costs you nothing: it leaves both sides of the sum, so the rest decide it between themselves. That is the right default for most signals. Reach for `agree` when no reading should be read as the quiet answer, the classic case being a TV that has fallen off the network, where "not reachable" really does mean "off." Reach for `disagree` for a signal whose silence should hold the result back rather than be waved through.

## Reading the attributes

The sensor carries six attributes, and they turn tuning from guesswork into reading:

- **confidence**: the current agreeing share, as a number out of 100.
- **score**: the agreeing weight over the available weight, such as `14 / 18`.
- **threshold**: the percentage it is checking against, echoed back.
- **contributing**: the names of the available signals currently agreeing.
- **not_met**: the names of available signals that are not agreeing.
- **unavailable**: the entities that are unavailable or unknown right now.

When the sensor stubbornly stays off, `not_met` tells you why in one glance, and `confidence` tells you how close you were. When it turns on too easily, `contributing` shows you which light signals are carrying it and want more weight on the firmer ones.

## Example use

A "house at bedtime" sensor is the natural fit. It leans on a Recently Active sensor for the settled door, then weighs that against the everyday signs that the day is over, including a couple of the newer tricks: a dark room read off a lux sensor, and a TV whose absence counts as off.

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
      path: TheThinkingHome/weighted_confidence_beta.yaml
      input:
        unique_id: house_at_bedtime
        sensor_name: House At Bedtime
        threshold_percent: 80
        signals:
          - entity: binary_sensor.front_door_recently_used
            state: "off"
            weight: 4
            required: true
            name: Front door settled
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
          - entity: media_player.living_room_tv
            state: "off"
            unavailable: agree
            weight: 2
            name: TV off
          - entity: sensor.bedroom_lux
            operator: below
            value: 5
            weight: 1
            name: Bedroom dark
          - entity: binary_sensor.house_recently_active
            state: "off"
            weight: 1
            name: No recent movement
```

The settled door is required, so an open front door keeps the answer off however sleepy everything else looks. The TV counts as off when it is unreachable rather than dropping out. The lux signal reads the room as dark below five. Everything else is weighed, and once the agreeing share clears 80 percent the sensor turns on. Hang your goodnight scene, your away checks, or your overnight automations off that single sensor, and they all share one honest read on whether the house has actually turned in.

## License

Use it, adapt it, share it. A link back to https://xeazy.com is appreciated but not required.
