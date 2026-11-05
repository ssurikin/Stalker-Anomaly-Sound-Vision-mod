# Sound Vision — Technical Specification

Developer-facing documentation for the Sound Vision mod. Everything on this page is in
English on purpose; player-facing instructions live in [README.md](../README.md) (English)
and [README.ru.md](../README.ru.md) (Russian).

Contents:

1. [What the mod does](#1-what-the-mod-does)
2. [Architecture](#2-architecture)
3. [How the mod learns about sounds](#3-how-the-mod-learns-about-sounds)
4. [Audibility model](#4-audibility-model)
5. [Rendering layers](#5-rendering-layers)
6. [Explosions (v2.9)](#6-explosions-v29)
7. [Rings stay while aiming (v2.9.1)](#7-rings-stay-while-aiming-v291)
8. [Sound classification](#8-sound-classification)
9. [Shot marker design history (v2.1–v2.7)](#9-shot-marker-design-history-v21v27)
10. [Mod contents](#10-mod-contents)
11. [Constants and module responsibilities](#11-constants-and-module-responsibilities)
12. [In-game debug facade](#12-in-game-debug-facade)
13. [Diagnostics (debug log)](#13-diagnostics-debug-log)
14. [Compatibility](#14-compatibility)
15. [References](#15-references)

---

## 1. What the mod does

Sound Vision visually indicates the sources of sounds in **S.T.A.L.K.E.R.: Anomaly**
(1.5.1 / 1.5.2 / 1.5.3, including GAMMA and other builds) for deaf and hard-of-hearing
players: expanding rings are drawn on the screen at the point of any sound source — even
through walls.

**Accessibility contract (since v2.7): every sound delivered to the mod is always shown.**
This covers both everything the engine broadcast over the AI hearing channel and every NPC
shot (the mod reads stalker ammo counters itself — the channel depends on neither sound
packs nor game executables). The optional "Hearing threshold" MCM option (v2.8.2, default
10 ppt) can hide barely audible distant sounds — shot markers are never filtered.

**Layered indication (since v2.8).** Rings dim in the screen center (that is where you
look with your own eyes), directly visible sources turn translucent, while sources behind
a wall are the opposite — the most noticeable: blurred halos brighter than visible ones,
and the center vignette does not hide them (v2.8.1). A hearing person hears the enemy
behind the wall — a deaf player must see him.

Ring → color mapping:

| Event | Visual |
|---|---|
| Someone walks/runs (including behind a wall) | yellow rings at the footstep point — close range only |
| Someone talks, shouts, groans | white rings |
| Gunshots, reloads, bullet hits | red rings — visible from afar, just like they are audible |
| Grenade, underbarrel and barrel explosions | large orange-red rings (v2.9) |
| Mutants (steps, growling, attacks) | purple rings |
| Anomalies: idle hum, activation hiss | turquoise rings |
| Physics: pushed barrels, hits, falling objects | blue rings |

## 2. Architecture

The mod **does not overwrite a single game file** — it only adds its own.

```
gamedata/
├── configs/
│   ├── text/eng/ui_st_sound_vision.xml      MCM labels (English)
│   ├── text/rus/ui_st_sound_vision.xml      MCM labels (Russian, CP1251)
│   └── ui/ui_sound_vision.xml               XML templates of 64 rings (32 hard + 32 soft, v2.8)
├── docs/
│   ├── references/Звук в Аномалии.md        sound system reference (in Russian)
│   └── specs.md                             this document
├── scripts/
│   ├── sound_vision.script                  root: source callbacks + registration
│   ├── sound_vision_config.script           settings: cache of MCM values
│   ├── sound_vision_classify.script         adaptive sound classification, colors
│   ├── sound_vision_overrides.script        manual user type pool (edited per build)
│   ├── sound_vision_events.script           event queue: audibility, anti-spam, scale
│   ├── sound_vision_detect.script           world polling: footsteps, physics, zones, bolt (4 Hz)
│   ├── sound_vision_projection.script       world -> HUD: projection and edge pinning
│   ├── sound_vision_render.script           ring pool, animation, transition smoothing,
│   │                                        center vignette, occlusion (v2.8)
│   ├── sound_vision_occlusion.script        line of sight: ray_pick, cache, ray budget (v2.8)
│   ├── sound_vision_log.script              debug log (tags HEAR/GATE/QUEUE/RENDER/OCC)
│   └── sound_vision_mcm.script              MCM tree + get_config()
└── textures/ui/
    ├── sv_ring.dds                          white ring (uncompressed DDS 128×128)
    └── sv_ring_soft.dds                     blurred ring for behind-wall halos (v2.8,
                                             peak = hard one since v2.8.1)
```

SRP: one module — one responsibility; dependencies are unidirectional and link lazily,
as in Twitch Integration: every script declares its global table at the end
(`_G.sound_vision_<module> = {...}`), and dependencies are resolved at the first call
via `_G.<module name>` — the X-Ray engine loads the script itself at the first mention
of its name.

All callback registrations are wrapped in `pcall` — on any game build (with MCM or
without, with modded exes or without) an unavailable callback does not break the others.

## 3. How the mod learns about sounds

Event sources:

1. **`npc_on_hear_callback`** (vanilla Anomaly callback) — the engine "hearing": when any
   NPC or monster **hears** a sound, the engine reports its type (shot/step/voice/physics —
   strings from ogg comments, e.g. `WPN_hit`, `MST_attack`), **loudness at the listener**
   and position. This covers shots, voices, monsters and physics at long range. Nuance
   (v2.3): for weapon types the event position is the bullet, so the ring is placed on the
   shooter (the source object `who_id`). Stalker footsteps are not broadcast by the engine
   hearing (they play "without feedback") — world polling catches them.
2. **`anomaly_on_before_activate`** (vanilla) — an anomaly activates (a bolt, an NPC or a
   monster entered the zone) → hissing at the zone point.
3. **`actor_on_feeling_anomaly`** (vanilla) — the player entered an anomaly field → instant
   rings at its center.
4. **`physic_object_on_hit_callback`** (vanilla) — a hit on a physical object: a knife on a
   barrel, a bullet on a crate, a fist on a box → ring at the impact point.
5. **`actor_on_item_drop`** (vanilla) — a dropped item → a thud at the landing point.
6. **World polling** (`sound_vision_detect`, every 250 ms, only objects near the player) —
   where the engine reports nothing, loudness is estimated by a "virtual anchor" (4 numbers
   for everything, not a calibration):
   - NPC/monster movement → **footsteps**: loudness by **movement speed** — sneaking is
     quiet, walking medium, running loud;
   - movement of corpses, barrels, boxes, thrown grenades → **physics**;
   - anomaly zones (`zone_*`) → periodic **idle hum** (imitates the cyclic idle sound, the
     rate is configurable);
   - a bolt: flies past and stops next to a zone → **anomaly activation**.
7. **`actor_on_footstep`** (vanilla) — the player's own footsteps. **Off** by default,
   can be enabled in the MCM.
8. **Ammo polling** (pure Lua, every 200 ms) — the **guaranteed shot channel** (v2.7): for
   every living stalker within 250 m the mod reads the active weapon and the ammo counter —
   the counter dropped, a shot happened (the queue merges a burst into one living shooter
   ring; a rising counter is a reload, no ring). Depends on neither a pack's sound
   metadata, nor the presence of AI listeners, nor executables. Suppressors are respected.
   The `npc_on_weapon_shot_start` callback remains a bonus of executables that have it
   (vanilla Anomaly 1.5.2 does not).
9. **`actor_on_hit_callback`** (vanilla) — the player **got hit** → a ring immediately on
   whoever struck: a guaranteed attack marker at the moment of impact. A shooter beyond
   250 m — the ring is placed at 250 m in the same direction (the renderer clamps far
   rings to the screen edge anyway).
10. **Explosion channels** (v2.9) — see [section 6](#6-explosions-v29).

## 4. Audibility model

The engine already decides whether a sound is audible: every source carries ogg comments
(loudness, AI hearing range), and on playback the sound is **attenuated linearly with
distance and muffled by walls** (xray-monolith, `IGame_Level::SoundEvent_Register`:
`Power = (1 − dist/max_ai_distance) · volume · occlusion`).

The mod reads the finished result via `npc_on_hear_callback`: the callback reports the
engine loudness at the AI listener's ears and the source position. The loudness at the
**player** is reconstructed with the inverse square from that "anchor" (the decibel model
of wave divergence: −6 dB per distance doubling):

```
player_loudness = ai_loudness · (ai_dist / player_dist)²
```

Threshold policy history: in v2.7 the loudness gate was removed entirely (hearing-channel
sounds — engine facts — were always shown). In v2.8.2 the threshold returned as an MCM
setting (default 10 ppt: the anomaly hum at 48 m — 0.0135 — passes, while barely audible
distant footsteps can be hidden; 0 — show everything). Weapon, reload and hit markers
ignore it.

What is visible up to (with defaults):

| Sound | Visible up to about |
|---|---|
| Assault rifle shot | ~246 m (shot anchor, v2.6; the shot fact itself — always, v2.7) |
| Shot behind a thick wall | shown if the engine delivered it to AI (occlusion handled by the engine) |
| Scream, loud speech | ~80 m |
| Running footsteps | ~35 m |
| Walking footsteps | ~29 m |
| Sneaking footsteps | ~25 m |
| Physics (pushing a barrel) | ~20 m |

Distance drives the ring **size** (farther — smaller) and **transparency** (farther —
paler). The old single-radius mode is available in the MCM (Hearing → "Show only audible
sounds" = off).

Note: loudness at AI and at the player are estimates from the same engine model; the model
may show a sound slightly before you actually hear it in your headphones — for
accessibility that is the right side to err on.

## 5. Rendering layers

### 5.1 Center vignette (v2.8)

If you look straight at a source, you already see what happens: indication in the center
of the screen is unnecessary and only gets in the way. Ring brightness depends on the
angular distance of the drawing point from the screen center:

- in the very center (the crosshair zone) only **10 % of the current brightness** remains
  (configurable);
- farther out it grows smoothly (smoothstep), reaching full brightness at the screen edges;
- rings pinned to the edge (a source behind you or out of frame) are not affected — the
  direction of a sound behind your back is visible as before;
- sources **behind a wall** are never dimmed by the vignette (v2.8.1); the release is
  synchronized with the texture crossfade. The "Dim behind-wall by center vignette"
  checkbox restores v2.8 behavior.

### 5.2 Walls and visibility (v2.8 / v2.8.1)

The mod checks the line of sight to every sound source with a level ray
(`level.ray_pick`, static geometry only) from the camera to the sound point:

| Source | Look |
|---|---|
| Visible directly (no wall) | translucent at **60 %** (brightness 40 %) — a hint that does not cover the enemy |
| Behind a wall | **blurred halo** — soft texture + size boost; brightness **70 %** (configurable) |
| No sign (old build without `ray_pick`) | as in v2.7, an honest line in the log |

The ray is static-only: NPCs and physics never flicker the state, the source itself is not
hit by the ray. Closed doors are dynamic geometry — the ray does not see them: a source
behind a closed door shows as "visible" (for accessibility it is better to show a bit too
much than to hide the visible). A ray hit counts as a wall only if it is clearly short of
the source point.

Performance: the state is cached per source (350 ms) and invalidated if the sound point
moved farther than 2 m; the budget is no more than two rays per frame (a new event gets
one priority ray), other sources keep their previous state without pulsing.

**Anti-flicker (hysteresis).** Near a corner the ray result changes from frame to frame —
so the raw ray result is only a candidate: a state change is accepted after confirmation
(~0.3 s of steady repetition). Ray errors and garbage instead of distance count as "no
data" (the previous state is kept); with a persistently broken binding the feature
disables itself with an honest `OCC` line in the log.

**Smooth texture switching (crossfade).** Textures are not swapped but drawn
**simultaneously** with complementary weights: the hard one fades out in proportion to the
soft one coming in (both weights share one exponent, ~0.4–0.6 s for a full transition).
The textures are concentric (the soft one is a Gaussian blur of the hard one with its peak
normalized to the hard one — v2.8.1), so the transition is a continuous morph without a
jump of brightness, position or wave phase. Measured on the real textures: the worst
visible inter-frame image jump is ~2.5 times smaller than with a texture swap.

**Behind-wall brightness (v2.8.1).** The blurred texture's peak is normalized to the hard
one's peak; the behind-wall brightness is configurable (0–100 %, default **70 %**); the
center vignette does not dim behind-wall sources. Measured on defaults: the peak
brightness of a behind-wall halo in the center of the screen grew from 6 % to 60 %
(**10.6×**).

### 5.3 Edge pinning

If the source is beyond the screen edge or behind your back, the ring smoothly slides to
the screen edge **on its own half** (like quest markers): a sound on the left — the ring
at the left edge, and so on. The transition is smooth in both directions (EDGE_BLEND_TAU);
a source strictly behind your back keeps the last known side. The ring size at the edge is
the same as in frame — it depends only on the distance to the source.

## 6. Explosions (v2.9)

An explosion is a "where did it boom" event: its ring does **not shrink with distance** —
every explosion starts at full ring size, and the loudness at your position adds size (up
to 2x). Explosions ignore the hearing threshold (like shot markers), have their own 200 m
range cap and are protected from queue eviction. Governed by the "Enlarged explosion
rings" MCM checkbox (on by default); turning it off returns the ordinary distance-based
sizing, while explosions keep their color and channels.

Three channels:

1. **Hearing channel**: the engine plays every `snd_explode` with the WEAPON_SHOOTING type
   mask (`Explosive.cpp:59`), so an explosion arrives as `WPN_shoot`; the mod tells it
   from a shot by the source object (not a stalker, not a zone, not a gun) and keeps the
   honest engine loudness.
2. **Grenade flight polling** (every 250 ms): a flying projectile is a live parentless
   object; when it disappears — it exploded. Works without AI listeners on any build.
3. **`explosive_item_on_explode`** (modded exes only) — barrels/mines/dynamite; vanilla
   Anomaly 1.5.x has no such callback, so the mod just skips it (an honest INIT line in
   the log).

All channels share one anti-spam key — the numeric id of the exploding object — so copies
merge into a single ring; blue shock-wave rings near a fresh explosion merge into it too.

## 7. Rings stay while aiming (v2.9.1)

Critical bug fixed: the whole indicator interface disappeared while aiming.
The cause is vanilla, not the engine and not GAMMA: `axr_main.actor_on_weapon_zoom_in`
sends `GUI_on_show("UIZoom")` — *"to force hiding HUD UIs on Zooming"* — and the mod dimmed
its rings on every such event. The zoom signal is now ignored (symmetrically on hide, so
an open inventory is never overlaid), and as a belt-and-suspenders measure the renderer
re-asserts its window every frame via the engine's idempotent `AddDialogToRender` — any
foreign HUD-hiding addon restores within one frame.

## 8. Sound classification

Sound type strings of modpacks are not standardized — a hard-coded list is impossible. The
classifier (`sound_vision_classify`) recognizes a new string in a cascade:

1. **user overrides** — `sound_vision_overrides.script` (highest priority);
2. **substring heuristics** — `wpn/weapon/shoot`, `mst/monster`, `voice/talk/scream`,
   `step/walk`, `anom/zone`, `obj/drop/door/hit` — covers vanilla and most packs;
3. **inference from the source** — an unrecognized sound from a mutant → purple "monster";
4. otherwise — neutral blue "other".

The result is remembered in a registry (a string is recognized once per session).

### How to teach the mod the sounds of your pack

1. In the MCM (the "Debug" section) enable **"Sound type log"**.
2. Play: shoot, reload — anything.
3. Open the game log (`appdata\logs\xray_<name>.log`) and find the lines
   `[sound_vision][TYPES] s_type '...' -> category`.
4. Put the needed string into `sound_vision_overrides.script`:

```lua
sound_vision_overrides = {
        map = {
                ["my_pack_custom_fire"] = "weapon", -- red shot ring
        },
}
```

To add a new color/category: extend `KIND_COLOR` + `KIND_CAT` + `classify()` in
`sound_vision_classify.script` and add a checkbox in `sound_vision_mcm.script` (do not
forget the keys in both text XML files).

## 9. Shot marker design history (v2.1–v2.7)

Why this exists: the shot marker went through several rounds of fixes, and the reasons are
useful context for any work on the hearing channel.

### Why some packs had no halo (v2.1)

The engine hearing channel only works with sounds whose ogg file carries a correct ogg
comment: without it the engine assigns the sound `game_type = 0` (the
`CSoundRender_Source` constructor) and **does not broadcast it to AI at all** — the player
hears the sound, but the mod does not receive it via `npc_on_hear_callback`.
Rebuilt/converted sounds of weapon packs are the classic case. On top of that, the
motivator skips self-sounds: for a shooter with no other NPCs nearby the hearing channel
has nobody to collect events from. That is why the **guaranteed shot channel** was added
(source 8 above).

### Shot marker — on the shooter, not on the bullet (v2.3)

The engine binds a hearing event `WPN_*` to the **bullet**, not to the shooter: the
callback's `sound_position` is the position of the projectile / the impact point at the
pulse moment (every 0.5 s). Bullets flying at the player gave `pos` 1.7–3.5 m from the
player; misses flew 79–85 m past the player and the loudness collapsed below the gate.
Since v2.3 the mod anchors `WPN_*` events to the live position of the source object
(`who_id`) — **the ring is placed on the shooter**. Grenade and explosion positions are
not moved — there the explosion point is more useful.

### Marker stability (v2.4)

1. **Loudness depended on a random listener.** A moved event carried the reading at the AI
   listener of the bullet. Now the loudness of a moved event is taken from the **virtual
   anchor of the sound type** — identical in every fight.
2. **The queue overflowed.** `weapon` events of one source **merge** into one living ring
   (`QUEUE refreshed` in the log).
3. **A shared anti-spam key.** The key is now separate: category + source.

Bonus: human footsteps and speech arrive with the `MST_` prefix (like mutants) and glowed
purple "monster" — now humans get yellow footsteps and white speech.

### Incoming fire, helicopters and a clean log (v2.5)

**`cannot access class member` errors**: `active_item` is only probed on humans
(`IsStalker`), vehicles immediately get the full-shot anchor — the log stays clean.

**Helicopters**: for `WPN_*` the loudness decides — whoever is louder. The shooter within
shot-hearing range → ring on the shooter; the shooter not audible but the impact near the
player audible → an **incoming fire ring** at the impact point. The rotor is a looping
sound the engine does not broadcast, but the helicopter's bullets arrive with the
vehicle's `who_id`: for 90 seconds after the last bullet the mod pulses an orange
"vehicle" ring (`kind=vehicle`) at its live position every 3 s (up to 600 m).

### The shot lives at the barrel (v2.6)

**`WPN_shoot` — the shot sound itself, emitted at the barrel — is anchored to the shooter
always**, without comparing with the bullet (the suppressor is still read from the
shooter-stalker). The "who is louder" rule is kept for `WPN_hit` — there the engine point
means an impact. `KIND_CAP.weapon = 250` so shots are visible across the anchor's whole
range. Regression checks: log replay in `verify_loudness.py`, sections 12–15.

### Attack marker — always (v2.7)

The combat log (2629 lines) exposed three independent causes of misses: the
`npc_on_weapon_shot_start` callback does not exist in vanilla Anomaly 1.5.2
(`RegisterScriptCallback` does not check names — "registration" succeeded, but the engine
never invoked the callback); the hearing channel is silent without a valid ogg comment or
without AI listeners near the shooter; the queue and gates finished the rest.

The solution — the "**engine fact → ring**" contract plus mining the fact ourselves:

- **ammo polling** (source 8): the counter dropped — a shot happened, a ring on the
  shooter; a reload gives no ring; works on any build;
- **a ring on hit** (source 9): the player got hit — a marker immediately on the attacker;
- everything arriving over the **hearing channel** is shown **always** (the `fact` flag in
  `push_event`); the only limit is the category range cap (weapon 250 m, others 150 m);
- **anti-spam does not act on weapon**: copies of one sound from different listeners merge
  into one living shooter ring;
- **weapon rings are protected from eviction**: on overflow the queue pushes out the
  oldest NON-weapon event, and only if the queue is entirely weapon — the oldest weapon;
- **the loudness gate was removed entirely** (in v2.8.2 the threshold returned as an
  option, default 10 ppt, shot markers untouched);
- **anomalies no longer "shoot"**: a `WPN_*` sound from someone who is not a
  stalker/mutant/vehicle is recolored into a turquoise "anomaly".

Checks: queue simulation (`sim_queue_interference.py`) — v2.7 (polling 100% + merging +
protection) gives **100% visibility and 0% evictions**, including rings=3 (queue 10) and
ten zones nearby.

## 10. MCM integration

The MCM tree is built in `sound_vision_mcm.script` (`on_mcm_load`), strictly following the
MCM 1.7 documentation (github.com/RAX-Anomaly/Anomaly-Mod-Configuration-Menu; see the
comment at the top of the script). Option captions are taken from
`configs/text/<lang>/ui_st_sound_vision.xml` by the keys
`ui_mcm_sound_vision_<section>_<option>` (+ `_desc`).

The single point of reading settings is `sound_vision_mcm.get_config(key)`: it reads the
value from `ui_mcm.get("sound_vision/<group>/<key>")` under `pcall`, and falls back to the
built-in defaults when MCM is not installed. Values are cached in
`sound_vision_config.mc` and refreshed by `refresh_config()`, which is invoked on
`on_game_start`, `actor_on_first_update` and `on_option_change` (MCM "apply").

The Russian label file is **CP1251-encoded** — keep that encoding when editing.

### 10.1 Hotkey: show/hide halos (v2.9.2)

The "General" MCM section contains a `key_bind` option (`toggle_key`, stored as a DIK
code, default `DIK_keys.DIK_MINUS` = 12 — the "-" key to the right of 0). Per the MCM
documentation, `key_bind` options must have `val = 2`, and the stored value is the
`DIK_keys` value of the key:

```lua
{ id = "toggle_key", type = "key_bind", val = 2, def = defaults.toggle_key }
```

Key presses are caught with the standard Anomaly script callback:

```lua
RegisterScriptCallback("on_key_press", fn)  -- signature: fn(dik)
```

(the axr_main callback table documents `on_key_press — Params: (<number>)`; the number is
the DIK code). The engine dispatches it from `CLevel::IR_OnKeyboardPress`, i.e. only
during gameplay: the main menu (including the MCM while the player rebinds the key), the
console and dialogs do not feed it — so rebinding cannot cause a phantom toggle, and no
extra menu guard is needed (`db.actor` is kept as a cheap sanity check). The handler
compares the pressed key with the bound one read via
`sound_vision_mcm.get_config("toggle_key")`
(`ui_mcm.get("sound_vision/general/toggle_key")`; without MCM the built-in default is
used) and calls `sound_vision_config.toggle_enabled()`.

Toggle semantics: the key flips the same `mc.enabled` flag that the "Enable mod" checkbox
controls. The state is kept as a runtime override in `sound_vision_config`:

- the override survives `refresh_config()` — a save/load or applying unrelated MCM
  options does not resurrect halos the player just hid;
- it resets when the player changes the "Enable mod" checkbox itself in the MCM — the
  checkbox is authoritative;
- a game restart (fresh Lua state) starts from the checkbox value again.

Every toggle is logged as `CONFIG hotkey toggle: enabled=... (mcm checkbox=...)`. MCM has
no `ui_mcm.set`, so the checkbox itself is not rewritten by the hotkey — on purpose: the
checkbox remains the persistent setting, the hotkey is a per-session switch.

## 11. Constants and module responsibilities

- `POOL = 32` (ring pool capacity) and `HEAR_CAP = 150` (loudness show limit, m) — in
  `sound_vision_events`;
- `MOVE_POLL_MS = 250`, `STEP_MIN_D2 = 0.06`, `ZONE_NEAR_M = 3.0`, `BOLT_STOP_MS = 450`
  and the virtual loudness anchors `ANV_*` — in `sound_vision_detect`;
- the guaranteed channel anchors `ANV_SHOT` / `ANV_SHOT_SIL` / `ANV_RELOAD`, the ammo
  polling period `WPN_POLL_MS = 200` and its radius `WPN_POLL_D = 250` — in
  `sound_vision.script`;
- `PIN_MARGIN = 40` (edge margin) — in `sound_vision_projection`;
- `EDGE_BLEND_TAU = 120` (edge↔frame transition smoothness, ms) and `OCC_TAU_MS = 120`
  (smoothing of the "visible ↔ behind the wall" switch, ms) — in `sound_vision_render`;
- `OCC_TTL_MS = 350` (source state cache life), `RAYS_FRAME = 2` (ray budget per frame),
  `NEAR_M = 1.5` (close sources are seen without a ray), `LOS_EPS_M = 0.45` (a hit right
  next to the point does not count as a wall) — in `sound_vision_occlusion`.

## 12. In-game debug facade

The global table `sound_vision_debug`:

- `.events` — the current event queue; each event carries `loud` (loudness at the player)
  and `occ_a`/`occ_s` (smoothed occlusion multipliers);
- `.classify(s_type, who_id)` — the classifier;
- `.types()` — the registry of seen types: string → category and counter;
- `.push_event(x, y, z, "step", nil, nil, 0.5)` — force an event with loudness 0.5;
- `.refresh_config()` — re-read settings;
- `.occlusion_stats()` — `ray_pick` availability, cache size, ray counter (v2.8).

The debug log lives in `sound_vision_log.script` (Twitch Integration style):
`dbg(tag, fmt, ...)` → `printf("[sound_vision][TAG] ...")`; the switch is `mc.debug_logs`,
the rate filter is `rate(key, ms)`.

## 13. Diagnostics (debug log)

If "the sound is audible but there are no rings" — enable in the MCM (Sound Vision →
**Debug**) the **"Debug log"** and open the game log (`appdata\logs\xray_<name>.log`).
Every event passes a pipeline, and every layer writes its own line prefixed
`[sound_vision]`:

| Tag | What it writes |
|---|---|
| `INIT` | mod start: which callbacks registered and the final settings, including the guaranteed channel parameters, the vignette and occlusion |
| `CONFIG` | settings re-read (the "MCM applied" marker) |
| `HEAR` | hearing channels CAUGHT a sound: type, source `id(section)`, loudness at the listener and the estimate at the player; for `WPN_*` — the suffix `(anchored: pos … -> shooter …)` if the ring was moved from the bullet to the shooter |
| `SHOT` | the guaranteed channel: `poll: npc=… ammo 30->29` — ammo polling caught a shot; `actor hit: who=…` — the player got hit, marker on the attacker |
| `DETECT` | world polling saw footsteps/physics/zones (footsteps — at most once per second per character); `grenade gone (boom)` — a flying projectile disappeared, explosion ring at its last position (v2.9) |
| `BOOM` | `explosive_item_on_explode` callback (modded exes): a barrel/mine exploded — ring at its position (v2.9) |
| `GATE` | the queue **rejected** the event and why: `too far`, `below threshold` (weapon is never cut), `no loudness data`, `throttle` (weapon is never cut), `merged into explosion` (v2.9), `category off`, `mod disabled` |
| `QUEUE` | the queue **accepted** the event — a ring is placed; `evicted oldest … weapon protected` (v2.7); `absorbed by explosion` (v2.9) |
| `OCC` | occlusion (v2.8): `ray_pick` availability; ACCEPTED state changes — `visible (clear LOS)` / `occluded (wall)`; `flap suppressed` — corner/fence flicker suppressed; feature disabled on a broken binding |
| `RENDER` | `shown` — the indicator is on screen (+ `occ=seen/wall/?`); `FAILED` — the event died without ever being projected; `HUD hidden` — a menu on top |
| `TYPES` | sound type log (its own checkbox, independent of the master switch) |

How to read the chain using "a bot shoots, no ring":

```
neither HEAR nor SHOT (poll)               -> the shot could not have happened; for other sounds:
                                              an ogg comment without game_type is not
                                              broadcast by the engine (see the reference)
HEAR/SHOT present, GATE too far            -> beyond the category range cap
                                              (weapon 250 m / others 150 m)
GATE throttle                              -> world polling anti-spam (footsteps/physics/zones);
                                              weapon is never cut (v2.7)
GATE below threshold                       -> loudness at the player below the "Hearing threshold"
                                              (MCM "Hearing"); weapon is never cut
QUEUE present but no RENDER shown          -> the HUD is hidden by a menu or the projection
                                              is not calibrated
OCC: ray_pick NOT available                -> a build without ray_pick: occlusion
                                              is not applied, rings as in v2.7
the ring is pale on a visible enemy        -> this is v2.8: a directly visible
                                              source is translucent (MCM:
                                              "Visible source brightness")
the ring behind a wall is pale in the center -> v2.8.1: the vignette does not dim
                                              behind-wall sources; check the MCM
                                              "Behind-wall brightness"
                                              and the "Dim behind-wall by center
                                              vignette" checkbox
```

Lines of identical rejections are deduplicated (at most once per second per source) so the
log stays readable. Turn it off after diagnosing.

## 14. Compatibility

- The mod consists **only of new files** with unique names (`sound_vision*`): no vanilla
  scripts are overwritten, no conflicts with other add-ons.
- A modified `xr_danger.script` (Stealth Overhaul etc.) — compatible: both add-ons only
  *add* `npc_on_hear_callback` handlers, they do not interfere with each other.
- Works on vanilla executables 1.5.1+; works on modded exes too.

## 15. References

- `docs/references/Звук в Аномалии.md` — engine facts about sound (the audibility model,
  ogg comments, types, callbacks), in Russian; extended as findings arrive.
- MCM (Mod Configuration Menu) documentation: github.com/RAX-Anomaly/Anomaly-Mod-Configuration-Menu.
- Anomaly modding guide (callbacks reference): robe127.github.io/stalker-anomaly-modding-guide.
- Engine sources: github.com/themrdemonized/xray-monolith (`xr_hear.script`,
  `xr_motivator.script`, `IGame_Level.cpp`, `step_manager.cpp`, `Explosive.cpp`,
  `key_binding_registrator_script.cpp`, `Level_input.cpp`).
