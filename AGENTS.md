# crossover-modder: instructions for agents

You're helping someone put one game inside another. Start by reading
`skills/game-crossover/SKILL.md` and follow it. It sends you to the other skills when you need
them:

- `skills/translate-world/SKILL.md`: the host's world as the guest's ground and collision
- `skills/translate-objects/SKILL.md`: the guest's objects drawn by the host, with damage both ways
- `skills/crossover-link/SKILL.md`: the protocol between the two games
- `skills/prove-it-in-game/SKILL.md`: how to show that something works
- `knowledge/`: field notes from finished crossovers. Search them before you start.

## Hard rules

These come from mistakes that cost real time. Don't bend them.

1. **Test in a separate profile.** Run the game on its own user folder or profile for testing
   (BeamNG: `-userpath`; Minecraft: a dev client with its own `run/` folder). Never install into,
   edit or delete the player's real profile, saves or mods unless they ask for that exact thing.
2. **Check the human isn't playing** before you move their character or car, run a scripted
   drive, restart a game or change its settings. Ask the game for its status twice, a few seconds
   apart. A scripted input can land on the car they are driving.
3. **No key injection or focus stealing.** Don't send keystrokes to a game window or pull it to the
   front. Windows won't let a background process take the foreground, so injected keys land in
   whatever window has focus. Drive the game through its own scripting instead.
4. **Screenshots show the game image only**, taken through the game's own screenshot call or the
   client area. No desktop, no taskbar, no notifications.
5. **Nothing is done until you've seen it in the running game.** Unit tests and a mock of the
   other game catch crashes and protocol bugs; they say nothing about how the engine behaves.
   Report three states separately: proven in game, tested offline, not yet seen.
6. **Don't invent engine APIs.** Read the engine's own scripts and cite the file and line of a real
   call site for every call you use. Keep a notes file with each API and its state (tested,
   confirmed in source, unverified, rejected).
7. **Don't rebuild under a running game.** Compiling the tree a running client loads from swaps
   class files under it. Build in a separate checkout or git worktree, then move the result over
   when the game is closed.
8. **No game files in commits.** Exports, textures, maps, decompiled code and converted assets stay
   in the user's own folders. Publish code and converters only.
9. **Single player, offline.** Never connect a modded game to online services or multiplayer
   servers, and don't touch games with anti-cheat.

## How to report

Give numbers: frame rates, rebuild times, how many hits on a route before and after. Say what you
ran to get them. If something is guessed rather than measured, say so.
