---
name: game-crossover
description: Use when someone wants two games to cross over, such as one game's cars, creatures, items or world inside another, or asks for a mashup done "properly" instead of an overlay. The entry point. Decides which game owns what, plans one coordinate mapping and the stage gates, then hands off to translate-world, translate-objects, crossover-link and prove-it-in-game.
---

# Game crossover: translate, don't overlay

Laying one game's picture over the other's gets a crossover on screen quickly. This project's
first version did that: Minecraft in a transparent window over BeamNG, always in front of BeamNG's
world, each game lighting its own half. What worked better in the BeamNG.drive x Minecraft project
([minecraft-beamng](https://github.com/JASGENIUS/minecraft-beamng)) is to let one game draw
everything, let each game simulate what it's good at, and hand each side a translation of the
other's part.

## 1. Recon before code

- Write down exact versions. The worked example: BeamNG.drive 0.39.4.0 (build 20972), Minecraft
  1.21.1 with Fabric, Java 21, Windows 11.
- Find each game's scripting surface. BeamNG has game-engine Lua and per-vehicle Lua, with
  LuaSocket built in, loaded from an unpacked mod in the user folder. Minecraft has Fabric mods.
  With both scriptable, no third process and no injected DLL was needed.
- Read the engine's own scripts for every call you plan to use and keep an API notes file: the
  call, the file and line where the engine itself uses it, and a state (TESTED, CONFIRMED,
  UNVERIFIED, REJECTED). The example's notes are in
  `beamng/docs/beamng-api-notes.md`. If the game's scripts are under a licence that forbids
  copying, cite the lines and don't paste the code.
- Check what the plain game can't do. BeamNG's camera sensors and shared-memory frames need a
  BeamNG.tech licence; the normal game doesn't have them.

## 2. Decide who owns what

Each thing has one owner that simulates it. The other side gets a translation and sends requests
back (inputs, hits, explosions). In the Minecraft-hosts direction:

| Thing | Owner | The other side gets |
|---|---|---|
| Car physics, damage, engine, gearbox, sound | BeamNG | node positions every frame, car boxes at 20 Hz, dashboard values |
| World, blocks, mobs, items, the player, rendering | Minecraft | a terrain heightfield, boxes for posts and blocks, a water plane |
| Driving input | Minecraft (keys, controller) | BeamNG's own vehicle input events |
| Hits and explosions | whoever caused them | the owner's own damage calls (dents, explode, fire) |

BeamNG's world render is switched off (`setRenderWorldMain(false)`, GPU load 83% to 18%). BeamNG
keeps running the physics, and Minecraft draws everything.

The first version went the other way (BeamNG hosts, Minecraft drawn over it in a transparent
window). It still ships as a mode, and building it first gave the link, the coordinates and the
camera sync that the translated direction reused.

## 3. One coordinate mapping, in one file

- Pick a canonical frame and let one side speak only that. In the example it's BeamNG's world
  frame (metres, X east, Y north, Z up); the Lua side never converts, and every conversion lives in
  one Java class with its own tests (`CrossoverCoords`).
- Minecraft to BeamNG is `(x, y, z) -> (x, -z, y)`, a proper rotation, so angular velocity maps
  like any vector. Minecraft velocities are blocks per tick: times 20 first.
- Measure conventions, don't assume them. BeamNG's `LuaQuat` rotates vectors the inverse way from a
  Hamilton quaternion with the same components (`quatFromAxisAngle(Z, +90) * X = -Y`), so the
  conversion is a conjugate. Both games' FOV is vertical degrees.
- Choose the scale on purpose. 1 block = 1 m when BeamNG hosts. When Minecraft hosts it's 1.4
  blocks per metre, so a car's roof comes up to Steve's head. Then everything goes through that
  scale: drawing, hitboxes, the chase camera, exit spots, collision cubes and ground boxes, so what
  is drawn is what collides.
- Test each axis both ways, round trips, the four compass yaws and FOV conversion.

## 4. The link

UDP JSON on 127.0.0.1, one message per datagram, latest state wins, acks only where a message
must arrive. Read `crossover-link` before writing it.

## 5. Stage gates

Each stage is DONE only when its pass condition was met in the running games, not just in tests.
The order that worked:

1. The mod loads in a clean, separate profile with no errors in the log.
2. Two-way telemetry at 30 Hz or more, with ping, reconnect and stale-session tests (measured:
   60.3 Hz, 0.66 ms round trip, 0 drops).
3. One object from game A shows in game B at the right spot.
4. Positions every frame; the camera of one drives the other's (checked to within 2.6 px).
5. Collision: the guest's ground and walls hold in the host (`translate-world`).
6. Stand-ins: invisible solid boxes for the guest's objects (`translate-objects`).
7. Input: drive, steer, brake, shift, from keyboard and controller.
8. Crashes, explosions, mobs.
9. Real content: the guest's objects drawn natively, real maps, cities.

Write the stage list down with a goal, an automated test, a manual test and a pass condition
each, and keep its status honest.

## 6. Expect the flat world to lie

The crossover worked on a flat world long before it worked in a city. On a real map, the player
reported "hitting random shit", and an autopilot found seven separate causes (streets read as
roofs over a subway, physics lagging terrain updates, smoothing ridges, signs as pillars, the wrong
level picked under a highway, a monorail block as a floating wall, a park tunnel). Plan the audit
tools from `prove-it-in-game` before you get there.

## 7. Safety

Follow the hard rules in `AGENTS.md`: a separate test profile, check nobody is playing before you
move anything, no key injection, game image only in screenshots, nothing claimed before it's seen
in game, no game files in commits, single player only.
