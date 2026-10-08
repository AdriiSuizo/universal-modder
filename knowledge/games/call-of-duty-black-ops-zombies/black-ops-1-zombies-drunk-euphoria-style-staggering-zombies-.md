---
kind: game
title: 'Black Ops 1 Zombies: drunk (Euphoria-style) staggering zombies from the decompiled engine'
game: 'Call of Duty: Black Ops (Zombies)'
games_also: []
game_version: 'IW 3.0 / Black Ops engine. Steam Call of Duty: Black Ops, Zombies, run through the BO1Zombies decompiled-engine build (repo adriisuizo/bo1-zombies-decompiled, branch claude/lucid-turing-mfnut4) over a copy of the Steam install'
platform: windows
engine: native
route: decomp-recomp
tools:
- CMake + MSVC x86 (Visual Studio 2019/2022), DirectX SDK June 2010
- the repo's tools/headless.ps1 (windowless runs, log checks)
anti_cheat: none in this path (the build runs its own listen server from a copy of the install; the Steam client exe is never used)
status: in-progress
agents:
- Claude Code (Claude Fable 5.1)
humans: []
date: '2026-10-08'
links:
- https://github.com/adriisuizo/bo1-zombies-decompiled
tags:
- zombies
- procedural-animation
- actor-ai
- decompiled-engine
---
# Black Ops 1 Zombies: drunk (Euphoria-style) staggering zombies from the decompiled engine

> A mod (`mods/euphoria` in the BO1Zombies decompiled-engine repo) that makes the zombies stagger like GTA IV's drunk
> companions while keeping their own walk / run animations: they weave off the path, lurch sideways, lean into it,
> catch themselves, and one stumble in four is a near-fall. Route: a ~250-line procedural layer in the engine's actor
> move code, driven by one new script field per zombie (`self.drunk`), plus a GSC script that sets it. Written and
> desk-checked against the engine source in a Linux session; **not yet built or run** (needs the Windows build). The
> headless self-test it ships with (`euphoria PASS` in games_mp.log) is the oracle for whoever builds it next.

## Setup
- The repo is the engine: decompiled BO1 Zombies source, `CMakeLists.txt` (MSVC, x86, C++20, DirectX SDK June 2010 at
  `DXSDK_DIR`), `setup.ps1` copies the Steam install next to the built exe (zone/, iwds, videos; never BlackOps.exe).
  Mods are folders `mods/<name>` copied next to the exe on every build and loaded only with `+set fs_game mods/<name>`
  or stacked with `+set fs_mods "a b c"`.
- Mod hooks: `mods/_stack/maps/_modstack.gsc`. A mod ships `maps/<name>/_hooks.gsc` with `register()` adding functions
  to `"zombie_pentagon_ffotd::main_start"` (Five) or `"_zombiemode_ffotd::main_start"` (every map). The engine writes
  `maps/_modstack_list.gsc` from `fs_mods` (`src/clientscript/cscr_parser.cpp`).
- Headless runs: `tools\headless.ps1 -Zombies -Commands "..." -AutoQuitMs N` (private desktop, no window; logs in
  `build\Release\<fs_game>\games_mp.log` and `console_mp.log`).
- Nothing was installed in this session; the Windows toolchain is the repo's documented one.

## Route and why
- **Chosen: engine (server actor code) + script.** The retail zombie scripts and the animtree live in the game's
  fastfiles, not in the repo, so a script-only mod cannot reference stumble / fall animations by name (`%anim` not in
  the animtree fails the compile) and GSC has no way to bend an actor's heading or push it sideways without fighting
  the AI every frame. The engine has the hooks: the actor's move direction and stride are built in one function and
  its body angles in another.
- **Rejected: real active ragdoll.** NaturalMotion's Euphoria is behaviour-driven ragdoll; the engine's ragdoll
  (`src/ragdoll`) is a passive client-side corpse ragdoll, and live AI is a server actor. Weeks of work for a worse
  result than a tuned procedural layer.
- **Rejected: client bone post-processing.** Possible (`DObjCalcSkel` in `src/xanim/dobj_skel.cpp`), but needs the
  drunk amount on the client (a netfield or a hack) and doesn't move the actor. Whole-entity pitch / roll are already
  networked and rendered, so the server alone is enough.

