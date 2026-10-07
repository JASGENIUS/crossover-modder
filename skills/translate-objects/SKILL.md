---
name: translate-objects
description: Use when one game's objects (cars, creatures, props) should exist inside another game with their own physics, drawn by the host engine. Covers stand-in hitboxes, exporting through the guest engine's own exporter, binding the mesh to physics nodes and bending it every frame, matching materials, damage both ways, controls, doors and stalls. Examples from BeamNG cars drawn in Minecraft.
---

# Translate objects: the host draws the guest's things

The guest engine keeps simulating its object. The host gets enough to collide with it, draw it and
send damage back. Build it in the order below; each step was useful on its own.

## 1. Stand-ins first

- One invisible solid entity per guest object, placed straight from the guest's data every update.
  BeamNG sends each car's oriented box (`be:getObjectOOBBCenterXYZ`, `getObjectOOBBHalfAxisXYZ`) at
  20 Hz, for cars within 400 m of the camera, at most 64.
- That alone gives TNT, punches, mobs and standing on the roof, before any drawing exists.
- Server and client read the same parsed snapshot, so both copies of the box agree.
- Ignore impossible values. A freshly spawned BeamNG car reports about 52,000 m/s for one message;
  speeds over 150 m/s aren't extrapolated.
- An entity in a chunk that isn't ticking doesn't tick. A proxy that followed its car in its own
  tick was stranded 1100 blocks back and the car had no hitbox. Move non-ticking proxies from the
  server tick.
- The guest stalls (a BeamNG car swap holds it about 2 s, a big export much longer). Keep stand-ins
  and seated riders through silence (90 s in the example) and keep the last pose instead of removing
  them.

## 2. Export with the guest engine's own exporter

- BeamNG's `util_export` writes a glTF of the player car into the user folder. Nothing of the game's
  is copied into the mod. It only exports player 0's car, so lend the slot with `be:enterVehicle`
  for the length of the synchronous call and give it back.
- A model's first export converts its textures and froze BeamNG for 20-30 s. Converting the DDS
  textures a few per frame beforehand fixed most of that (22 textures in 3.4 s). The export itself
  still holds BeamNG about 2 s on a big car, over 8 s on some, 17-29 s for big mods.
- Cache by model plus part configuration (`veh.partConfig`), and share an export only when it was
  made undamaged. Answer a repeated question from the last export's files, so a host restart
  reloads every car without exporting again.
- Write a side file next to the export with what the exporter leaves out: node offsets at export
  time, the flexbody node groups, paint, and per-material fields read with `mat:getField`.

## 3. Bind the mesh to physics nodes and bend it

- BeamNG flexbodies follow nodes. Bind each vertex to three nodes of its own flexbody's node group,
  at least 10 cm apart with a wide angle where the part has them. Nodes 1 cm apart made the mesh
  shake.
- Stream node offsets every frame per car in range, in millimetres (`q` = 1000). Whole centimetres
  shook body panels by up to 21 cm; millimetres leave 4-7 mm. A Scintilla is about 17 KB per message.
- Keep the last 8 snapshots and draw 25 ms in the guest's past, blended between the two around that
  time. Uneven UDP arrival stops showing as stutter. The camera reads the same blended pose, so car
  and camera never disagree.
- Bend on the GPU. Each vertex carries its node frame and its position in it; per frame only the
  nodes go up, as a buffer texture. CPU time for an 809k-triangle car went from 20.2 ms to 0.2 ms a
  frame. Skip cars outside the view.
- Drop triangles stretched past twice their rest length in a geometry shader, as BeamNG hides a
  broken flexmesh, so parts that break off don't trail strands.
- Props (steering wheel, mirrors, pedals) have their own vertex buffers and node transforms in the
  export. Place them through their node chain.
- Check which GPU you're on. The dev JDK ran on the laptop's Intel chip until Windows got a
  high-performance preference for that `java.exe`.

## 4. Match the materials

Read the guest's shader source and follow it. BeamNG's layered materials are built in
`defaultMat.hlsl` (`processLayers`, lines 212-424). What went wrong:

