# Sound Vision — Visual Sound Indicators

An audio accessibility mod for S.T.A.L.K.E.R. that visually indicates the sources of sounds in the game, helping deaf and hard-of-hearing players better understand their surroundings.

[English](README.md) | [Русский](README.ru.md)

---

An accessibility mod for **S.T.A.L.K.E.R.: Anomaly** (1.5.1 / 1.5.2 / 1.5.3, including GAMMA
and other builds).
Built for deaf and hard-of-hearing players: the game draws **expanding rings right on the
screen** at the point of any sound source — even **through walls**. Since v2.7 the mod follows
an accessibility contract: **every sound delivered to the mod is always shown** (the optional
"Hearing threshold" in the MCM can hide barely audible distant sounds — shot markers are never
filtered) — both everything the engine broadcast over the AI hearing channel and **every NPC
shot** (the mod reads stalker ammo counters itself — the channel depends on neither sound
packs nor game executables). Since v2.8 the indication is **layered** and never interferes
with shooting: rings dim in the screen center (that's where you look with your own eyes),
directly visible sources turn translucent, while sources behind a wall are the opposite — the
most noticeable: blurred halos brighter than visible ones, and the center vignette does not
hide them (v2.8.1) — a hearing person hears the enemy behind the wall, a deaf player must see
him. Everything is configurable in the **MCM**.

---

## What you will see

| Event | Visual |
|---|---|
| Someone walks/runs (including behind a wall) | yellow rings at the footstep point — close range only |
| Someone talks, shouts, groans | white rings |
| Gunshots, reloads, bullet hits | red rings — visible from afar, just like they are audible |
| Grenade, underbarrel and barrel explosions | large orange-red rings (v2.9): full size at any distance; the louder the boom, the larger the ring (up to 2x) |
| Mutants (steps, growling, attacks) | purple rings |
| Anomalies: idle hum, activation hiss | turquoise rings |
| Physics: pushed barrels, hits, falling objects | blue rings |

- A ring **expands and fades** like a wave (1–3 waves per sound, configurable).
- **Audibility as in the game**: a loud shot at 100 m is shown (you hear it), a quiet footstep
  at 60 m is not (you don't hear it). Distant sources get smaller, paler rings (configurable).
- If the source is **beyond the screen edge** or **behind your back**, the ring smoothly slides
  to the screen edge **on its own half** (like quest markers in modern games): a sound on the
  left — the ring at the left edge, a sound on the right — at the right edge, above/below — at
  its own edge. The transition is smooth in both directions: the source leaves the frame — the
  ring rides to the edge; comes back into frame — it rides back, with no position or size
  jumps. Turn away from the source — the ring neither disappears nor "mirrors", it waits at
  the edge until you turn back; a source strictly behind your back keeps the last known side.
  The ring size at the edge is the same as in frame — it depends only on the distance to the
  source.
- The ring color is set by the sound category; brightness is a general setting.

## Center vignette and walls (v2.8 / v2.8.1)

The indication became layered: combat is seen with your own eyes, and the rings cover neither
the crosshair, nor enemies, nor your weapon.

**Center vignette.** If you look straight at a source, you already see what happens: gunfire,
enemies, their movement. Indication in the center of the screen is unnecessary and only gets
in the way. So ring brightness depends on the angular distance of the drawing point from the
screen center:

- in the very center (the crosshair zone) only **10 % of the current brightness** remains
  (configurable);
- farther out it grows smoothly (smoothstep), reaching full brightness at the screen edges;
- rings pinned to the edge (a source behind you or out of frame) are not affected — the
  direction of a sound behind your back is visible as before. The vignette releases smoothly
  as the ring rides to the edge (works even with a "Full brightness radius" above 100 %);
- sources **behind a wall** are never dimmed by the vignette (v2.8.1): for a hard-of-hearing
  player this is key information — a hearing person hears the enemy behind the wall, a deaf
  player must see him. The release is synchronized with the texture crossfade, without a
  jump. The "Dim behind-wall by center vignette" checkbox restores v2.8 behavior.

**Walls and visibility.** The mod checks the line of sight to every sound source with a level
ray (`level.ray_pick`, static geometry only) from the camera to the sound point. The rules:

| Source | Look |
|---|---|
| Visible directly (no wall) | translucent at **60 %** (brightness 40 %) — a hint that does not cover the enemy |
| Behind a wall | **blurred halo** — soft texture + size boost; brightness **70 %** (configurable) — noticeably brighter than visible |
| No sign (old build without `ray_pick`) | as in v2.7, an honest line in the log |

The ray is static-only: NPCs and physics never flicker the state, the source itself is not
hit by the ray. Closed doors are dynamic geometry — the ray does not see them: a source behind
a closed door shows as "visible" (for accessibility it is better to show a bit too much than
to hide the visible). A ray hit counts as a wall only if it is clearly short of the source
point — a source standing next to a wall and a bullet impact on that wall do not become
"occluded".

Performance: the state is cached per source (350 ms) and invalidated if the sound point moved
farther than 2 m (a walking NPC, a flying grenade); the budget is no more than two rays per
frame (a new event gets one priority ray so its halo is born correct right away), other
sources keep their previous state without pulsing.

**Anti-flicker (hysteresis).** Near the corner of a house, a fence or a doorway the ray result
changes from frame to frame — without a filter the halo pulses "visible ↔ behind the wall" and
the sound wave drowns in flicker. So the raw ray result is only a candidate: a state change is
accepted after confirmation (~0.3 s of steady repetition). At a corner the ring stays stable
(pulse 0.86 → 0.00 measured), while a real "came out from behind the wall" change arrives
within ~0.35–0.7 s — as one smooth crossfade. Ray errors and garbage instead of distance count
as "no data" (the previous state is kept) — before, a failure counted as "path clear" and a
source behind a wall looked translucent and flickered; with a persistently broken binding the
feature disables itself with an honest `OCC` line in the log.

**Smooth texture switching (crossfade).** A dynamic source — an NPC at the corner of a house,
running through a doorway — can flip "visible ↔ behind the wall" many times during one ring's
life. So the textures are not swapped, but drawn **simultaneously** with complementary
weights: the hard one fades out in proportion to the soft one coming in (both weights share
one exponent, ~0.4–0.6 s for a full transition). The textures are concentric (the soft one is
a Gaussian blur of the hard one with its peak normalized to the hard one: blurring changes
the halo width, not its brightness — v2.8.1), so the transition is a continuous morph "the
crisp ring dissolves into a blurry halo" and back: no jump of brightness, position or wave
phase. Measured on the real textures: the worst visible inter-frame image jump is ~2.5 times
smaller than with a texture swap, and the transition itself is continuous. Verified
numerically: in the steady state only one texture is drawn (the other is hidden below the
brightness threshold), and a source oscillating at a corner "breathes" instead of popping.

**Behind-wall brightness (v2.8.1).** A source behind a wall is key information for a
hard-of-hearing player, so since v2.8.1 it is drawn "as is" in brightness:

- the soft texture no longer dims the halo — the blurred texture's peak is normalized to the
  hard one's peak;
- the brightness of a behind-wall source is configurable in the MCM ("Behind-wall brightness",
  0–100 %, default **70 %** — noticeably brighter than visible sources; 100 % = like a normal
  ring);
- the center vignette **does not dim** behind-wall sources (the "Dim behind-wall by center
  vignette" checkbox, off by default, restores v2.8 behavior).

Measured on defaults: the peak brightness of a behind-wall halo in the center of the screen
grew from 6 % to 60 % (**10.6×**) — the center vignette used to dim it to near-invisibility,
which is exactly what a player noticed.

## Enlarged explosion rings (v2.9)

An explosion is a "where did it boom" event: its ring does **not shrink with distance** —
every explosion starts at full ring size, and the loudness at your position adds size (a
loud barrel or rocket blast is larger than a distant grenade, up to 2x). Explosions ignore
the hearing threshold (like shot markers), have their own 200 m range cap and are protected
from queue eviction. Governed by the **"Enlarged explosion rings"** MCM checkbox (on by
default); turning it off returns the ordinary distance-based sizing, while explosions keep
their color and channels.

## Loudness-based audibility (v2.0)

The engine already decides whether a sound is audible: every source carries ogg comments
(loudness, AI hearing range), and on playback the sound is **attenuated linearly with distance
and muffled by walls** (xray-monolith, `IGame_Level::SoundEvent_Register`:
`Power = (1 − dist/max_ai_distance) · volume · occlusion`).

The mod reads the finished result via `npc_on_hear_callback`: when any NPC or monster
**hears** a sound, the callback reports the engine loudness at its ears and the source
position. The loudness at the **player** is reconstructed with the inverse square from that
"anchor" (the decibel model of wave divergence: −6 dB per distance doubling):

```
player_loudness = ai_loudness · (ai_dist / player_dist)²
```

In v2.7 the loudness threshold was removed entirely: hearing-channel sounds — engine facts —
were always shown, and world polling also showed everything it caught (loudness remained only
in ring size and brightness; see v2.7 below). In v2.8.2 the threshold returned as an MCM
setting (default 10 ppt: the anomaly hum at 48 m — 0.0135 — passes, while barely audible
distant footsteps can be hidden; 0 — show everything). The result works **without calibrating
individual sources**:

| Sound | Visible up to about |
|---|---|
| Assault rifle shot | ~246 m (shot anchor, v2.6; the shot fact itself — always, v2.7) |
| Shot behind a thick wall | shown if the engine delivered it to AI (occlusion handled by the engine; v2.7) |
| Scream, loud speech | ~80 m |
| Running footsteps | ~35 m |
| Walking footsteps | ~29 m |
| Sneaking footsteps | ~25 m |
| Physics (pushing a barrel) | ~20 m |

Distance did not go anywhere: it drives the ring **size** (farther — smaller) and
**transparency** (farther — paler). The old mode with a single detection radius is available
in the MCM (Hearing → "Show only audible sounds" = off).

Note: loudness at AI and at the player are estimates from the same engine model; the model may
show a sound slightly before you actually hear it in your headphones — for accessibility that
is the right side to err on.

## How the mod learns about sounds

The mod **does not overwrite a single game file** — it only adds its own. Event sources:

1. **`npc_on_hear_callback`** (vanilla Anomaly callback) — the engine "hearing": when any NPC
   or monster **hears** a sound, the engine reports its type (shot/step/voice/physics —
   strings from ogg comments, e.g. `WPN_hit`, `MST_attack`), **loudness at the listener** and
   position. This covers shots, voices, monsters and physics at long range — with honest
   loudness. Nuance (v2.3): for weapon types the event position is the bullet, so the ring is
   placed on the shooter (the source object `who_id`).
   Stalker footsteps are not broadcast by the engine hearing (they play "without feedback") —
   world polling catches them.
2. **`anomaly_on_before_activate`** (vanilla) — an anomaly activates (a bolt, an NPC or a
   monster entered the zone) → **hissing** at the zone point.
3. **`actor_on_feeling_anomaly`** (vanilla) — the player entered an anomaly field → instant
   rings at its center.
4. **`physic_object_on_hit_callback`** (vanilla) — a hit on a physical object: a **knife on a
   barrel**, a bullet on a crate, a fist on a box → ring at the impact point.
5. **`actor_on_item_drop`** (vanilla) — a dropped item → a thud at the landing point.
6. **World polling** (every 250 ms, only objects near the player) — where the engine reports
   nothing, loudness is estimated by a "virtual anchor" (4 numbers for everything, not a
   calibration):
   - NPC/monster movement → **footsteps**: loudness by **movement speed** — sneaking is quiet,
     walking medium, running loud;
   - movement of corpses, barrels, boxes, thrown grenades → **physics** (pushing, sliding,
     falling — the same barrel pushing the AI cannot hear);
   - anomaly zones (`zone_*`) → periodic **idle hum** (a real zone idle sound plays in cycles
     with pauses — the mod imitates it with pulses, the rate is configurable);
   - a bolt: flies past and stops next to a zone → **anomaly activation**.
7. **`actor_on_footstep`** (vanilla) — the player's own footsteps. **Off** by default (you
   always feel your own sounds), can be enabled in the MCM.
8. **Ammo polling** (pure Lua, every 200 ms) — the **guaranteed shot channel**: for every
   living stalker within 250 m the mod reads the active weapon and the ammo counter — the
   counter dropped, a shot happened (the queue merges a burst into one living shooter ring; a
   rising counter is a reload, no ring). Depends on neither a pack's sound metadata, nor the
   presence of AI listeners, nor executables. Suppressors are respected: a quiet barrel shows
   at a shorter range (~77 m versus ~246 m for an open barrel). The
   `npc_on_weapon_shot_start` callback remains a bonus of executables that have it (vanilla
   Anomaly 1.5.2 does not — see v2.7 below).
9. **`actor_on_hit_callback`** (vanilla) — the player **got hit** → a ring immediately on
   whoever struck (for a shot it is the shooter, for a grenade — the grenade): a guaranteed
   attack marker at the moment of impact, even if every other channel stayed silent. A shooter
   beyond 250 m — the ring is placed at 250 m in the same direction (the renderer clamps far
   rings to the screen edge anyway).

10. **Explosion channels** (v2.9) — grenades, underbarrel rounds, barrels and mines come as a
    separate orange-red **explosion** category:
    - the engine plays every `snd_explode` with the WEAPON_SHOOTING type mask
      (`Explosive.cpp:59`), so an explosion arrives as `WPN_shoot`; the mod tells it from a
      shot by the source object (not a stalker, not a zone, not a gun) and keeps the honest
      engine loudness — a louder barrel is drawn larger;
    - **grenade flight polling** (every 250 ms): a flying projectile is a live parentless
      object; when it disappears — it exploded. Works without AI listeners on any build;
    - **`explosive_item_on_explode`** (modded exes only) — barrels/mines/dynamite; vanilla
      Anomaly 1.5.x has no such callback, so the mod just skips it (an honest INIT line in
      the log).
    All channels share one anti-spam key — the numeric id of the exploding object — so
    copies merge into a single ring; blue shock-wave rings near a fresh explosion merge
    into it too.

All callback registrations are wrapped in `pcall` — on any game build (with MCM or without,
with modded exes or without) an unavailable callback does not break the others.
## Why some packs had no halo (v2.1)

The engine hearing channel only works with sounds whose ogg file carries a correct ogg
comment: without it the engine assigns the sound `game_type = 0` (the `CSoundRender_Source`
constructor) and **does not broadcast it to AI at all** — the player hears the sound (the
audio channel has its own ranges), but the mod does not receive it via
`npc_on_hear_callback`. Rebuilt/converted sounds of weapon packs are the classic case.
On top of that, the motivator skips self-sounds: for a shooter with no other NPCs nearby the
hearing channel has nobody to collect events from.

That is why the **guaranteed shot channel** was added (source 8 above): since v2.7 it fires
by the fact of shooting via **ammo polling** — pure Lua on any build, with any weapon pack and
without AI listeners. (In v2.1–v2.6 the channel was the `npc_on_weapon_shot_start` callback;
the combat log of 05.11 showed that this callback does not exist in Anomaly 1.5.2 at all and
the channel was silent — see v2.7 below.) On any build shots are also drawn by the hearing
channel: since v2.3 it places the marker on the shooter itself (see below).

### Shot marker — on the shooter, not on the bullet (v2.3)

The debug log on GAMMA exposed the second cause of missing halos. The engine binds a hearing
event `WPN_*` to the **bullet**, not to the shooter: the callback's `sound_position` is the
position of the projectile / the impact point at the pulse moment (every 0.5 s):

- bullets flying at the player gave `pos` 1.7–3.5 m from the player (the shooter at ~23 m) —
  rings flashed at the player's feet;
- misses flew 79–85 m past the player — the loudness `power·(dl/dp)²` collapsed to
  0.013 < threshold 0.02, and the event was rejected by the gate.

In both cases the shooter was left without a marker. Since v2.3 the mod anchors `WPN_*`
events to the live position of the source object (`who_id`) — **the ring is placed on the
shooter**, with loudness recomputed to him (since v2.4 — from the sound type anchor, see
below). Works with any pack and on any executables: the engine always broadcasts impact
sounds, no weapon sections need to be known. Grenade and explosion positions (strings without
the `WPN_` prefix) are not moved — there the explosion point is more useful. In the log the
move shows as the suffix `(anchored: pos 85m -> shooter 23m, anv=shot)`.

### Marker stability (v2.4)

A GAMMA test showed jitter: markers appeared every other time. Causes and fixes:

1. **Loudness depended on a random listener.** A moved event carried the reading at the
   AI listener of the bullet: for a listener 3 m from the bullet the same shot gave loudness
   0.010 (below threshold — no marker), for a listener 19 m away — 0.170 (marker present).
   Now the loudness of a moved event is taken from the **virtual anchor of the sound type** —
   as in the guaranteed channel: a shot is visible up to ~150 m, with a suppressor up to
   ~77 m (the suppressor is read from the shooter's active weapon), a reload up to ~40 m, a
   dry click and weapon draw — up close. Identical in every fight, no matter who heard the
   bullet.
2. **The queue overflowed.** Every hearing event created a new ring: in combat the queue
   (16 waves) filled up and markers evicted each other before living out their wave. Now
   `weapon` events of one source **merge** into one living ring: it pulses and follows the
   shooter, and the queue is enough for everyone. In the log the merge shows as a
   `QUEUE refreshed` line.
3. **A shared anti-spam key.** Footsteps and shots of one NPC shared one anti-spam pause and
   ate each other's waves. The key is now separate: category + source.

Bonus: human footsteps and speech arrive with the `MST_` prefix (like mutants) and glowed
purple "monster" — now humans get yellow footsteps and white speech; purple stays with
mutants.

### Incoming fire, helicopters and a clean log (v2.5)

A GAMMA test against a helicopter exposed two problems.

**`cannot access class member` errors.** The loudness anchor choice probed `active_item` of
the sound source to read the suppressor. Stalkers have the method, helicopters do not: the
engine binding printed an error into the log on **every** hit, even under `pcall`. Now the
method is only probed on humans (`IsStalker`), vehicles immediately get the full-shot anchor —
the log is clean. (The `!MCM given bad path: rax_ammo_check/...` spam from the same log is
another mod, unrelated to Sound Vision.)

**The helicopter had no halos.** The v2.4 anchor loudness is computed from the shooter's
distance: for a helicopter at 300–550 m the shot is "inaudible" (0.004 < threshold 0.02), and
even when its bullets chewed the ground 7–25 m from the player, there was no ring. Now for
`WPN_*` the loudness decides — whoever is louder:

- the shooter is within shot-hearing range → ring on the shooter (as in v2.4);
- the shooter is not audible but the impact near the player is → an **incoming fire ring** at
  the impact point (loudness — the AI listener reading moved to the player).

A regression of a normal fight is excluded: for a close shooter the anchor is louder than the
impact, and the ring still stands steadily on him. In the log incoming fire shows as the
suffix `(incoming fire: shooter 299m below threshold -> impact ring, im=0.517)`.

**Helicopter flight sound.** The rotor is a looping sound, the engine does not broadcast it to
AI, the hearing channel is silent about the flight. But the helicopter's bullets arrive with
the vehicle's `who_id`: the mod remembers the object and for 90 seconds after the last bullet
pulses an **orange "vehicle" ring** (`kind=vehicle`) at its live position every 3 s (up to
600 m). The category shares the MCM toggle with "Weapons".

### The shot lives at the barrel (v2.6)

A test in a regular fight: an enemy at ~43 m pours fire at the player — muzzle flashes are
visible, shots are audible, but there are no rings. In the log:
`[GATE] rejected (below threshold): loud=0.0039/0.0046 < 0.0200 d=42–44 m`.

The cause is a hole in the v2.5 rule. The "who is louder" comparison only ran when the bullet
point diverged from the shooter by more than 2 m; while the bullet was still at the barrel,
the event went into the old reading move: `loud = p·(dl/dp)²`. And the engine reading at an AI
listener standing right next to the shooter (dl ≈ 3 m, p ≈ 0.98), crushed by the inverse
square, gives `0.98·(3/43)² ≈ 0.004 < 0.02` — no ring, although at that moment the shot is
loud at the player.

Fix: **WPN_shoot — the shot sound itself, emitted at the barrel — is always anchored to the
shooter**, without comparing with the bullet (the suppressor is still read from the
shooter-stalker). The "who is louder" rule is kept for `WPN_hit` — there the engine point
means an impact, and the v2.5 incoming fire works as before.

Additionally, a gate/queue mismatch was removed: the shot anchor (0.60@45 m) is audible up to
~246 m, while the queue cut everything beyond 150 m. Now `KIND_CAP.weapon = 250` — shots are
visible across the anchor's whole range (up to ~246 m, with a suppressor up to ~77 m, as
before).

Regression checks (log replay in `verify_loudness.py`, sections 12–15): the log fight — 2/2
rings instead of 0/2; a bullet in flight and misses — as in v2.5; `WPN_hit`/helicopter —
unchanged.

### Attack marker — always (v2.7)

A test against bandit rejects: "they are shooting, but the red marker is there, then it is
not — while it must always be there". The combat log (2629 lines) exposed three independent
causes of misses:

1. **The shot callback channel does not exist.** Vanilla Anomaly 1.5.2 has no
   `npc_on_weapon_shot_start` callback (the anomaly-modding-book callback list), and
   `RegisterScriptCallback` does not check names — "registration" succeeded, but the engine
   never invoked the callback: over the whole fight the `SHOT` channel produced not a single
   line.
2. **The hearing channel is silent for independent reasons.** A shot reaches the mod through
   AI hearing only if the shot sound has a valid ogg comment (otherwise `game_type = 0` and
   the engine does not broadcast it) and there is at least one AI listener near the shooter.
   In the fight log the rejects with carbines produced 3 `WPN_hit` events for the entire
   firefight — hence "sometimes there, sometimes not".
3. **The queue and gates** finished the rest: eviction of live weapon rings by the background
   flow (62–84% in simulation), the loudness threshold and anti-spam.

The solution is the "**engine fact → ring**" contract plus **we mine the fact ourselves**:

- **ammo polling** (source 8 in the list above): every 200 ms the mod reads the ammo counters
  of the active weapons of all living stalkers within 250 m; the counter dropped — a shot
  happened, a ring on the shooter. A reload (the counter rose) gives no ring. Works on any
  build;
- **a ring on hit** (source 9): the player got hit — a marker immediately on the attacker,
  even if every other channel was silent;
- everything arriving over the **hearing channel** (`npc_on_hear_callback`) is shown **always**
  (the `fact` flag in `push_event`); the only limit is the category range cap
  (weapon 250 m, others 150 m);
- **anti-spam does not act on weapon**: copies of one sound from different listeners merge
  into one living shooter ring;
- **weapon rings are protected from eviction**: on overflow the queue pushes out the oldest
  NON-weapon event (footsteps/physics/zone hum) and only if the queue is entirely weapon —
  the oldest weapon (`evicted oldest … weapon protected` in the log);
- **the loudness gate was removed entirely** — including for world polling: the zone hum at
  48 m (0.0135 < 0.02) and crows at 24 m are no longer cut. Loudness remained only in ring
  size and brightness; the "Hearing threshold" MCM option was replaced by this policy
  (in v2.8.2 the threshold returned as an option, default 10 ppt, shot markers untouched);
- **anomalies no longer "shoot"**: a zone's thermal discharge arrives as type `WPN_shoot`,
  and previously an anomaly drew a red attack ring from 150 m away (in the log:
  `who=zone_mine_thermal_weak → weapon, 168 m`). A `WPN_*` sound from someone who is not a
  stalker/mutant/vehicle is now recolored into a turquoise "anomaly".

Checks: queue simulation (`sim_queue_interference.py`) — with rejects shooting (v2.6: hearing
delivered ~15% of shots) the ring was visible 17–25% of the firing time; v2.7 (polling 100% +
merging + protection) — **100% visibility and 0% evictions**, including rings=3 (queue 10) and
ten zones nearby; syntax — no errors.

### Adaptive type pool

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

## Installation

### Mod Organizer 2
1. Download the `SoundVision_v2.8.2.zip` archive.
2. In MO2: the *Archives* tab (on the right) → *Install From file* → pick the archive.
3. Make sure the mod is enabled and sits **above** mods that could change `sound_vision.*`
   (there are none — no name conflicts exist).
4. Launch the game.

### Manually
Copy the `gamedata` folder from the archive into the game root (where `AnomalyLauncher.exe`
lives), merging folders. No existing files will be overwritten.

### Upgrading from previous versions
Just remove the old version (deactivate in MO2) and install the fresh one on top: file names
did not change, nothing needs manual editing.

### Requirements
- **MCM (Mod Configuration Menu)** — optional. Without it the mod runs with default settings
  (enabled, every caught sound is shown).
- **Everything works on vanilla executables 1.5.1+**: the guaranteed shot channel is pure Lua
  (ammo polling + `actor_on_hit_callback`), modded exes are not required. The
  `npc_on_weapon_shot_start` callback, if the build has it, merely speeds up the reaction.

## Settings (MCM → Sound Vision)

### General
| Option | Default | Description |
|---|---|---|
| Enable mod | on | show/hide all rings |
| Show my own sounds | off | rings for the player's own footsteps/shots |
| Detection range | 45 m | world polling coverage for footsteps/physics/zones; in the "by distance" mode — the single show limit (5–100) |
| Wave lifetime | 1.2 s | how long one ring expands and fades |
| Per-source delay | 0.3 s | anti-spam: at most one wave per this time from one source in each category; shots (weapon) are not limited — copies merge into one living ring (v2.7) |

### Hearing
| Option | Default | Description |
|---|---|---|
| Show only audible sounds | on | every caught sound is always shown; the option only affects world polling: on — the category range cap, off — the single "Detection range" radius |
| Hearing threshold | 10 ppt | minimum loudness at the player for a ring to be drawn, in thousandths of full (0–100); 0 — show everything. Shot, reload and hit markers ignore it (v2.8.2) |

### Size and animation
| Option | Default | Description |
|---|---|---|
| Ring size | 96 px | ring size for a close source (HUD virtual pixels) |
| Reference distance | 8 m | at this distance a ring has full size |
| Minimum size | 12 % | shrink limit for the farthest sounds (5–50 %) |
| Waves per sound | 2 | 1–3, "sonar" |
| Brightness | 85 % | overall opacity |
| Enlarged explosion rings | on | explosion rings keep full size at any distance; loudness adds size (up to 2x). Off = explosions size like ordinary sounds |

### Center vignette (v2.8)
| Option | Default | Description |
|---|---|---|
| Dim the screen center | on | vignette mask: rings almost die out in the crosshair zone, returning to full brightness toward the edges |
| Center brightness | 10 % | how much of the usual brightness remains in the very center (2–50 %) |
| Dim zone radius | 50 % | how far the dimmed zone stretches from the center, % of the way from center to screen edge (10–80 %) |
| Full brightness radius | 100 % | at what distance from the center rings are full again, % of the way to the edge (40–150 %) |

### Walls and visibility (v2.8)
| Option | Default | Description |
|---|---|---|
| Wall awareness | on | line-of-sight check by ray: a directly visible source — translucent, behind a wall — a blurred halo |
| Visible source brightness | 40 % | brightness of rings for sources with no wall between you (10–90 %); 40 % = 60 % transparency |
| Behind-wall brightness | 70 % | brightness of rings for sources hidden by a wall (0–100 %); key information for the hard-of-hearing, so brighter than visible |
| Blurred halos behind walls | on | soft blurred halo texture for behind-wall sources; the texture peak equals a normal ring — blurring does not reduce brightness |
| Blurred halo size | 130 % | how much wider the halo of a behind-wall source is (100–180 %) |
| Dim behind-wall by center vignette | off | if enabled, the center vignette also dims behind-wall sources (v2.8 behavior) |

### Sound types
Six category toggles: gunshots/weapons (also governs explosions — orange-red, v2.9),
voice/speech, footsteps/movement, mutants, anomalies, physics/other. Each category can be disabled separately.

### Detection sources
| Option | Default | Description |
|---|---|---|
| Anomaly hum pause | 4 s | how often each zone pulses while idle (2–10 s) |
| Track moving objects | on | pushed barrels, sliding boxes (grenades are shown as explosions, v2.9) |
| Object move threshold | 12 cm | sensitivity: lower = more rings (5–50 cm) |
| Guaranteed shot channel | on | ammo polling of stalkers within 250 m: the counter dropped — a shot; plus a ring on the attacker when you get hit. Works on any build |

### Debug
| Option | Default | Description |
|---|---|---|
| Debug log | off | full trace of the sound pipeline into the game log (see "Diagnostics" below) |
| Sound type log | off | new type strings → game log, for tuning `sound_vision_overrides` |

> **How to fit the size to yourself:** "Ring size" + "Reference distance" + "Minimum size"
> together define how the ring shrinks with distance: sources closer than 8 m get the full
> ring (96 px), farther — it smoothly shrinks down to 12% of the size. Want "farther = more
> visible" — raise the reference distance and the minimum size.

## Diagnostics (debug log)

If "the sound is audible but there are no rings" — enable in the MCM (Sound Vision →
**Debug**) the **"Debug log"** and open the game log (`appdata\logs\xray_<name>.log`). Every
event passes a pipeline, and every layer writes its own line prefixed `[sound_vision]`:

| Tag | What it writes |
|---|---|
| `INIT` | mod start: which callbacks registered and the final settings, including the guaranteed channel parameters (`ammo-poll 200ms/250m + actor hit`), the vignette and occlusion |
| `CONFIG` | settings re-read (the "MCM applied" marker) |
| `HEAR` | hearing channels CAUGHT a sound: type, source `id(section)`, loudness at the listener and the estimate at the player; for `WPN_*` — the suffix `(anchored: pos … -> shooter …)` if the ring was moved from the bullet to the shooter |
| `SHOT` | the guaranteed channel: `poll: npc=… ammo 30->29` — ammo polling caught a shot; `actor hit: who=…` — the player got hit, marker on the attacker |
| `DETECT` | world polling saw footsteps/physics/zones (footsteps — at most once per second per character); `grenade gone (boom)` — a flying projectile disappeared, explosion ring at its last position (v2.9) |
| `BOOM` | `explosive_item_on_explode` callback (modded exes): a barrel/mine exploded — ring at its position (v2.9) |
| `GATE` | the queue **rejected** the event and why: `too far` (beyond the category range cap: weapon 250 m, others 150 m), `below threshold` (loudness at the player below the "Hearing threshold"; weapon is never cut), `no loudness data`, `throttle` (world polling anti-spam; weapon is never cut), `merged into explosion` (a shock-wave ring near a fresh explosion, v2.9), `category off`, `mod disabled` |
| `QUEUE` | the queue **accepted** the event — a ring is placed (id assigned); `evicted oldest … weapon protected` — on overflow the oldest background event was pushed out, the red marker survived (v2.7); `absorbed by explosion` — a wave ring merged into a fresh explosion (v2.9) |
| `OCC` | occlusion (v2.8): `ray_pick` availability on this build (one line at the first ray); ACCEPTED state changes — `visible (clear LOS)` / `occluded (wall)` with distance and confirmation time; `flap suppressed` — corner/fence flicker suppressed; feature disabled on a broken binding |
| `RENDER` | `shown` — the indicator is on screen (+ `occ=seen/wall/?` — the occlusion state); `FAILED` — the event died without ever being projected; `HUD hidden` — a menu on top |
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

## Features and limitations

- **Speech**: a voice sound is visible when at least one NPC nearby hears it (the second
  interlocutor, a companion, a mutant). A lone stalker muttering to himself in an empty room
  gives no rings — the engine does not broadcast "sound into nowhere". In combat/at a base
  (the typical cases) speech is displayed.
- **Anomaly hum** — approaching: the mod pulses rings on the zone's schedule (default every
  4 s; a zone right next to you hisses more often — every ~1.1 s). The moment of a real hiss
  (activation) is drawn exactly by the `anomaly_on_before_activate` callback.
- Rings are an **event indicator**: they are drawn at the point where the sound happened and
  live ~1 second. A walking character "follows" themselves with a chain of rings; a pushed
  barrel — with a chain of blue ones.
- In combat there are many sources: anti-spam and the limit of simultaneous waves (16 at 2
  waves per sound) keep the screen from turning into porridge; tuned by the "per-source
  delay".
- Rings are not drawn over inventory/dialogs/maps (the HUD hides when menus are open).

## Compatibility

- The mod consists **only of new files** with unique names (`sound_vision*`): no vanilla
  scripts are overwritten, no conflicts with other add-ons.
- A modified `xr_danger.script` (Stealth Overhaul etc.) — compatible: both add-ons only
  *add* `npc_on_hear_callback` handlers, they do not interfere with each other.
- Works on vanilla executables 1.5.1+; works on modded exes too.

## Mod contents

```
gamedata/
├── configs/
│   ├── text/eng/ui_st_sound_vision.xml      MCM labels (English)
│   ├── text/rus/ui_st_sound_vision.xml      MCM labels (Russian, CP1251)
│   └── ui/ui_sound_vision.xml               XML templates of 64 rings (32 hard + 32 soft, v2.8)
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
├── docs/references/
│   └── Звук в Аномалии.md                   sound system reference for development (in Russian)
└── textures/ui/sv_ring.dds                  white ring (uncompressed DDS 128×128)
    textures/ui/sv_ring_soft.dds             blurred ring for behind-wall halos (v2.8, peak = hard one since v2.8.1)
```

Modules depend on each other unidirectionally and link lazily, as in Twitch Integration:
every script declares its global table at the end (`_G.sound_vision_<module> = {...}`), and
dependencies are resolved at the first call via `_G.<module name>` — the X-Ray engine loads
the script itself at the first mention of its name.

## For developers

- Constants by module responsibility: `POOL = 32` (ring pool capacity) and `HEAR_CAP = 150`
  (loudness show limit, m) — in `sound_vision_events`;
  `MOVE_POLL_MS = 250`, `STEP_MIN_D2 = 0.06`, `ZONE_NEAR_M = 3.0`, `BOLT_STOP_MS = 450`
  and the virtual loudness anchors `ANV_*` — in `sound_vision_detect`;
  the guaranteed channel anchors `ANV_SHOT` / `ANV_SHOT_SIL` / `ANV_RELOAD`, the ammo polling
  period `WPN_POLL_MS = 200` and its radius `WPN_POLL_D = 250` — in `sound_vision.script`;
  `PIN_MARGIN = 40` (edge margin) — in `sound_vision_projection`;
  `EDGE_BLEND_TAU = 120` (edge↔frame transition smoothness, ms) and `OCC_TAU_MS = 120`
  (smoothing of the "visible ↔ behind the wall" switch, ms) — in `sound_vision_render`;
  `OCC_TTL_MS = 350` (source state cache life), `RAYS_FRAME = 2` (ray budget per frame),
  `NEAR_M = 1.5` (close sources are seen without a ray), `LOS_EPS_M = 0.45` (a hit right next
  to the point does not count as a wall) — in `sound_vision_occlusion`.
- Add your own color/type: extend `KIND_COLOR` + `KIND_CAT` + `classify()` in
  `sound_vision_classify.script` and add a checkbox in `sound_vision_mcm.script` (do not
  forget the keys in both text XML files).
- Sound classification is a cascade: user overrides (`sound_vision_overrides.script`) →
  substring heuristics (`WPN_*`, `MST_*`, `ITM_*`, `voice`, `drop`…) → inference from the
  source (a mutant → "monsters") → "other" (blue). The result is cached in a registry; with
  the MCM log every new type string is written to the game log.
- In-game debugging: the global table `sound_vision_debug` — `.events` (the current event
  queue; each event carries `loud` — loudness at the player, and `occ_a`/`occ_s` — smoothed
  occlusion multipliers), `.classify(s_type, who_id)` (the classifier), `.types()` (the
  registry of seen types: string → category and counter), `.push_event(x, y, z, "step", nil,
  nil, 0.5)` (force an event with loudness 0.5), `.refresh_config()` (re-read settings),
  `.occlusion_stats()` (`ray_pick` availability, cache size, ray counter — v2.8).
- The debug log — `sound_vision_log.script`, Twitch Integration style:
  `dbg(tag, fmt, ...)` → `printf("[sound_vision][TAG] ...")`; the switch is `mc.debug_logs`,
  the rate filter is `rate(key, ms)`. Tags and the chain interpretation are in the
  "Diagnostics" section above.
- Engine facts about sound (the audibility model, ogg comments, types, callbacks) — in
  `docs/references/Звук в Аномалии.md` (in Russian); extended as findings arrive.
- The MCM tree structure follows the MCM 1.7 documentation strictly (see the comment at the
  top of `sound_vision_mcm.script`).