## How the game works (what we had to learn)
All in `src/game_mp/actor_mp.cpp` unless noted.
- Actors think every 50 ms (`Actor_Think`). Per frame: `Actor_UpdateOriginAndAngles` → `Actor_UpdateAnglesAndDelta`
  (move delta and orientation) → `Actor_DoMove` (physics, `Actor_PhysicsAndDodge`).
- Anim modes (`eAnimMode`): in `AI_ANIM_MOVE_CODE` (normal pathing) the stride LENGTH comes from the animation's root
  motion (`XAnimCalcDelta` via `Actor_GetAnimDeltas`) but the DIRECTION comes from the path lookahead:
  `Path_UpdateMovementDelta` sets `Physics.vWishDelta = fMoveDist * lookaheadDir` and stores a 2D look direction into
  `moveHistory`, which `Actor_FaceMotion` (`src/game/actor_orientation.cpp`) averages to turn the body
  (`ai_angularYawEnabled` is off by default, so the body follows the history, not the wish delta). The other modes
  (`USE_POS_DELTAS`, `USE_BOTH_DELTAS`, ...) are scripted animations (window climbs, traversals, attacks) and rotate
  the animation's delta by `fDesiredBodyYaw` in `Actor_DoMove`: perturb yaw there and the climb misses the window.
- `Actor_SetBodyAngle` writes `currentAngles[0] = 0` and `[2] = 0` every frame: retail actors never pitch or roll.
  Anything written after `Actor_UpdateBodyAngle` survives to `Actor_Think`'s copy into `s.lerp.apos.trBase`.
- All three `lerp.apos.trBase` components are netfields (`src/qcommon/msg_mp.cpp`, `MSG_FIELD_ES_ANGLE`) and the
  client builds the actor's axis from all three (`CG_CalcEntityLerpPositions` → `BG_EvaluateTrajectory(apos)` →
  `AnglesToAxis(cent->pose.angles)`, `src/cgame_mp/cg_ents_mp.cpp`). The client's actor controller
  (`CG_Actor_DoControllers`, `cg_pose_mp.cpp`) bends the spine by `pose.actor.pitch`, which the actor path never sets
  (it aliases the turret barrel pitch zeroed in `CG_Actor_PreControllers`).
- `s.animState.fLeanAmount` IS networked for actors (`Actor_UpdateNetworkLeanAmount`) but the client only uses it for
  the dog run-blend weights (`bg_dog_animations_mp.cpp`); human actors ignore it. Not a free sway channel.
- Per-actor extra state: `actor_s` keeps its decompiled size (0x2780, strides hard-coded), so SP-only / mod state goes
  in the side table `actor_sp_ext_t` (`src/game_sp/actor_sp_ext.h`, one slot per actor, zeroed in `Actor_SetDefaults`
  on every spawn). A script field on it is one row in `aifields` (`src/game/actor_fields.cpp`, explicit array size,
  92 → 94 here) with ofs `AF_SP_EXT` and the generic ext setter / getter, plus a name → offset row in
  `g_actorSpExtFields` (`actor_sp_ext.cpp`). Rows register as script class fields at init (`Scr_AddClassField`), so
  `self.drunk = 0.8` in GSC lands in the engine struct.
- Mods' per-zombie threads: poll `GetAiSpeciesArray("axis", "all")`, mark each with a script var, skip specials by
  `animname != "zombie"` (a special has no `animname` in its first frames, so wait 0.5 s before deciding), dogs by
  the `isdog` field, crawlers by `has_legs == false`.

## Build steps
1. In `adriisuizo/bo1-zombies-decompiled` on the branch, the change is: `src/game_sp/actor_sp_stagger.{h,cpp}` (new),
   three hook lines in `src/game_mp/actor_mp.cpp` (`Actor_Stagger_Move` / `_Push` in `Path_UpdateMovementDelta`,
   `Actor_Stagger_Tilt` at the end of `Actor_UpdateAnglesAndDelta`), the `drunk` / `drunkstumbles` field rows, the
   side-table member, `cmake_files.cmake`, `mods/euphoria/`, `tools/euphoria_flags.js` and the launcher select.
