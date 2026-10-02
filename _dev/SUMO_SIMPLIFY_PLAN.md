# Sumo Arena: simplify pass

**Status:** plan only. Nothing in the game has changed yet. Written 2 October 2026.

**Goal:** make the game less complicated before testing online servers.

- Keep skins and accessories.
- Turn off ability levelling. Everyone plays at one fixed ability level.
- Put the camera back to the original angle.

**How to start the next session:** open the Sumo Arena game project (it was not connected to the session that wrote this plan) and say:
"Follow `_dev/SUMO_SIMPLIFY_PLAN.md` from the tumblewake/website repo."

---

## The rule: nothing gets deleted

Everything that is cut is switched off and stored, so it can come back with a one-line change.

1. **Snapshot first** (step 1). A tagged copy of the full game exists before any edit.
2. **Switches, not removals.** Levelling code, level tables and the newer camera values stay in the project. A switch turns them off.
3. **Hide UI, don't delete it.** Upgrade buttons and level labels get `visible = false` through the switch. Their nodes stay in the scenes.
4. **Don't move or rename files.** Moving files outside the Godot editor breaks `res://` paths and UIDs.
5. **Don't touch save data.** Ability levels already in a save file stay there and are simply ignored while levelling is off. No migration.

---

## Decisions

| Area | Decision |
|---|---|
| Skins and accessories | Keep exactly as they are: shop, unlocks, equipping, looks in matches. |
| Ability levelling | Off. No upgrade button, cost, level label, level pips or "NEXT" row. |
| Ability level | **Level 5 of 10** for every ability, for you and for every bot. Set by one constant. |
| Picking abilities | Keep. Each wrestler still takes two abilities in. |
| Camera | Original angle by default. The newer angle is kept as a stored preset. |
| Kensho, gold, diamonds, chests, Season Pass | Keep. Gold now only buys things in the shop (see step 5). |

**Why level 5:** level 1 is the starting point that upgrades are meant to improve on (for example, Jump at level 1 recharges in 8.75 seconds). Level 10 stacks every extra, such as Jump's shockwave, which is more for a new player to learn. Level 5 sits in the middle: snappier than level 1 without the full set of extras. It is one number, so changing it after playtesting is a one-line edit.

One level for everyone also helps the online tests. Nobody can out-level an opponent, and there are no per-player stats to sync.

**Check in step 3:** see what level 5 actually gives each ability. If level 5 unlocks a big extra (a shockwave or a second effect) that makes an ability feel busy, use the level just before that extra instead, and write down which level was chosen.

---

## Steps

### 0. Find the code (about 5 minutes)

Run these from the game project folder, then fill in the **Found files** table at the bottom before editing anything.

```sh
# Where ability levels are read, stored or upgraded
grep -rn -i -E "ability_?level|upgrade|level_?up|max_?level" --include=*.gd .

# Level tables (stats per level, upgrade costs)
grep -rln -i -E "level" --include=*.gd --include=*.tres --include=*.json --include=*.cfg .

# Upgrade UI ("LEVEL 2", "NEXT", "Need N more gold")
grep -rn -i -E "LEVEL |NEXT|more gold|upgrade" --include=*.tscn --include=*.gd --include=*.csv --include=*.po .

# Camera
grep -rn -E "Camera3D|Camera2D|fov|look_at|rotation_degrees|camera_|cam_" --include=*.gd --include=*.tscn .

# Bots: does difficulty change bot ability levels?
grep -rn -i -E "difficulty|yokozuna" --include=*.gd . | grep -i -E "level|abilit"

# Save file fields
grep -rn -i -E "save|load" --include=*.gd . | grep -i -E "level|abilit"
```

### 1. Snapshot (about 2 minutes)

If the project is in git:

```sh
git add -A && git commit -m "Snapshot before simplify pass"
git tag pre-simplify
git branch archive/full-features
git push origin pre-simplify archive/full-features   # if it has a remote
```

If it is not in git: zip the whole project folder to `SumoArena-full-features-2026-10.zip` and keep it outside the project folder.

### 2. Add the switches

Put these in the existing global settings or autoload script. Only create a new script if there is no such script.

```gdscript
# Simplify pass (October 2026): these features are switched off, not removed.
const ABILITY_LEVELING_ENABLED := false
const FIXED_ABILITY_LEVEL := 5
const CAMERA_PRESET := "original"   # "original" or "new" (the newer, steeper angle)
```

### 3. Abilities: one fixed level

- [ ] Find the place the game turns "which level is this ability?" into stats. When levelling is off, have it return `FIXED_ABILITY_LEVEL`. If levels are read in many places, add one function such as `get_ability_level(ability)` and point those places at it.
- [ ] Bots use the same function. If bot difficulty raised bot ability levels, that stops. Difficulty then comes only from how bots play.
- [ ] Leave the level 1 to 10 stat tables and upgrade costs untouched.
- [ ] Leave the save file alone. Saved levels are kept but not read.
- [ ] Note what level 5 gives each ability (see the check under Decisions).

