---
layout: post
title: "Chasing a phantom Zigbee flood"
description: "Debugging a Zigbee2MQTT device that spammed my MQTT broker once a second — and why the debounce setting that should have fixed it never stood a chance."
comments: false
keywords: "zigbee2mqtt, tuya, mqtt, home assistant, homelab, debugging, zigbee"
published: false
---

I noticed something off in my Zigbee2MQTT logs recently: one specific device — a Moes 2-gang dimmer switch in the kitchen — was publishing to MQTT roughly once every single second, around the clock. Every other device in the house (~28 of them) combined didn't come close to that rate. In a two-minute log sample, that one switch accounted for **97% of all Zigbee2MQTT traffic**.

Nothing was actually happening in the kitchen. The light wasn't flickering, no one was touching the switch. The published payload was byte-for-byte identical every time — same state, same brightness — except for the timestamp.

## The obvious fix that didn't work

Zigbee2MQTT has a per-device `debounce` setting for exactly this kind of thing: "don't publish more than once every N seconds." It was already configured — `debounce: 2` — and I confirmed via a live query that the running instance genuinely had it loaded. Not a stale config, not a typo.

Didn't matter. Restarting the service didn't help. Power-cycling the device at the breaker didn't help. The messages kept coming, exactly once a second, like clockwork.

## Turning up the volume

Rather than keep guessing, I bumped Zigbee2MQTT's log level to `debug` live over MQTT (no restart needed) and watched the raw Zigbee frames arriving from the device. That's when it got interesting:

```
Received Zigbee message from 'Kitchen', type 'commandDataReport',
cluster 'manuSpecificTuya', data '{"dpValues":[{"dp":20,"datatype":2,
"data":[40,100,55,136]}]}'
```

Every second, another one of these, with the last byte ticking down by exactly 1 each time: `136, 135, 134, 133...`. Some kind of counter, buried in a Tuya-proprietary "datapoint" (DP) — Tuya devices tunnel their own manufacturer-specific data through a generic Zigbee cluster rather than using standard attributes, which is exactly what let this fly under the radar.

Datapoint 20. I checked the actual converter code running on my system against the device's real, documented feature set — state, brightness, min/max brightness, countdown timers, power-on behavior, backlight mode. Those map to datapoints 1, 2, 3, 5, 6, 7, 8, 9, 11, 12, 14, and 21.

**Datapoint 20 isn't one of them.** It's not documented, not exposed, not a real feature — near as I can tell, an internal firmware heartbeat the device pushes unconditionally, once a second, forever.

## Why "debounce" never stood a chance

This is the part I actually found satisfying to dig into. I went and read Zigbee2MQTT's own source for how it decides whether to publish a message. Simplified, the logic is:

1. Run the message through the device's converters.
2. If a converter produces a recognized change → publish it, respecting `debounce`/`throttle`.
3. If **no converter recognizes anything** (exactly the datapoint-20 case) → skip straight to a separate code path that exists purely to keep the device's "last seen" timestamp fresh. That path has **no debounce hook at all**. It's not optional, not configurable per-device — it just always fires.

So `debounce: 2` was never going to touch this. It's not a misconfiguration on my end — the setting simply doesn't apply to messages the software doesn't understand. Every "unknown data" message takes an entirely different, un-throttled road to the broker.

## The actual fix

Once I understood *why* it was happening, the fix became obvious: make datapoint 20 a *known* property, even a meaningless one, so it stops being "unrecognized" and starts flowing through the path that actually respects debounce.

Zigbee2MQTT supports live-loaded external converters — small JavaScript files that can extend or override a device's built-in definition. I wrote one that copies the device's real definition verbatim (all the actual DPs have to be replicated — an external definition fully replaces the built-in one for a matching device, so leaving anything out would silently break a real feature) and adds one new line: datapoint 20, mapped to a hidden, read-only diagnostic property.

```js
tuya.modernExtend.dpNumeric({
    name: "internal_dp20_unmapped",
    dp: 20,
    type: tuya.dataTypes.number,
    readOnly: true,
    expose: e.numeric("internal_dp20_unmapped", ea.STATE)
        .withDescription("Undocumented Tuya datapoint, appears to be a firmware heartbeat.")
        .withCategory("diagnostic"),
}),
```

I also added `debounce_ignore` for the switch's real, user-facing properties (state, brightness, and so on) — that way an actual light toggle still publishes instantly, and only the meaningless heartbeat gets held back and coalesced.

## Results

Same two-minute measurement, before and after:

| | Before | After |
|---|---|---|
| Messages from this device | 372 / 384 lines (~97%) | 72 / 2,831 lines (~2.5%) |
| Cadence | rigid, 1/sec | irregular, every 3–10s |

Roughly a **5x reduction**, and — I checked carefully — every real property (state, brightness, countdown, power-on behavior) is byte-for-byte identical to what it reported before. The fix touches nothing about how the light actually works; it only stops one specific, meaningless internal counter from drowning everything else out.

It's not perfect silence — a continuous stream faster than the debounce window will still leak a bit through — but going from *dominating the entire broker* to *roughly in line with my busiest legitimate sensor* is a win I'll happily take.

## Is anyone else hitting this?

I went looking before writing this up. I found a near-identical symptom reported for a completely different device — an unrecognized datapoint spamming messages once a second — but the maintainers closed that one as "won't fix": debounce genuinely isn't designed to reach this code path, and that's apparently accepted as expected behavior rather than a bug.

I also checked the original pull request that added support for this exact dimmer model. Datapoint 20 is never mentioned anywhere in it — nobody flagged it when the device was first added. So while the general *pattern* (unmapped datapoint bypassing debounce) is known and accepted as "working as intended," the *specific* fix for this device doesn't exist upstream yet.

That feels worth contributing back — not as "please redesign your debounce system" (already asked, already declined), but as a small pull request adding datapoint 20 to the official device definition, the same one line I've been running locally. I still don't know what datapoint 20 actually *represents* — just that it counts down once a second and means nothing to me. Sometimes the most honest bug report is "I don't know what this is, but here's how to make it stop shouting."