| Symptom | Cause | Fix |
|---|---|---|
| Cars too dark and too shiny | the exporter writes positions in glTF axes (y up) and normals still z up (`util/export.lua:864-879`) | turn normals (x, y, z) -> (x, z, -y); they then match their faces at 0.91-0.97 |
| Every face drawn from both sides | glTF `doubleSided` is true for every material (`export.lua:270`) | use BeamNG's own flag in the extras (`:800`) |
| Black trims came out grey | material colour factors are picked as they look (gamma) | apply a 2.2 power before linear lighting |
| Tyres and carbon drawn as white chrome | the exporter embeds mod JPEGs as they are, and Minecraft's `NativeImage` reads PNG only (32 of one car's 90 images) | decode non-PNG images with stb; log any that still fail |
| Black rims came out silver | BeamNG blends a detail map over white where a layer has no base map (`defaultMat.hlsl:294-307`) | do the same |
| Glass all at one opacity | opacity map dropped | follow BeamNG: bottom layer factor times opacity map, upper layers by their own, premultiplied |
| Car paint drawn as glass | `translucent = 1` with blend op "None" is solid in BeamNG | treat it as solid |
| Body paint came out white | palette paint rules | mask alpha is how much paint, mask rgb which of the three paints, applied when `paletteBaseColor` is on |
| GPU driver reset | car mods ship 4K and 8K maps; one car's 123 images took 1.8 GB of an 8 GB card | scale car textures to 1024 px on load (that car: 326 MB) and log each car's texture memory |

Compare side by side with the guest's own picture from the same camera before calling it done.

## 5. Damage both ways

Use the guest's own damage calls, the ones its menus and tools use:

| Host event | Guest call (BeamNG vehicle Lua) |
|---|---|
| Explosion within 3 m of the body (0.75 m per unit of power) | `fire.explodeVehicle()` then `beamstate.breakAllBreakgroups()`, as BeamNG's "explode vehicle" |
| Farther explosion | a push from a short-lived "planet" at the blast centre: `beamstate.addPlanet(center, radius, mass, dt)` |
| Sword, arrow, punch | a dent: nodes within 0.3 m get their own mass times a speed change over 15 ms (`obj:getNodeMass`), fading to the edge |
| Edge or point near a wheel | `beamstate.deflateTire` |
| Fire, lava, flint and steel / water bucket | `fire.igniteVehicle()` / `fire.extinguishVehicle()` |

A fixed point force barely dented heavy panels and tore light nodes off; mass-scaled pushes fixed
it. Going the other way, a crash hurts the driver in the host and the car hits mobs. Cancel the
host's suffocation for anyone seated: the driver sits inside the car's box.

## 6. Controls

- Send driving input every frame (at most 60 Hz) through the guest's own input events with its
  keyboard or gamepad filter. Have the guest zero the inputs if they stop arriving.
- Every other control runs the same vehicle Lua as the guest's own key binding (shift, gearbox
  mode, ignition, horn, lights, reset, rewind).
- Holding the brake at rest makes BeamNG's arcade gearbox select reverse, and from neutral it waits
  for the car to stand still. Scripted stops should use the parking brake.

## 7. Doors, pickers and other extras

- **Doors.** BeamNG's own click raycasts from BeamNG's camera, which isn't where the host player
  looks. Send the host's eye ray, test it against the trigger boxes placed as BeamNG places them,
  and fire `core_vehicleTriggers.triggerEvent` like BeamNG's click. Many mods have no door
  triggers but shut their doors on couplers: toggle the `advancedCouplerControl` group instead.
  One mod's triggers were 1 cm boxes at the world origin, not on the car.
- **Picker.** List models from the guest's own list (`core_vehicles.getModelList`, its configs and
  preview pictures). The first call on a fresh BeamNG blocks about 4 s.
- **Shader packs.** A pack draws the world through its own buffers. Draw the cars into a target of
  your own with a copy of the world's depth and composite after the pack. Iris also caches GL state,
  so bind programs and set masks with plain GL and restore them.