### 4. Abilities: hide the levelling UI

In the ability menu (the screen in `images/ability-menu.jpg` on the website):

- [ ] Hide "LEVEL N" on each ability card.
- [ ] Hide the level pips next to the ability name.
- [ ] Hide the "NEXT" row and the "Need N more gold" line.
- [ ] Hide the "LEVEL 2 / 5 gold" upgrade button.
- [ ] Keep the ability cards, picking two, the Equipped button, and the NOW and RECHARGE rows. Those rows now show the fixed level's numbers.
- [ ] Ability descriptions mention upgrades ("Upgrades add a quicker hop..."). Show a version without that sentence while levelling is off, and keep the old text.

Everywhere else, hide or swap anything about levelling:

- [ ] "Upgrade available" badges or dots on the main menu
- [ ] Daily quests such as "upgrade an ability" (swap for a quest that doesn't need levelling)
- [ ] Trophies or stats about ability levels
- [ ] Profile screen ability levels
- [ ] Tips, tutorial or hint text about levelling up

### 5. Economy check (report only)

Upgrades were a main use for gold (Jump level 2 costs 5 gold). With them gone, gold only goes to the shop.

- [ ] Check what skins and accessories cost: gold, diamonds or both. If nothing in the shop costs gold, gold has nothing to buy. Tell the user; don't redesign prices in this pass.
- [ ] Check chest and Season Pass rewards for anything ability-specific. If there is any, give gold instead while levelling is off.

### 6. Camera: back to the original angle

- [ ] Find the gameplay camera values: position or offset, rotation or pitch, distance or height, FOV, and any zoom or follow settings.
- [ ] Find when they changed: `git log -p -S"<property name>" -- <camera file>`. The values before that commit are the original ones.
- [ ] Store both sets as named presets, with the original as the default:

  ```gdscript
  const CAMERA_PRESETS := {
  	"original": {"pitch_deg": 0.0, "distance": 0.0, "fov": 0.0},  # fill in from git history
  	"new":      {"pitch_deg": 0.0, "distance": 0.0, "fov": 0.0},  # current values, kept for later
  }
  ```

- [ ] If the values live in a `.tscn` scene rather than a script, put the original values back in the scene and copy the newer values into the `"new"` preset.
- [ ] Check every mode: Duel, Free for all, Free for all 16 (the big wheel), Teams and World Tournament. Leave the main menu podium camera alone unless it also changed.
- [ ] Visual reference: the website screenshots `images/match-4-players.jpg` and `images/wheel-16-players.jpg` were uploaded on 27 September 2026. If the camera changed after that date, these show the original angle. If unsure, ask the user.

### 7. Test

- [ ] The game launches with no errors in the Godot output.
- [ ] Shield, Jump, Hook, S-Gun and Dash all work and show the fixed level's numbers.
- [ ] No upgrade button, cost or level text appears anywhere.
- [ ] Bots on Easy and on Yokozuna still feel clearly different.
- [ ] Buy and equip a skin and an accessory. They show in a match.
- [ ] A chest opens and a Season Pass tier pays out.
- [ ] The camera looks like the original in all five modes, at iPhone and iPad screen shapes.
- [ ] An old save with levelled-up abilities loads without errors.
- [ ] **Nothing was lost:** set `ABILITY_LEVELING_ENABLED = true` and `CAMERA_PRESET = "new"` once, check the full game comes back, then set them back.

### 8. Commit

Make one commit per part (switches, abilities, ability UI, camera), so any part can be undone on its own.

---

## Next: online servers

- Fixed ability levels make online matches even, with no per-player stats to sync.
- **The website and privacy policy currently promise no online play.** They say "Fully offline", "No account needed" and "It sends nothing off your device". Testing on your own devices is fine. Before any public build with online play, update:
  - `index.html` lines 7 and 87
  - `sumo-arena.html` lines 7 and 149–150
  - `support.html` lines 73–78 and 82
  - `privacy.html` lines 47 and 50
  - the App Store privacy details

## Website copy to update once the cut is final (not yet)

- `sumo-arena.html` line 7: the meta description says "level up five abilities".
- `sumo-arena.html` lines 113–130: the "Level up" section and the `ability-menu.jpg` screenshot, which shows levels and an upgrade button.
- `support.html` lines 97–98: "How do I get chests and level up?"
- `support.html` line 82 and `privacy.html` line 47 list "ability levels" as saved data. That stays true while saves keep them.

---

## Found files (fill in during step 0)

| What | File | Line or node |
|---|---|---|
| Ability level → stats | | |
| Level tables and costs | | |
| Upgrade button and level labels | | |
| Ability descriptions | | |
| Bot difficulty → ability level | | |
| Save and load of ability levels | | |
| Quests, trophies, badges about levelling | | |
| Gameplay camera | | |
| Commit that changed the camera | | |
| Fixed level chosen (and why, if not 5) | | |
