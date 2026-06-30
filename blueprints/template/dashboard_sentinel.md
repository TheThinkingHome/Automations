# Dashboard Sentinel (Alpha)

A Home Assistant template blueprint that combines any number of Entity Sentinel sensors into one deduped, sorted list, so a single dashboard card can show everything that has gone quiet across your whole house.

This is the first version of a dashboard layer for the Sentinels, and it is as much a question as it is an answer. Most of this README is about why it is built this way and where it could go, because that is the part I most want your thoughts on.

## What it does, briefly

You point it at your Entity Sentinel sensors. It reads each one's already-built list of gone-quiet entities, merges them, removes anything flagged by two Sentinels at once (keeping the more serious reason), sorts the result so it groups by domain, and presents it as one sensor. A dashboard card then reads that one sensor instead of three or five. It carries a health view of its sources alongside the list: a real Entity Sentinel that is currently broken is reported as a down source while the rest still combine, and a wrong input (a Battery Sentinel added by mistake, a plain sensor, another aggregate, or a typo) is surfaced as a rejected input rather than silently dropped.

That is the whole job. It does not watch anything itself; the Entity Sentinels do the watching, and this only combines their results for display.

## Why it is built this way, and the honest reasoning

I want to be straight about what this is, because the shape of it was not the goal. It is a workaround, and a good one, but a workaround.

The goal was a single dashboard card that shows every gone-quiet entity across all my Entity Sentinels, as one clean list. I run several Entity Sentinels (one for lively entities, one for quiet ones, one for sleepy ones), and a dashboard that shows three separate lists, with the same entity sometimes appearing in two of them, is not the single pane of glass I wanted.

The natural place to do the combining is in the card itself: read all three sensors, merge their lists, dedupe, sort, render. But the stock dashboard cards cannot do that. There is no built-in card that takes several sensors, unions their attribute lists, removes duplicates with a severity rule, and sorts the result. You can get close with custom cards and templating, but not cleanly, and not in a way I would hand to someone else to set up.

So I moved the combine-and-dedupe logic out of the card and into a sensor. That is what this blueprint is: the merge that I could not do in the dashboard, done in a template sensor instead, so the card only has to render one already-clean list. The sensor is the means; the dashboard is the end.

This is why it identifies itself as an aggregate (`sentinel_type: entity`, `sentinel_role: aggregate`). It is a faithful mirror of an Entity Sentinel, same state meaning, same `devices` shape, so any card or automation reads it exactly like a single Entity Sentinel. The `aggregate` role marks that it is the combined one, so anything that walks your Entity Sentinels can include it or leave it out and never double count.

## The decisions that fell out of that

A few choices follow directly from "this is a sensor that exists to feed a dashboard."

It validates its inputs by marker, and surfaces mistakes instead of hiding them. Because a person points it at "their Entity Sentinels" by hand, they might add the wrong thing. So each source is checked: a healthy Entity Sentinel is merged, a broken one is reported as down, and anything that is not an Entity Sentinel is rejected and named, with the reason, so a misconfiguration shows up on the dashboard rather than quietly doing nothing.

A broken source never silences the healthy ones. If one Entity Sentinel is in a setup error, the others still combine, and the broken one is reported separately. The list you can build is always built; the part you cannot see is reported as missing, not pretended away.

It does no detection of its own, on purpose. Every Entity Sentinel has already done the expensive work and published it. This sensor only reads those finished lists and combines them, so it runs no entity scan and no detection engine. It re-evaluates only when one of its named sources changes. Keeping it a light reader, not a second engine, was a deliberate line.

Down sources and rejected inputs are kept as two separate lists. A down monitor is an operational problem that will clear on its own; a rejected input is a configuration mistake that persists until you fix it. They are different kinds of trouble for different audiences, so the dashboard can treat them differently.

## What I am asking for

This is an alpha, and unlike the Sentinels themselves, it is the newest and least-settled idea in the set. I am putting it out because it works and I use it, but I am genuinely unsure it is the right long-term shape, and I would rather hear that now than later.

So, plainly:

Ideas and refinements are welcome. If you see a cleaner way to get a unified, deduped Sentinel list onto a dashboard, I want to hear it. If the attribute shape is missing something your card needs, tell me.

If you think this should not be a sensor at all, say so. I built an aggregate sensor because I could not combine and dedupe inside the dashboard. If there is a card-side approach I missed that does it properly, that would be the better answer, and I would happily retire this in favor of it.

And the bigger one: if someone wants to take this over and build a proper HACS card for it, I think that is the real destination. A custom card could do the combine and dedupe itself, read the Sentinels directly, and skip the intermediate sensor entirely. That is the version of this I could not build at the dashboard layer, and it is very likely the right one. If that is a project you would want to own, please reach out. I will help however I can, and I would rather see the better tool exist than hold on to the workaround.

## Setup

Import the blueprint, create the sensor, and point it at your Entity Sentinel sensors. Name it, give it a unique id, and add it to a dashboard card. It needs at least one Entity Sentinel to read; build those first (see the Entity Sentinel blueprint).

This companion comes from **The Thinking Home** at [xeazy.com](https://xeazy.com). The Entity Sentinel and Battery Sentinel blueprints, and the community thread, are linked from the [repository](https://github.com/TheThinkingHome/Automations).

## License

Copyright (C) 2026 James Lander, The Thinking Home (<https://xeazy.com>). This blueprint is free software: you may use, modify, and redistribute it under the terms of the GNU General Public License, version 3 or later (GPL-3.0-or-later). It is provided with no warranty. See the LICENSE file in this repository for the full text. If you redistribute or adapt it, keep this copyright and license notice intact.
