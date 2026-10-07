---
name: translate-world
description: Use when one game's world has to become ground and collision in another engine, for example a Minecraft map as BeamNG terrain so real cars can drive on it. Covers cubes vs boxes vs heightfields, column surface rules for cities (roofs, streets, decks, tunnels), smoothing, thin posts as exact boxes, keeping the physics in step with edits, object budgets, lost patches and the edge cases that broke a city map.
---

# Translate the world into the other engine's ground

Examples are from Minecraft hosting BeamNG cars (scale 1.4 blocks per metre, so one block is
0.71 m). The rules carry over to any voxel or level world fed into a physics engine.

## Pick the representation

| Shape | Use it for | Cost in the example |
|---|---|---|
| Collision cubes (BeamNG TSStatic `gm_cube_1m.dae`, scaled) | single blocks, merged ground boxes, posts | a new TSStatic has no collision until `be:reloadCollision()`, which rebuilds the whole level's static collision: about 0.53 s on gridmap_v2 (~20,900 instances), about 18 us per object on an empty level |
| Heightfield (BeamNG `TerrainBlock` from `util/terrainGenerator.lua`) | natural ground, roads, anything a wheel rolls over | 1024 x 1024 samples built in 450-540 ms |
| Engine's own floor | bedrock | smallgrid's floor at z = 0 can't be lowered: moving or deleting its GroundPlanes leaves the collision. Use it as the bottom and raise everything else |

A Minecraft hillside climbs one block at a time, and 0.71 m is a wall to a car, not a step. Boxes
gave a flat world with holes; a real world needed a heightfield with the steps turned into slopes.

## Build the heightfield

- One sample per block for smooth ground, or 2 x 2 per block for "real blocks" (a one-block step
  then rises 0.70 m within 0.35 m, measured with `getTerrainHeight` every 5 cm). In real-block mode
  a half-block step becomes a ramp, since a sharp 0.36 m edge is about a sports car's wheel radius.
- A column's height is the top of what its block collides with (a slab is half a step, a dirt
  path 15/16), not the block's full cube.
- Smooth only between neighbours a step apart: one block, or two between natural columns on a
  hillside. A blur over every neighbour lifted roads two blocks from a barrier by up to 0.7 m;
  limiting it took audited ridges from 1689 road blocks to 551, the rest being kerb slopes meant to
  be there. Columns the blur would move by more than a block keep their height, so pits, cliffs
  and walls stay sharp.
- Write the heightmap as a 16-bit PNG into the engine's user folder and send the path. BeamNG's
  generator deletes every `*heightMap.png` under `temp/` on init, so name yours differently.

## Column rules for cities

A heightfield has one height per column. In a city a column can hold a roof, a street, a subway
and a highway deck. Scan each column top down and decide which run of blocks is ground. What went
wrong and what fixed it:

| Symptom | Cause | Fix |
|---|---|---|
| A house reached BeamNG as one solid lump the height of its roof | no overhangs in a heightfield | a solid run with a built block in it and at least 2 blocks of air under it is a roof; the ground is further down |
| 36% of audited street blocks were the subway floor 19 blocks down | "any built run with room under it is a roof" caught streets laid on dirt over a tunnel | loose soil (dirt, grass, sand, gravel) right under a built block belongs to a street; more than 3 blocks of it is ground whatever is under it |
| A garden on a garage roof filled the garage | lawn read as ground | up to 3 natural blocks over a built ceiling with room under it are still a roof |
| A twelve-block wall across the highway | one stone block of a monorail read as a floating island | natural blocks over a gap are ground only if no built floor with headroom comes first further down |
| The park road was blocked | a hill over a built tunnel lining | up to 16 natural blocks over a built lining is a tunnel |
| White walls didn't count as buildings | clay counted as natural | clay is a building block |
| Roads under towers were walls | the scan stopped at 128 blocks | scan 256 blocks down |
| Downloaded old maps read as the world floor | their heightmaps had the wrong length and Minecraft left them empty | find the top block yourself when the heightmap is empty |

## Several levels: pick the one the car can reach

Keep up to 3 decks per column besides the ground. Around each car, choose a column's level by a
breadth-first search over (column, level) from the car's column: starts within 1.5 below to 0.5
above the car, each step climbs at most 1.25 blocks, fewest steps wins, ties go to the level
nearest the car. Radius: 32 blocks plus 2 s of travel, at most 128. The earlier rule (the level
nearest the car's height carried along its nose's pitch) put lamp arms and awnings across the
road whenever the car pitched.