2. Build as usual (`cmake --build build --config Release`); the build copies `mods/` next to the exe.
3. Play: launcher "Drunk zombies" (Tipsy 0.4 / Drunk 0.75 / Wasted 1), or
   `+set fs_game mods/euphoria +set fs_mods euphoria +set euphoria_level 0.75 +devmap zombie_theater`.
4. Tune with `bo1_mod_stagger_weave` (deg), `_sway` (deg), `_rate` (stumbles/s), `_push` (units/s), `_stride` (0..1),
   `_debug <entnum>` (per-frame console line).

## Verification
- **Not run.** The session was a Linux container with the source only: no MSVC, no DirectX SDK, no game. Every engine
  call in the new code was checked against its declaration in the repo's headers, and the hook points were read in
  full (the functions above), but the first Windows build is still the first compile.
- The oracle shipped with the mod: `tools\headless.ps1 -Zombies -Commands "+set fs_game mods/euphoria +set fs_mods
  euphoria +set euphoria_selftest 1 +devmap zombie_theater" -AutoQuitMs 150000`, then grep `euphoria` in
  `build\Release\mods\euphoria\games_mp.log`: one line every 5 s with drunk zombies alive, the largest body roll and
  pitch seen (`z.angles[2]` / `[0]` of the AI entities, retail always 0) and the engine's stumble count
  (`z.drunkstumbles`); `euphoria PASS` once a zombie has stumbled and leaned > 2°, `euphoria FAIL` after 180 s.
- What a human still has to judge: whether the feel is right (the amounts are a first guess), whether the feet slide
  too much during a lurch (then lower `bo1_mod_stagger_push` / `_stride`), and that window climbs / doors / attacks
  are unaffected (the engine gates on `AI_ANIM_MOVE_CODE`, by reading, not by test).

## Gotchas
1. **Symptom.** A `%ai_zombie_...` animation name in a mod script that is not in the map's animtree. **Cause:** `%name`
   resolves at compile time against `#using_animtree`; the retail animtree is in the fastfile, not the repo. **Fix:**
   don't guess names; drive what you can from the engine (this mod), or read the names from the fastfile's scripts
   with the game at hand before using them.
2. **Symptom.** Pitch / roll written to an actor's `r.currentAngles` vanish. **Cause:** `Actor_SetBodyAngle` zeroes
   them on every orientation update. **Fix:** write them after `Actor_UpdateBodyAngle` in
   `Actor_UpdateAnglesAndDelta`; they are then copied to the networked `apos` in `Actor_Think`.
3. **Symptom.** Turning the move direction in `Path_UpdateMovementDelta` makes the zombie slide sideways while facing
   the path. **Cause:** the body yaw follows `moveHistory` (filled from the path's look direction), not the wish delta.
   **Fix:** rotate the look direction that goes into `moveHistory` by the same angle.
4. **Symptom.** A heading perturbation breaks window climbs / traversals. **Cause:** those run in other anim modes
   whose deltas are rotated by `fDesiredBodyYaw` in `Actor_DoMove`. **Fix:** only perturb in `AI_ANIM_MOVE_CODE`
   (the `Path_UpdateMovementDelta` branch is only reached there) and fade the lean out in the other modes.
5. **Symptom.** `self.myfield = x` on an AI is just a script variable. **Cause:** engine fields need a row in `aifields`
   (and, for side-table storage, in `g_actorSpExtFields`); the array has an explicit size. **Fix:** add the rows and
   bump the size; the rows register as class fields at init.
6. **Symptom.** The networked `fLeanAmount` looked like a free "lean" channel. **Cause:** the client only reads it for
   dog run blends. **Fix:** use the entity's pitch / roll (all three apos angles are netfields and rendered).

## Assets
None: the mod reuses the game's own animations.

## Cost and time
One session, reading ~15 engine functions in full plus the mod conventions; ~250 lines of C++, ~150 of GSC, docs.

## Open questions
- Real falls and get-ups: BO1 has knock-down / get-up clips (the Thundergun uses them at high rounds); read their names
  from the fastfile's `animscripts\zombie_utility.gsc` / `maps\_zombiemode_spawner.gsc` with the game, then trigger
  them from the engine's near-fall (a notify) so the zombie goes down and gets up.
- Foot IK (`src/ik`) to pin the feet during a lurch and remove the slide.
- The amounts (weave 20°, sway 8°, 0.45 stumbles/s, 120 u/s push) are untested first guesses.
