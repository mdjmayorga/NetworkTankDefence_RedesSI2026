# Multiplayer Top-Down Shooter — Client/Server over UDP

A real-time multiplayer arena game (2–4 players) built in Python with an **authoritative UDP server** and a **Pygame client** featuring client-side prediction. Developed for the *Computer Networks* course at the Costa Rica Institute of Technology (TEC), 2026.

## Highlights

- **Authoritative server**: all game logic (movement, collisions, damage, scoring) runs on the server; clients only send inputs, which prevents cheating and keeps every player in sync.
- **Custom application-layer protocol**: compact JSON messages over non-blocking UDP sockets.
- **Fixed-rate networking**: server simulation at 60 Hz, state broadcast at 20 Hz, client input at 30 Hz.
- **Client-side prediction and reconciliation**: the client predicts local movement instantly and smoothly corrects toward the server's state, so the game feels responsive even with latency.
- **Connection management**: handshake with retries, "server full" rejection, explicit disconnect, and a 10-second inactivity timeout.
- **Multithreaded client**: separate send and receive threads, with shared state protected by a lock.

## Architecture

```
┌────────────┐   input (30 Hz)    ┌──────────────────────┐
│  Client 1  │ ─────────────────▶ │                      │
│  (Pygame)  │ ◀───────────────── │   Authoritative      │
└────────────┘   state (20 Hz)    │   Server (UDP)       │
      ...                         │   60 Hz game loop    │
┌────────────┐                    │                      │
│  Client 4  │ ◀────────────────▶ │                      │
└────────────┘                    └──────────────────────┘
```

| File | Role |
|------|------|
| `Proyecto2/server.py` | Authoritative server: game loop, physics, collisions, match state machine, broadcasting |
| `Proyecto2/client.py` | Pygame client: rendering, input capture, local prediction, network threads |
| `Proyecto2/common.py` | Shared constants, vector math, obstacle generation, JSON encode/decode |
| `DOCUMENTACION.md` | Full technical documentation (Spanish): every constant, formula and message |

## Network Protocol

**Transport:** UDP · **Format:** JSON (UTF-8, compact separators)

| Direction | Message | Purpose |
|-----------|---------|---------|
| Client → Server | `connect` | Join request, retried every ~1 s until welcomed |
| Client → Server | `input` | Movement keys, shoot flag and aim vector (30 Hz) |
| Client → Server | `ready` | Player confirms the match can start |
| Client → Server | `disconnect` | Clean exit |
| Server → Client | `welcome` | Assigns a player ID |
| Server → Client | `full` | Rejects the 5th player |
| Server → Client | `state` | Full snapshot: phase, timer, players, bullets, pickups, ranking (20 Hz) |

Example state message:

```json
{"type":"state","phase":"playing","time_left":95,
 "players":[{"id":1,"name":"P1","x":100.5,"y":270.0,"hp":80,"score":1,"aim":[1.0,0.0]}],
 "bullets":[{"owner_id":1,"x":200.0,"y":270.0}],
 "pickups":[{"id":1,"type":"weapon","x":480.0,"y":270.0}]}
```

## Match Flow

```
WAITING ──(≥2 players)──▶ READY_CHECK ──(all ready)──▶ COUNTDOWN (3 s) ──▶ PLAYING
   ▲                                                                         │
   └────────────────── FINISHED (ranking, 8 s) ◀──(3 kills or 120 s)─────────┘
```

If players drop below two at any point, the server returns to `WAITING`.

## Gameplay

- **Move:** `W A S D` or the arrow keys. **Aim:** mouse. **Shoot:** left click.
- **Win condition:** the first player to reach 3 kills, or the best ranking when the 120-second timer ends.
- **Pickups:**
  - *Power weapon*: automatic fire for 15 s.
  - *Health*: restores full HP.
- Obstacles block both players and bullets, using circle-vs-AABB collision.
- **Ranking order:** kills, then fewest deaths, then total damage dealt.

## Getting Started

**Requirements:** Python 3.8+ and Pygame.

```bash
pip install pygame
cd Proyecto2
```

**Start the server** (the default port is 5000):

```bash
python server.py 5000
```

**Start each client** (in separate terminals or on other machines on the same network):

```bash
python client.py <server_ip> <port> <player_name>
# example
python client.py 192.168.1.10 5000 Mariano
```

To play on one computer, use `127.0.0.1` as the server IP. When playing across machines, make sure the server's firewall allows inbound UDP on the chosen port.

## Tech Stack

Python · `socket` (UDP) · `threading` · Pygame · JSON


