# PvPCore 3.0.9 Changelog

## Fixes

### 1. FFA player count no longer covers the NPC name
The 3.0.8 count hologram floated too high - the vanilla player nametag occupies
feet+2.075..2.3 blocks, and the old count line rendered at feet+2.02..2.245,
covering most of the NPC's name.

- The NPC name and the live player count are now ONE two-line text-display
  hologram: the name sits exactly where the vanilla nametag used to be, the
  count line hangs right under it, clear of the NPC's head.
- While the hologram is up the vanilla nametag is hidden via the NPC's
  scoreboard team (rule nametagVisibility=never); it is restored automatically
  when the count line is blanked (empty `zone.npc-count` message).
- The name line renders the full colored display name (spaces preserved),
  slightly nicer than the vanilla nametag, which had to squash spaces out of
  the profile name.
- A leftover NEVER visibility rule from a crashed session can no longer hide
  the name forever: spawning resets the team rule to ALWAYS before the
  hologram re-hides it.

### 2. Fast heal in sword fights returned to normal
Every duel/round start and FFA respawn used to set hunger saturation to 5.0.
Vanilla's HungerManager treats `food == 20 && saturation > 0` as a license for
SATURATED FAST REGENERATION - up to 1 HP every 10 ticks (0.5s) - so sword
fights healed absurdly fast, and the mid-duel hunger top-up re-armed that fast
regen roughly every 10 seconds of a long fight.

- Fighters now start with saturation 0: healing is the normal vanilla natural
  regeneration (1 HP per 4 seconds at food >= 18).
- The in-duel hunger loop tops the food level back up but never re-adds
  saturation.
- FFA respawn applies the same fix, so sword FFA fights heal at the normal
  rate too.

## New features

### 3. Per-kit auto potions at the start of every duel round
Admins can now attach potion effects (and items) to a kit; every fighter
receives them automatically when a duel starts and again at the start of each
further round.

Commands (all under `/pvpadmin kit autopotions <kit>`):
- `/pvpadmin kit autopotions <kit>` - list the configured auto potions
- `/pvpadmin kit autopotions <kit> addeffect <effect> <seconds> [amplifier]`
  - e.g. `addeffect speed 30 1` = Speed II for 30s; `addeffect night_vision -1`
    = infinite Night Vision; use `-1` seconds for infinite duration
- `/pvpadmin kit autopotions <kit> additem` - adds the potion/item held in
  your main hand (with its exact NBT: splash potions, custom potions, golden
  apples...) to be handed out each round
- `/pvpadmin kit autopotions <kit> remove <index>` - remove one entry
- `/pvpadmin kit autopotions <kit> clear` - remove all entries

Notes:
- Effects support the full vanilla effect id list with tab completion
  (speed, fire_resistance, strength, resistance, night_vision, ...).
- Auto items go into the first free hotbar/inventory slot; if the inventory is
  full they are dropped at the fighter's feet so nothing is lost.
- A kit template reset (`/pvpadmin kit reset`) keeps the auto potion config.
- Auto potions apply to DUELS (unranked, ranked and /duel) - FFA zones keep
  their own kit layout system.

### 4. Command tree repair (found while testing #3)
The decompiled command chain from 3.0.4 silently attached the kit `reset`
subcommand at `/pvpadmin reset` instead of `/pvpadmin kit reset` (closing
parens in the one-liner re-exposed the outer builder - confirmed against the
original 3.0.4 bytecode). The `/pvpadmin` tree was rebuilt cleanly:
- every kit operation, including the new `autopotions` and `reset`, now
  verifiably hangs under `/pvpadmin kit ...`
- `/pvpadmin reset <kit>` keeps working as a back-compat alias
- all other subcommands (reload, save, setlobby, scoreboard, audit, build)
  unchanged

## Compatibility
- Minecraft 1.21.11, Fabric Loader 0.19.3+, Fabric API 0.141.x, Java 21
- Configs, kits, zones, NPCs and stats from 3.0.4 - 3.0.8 load as-is
  (old kits.json simply gains the empty auto-potion lists on first save)
