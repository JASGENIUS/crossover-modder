# BeamNG.drive x Minecraft

Real BeamNG cars driven through Minecraft worlds, and the older reverse mode with Minecraft drawn
over BeamNG. Code: [minecraft-beamng](https://github.com/JASGENIUS/minecraft-beamng).

**Status:** playable on Windows in both directions. Proven in the running games: driving on flat
worlds, generated terrain, a house map with a garage, and a 4.5 km² city map with a subway, an
elevated highway and towers; crashes, dents, TNT, water and a flooded engine, opening doors, any
car from BeamNG's own list plus mod cars, shader packs.

## Versions

| | |
|---|---|
| BeamNG.drive | 0.39.4.0, build 20972, Steam, Direct3D 12 |
| Minecraft | Java Edition 1.21.1, Fabric Loader, Loom 1.11, Java 21 |
| Shaders | Sodium 0.6.13 + Iris 1.8.8 (newer builds need a newer Loom) |
| OS | Windows 11 |

## The route, and why

- **No third process.** BeamNG's game-engine Lua has the camera, raycasts, vehicles, physics and
  LuaSocket, so the BeamNG side is an extension in an unpacked mod in a user folder. The Minecraft
  side is a Fabric mod. They talk over UDP JSON on 127.0.0.1:47020. Nothing in either install is
  modified.
- **BeamNG runs on its own user folder** (`-userpath`), so the player's own mods and settings are
  never touched.
- **Who owns what.** BeamNG owns the cars (soft-body physics, damage, engine, gearbox, sound).
  Minecraft owns the world, the player, mobs and items, and draws everything. BeamNG's world render
  is switched off.
- **The world goes to BeamNG as a TerrainBlock** built by BeamNG's own generator from a 16-bit PNG,
  1024 x 1024 blocks around the player, patched live as blocks change. Posts and tree trunks near
  cars go as boxes. BeamNG runs the empty `smallgrid` level, whose floor is bedrock.
- **Cars come to Minecraft through BeamNG's own glTF exporter**, into BeamNG's user folder. Minecraft
  binds the mesh to the car's physics nodes and bends it on the GPU from a node stream every frame.
- **Scale 1.4 blocks per metre**, so a car's roof comes up to Steve's head.

## What the engines really do

- BeamNG rebuilds the whole level's static collision for any new collision object
  (`be:reloadCollision`), about 18 us per object, and a new object has no collision without it.
- BeamNG's physics takes new terrain heights about 0.7 s after the last `tb:updateGrid` anywhere;
  each call restarts the wait. `be:reloadCollision` pushes them in at once.
- `util_export` only exports player 0's car, writes normals in BeamNG's z-up axes next to y-up
  positions, and marks every material double-sided.
- BeamNG's `LuaQuat` multiplies the inverse way from Hamilton's; its camera FOV is vertical, like
  Minecraft's.
- Minecraft signs, banners and pressure plates block motion but have no collision shape.
- Minecraft's `NativeImage` reads PNG only.

## How it was verified

Each stage passed in the running games before the next. For the city: an autopilot driving
BeamNG's car along Minecraft's roads and logging every unexplained stop or jolt, a static audit of
every road block in a 370 x 360 block area, timing probes for engine behaviour, object counts in
BeamNG's scene, and side-by-side screenshots for materials. Lua modules have 85 unit tests run
through lupa; the Java side has unit tests for coordinates, the link, terrain rules and damage.

## Gotchas

| Symptom | Cause | Fix |
|---|---|---|
| BeamNG crashes at startup | `-windowed` calls a missing global in 0.39.4 (`parseArgs.lua:58`) | never pass it; set window mode in the profile's settings |
| Settings file ignored | settings JSON written with a BOM | write UTF-8 without a BOM |
| A first-run wizard covers the game | fresh profile asks about online features | seed `onlineFeatures` and `telemetry` as `disable` |
| BeamNG at 30 fps behind Minecraft | background fps limit | hold it at 120 while connected (turning it off let BeamNG run 700+ fps and starve Minecraft) |
| The car stutters | uncapped Minecraft (about 260 fps) left BeamNG short of GPU | cap Minecraft at 120 fps |
| BeamNG cars silent | BeamNG mutes when unfocused | turn off `AudioMuteOnWindowLoseFocus` in the test profile |
| Car falls through a road BeamNG already reports | physics lags `updateGrid` while patches stream | `be:reloadCollision()` after a terrain flush, at most every 0.25 s |
| Car hits things in a city that aren't there | streets read as roofs, smoothing ridges, signs as pillars, wrong level under a highway, a stone block read as a floating wall | the column rules in the `translate-world` skill |
| A lamp post stops cars a metre early | a thin post filled a whole heightfield sample | posts and trunks as exact boxes near cars |
| Collision rebuilds take 145 ms | about 8000 objects (a cube per log ever scanned) | trunks as posts near cars only: 4-5 ms |
| 2996 unowned cubes in BeamNG | Minecraft sent blocks while the level was loading; the level-loaded reset forgot them | delete owned objects on that hook, delete strays on load |
| City towers became pits | a height of exactly the terrain's top wraps to 0 (16-bit) | clamp a step under the top |
| A city street chunk 0.7 m high | about 1 in 150 terrain patches lost while BeamNG was busy | ack every patch, resend after 3 s |
| Parked cars fell 62 m | a new terrain replaced the one they stood on | a flat stand under each car left outside |
| A car spawned under the terrain | `spawn.safeTeleport` only searches 10-50 m down | spawn over the terrain, rescue cars more than 2 m inside it |
| The link drops while a car loads | a big mod's export holds BeamNG 17-29 s | expect the stall before asking and keep the session |
| Cars look dark and shiny | exporter normals in z-up axes | turn them (x, z, -y) |
| Tyres drawn as white chrome | mod JPEG textures dropped by a PNG-only reader | decode with stb |
| Black trims come out grey | colour factors are gamma | apply a 2.2 power |
| A car's paint drawn as glass | `translucent` with blend op None is solid in BeamNG | treat it as solid |
| GPU driver reset | full-size 4K and 8K car textures filled 8 GB | scale to 1024 px on load |
| A panel shakes on a parked car | node stream rounded to centimetres | send millimetres |
| A car without a hitbox far from the player | proxy entity in a chunk that isn't ticking | move it from the server tick |
| A scripted stop creeps backwards | brake held at rest selects reverse in the arcade gearbox | stop, then parking brake |
| The dev client runs slowly on a laptop | the Gradle JDK ran on the Intel iGPU | set a Windows high-performance GPU preference for that `java.exe` |
| Iris breaks custom drawing | Iris caches the bound program and holds back masks | bind and set masks with plain GL, restore after |
| Iris mixins fail in a dev client | Iris loaded from `run/mods` | add Sodium and Iris through Gradle |
| Controlify breaks the Gradle build | its newer class tweaker format | put its jar in `run/mods` instead |

## Open

Railings stay terrain, so their face is up to about half a metre out from the bars. Terrain has no
grip materials (it all drives as asphalt). Lakes above sea level get no water.
