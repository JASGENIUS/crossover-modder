---
name: crossover-link
description: Use when designing or debugging the connection between two running games in a crossover. Covers a localhost UDP JSON protocol with sessions and sequence numbers, which messages may be lost and which need acks, holding the link through engine load stalls, threading so neither game waits on the network, dev commands for driving both games from a terminal, and a mock of the other game for offline tests.
---

# The link between the two games

The example protocol is in `beamng/docs/protocol.md` of
[minecraft-beamng](https://github.com/JASGENIUS/minecraft-beamng). It has three implementations
kept in step (BeamNG Lua, Minecraft Java, a Python diagnostic client); bump the version number on
any incompatible change.

## Transport

- UDP on 127.0.0.1 only, the game with the fixed address binds a known port (47020), clients bind
  an ephemeral one. Nothing listens on other interfaces.
- One JSON object per datagram. Datagrams to BeamNG stay under 8000 bytes because LuaSocket reads
  at most 8192 per receive; datagrams from BeamNG can be up to 64 KB.
- Why UDP: almost everything is latest state wins. A lost camera pose or car state is replaced
  16 ms later, and nothing queues behind a lost packet.
- Why JSON: you can read it in logs and captures, and the volume was cheap (BeamNG spent about
  0.1 ms a frame in the extension). A binary encoding can come later behind the same envelope.
- The game side never blocks: non-blocking socket (`settimeout(0)`), drained up to 256 datagrams
  per frame, and every message handler wrapped in `pcall` so one bad message can't break the game.

## Envelope and sessions

Every message carries `v` (protocol version), `t` (type), `sid` (session id), `seq` (per-sender
counter) and `ts` (sender's monotonic clock, ms). Receivers drop other versions, other sessions and
anything not newer than what they already applied, and count gaps as drops.

1. Client sends `hello` every 500 ms until answered.
2. The game answers `welcome` with a new session id, its version, the level, `ready` (BeamNG's
   `worldReadyState == 2`), and the user folder path the client needs for file exchange.
3. Client pings every 250 ms; the pong echoes the ping's `ts`, so the round trip needs no shared
   clock.
4. The game drops a silent client after 3 s; the client reconnects after 8 s without data.
5. An unknown session id gets an error and the client says hello again at once. That covers game
   restarts and extension reloads. Start session ids from the clock so a restarted game doesn't
   reuse old ones.

## Rates in the example

| Message | Rate |
|---|---|
| Player car state, camera, level | every frame, at most 60 Hz |
| All car boxes | 20 Hz |
| Dashboard (gear, rpm, mode, water) | 20 Hz |
| Node stream for drawing | every frame, per car in range |
| Camera pose from the host | every host frame, at most 240 Hz |
| Driving input | every frame, at most 60 Hz |

Measured on the link: 60.3 Hz state, 0.66 ms round trip, 0 drops.

## What needs to arrive

Latest-wins covers state. These needed more:

- **Terrain patches:** an id per patch, an ack from the game, a resend after 3 s. About 1 in 150 was
  lost while BeamNG was busy.
- **Big one-off requests (a terrain load, an export):** an id, resent on a timer (10 s for loads,
  5 s for exports), and idempotent answers. The game answers a repeat from what it already built
  instead of doing the work again.
- **Messages to a frozen game:** a busy BeamNG doesn't read its socket, and Windows gives a UDP
  socket a 64 KB buffer. Hold messages that must arrive until the game has been heard from within
  the last second.

## Load stalls

The guest freezes on purpose sometimes: a car export held BeamNG 17-29 s for big mods, a spawn or
car swap about 2 s. With a 2 s timeout the export's answer arrived during the reconnect and was
thrown away; with 8 s the link still dropped mid-export and the picker said the game wasn't
connected. Before a request
that stalls, the client says it expects a stall (90 s for an export, 60 s for a spawn) and keeps
the session through that silence. Count held stalls in the status display.

## Threads

The link runs on its own I/O thread and keeps the newest message of each type. Game threads read
immutable snapshots and send; nothing on a game thread waits for the network. Parse a car message
once into an immutable map that every consumer reads, so the server's and client's copies agree.
Send the camera pose right after the host sets up its camera, so the pose the guest renders is the
one the host is rendering.

## Driving both games from a terminal

You'll need this for every test:

- Host side: a command file the mod reads each tick (`mc.ps1 "status"`, `look`, `screenshot`,
  terrain dumps, settings). Answers go to a file next to it.
- Guest side: a small client that speaks the protocol (`bngdiag.py telemetry | bench | ping | place
  | reload | raw`). A diagnostic client must keep pinging or the game drops it after 3 s.
- Dev messages in the guest: reload the extension from disk, teleport and repair a car, read
  terrain heights at points, count our objects in the scene, take a screenshot of the game image.
- Don't trust BeamNG's log for live state: `beamng.log` reaches the disk minutes late and still
  holds the previous run's lines at startup. Ask over the socket.

## Without the game

- A mock of the guest: `mock_beamng.py` runs the real extension Lua under LuaJIT with stubbed engine
  calls (flat ground, a ramp, a wall, two cars) and a real UDP socket. It catches Lua errors and
  protocol bugs. It says nothing about how BeamNG behaves.
- Unit tests for the pure Lua modules through `lupa` (85 tests in the example), and Java unit tests
  for coordinates, the link, terrain rules and damage.
- A health check with thresholds: game fps at least 30 on both sides, state at least 30 Hz, worst
  state gap at most 250 ms, round trip at most 10 ms, drops at most 1%, extension cost at most 1 ms
  a frame, camera driven on at least 95% of frames.