## Things that block movement but have no shape

Minecraft signs, banners and pressure plates are `forceSolidOn` (they block motion) with an empty
collision shape. A sign over the road stood in BeamNG as a three-block pillar. Test the collision
shape, not the solid flag.

## Thin things as exact boxes

A 0.36 m lamp post filled a whole sample, and the slope between samples spread it out another block
each way: about 1.2 m wide at bumper height. Now:

- A block whose collision shape is narrower than 0.75 blocks both ways, or a log, is left out of the
  terrain.
- Each chunk read keeps the ones standing on something as boxes of their own size (at least 0.28
  blocks a side). A chain hanging over air isn't kept.
- Only chunks within 24 blocks plus 1 s of a car's travel (at most 64) are sent, 16 chunk sends per
  refresh at most, and they leave again when no car is near.
- Connected fences, railings and window panes stay terrain. Boxing connected panes would open
  every building wall at its windows.

Proven at a city lamp post: a car passing 40 cm from it drove on, a car aimed at it stopped.

## Keep the physics in step

- BeamNG's `getTerrainHeight` reads new heights at once after `tb:updateGrid`, but the physics
  takes them about 0.7 s after the last `updateGrid` anywhere, and every call restarts that wait.
  With patches streaming as you drive, the physics stayed on the old ground and a car fell through
  a highway deck that BeamNG already reported. `be:reloadCollision()` after a terrain flush pushes
  them in at once (a parked car fell 0.08 s after it). Do it at most every 0.25 s and switch it off
  if a rebuild takes over 30 ms.
- `updateGrid` costs 8-12 ms a call. Several patches in one frame stalled BeamNG 50-70 ms and the car
  visibly lagged back. Set all heights, then one `updateGrid` over the union.

## Keep the object count low

Every collision object makes each rebuild slower (about 18 us each). With a cube per tree trunk ever
scanned plus the posts, BeamNG held about 8000 objects and rebuilds took 145 ms, which also tripped
the 30 ms cutoff and switched the terrain rebuilds off. With trunks as posts near cars only: about
500 objects and 4-5 ms (3.5-5.3 ms over a full drive). Leaves would be thousands of cubes: leave
them out.

## Deliver patches reliably

- Patches to BeamNG must fit in 8000 bytes (LuaSocket reads 8192 per datagram): at most 512 samples
  each.
- Give each patch an id and have the engine ack it; resend a chunk after 3 s without one. About 1
  patch in 150 was lost while BeamNG was busy (5 of 746 after a teleport), which left a street chunk
  0.7 m high.
- A full terrain load gets an id and is resent every 10 s until answered. The engine answers a load
  it already built with `same: true` instead of building it again and losing the patches since.
- The terrain covers 1024 x 1024 blocks around the player and is built anew within 192 blocks of the
  edge (about 0.6 s each time over an 1100-block drive).
- The terrain's grid origin moves on every rebuild. Anything that sends patches by grid position has
  to read the origin live: a stale one made two probe tests patch the wrong place without any error.

## Edge cases that bit

- **16-bit height wrap.** A sample set to exactly the terrain's height wraps to 0, so city towers
  became pits. Clamp a step under the top and build at least 320 blocks of height range.
- **Tall dimensions.** With the world raised to y -2032..2031, Minecraft's flat generator stacked
  its layers from the floor and made a 1970 m deep plain around a map. Set the flat air layer so the
  ground lands where the map is.
- **Cars left behind.** When a new terrain replaces the old, cars parked outside it fell 62 m. Give
  each one a flat stand at its height until a terrain covers it again. Keep the terrain when the
  host game quits, or cars fall and the next load lifts them onto roofs.
- **Spawning.** `spawn.safeTeleport` only looks 10-50 m down for ground (`spawn.lua:260-296`).
  Never ask for a car under the terrain; start spawns 0.4 m over it, and put back any car more than
  2 m inside the ground.
- **Water.** A `WaterPlane` at the host's sea level is real BeamNG water: vehicle code finds it with
  `obj:inWater`, and a sunk engine hydrolocks. It has no edges, so dry ground below sea level should
  count as water too (expected, not seen).
- **Grip.** A TerrainBlock with no materials drives as asphalt everywhere (BeamNG logs
  `ground model not found ... using asphalt`).
