# Dashboard Sentinel (Beta)

A dashboard for your Sentinels: a status board that shows what has gone quiet and what is running low across your whole house, and the optional aggregator sensor that some of those dashboards need.

This is the first version of a dashboard layer for the Sentinels, and it is as much a question as it is an answer. Most of the reasoning below is about why the aggregator is built the way it is and where it could go, because that is the part I most want your thoughts on.

**This is in active development. I make no warranty, and I am not even claiming it works. It is a direction I am exploring and asking for help on, not a finished tool. Do not rely on it for anything. Use at your own risk.**

## Putting it on a Dashboard

This is the payoff, and for most people it is the whole reason to be here. Here is the status board shown below, built from stock `markdown` cards so it is portable and needs nothing installed beyond the one layout helper noted after. Each card reads your Sentinel sensors directly and shows a calm "all healthy" state when there is nothing to report and a formatted list when there is.

![Sentinel Status dashboard showing one low battery and three quiet entities, arranged as a header summary above two columns with a footer](https://xeazy.com/wp-content/uploads/sentinel-status-detected-todo.webp)

The four cards are shown one at a time below. The responsive arrangement, the header and footer spanning the full width with the two columns side by side on a wide screen and stacked on a phone, is done with [layout-card](https://github.com/thomasloven/lovelace-layout-card), the one HACS helper this board depends on. In a `custom:grid-layout` view each card also carries a `view_layout` line naming its grid area (`header`, `battery`, `entity`, `footer`); drop a card into an ordinary dashboard without layout-card and it still renders, it simply stacks in normal order. The card content below is what matters; the layout is yours to arrange.

The Batteries column reads `sensor.battery_sentinel`. The Entities column reads whatever entity monitor you have: point it at a single Entity Sentinel if you run one, or at the aggregator (`sensor.dashboard_sentinel`) if you run several and combined them. The `devices` shape is the same either way, so the same card works whichever you feed it.

### Header: the one-line summary

Reads both sensors and states the situation in a sentence: all clear when both are healthy and empty, or the two counts when something needs attention.

```yaml
type: markdown
content: |
  {%- set b = states('sensor.battery_sentinel') | int(0) %}
  {%- set e = states('sensor.dashboard_sentinel') | int(0) %}
  {%- set bok = is_state_attr('sensor.battery_sentinel', 'ok', true) %}
  {%- set eok = is_state_attr('sensor.dashboard_sentinel', 'ok', true) %}
  # 🛡️ Sentinel Status

  {% if b == 0 and e == 0 and bok and eok -%}
  ### All systems healthy. Nothing needs attention.
  {%- else -%}
  ### {{ b }} {{ 'battery' if b == 1 else 'batteries' }} low&nbsp;&nbsp;·&nbsp;&nbsp;{{ e }} {{ 'entity' if e == 1 else 'entities' }} quiet
  {%- endif %}
```

### Batteries column

Reads `sensor.battery_sentinel`. When batteries are low it lists each by name, level, type if the device exposes one, and area. When none are low it shows a green check and a short reassurance. The last line shows when the sensor last evaluated, so the refresh button below the board has a visible effect.

```yaml
type: markdown
content: |
  ## 🔋 Batteries

  ---

  {% set low = state_attr('sensor.battery_sentinel', 'devices') -%}
  {% if low %}
  {% for d in low -%}
  ### {{ d.name }}
  {{ d.level }}%{% if d.battery_type %} · {{ d.battery_type }}{% endif %} · *{{ d.area }}*

  {% endfor -%}
  {% else %}
  <ha-icon icon="mdi:check-circle" style="color: var(--success-color, #4caf50); --mdc-icon-size: 32px;"></ha-icon>

  ### All batteries healthy

  Every monitored battery is above its threshold.
  {% endif %}
  {%- set t = state_attr('sensor.battery_sentinel', 'last_evaluated') %}
  <div style="opacity: 0.55; font-size: 0.8em; margin-top: 8px;">{% if t and t not in ['unknown', 'unavailable'] %}Last checked {{ (t | as_datetime | as_local).strftime('%-I:%M:%S %p') }}{% else %}Last checked: unknown{% endif %}</div>
```

### Entities column

Reads your entity monitor. Point it at a single Entity Sentinel, or at the aggregator if you built one. When entities have gone quiet it lists each by name, reason, age, and area. When all are reporting it shows the same green check. The last line shows when the sensor last evaluated.

```yaml
type: markdown
content: |
  ## 📡 Entities

  ---

  {% set quiet = state_attr('sensor.dashboard_sentinel', 'devices') -%}
  {% if quiet %}
  {% for d in quiet -%}
  ### {{ d.name }}
  {{ d.reason }}{% if d.age and d.age != 'unknown' %} · {{ d.age }}{% endif %} · *{{ d.area }}*

  {% endfor -%}
  {% else %}
  <ha-icon icon="mdi:check-circle" style="color: var(--success-color, #4caf50); --mdc-icon-size: 32px;"></ha-icon>

  ### All entities reporting

  Every monitored entity is alive and current.
  {% endif %}
  {%- set t = state_attr('sensor.dashboard_sentinel', 'last_evaluated') %}
  <div style="opacity: 0.55; font-size: 0.8em; margin-top: 8px;">{% if t and t not in ['unknown', 'unavailable'] %}Last checked {{ (t | as_datetime | as_local).strftime('%-I:%M:%S %p') }}{% else %}Last checked: unknown{% endif %}</div>
```

### Footer

A refresh strip and a quiet attribution line that closes the board. The two tiles press the refresh buttons your Sentinels use, so you can force a re-evaluation before you trust the board; the "Last checked" line in each column updates to confirm it ran. Point each tile at whatever `input_button` you set as the refresh button on your own Sentinels.

```yaml
type: vertical-stack
cards:
  - type: horizontal-stack
    cards:
      - type: tile
        entity: input_button.battery_sentinel
        name: Check batteries now
        icon: mdi:refresh
        tap_action:
          action: perform-action
          perform_action: input_button.press
          target:
            entity_id: input_button.battery_sentinel
      - type: tile
        entity: input_button.entity_sentinel
        name: Check entities now
        icon: mdi:refresh
        tap_action:
          action: perform-action
          perform_action: input_button.press
          target:
            entity_id: input_button.entity_sentinel
  - type: markdown
    content: |
      <div style="text-align: center; opacity: 0.6; font-size: 0.85em;">Battery Sentinel + Dashboard Sentinel &nbsp;·&nbsp; updates live</div>
```

## Do you need the aggregator?

The rest of this page is about the aggregator sensor, `sensor.dashboard_sentinel`. Whether you need it depends on what you run.

**If you use Battery Sentinel, you do not need it.** One Battery Sentinel covers the whole house, so there is nothing to combine. Take the Batteries card above and you are done.

**If you run a single Entity Sentinel, you do not need it either.** Point the Entities card straight at that sensor. There is nothing to aggregate.

**If you run several Entity Sentinels and want them in one list, this is for you.** To reliably catch frozen entities without false reports, many homes run more than one Entity Sentinel (different scopes, freeze windows, and cadences, for example a lively tier, a quiet tier, and a sleepy tier). A dashboard card cannot merge several sensors' lists, remove duplicates, and sort the result, so the aggregator does that for you and hands the card one clean list.

**If you run several but want to keep them separate, you do not need it.** If you would rather see your lively, quiet, and sleepy tiers as their own columns, so you can tell at a glance which tier a failure fell in, skip the aggregator and give each Entity Sentinel its own card. Aggregation is a choice, not a requirement.

So the aggregator is optional for everyone. It earns its place in exactly one case: you run several Entity Sentinels and want a single unified list rather than several separate ones. If that is you, the rest of this page is how it works and how to build it.

## What the aggregator does

You point it at your Entity Sentinel sensors. It reads each one's already-built list of gone-quiet entities, merges them, removes anything flagged by two Sentinels at once (keeping the more serious reason), sorts the result so it groups by domain, and presents it as one sensor. A dashboard card then reads that one sensor instead of three or five. It carries a health view of its sources alongside the list: a real Entity Sentinel that is currently broken is reported as a down source while the rest still combine, and a wrong input (a Battery Sentinel added by mistake, a plain sensor, another aggregate, or a typo) is surfaced as a rejected input rather than silently dropped.

That is the whole job. It does not watch anything itself; the Entity Sentinels do the watching, and this only combines their results for display.

## Why it is built this way, and the honest reasoning

First, this template sensor is a workaround, a good one, but still a workaround.

The Sentinels expose clean, structured information, so building a dashboard from them is straightforward. _Battery Sentinel_ needs only one instance in most homes, so its card is simple. _Entity Sentinel_ is different: to reliably detect frozen entities and not false report, your home will require several (different scopes, different freeze windows, different cadences), and putting each one in its own card makes a poor dashboard. The same entity can appear in two cards at once, the lists sit in separate boxes instead of one view, and there is no single ordered list of everything that is quiet.

The fix would be to combine them in the card, but a dashboard card cannot union several sensors' lists, remove the duplicates, and alphabetize the result. That has to happen outside the card. So I moved the combine-and-dedupe logic out of the card and into a new template sensor. That is what this blueprint is: the merge that I could not do in the dashboard, done in a sensor instead, so the card only has to render one already-clean list. The sensor is the means; the dashboard is the end.

This is why it identifies itself as an aggregate (`sentinel_type: entity`, `sentinel_role: aggregate`). It is a faithful mirror of an _Entity Sentinel_, same state meaning, same `devices` shape, so any card or automation reads it exactly like a single _Entity Sentinel_. The `aggregate` role marks that it is the combined one, so anything that walks your _Entity Sentinels_ can include it or leave it out and never double count.

## The decisions that fell out of that

A few choices follow directly from "this is a sensor that exists to feed a dashboard."

It validates its inputs by marker, and surfaces mistakes instead of hiding them. Because a person points it at "their Entity Sentinels" by hand, they might add the wrong thing. So each source is checked: a healthy _Entity Sentinel_ is merged, a broken one is reported as down, and anything that is not an _Entity Sentinel_ is rejected and named, with the reason, so a misconfiguration shows up on the dashboard rather than quietly doing nothing.

A broken source never silences the healthy ones. If one _Entity Sentinel_ is in a setup error, the others still combine, and the broken one is reported separately. The list you can build is always built; the part you cannot see is reported as missing, not pretended away.

It does no detection of its own, on purpose. Each _Entity Sentinel_ has already done the work and published it. This sensor only reads those finished lists and combines them, so it runs no entity scan and no detection engine. It re-evaluates only when one of its named sources changes. Keeping it a light reader, not a second engine, was a deliberate line.

Down sources and rejected inputs are kept as two separate lists. A down monitor is an operational problem that will clear on its own; a rejected input is a configuration mistake that persists until you fix it. They are different kinds of trouble for different audiences, so the dashboard can treat them differently.

## Import the blueprint

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FTheThinkingHome%2FAutomations%2Fmain%2Fblueprints%2Ftemplate%2Fdashboard_sentinel.yaml)

Or paste this URL into Settings, Automations & Scenes, Blueprints, Import Blueprint:

```
https://raw.githubusercontent.com/TheThinkingHome/Automations/main/blueprints/template/dashboard_sentinel.yaml
```

## Parameters

| Input | Required | Default | What it does |
| --- | --- | --- | --- |
| Entity Sentinel sensors | Yes | none | The Entity Sentinel sensors to combine. Add as many as you like. Only healthy Entity Sentinels are merged; a broken one is reported as a down source; a wrong input is surfaced as rejected. |
| Sensor name | No | Dashboard Sentinel | The name for the sensor this blueprint creates. |
| Unique ID | Yes | none | A unique id so the sensor is registered and you can rename it or place it in an area from the interface. |
| Debug logging | No | off | Writes one diagnostic line per evaluation, naming the merged count and any down or rejected sources. Needs a `logger` entry at info level to appear. |

## Attributes

The point of this sensor is to be read, so these are the interface a card or automation consumes.

| Attribute | What it carries |
| --- | --- |
| state | The deduped count of gone-quiet entities across all accepted sources. |
| `devices` | The merged, deduped, entity-id-sorted list. Each entry mirrors Entity Sentinel: `name`, `entity_id`, `area`, `reason`, `since`, `last_seen`, `age`. |
| `ok` | `true` only when the setup is valid and every source was accepted (none down, none rejected). |
| `error` | Empty when the setup is valid. On a setup error (no sources selected, or the same source listed twice) it carries the diagnosis and the fix, and the sensor's state reads `setup_error` with all outputs empty. |
| `unavailable_count` / `frozen_count` | Sub-counts within `devices`. |
| `sources` | The valid Entity Sentinels that are currently down, each with `sensor`, `name`, `status` (`setup_error`, `unavailable`), and `error`. Empty when all sources are healthy. |
| `rejected` | The misconfigured inputs, each with `sensor`, `name`, `status` (`not_a_sentinel`, `wrong_family`, `aggregate`, `missing`), and a `note` explaining the problem. Empty when all inputs are valid. |
| `source_error_count` / `rejected_count` | Quick counts for a badge or an automation gate. |
| `total_monitored` | The number of accepted sources. |
| `sentinel_type` / `sentinel_role` | `entity` / `aggregate`, the identity markers. |
| `sentinel_version` | The running version. |
| `last_evaluated` | ISO-8601 timestamp of the last evaluation. |

## A sample configuration

This is a template blueprint, so it is normally set up in the UI. If you prefer YAML, a `use_blueprint` entry looks like this (the `path` is relative to your `config/blueprints/template/` folder and reflects where the blueprint was saved on import):

```yaml
template:
  - use_blueprint:
      path: dashboard_sentinel.yaml
      input:
        source_sensors:
          - sensor.entity_sentinel_lively
          - sensor.entity_sentinel_quiet
          - sensor.entity_sentinel_sleepy
        sensor_name: Dashboard Sentinel
        unique_id: dashboard_sentinel
        debug_enabled: false
```

Once it is built, point the Entities card above at `sensor.dashboard_sentinel` instead of an individual Entity Sentinel, and the board shows the combined list.

## What I am asking for

This is a beta. It graduated from alpha after an adversarial torture suite of 27 scenario checks, which found and fixed one real bug on the way (a down source misdiagnosed as not-a-Sentinel), and it runs live on my own system combining three tiers. The aggregator's shape, one deduped mirror sensor, has settled; what beta still asks of you is time in varied homes and your read on where the dashboard layer should go next.

So, plainly:

Ideas and refinements are welcome. If you see a cleaner way to get a unified, deduped Sentinel list onto a dashboard, I want to hear it. If the attribute shape is missing something your card needs, tell me.

If you think this should not be a sensor at all, say so. I built an aggregate sensor because I could not combine and dedupe inside the dashboard. If there is a card-side approach I missed that does it properly, that would be the better answer, and I would happily retire this in favor of it. If you have done union-and-dedupe across attribute lists in a card and it held up, I want to see how.

And the bigger one: if someone wants to take this over and build a proper HACS card for it, I think that is the real destination. A custom card could do the combine and dedupe itself, read the Sentinels directly, and skip the intermediate sensor entirely. That is the version I could not build at the dashboard layer, and it is very likely the right one. I would be happy to hand this project to someone who can make it simpler. If that is something you would want to own, please reach out.

## Changelog

This tracks the decisions and changes as the project develops, so the thinking behind each version is on the record.

| Version | Notes |
| --- | --- |
| 1.0.0-beta | Graduation from alpha, gated by an adversarial torture suite (27 checks) that found one real bug and drove two hardening rules. Down-source fix: an unavailable entity carries no attributes, so the identity checks read nothing and a genuinely down Entity Sentinel was misreported as `not_a_sentinel` in `rejected` instead of a down source in `sources`; the state check now comes first, so a dead or restarting source is truthfully reported as down. Fail-loud setup: an empty source list, or the same source listed more than once, now stops the sensor with state `setup_error`, `ok: false`, and a new `error` attribute carrying the diagnosis and the fix, naming each duplicated sensor, instead of a silent green board or a silently deduped duplicate that miscounted `total_monitored`. The `error` attribute joins the contract permanently, empty when healthy. Merge, dedupe, precedence, sort, and rejection behavior are otherwise unchanged from alpha.2 and re-proven by the suite. |
| 1.0.0-alpha.1 | First build. Reads any number of Entity Sentinel sensors and presents them as one deduped, sorted mirror. Validates each source by its marker attribute: a healthy Entity Sentinel is merged; a broken one is reported in the `sources` list while the rest still aggregate; a wrong input (a non-Sentinel, a wrong-family Sentinel such as Battery, another aggregate or itself, or a missing entity) is surfaced in the `rejected` list rather than silently dropped. Dedupes across sources with the genuinely-gone-beats-frozen precedence used by Sentinel Notify, sorts by entity id, and carries identity markers. Event-driven off the named sources and Home Assistant start, with no detection engine of its own. |

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
