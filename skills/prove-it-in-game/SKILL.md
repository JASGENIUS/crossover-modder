---
name: prove-it-in-game
description: Use before calling any crossover feature done, and whenever a player reports something vague like "it keeps hitting random stuff". Covers checking nobody is playing, an autopilot that drives the real object through the real world and logs every hit, a map-wide audit with before and after numbers, probes that time engine behaviour, counting objects in the engine's scene, performance numbers, clean screenshots and the fix-and-rerun loop.
---

# Prove it in the running games

Unit tests and a mock of the other game catch crashes and protocol bugs. Whether a car can drive
through a city only shows in the two running games. In the BeamNG x Minecraft project a stage
was marked DONE only after its pass condition was met live, and the build log keeps three states
apart for every feature: proven in game, tested offline, not yet seen.

## Before you touch the games

- Ask the game whether someone is playing, twice, a few seconds apart. Don't move their character
  or car, restart a game, or run a scripted drive while they are. BeamNG's `vehicle_drive` acts on
  the player car, so a test drive can steal the inputs of the car they're driving.
- Test in the separate profile. Never in their real one.
- Don't build the tree a running client loads from. Use a separate worktree.

## An autopilot for vague reports

"It's hitting random shit, I guess it only works good on a flat world" is not something you can fix
by reading code. What found the causes was a script that:

1. drives the guest's real car along the host's roads through the guest's own inputs, with the
   host player following as a spectator so chunks load and the terrain follows;
2. logs every hit: a sudden stop the inputs don't explain, a jolt upward, getting stuck;
3. for each hit saves the guest's terrain heights around the car next to the host's own column
   data (a dev command dumps the translator's raw height, its chosen height and its candidate
   levels per column), takes a screenshot from behind the car, then puts the car further along the
   road and carries on;
4. recovers by itself. BeamNG's arcade gearbox won't leave neutral until the car stands still, so
   the script settles the car with the parking brake first.

Seven causes came out of those logs, each with its own fix. Afterwards, random city drives of 3,
3.4 and 3.3 km hit only real things (lamp posts, fences, building corners).

## An audit over the whole map

The autopilot finds what it drives into. An audit finds the rest. For every road block in an area
(370 x 360 blocks of the city), compare three numbers: the guest engine's terrain height, the
translator's raw and chosen heights, and the road's own top. Sort the differences into named kinds:

| Kind | Meaning |
|---|---|
| sync | the engine differs from what was sent: a lost patch |
| ridge / dip | smoothing moved the road up or down |
| deck | a road on top that the car only gets when near (an elevated road, or a street wrongly read as a roof) |
| hole | the road is higher than the ground the engine has, and it's no deck |
| bump | something stands on the road in the engine only |
| wall | a road with room over it is buried under a structure |

Numbers before and after the fixes: bumps 4 to 0, holes 0, lost patches 0, ridges 1689 to 551 (the
rest are kerb slopes meant to be there), covered street walls 354 to 5 (a lava pit and a rail line,
neither a road).

## Probes that time the engine

When you suspect engine behaviour, measure it with the smallest test you can build:

- Lower the terrain 1.4 m under a parked car and time the fall: 0.77 s. Re-send an unchanged patch
  40 m away every 0.25 s and the fall waits until they stop. That's how the physics lag after
  `updateGrid` was found. With a collision rebuild after the patch the car fell after 0.08 s.
- Read live parameters at the start of every probe. The terrain's grid origin moves on each
  rebuild; a stale one made two tests patch the wrong place and report "never fell".

## Count what's in the engine

Ask the guest for its objects by name prefix and compare with what the bridge thinks it owns. That
count found 2996 cubes nobody owned, made while the level was still loading and orphaned by the
level-loaded reset.

## Performance numbers

Write them down before and after every change:

- collision rebuilds: 145 ms with about 8000 objects, 3.5-5.3 ms with about 500;
- the state stream: 59.5 Hz while driving with the rebuilds on;
- frame rates on both sides. Uncapped Minecraft (about 260 fps) held BeamNG to 68-111 fps and its
  state to 46-53 Hz, and the car stuttered; capped at 120 both ran at 120 and the state at 59-60 Hz;
- GPU memory before stacking heavy content (one driver reset came from full video memory).

## Screenshots and video

- Only the game image: the engine's own screenshot call (BeamNG `screenshot.doScreenshot`) or the
  game's client area. Windows notifications drawn over the game get captured too.
- Before recording, capture one frame and check it matches the game's own screenshot. One take
  captured the desktop on the wrong monitor.
- For materials, put the two engines' pictures side by side from the same camera.

## The loop

1. Reproduce it on a route you can run again.
2. Find the cause with numbers, not a guess.
3. Make one fix.
4. Rerun the same route and the audit, then a random drive.
5. Write the gotcha down as symptom, cause and fix, and add the engine call with its source file
   and line to the API notes.

## Don't

- Don't inject keys or pull a game window to the front.
- Don't leave a `tail -F` running on a game's log on Windows: it can outlive your monitor and lock
  the file.
- Don't say it works because the tests pass.
