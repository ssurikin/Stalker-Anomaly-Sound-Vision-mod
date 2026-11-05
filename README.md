# Sound Vision — Visual Sound Indicators

Sound Vision is an accessibility mod for **S.T.A.L.K.E.R.: Anomaly** (1.5.1 / 1.5.2 / 1.5.3,
including GAMMA and other builds). It is built for deaf and hard-of-hearing players: the
game draws **expanding rings right on the screen** at the point of any sound source — even
**through walls**. What other players hear, you see.

[English](README.md) | [Русский](README.ru.md)

---

## What you will see

| Event | Visual |
|---|---|
| Someone walks/runs (including behind a wall) | yellow rings at the footstep point — close range only |
| Someone talks, shouts, groans | white rings |
| Gunshots, reloads, bullet hits | red rings — visible from afar, just like they are audible |
| Grenade, underbarrel and barrel explosions | large orange-red rings |
| Mutants (steps, growling, attacks) | purple rings |
| Anomalies: idle hum, activation hiss | turquoise rings |
| Physics: pushed barrels, hits, falling objects | blue rings |

- Audibility works **like in the game**: a rifle shot is shown from ~250 m, a scream from
  ~80 m, running footsteps from ~35 m. Farther sources get smaller, paler rings.
- A sound off-screen or behind your back pins its ring to the screen edge **on its own
  side**, like quest markers in modern games.
- A source you see directly is translucent (never covers the enemy); a source **behind a
  wall** is a bright blurred halo — key information. In the very center of the screen
  (the crosshair zone) rings dim to almost nothing, so they never block your aim.
- Red shot markers are guaranteed: even a lone shooter with no one around, and the attacker
  when you get hit.

## Installation

### Mod Organizer 2 (recommended)
1. Download the `Stalker-Anomaly-Sound Vision-mod-main.zip` archive.
2. Open Mod Organizer 2, enable the "Show «Open folders» menu" option and click
   «Open folder "Mods"».
3. Place the mod folder from the downloaded archive into that mods folder.
4. Return to MO2 and click **Refresh**. The new mod will appear at the very bottom of the
   mod list — enable it (tick the checkbox).
5. Launch the game.

### Manually
Copy the `gamedata` folder from the archive into the game root (where `AnomalyLauncher.exe`
lives), merging folders. No existing files will be overwritten.

### Upgrading from previous versions
Just remove the old version (deactivate in MO2) and install the fresh one on top: file
names did not change, nothing needs manual editing.

### Requirements
- **S.T.A.L.K.E.R.: Anomaly 1.5.1 or newer.** Everything works on vanilla executables —
  modded exes are not required.
- **MCM (Mod Configuration Menu)** — optional. Without it the mod runs with default
  settings (enabled, every caught sound is shown).

## Settings (MCM → Sound Vision)

Everything is configured in the MCM, and every option has a description on hover:

- **General** — enable/disable the mod, your own sounds, detection range, wave lifetime,
  and the **"Sound Vision: show/hide halos" hotkey**: press it right in the game to show
  or hide all rings without opening any menu. Default key: `-` (the minus to the right of
  0); rebind it in the MCM;
- **Hearing** — audibility mode and the optional hearing threshold;
- **Size and animation** — ring size, waves per sound, brightness, large explosion rings;
- **Sound types** — six category toggles (weapon, voice, steps, mutants, anomalies, physics);
- **Detection sources** — anomaly hum pause, moving objects, guaranteed shot channel;
- **Center vignette** and **Walls and visibility** — how rings behave in the crosshair
  zone and behind walls;
- **Debug** — logs for troubleshooting.

## Good to know

- Rings are an **event indicator**: they are drawn where the sound happened and live about
  a second. A walking character leaves a chain of rings behind them.
- **Speech** is shown when at least one NPC nearby hears it (a second interlocutor, a
  companion, a mutant). A lone stalker muttering to himself in an empty room gives no
  rings — the engine does not broadcast "sound into nowhere". In combat and at bases
  speech is displayed.
- In combat with many sources, anti-spam and the wave limit keep the screen readable;
  red shot markers are never dropped.
- Rings are not drawn over menus (inventory, dialogs, maps).

## Compatibility

- The mod consists **only of new files** with unique names (`sound_vision*`): no vanilla
  file is overwritten, no conflicts with other add-ons.
- Works alongside other hearing addons (for example, mods that modify `xr_danger.script`).

## For developers

Architecture, sound channels, the occlusion and vignette internals, constants, diagnostics
and the debug log — in the technical specification:
**[docs/specs.md](docs/specs.md)**.
