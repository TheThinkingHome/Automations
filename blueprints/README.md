# The Thinking Home Blueprints

A collection of Home Assistant blueprints from [xeazy.com](https://xeazy.com). Each one is useful, kept deliberately simple, and tested on a real instance before it ships. Questions and discussion live on the forum: https://xeazy.com/logbook

## Blueprints

### Recently Active

Home Assistant triggers fire once, at the instant a state crosses a line, and then the moment is gone. Recently Active holds onto it. It builds a binary sensor that stays `on` while your source is on, and for a chosen number of seconds after it turns off, so a fired-and-gone event becomes a state your other automations can still ask about. Point it at a door contact, a motion sensor, a switch, anything that reads on or off. The usual job is suppressing a notification or an analysis for a short window right after something happens, such as skipping a camera's "a person walked in" alert for a minute after the door opens.

**[Full directions and one-click import ->](template/recently_active.md)**

### Weighted Confidence

Some states are not one sensor's job. Whether the house is at bedtime, whether a room is really in use, whether everyone has actually settled: each is a handful of weaker signals that mostly agree. Weighted Confidence takes a list of those signals, gives each its own weight, and turns a binary sensor on when the agreeing weight crosses a threshold you set. Signals that go unavailable drop out of the sum instead of dragging the score down, any signal can be made a hard gate that forces the answer off on its own, and the attributes show you the running score and exactly which signals are helping and which are holding it back. It pairs with Recently Active: feed in a "door shut for ten minutes" sensor as one of the weighted signals.

**[Full directions and one-click import ->](template/weighted_confidence.md)**

---

Everything below is the general guide for importing and using any template blueprint here: how one is imported, where it lives, and how you turn it into a working entity. Each blueprint's own page, linked above, adds the specifics unique to it.

## How the blueprints are organized

They live under `blueprints/`, sorted by the kind of entity they build: `template/` for template entities such as sensors and binary sensors, with `automation/` and `script/` to follow as the collection grows. Home Assistant works out which kind a blueprint is from inside the file, so the folder is only for tidiness.

## Importing a blueprint

Every blueprint page has a one-click import button. To do it by hand instead:

1. In Home Assistant, open Settings, then Automations & Scenes, then the Blueprints tab.
2. Click Import Blueprint at the bottom right.
3. Paste the blueprint's raw URL, listed on its page, and click Preview.
4. Confirm and import.

Importing only registers the blueprint. It does not create anything yet. A template blueprint lands at `config/blueprints/template/<author>/<name>.yaml`. Open that folder and note the exact `<author>` subfolder Home Assistant assigned, because you point at it when you build an entity.

## Turning a template blueprint into an entity

This is the step that catches people. Automation blueprints get a Create automation button on the Blueprints page. Template blueprints do not, and they never show up in the Create Helper list. That is expected, not a fault. You build a template entity from a blueprint with a short piece of YAML called `use_blueprint`:

```yaml
use_blueprint:
  path: <author>/<name>.yaml
  input:
    # the settings this blueprint asks for, listed on its page
```

`path` is relative to `config/blueprints/template/`, so it is the `<author>` folder plus the filename. The `input:` keys change from one blueprint to the next, and each blueprint's page spells out its own.

That `use_blueprint` block has to sit under Home Assistant's `template:` configuration. The tidy way to do that, and the way that holds up once you have several, is a package.

## What a package is, and how to use one

A package is a single YAML file that holds a bundle of related configuration. Rather than scattering entities across your main files, you keep everything for one purpose in one file, and Home Assistant folds it into your overall configuration at startup. Packages are optional, but they keep related things together and easy to find.

If you have never used packages, switch them on once. In `configuration.yaml`, add the two lines below. If a `homeassistant:` section already exists, just add the `packages:` line under it:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Then make a folder named `packages` next to your `configuration.yaml`. Every file you drop in there is a package.

Now create a file such as `packages/blueprints.yaml` and gather your `use_blueprint` blocks under one `template:` list:

```yaml
template:
  - use_blueprint:
      path: <author>/<name>.yaml
      input:
        # this entity's settings

  - use_blueprint:
      path: <author>/<another>.yaml
      input:
        # another entity's settings
```

Any mix of blueprints can share that list. One that builds a binary sensor and one that builds a sensor sit side by side, because each blueprint decides what kind of entity it makes.

If you already keep template entities in `configuration.yaml` under a `template:` key, you can drop a `use_blueprint` block into that list instead. The package is simply the cleaner home once you have a few.

## Loading your changes

A brand-new package file needs a full Home Assistant restart to register the first time. After that, adding more `use_blueprint` blocks to the same file only needs Developer Tools, YAML, Reload Template Entities.

## License

Copyright (C) 2026 James Lander, The Thinking Home (https://xeazy.com) This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
