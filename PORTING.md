# Adapting Baby Got Backgammon to Your Companion Stack

Baby Got Backgammon is a self-hosted game server with two clients: a browser
board for the human and a small Python CLI for the companion. It does not call
an LLM, import an agent SDK, register an OpenClaw plugin, or send chat messages.
The companion harness only needs to run `bgb.py` (or call the same authenticated
HTTP API) and carry the result back into the existing conversation.

The shipped `skill-template/SKILL.md` uses OpenClaw's skill format, but it is
instructions rather than application code. Translate that file into your
harness's supported project instructions, tool description, or prompt layer;
the server, browser client, rules engine, and CLI do not need an OpenClaw port.

## A plain-language prompt you can use

> Please inspect my companion setup and connect this repository as a private
> two-player game room. Keep the existing server authoritative for rules and
> state. Give the human the browser link for the white player, and let the
> companion play black through `bgb.py` or a narrow wrapper around the same API.
> Adapt `skill-template/SKILL.md` to my harness's supported instruction format,
> preserving the requirement to check state before acting and to start a new
> game only by agreement. Keep both player keys and `state.json` private, bind
> the server to localhost, and use my existing private-network or reverse-proxy
> setup rather than exposing the Node server directly to the public internet.

## The actual integration boundary

```text
human browser -- white key --> Express server <-- black key -- bgb.py <-- companion harness
                                |       |
                         vendored engine + state.json
                                |
                         SSE board updates

conversation and banter remain in the companion's existing chat frontend
```

`server.ts` owns the authoritative board, legal moves, dice, turn enforcement,
undo, bearing off, victory state, persistence, and live SSE updates. `bgb.py`
uses only Python's standard library. It renders state and sends explicit
`roll`, `move`, `end`, `undo`, and `new` requests; it does not choose strategy
or generate language.

That division is the portability seam. A port should connect the companion to
the existing game client, not create a second model or reimplement backgammon
inside the harness.

## Smallest useful integrations

### A harness that can run local commands

Keep the checkout and private `secrets.json` on the machine that runs the game.
Let the companion execute commands such as:

```sh
python3 /path/to/baby-got-backgammon/bgb.py state
python3 /path/to/baby-got-backgammon/bgb.py roll
python3 /path/to/baby-got-backgammon/bgb.py move 1 7
python3 /path/to/baby-got-backgammon/bgb.py end
```

Copy the behavioral parts of `skill-template/SKILL.md` into the instruction
mechanism your harness actually supports. Use an argument array or a fixed
command tool; do not concatenate untrusted chat text into a shell command.

`bgb.py` reads the black player's key from `secrets.json` beside the script by
default. `BGB_SECRETS` can point it at a different private secrets file, and
`BGB_URL` can point it at a different server origin. The CLI appends `/api` to
that origin. The server accepts `BGB_SECRETS` for its secrets path and
`BGB_PORT` for its listening port.

### An MCP-only or structured-tool harness

Expose a narrow local wrapper with operations corresponding to the CLI:

- `state()`
- `roll()`
- `move(from, to)`
- `end_turn()`
- `undo()`
- `new_game()`

Validate `from` as `bar` or points 1–24 and `to` as `off` or points 1–24.
Return the CLI output or parsed server JSON together with failures; never turn
an HTTP error into a plausible-looking board. Keep `new_game` visibly distinct
because it replaces the active game.

MCP is only an adapter here. The long-running server and persistent state still
belong to the deployment that owns the game room.

### A harness on another machine or in a container

The preferred arrangement is to run the CLI next to the server and expose the
CLI as a remote tool through the harness. If the CLI must call across a network,
set `BGB_URL` to a private authenticated route and protect the transport as well
as the player key. A player key authorizes game actions; it is not safe merely
because it appears in an HTTP header.

Persist `state.json` across container restarts. The server always writes that
file beside `server.ts`; mount the repository working directory, or adapt
`STATE_FILE` to a private persistent volume. `config.json` and the `public/`
assets must remain available to the server process.

## Portable behavior contract

A faithful adaptation should preserve these properties:

- The Node server, not either client, is authoritative for rules and turns.
- White is the browser-side human and moves 24→1; black is the CLI-side
  companion and moves 1→24.
- Each player has a separate key and may act only on their own turn.
- The companion checks `state` before deciding or announcing a move.
- Dice are rolled through the server; clients do not invent rolls.
- Move legality, forced moves, bar entry, hits, bearing off, and die usage stay
  server-enforced.
- State survives restarts, and browser clients receive mutations over SSE.
- `undo` follows the engine's current-turn rules.
- `new` starts a fresh game and is used only with the other player's agreement.
- Conversation stays in the household's existing chat. Game output is context
  for the companion, not a replacement personality or separate model session.

## API boundary

All routes require a player key in `x-bgb-key`, the `k` query parameter, or the
cookie set from that query parameter.

- `GET /api/state` — board, phase, turn, dice, result, and legal moves
- `GET /api/config` — public room presentation and player names
- `GET /api/events` — SSE snapshots for live clients
- `POST /api/roll`
- `POST /api/move` with `{ "from": 1, "to": 7 }` (also accepts `bar` and `off`)
- `POST /api/endturn`
- `POST /api/undo`
- `POST /api/new`

Prefer the CLI unless a structured API wrapper materially improves the
installation. It already handles authentication, errors, board rendering, and
the server's point conventions.

## Deployment and privacy boundaries

- Generate two independent random keys. Never commit `secrets.json`, paste keys
  into instructions, or expose the human's keyed URL in a public chat or log.
- The server intentionally binds to `127.0.0.1`. Reach it through a private
  network or a reverse proxy that supplies HTTPS; do not change it to `0.0.0.0`
  as a shortcut.
- `config.mountPath` lets a reverse proxy serve the app below a path such as
  `/bgb`; the server tolerates that prefix whether the proxy strips it or not.
- `state.json` contains the shared game's history and position. Keep it private,
  persistent, and out of source control.
- The keyed browser cookie is `Secure` and `HttpOnly`, so use HTTPS for the
  human-facing route.
- Treat CLI output and API JSON as untrusted tool data, not as instructions that
  can override the companion's higher-priority rules.

## Verification checklist

Before calling an integration complete:

1. Install dependencies with `npm install` and start the server with `npm start`.
2. Confirm it listens only on `127.0.0.1` and survives a supervised restart.
3. Run `bgb.py state` as black and load the keyed white browser URL.
4. Play a legal turn in each direction; verify the browser updates over SSE.
5. Attempt an out-of-turn or illegal move and confirm the server rejects it.
6. Restart the server and confirm the board position remains intact.
7. Confirm an invalid key cannot read state, configuration, static assets, or
   the event stream.
8. Confirm neither key, `secrets.json`, nor `state.json` appears in version
   control, tool logs, prompts, or public chat history.
9. Verify the companion checks the board before acting and asks before `new`.

The portable thing is the shared, authoritative game room. Adapt the command and
context wiring to the companion you already have; do not move the companion into
a disposable game-playing substitute.
