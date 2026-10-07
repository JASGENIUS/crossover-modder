# crossover-modder

Skills that teach an AI coding agent to put one game inside another by translating it. One game's
world becomes real collision in the other, the other game's objects keep their own physics and
damage, and one engine draws everything from your own installs. It works as a Claude Code plugin
and with any agent that reads `AGENTS.md` or the Agent Skills format.

Everything here comes from one long project,
[minecraft-beamng](https://github.com/JASGENIUS/minecraft-beamng): real BeamNG.drive cars driven
through Minecraft worlds, a whole city map included. Each rule in these skills was measured in
those two running games, or broke there and got fixed. It's a playbook for an agent, not a
program that mods games by itself.

## Overlay vs translation

A common way to mash up two games is passthrough: run both and combine their pictures. The guest
renders its own pixels, and they're composited into the host's frame, often depth-tested against
the host's walls and relit with the host's light. That's Pattern 2 in
[universal-modder](https://github.com/rehan-remade/universal-modder)'s `mashup-mods` skill, and
it's how [minecraft-crossover-bridge](https://github.com/justbustin/minecraft-crossover-bridge)
puts Minecraft into Monster Hunter and Elden Ring. It gets a demo on screen fast. The guest's
things still exist in the host only as far as the bridge carries them (terrain rays, monster
hitboxes, damage), and two renderers still draw the one picture.

These skills go the other way. One game draws everything, each game simulates what it's good at,
and each gets a translation of the other's part:

- **The world becomes ground.** A Minecraft city is rebuilt for BeamNG as a terrain heightfield
  plus exact boxes for posts and tree trunks. The car drives on the street, under the elevated
  highway and into a garage, and it stops against a lamp post.
- **Objects are drawn by the host.** BeamNG exports the car with its own glTF exporter into your
  BeamNG user folder. Minecraft draws that mesh with BeamNG's materials and bends it every frame
  from BeamNG's node positions, so a crumpled bonnet shows up in Minecraft.
- **Damage goes both ways.** Minecraft TNT within 3 m blows the car up through BeamNG's own
  explode call, a sword dents it, a crash hurts the player in the driver's seat, and a car hits
  mobs.
- **One place to play.** You drive from Minecraft's keyboard or controller, and each control runs
  the same vehicle Lua that BeamNG's own key binding runs.

## Install

| Agent | How |
|---|---|
| Claude Code | `/plugin marketplace add JASGENIUS/crossover-modder`, then `/plugin install crossover-modder@crossover-modder` |
| Skills only (any agent) | `npx skills add https://github.com/JASGENIUS/crossover-modder` |
| Anything else | `git clone https://github.com/JASGENIUS/crossover-modder` and start your agent inside it. `AGENTS.md` says where the skills are; `CLAUDE.md` points to it |

## Try it

> Put BeamNG cars in my Minecraft world properly. The car should drive on the blocks, crash into
> walls and get dented by swords.

> I want to drive cars from a racing game with Lua mods through a Minecraft map. Plan it stage by
> stage before writing code.

> My crossover car keeps hitting invisible things in a city map. Find out why and prove the fix.

> Bring Minecraft mobs into my driving game as real obstacles the car's physics can hit.

The agent starts with `game-crossover`. It works out which game should own what, then builds in
stages and only calls a stage done after seeing it work in the running games.

## Skills

| Skill | What it covers |
|---|---|
| `game-crossover` | The entry point: who owns what, one coordinate mapping, the stage gates, the safety rules |
| `translate-world` | Turning a voxel or level world into the other engine's collision: heightfields, roofs, decks, tunnels, posts, physics refresh, budgets |
| `translate-objects` | Drawing the other game's objects natively: stand-ins first, the engine's own exporter, node-driven bending, materials, damage, controls, doors |
| `crossover-link` | The localhost protocol: sessions, rates, what may be lost and what needs an ack, load stalls, dev commands, a mock of the other game |
| `prove-it-in-game` | Checking it for real: an autopilot, a map-wide audit, live probes, counting scene objects, before and after numbers |

`knowledge/` holds field notes. The first one is
[`knowledge/beamng-minecraft.md`](knowledge/beamng-minecraft.md): versions, the route, and a table
of gotchas with their causes and fixes.

## The worked example

[minecraft-beamng](https://github.com/JASGENIUS/minecraft-beamng) is the code these skills were
written from. It runs both ways on Windows: BeamNG cars in Minecraft (the translated direction),
and the older overlay mode with Minecraft inside BeamNG. Its build log,
[`beamng/docs/minecraft-host.md`](https://github.com/JASGENIUS/minecraft-beamng/blob/main/beamng/docs/minecraft-host.md),
records each step with what was proven in game and what wasn't. The BeamNG calls it uses are
listed with their source lines in
[`beamng/docs/beamng-api-notes.md`](https://github.com/JASGENIUS/minecraft-beamng/blob/main/beamng/docs/beamng-api-notes.md).

## Rules

- Only games you own, run from your own install. Nothing from either game goes into a commit:
  exports, textures and converted files stay in your own user folders.
- Single player and offline. Don't connect a modded game to online services or multiplayer
  servers, and leave games with anti-cheat alone.
- The agent works in a separate profile or user folder for testing, never your real one.

## Credits

- [universal-modder](https://github.com/rehan-remade/universal-modder) by rehan-remade (MIT): the
  idea of a skill pack that lets an agent mod any game, and the mashup patterns this builds on.
- [minecraft-crossover-bridge](https://github.com/justbustin/minecraft-crossover-bridge) by
  justbustin (MIT): where the BeamNG port started.

MIT licence, see [LICENSE](LICENSE). Made by JASGENIUS. Not affiliated with Mojang, Microsoft or
BeamNG GmbH.
