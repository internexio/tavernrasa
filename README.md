# TavernRasa

A mobile-first, multiplayer, top-down 16-bit browser game. Players are rival servers in a
tavern whose patrons start with no personality. How the room treats them moves five traits
(the OCEAN model), and who they have become decides what they pay. Phones join by QR code.
A settings switch shows the trait engine working, live.

- Design: [`docs/GDD.md`](docs/GDD.md)
- What was built, checked and left open: [`docs/BUILD_NOTES.md`](docs/BUILD_NOTES.md)

## Run it

Needs Node 20 or newer. Nothing to build.

```sh
npm install
npm start            # http://localhost:8080
```

Open `http://localhost:8080`, choose **Open a room**, and you are on the host screen with a
QR code. For phones on the same Wi-Fi the QR must point at your machine's LAN address:

```sh
PUBLIC_BASE_URL=http://192.168.1.20:8080 npm start
```

Desktop testing: open `/j/CODE` in a narrow window. Arrow keys or WASD walk, Space or Enter
presses the big button, E opens the full list of actions.

## Configuration

Everything is an environment variable. See [`.env.example`](.env.example).

| Variable | Default | What it does |
|---|---|---|
| `PORT`, `HOST` | `8080`, `0.0.0.0` | Where the server listens. |
| `PUBLIC_BASE_URL` | derived from the request | The origin the QR encodes, for example `https://tavernrasa.slacksurge.com`. Set it in production. |
| `TRUST_PROXY` | `1` | Read `X-Forwarded-For`, `-Proto` and `-Host`. Set `0` if the server is exposed directly. |
| `LLM` | off, unless a provider key is present | `off`, `openai`, `groq` or `gemini`. |
| `OPENAI_API_KEY`, `GROQ_API_KEY`, `GEMINI_API_KEY` | none | The key for the chosen provider. `LLM_API_KEY` works for any. |
| `LLM_MODEL`, `LLM_BASE_URL`, `LLM_PRICE_IN`, `LLM_PRICE_OUT`, `LLM_MAX_TOKENS` | per provider | Overrides. Prices are USD per 1M tokens and drive the cost fuse. |
| `LLM_RATE_PER_MIN` | `30` | Model lines per minute, per room. |
| `LLM_ROOM_FUSE_USD` | `0.25` | Past this a room gets stock lines only. |
| `LLM_DAILY_BUDGET_USD` | `2.00` | Past this the whole server gets stock lines only, until the next UTC day. |
| `LLM_TIMEOUT_MS` | `1500` | A slower reply is dropped for the stock line. |
| `MAX_ROOMS`, `ROOM_IDLE_MIN` | `40`, `10` | Room cap; minutes after the last socket before a room is collected. |
| `MAX_SOCKETS_PER_IP`, `MAX_WATCHERS`, `CREATE_PER_MIN` | `80`, `20`, `12` | Abuse limits: sockets per address (a venue shares one), host screens per room, rooms created per address per minute. |

The game is complete with the model off. With it on, only deliberate talk (chat, tale, lean,
a kept or broken promise, or anything typed) is sent to the model, and every line has a stock
fallback chosen by the same trait engine.

## Deploying

Not deployed yet: confirm the hosting target first (BUILD_NOTES, "Before it goes live").

```sh
docker build -t tavernrasa .
docker run -p 8080:8080 -e PUBLIC_BASE_URL=https://tavernrasa.slacksurge.com tavernrasa
```

The image is one Node process, no volumes, no database. State is in memory, so run a single
instance; a restart ends the rooms that are open. Whatever sits in front of it has to:

- terminate TLS,
- pass WebSocket upgrades on `/ws` (the game socket), with an idle timeout over 30 s,
- forward `X-Forwarded-Proto` and `X-Forwarded-For`.

Two examples of a proxy that does this. Adjust to whatever is actually in place:

```
# Caddy
tavernrasa.slacksurge.com {
    reverse_proxy 127.0.0.1:8080
}
```

```nginx
# nginx, inside the server block for tavernrasa.slacksurge.com
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_read_timeout 75s;
}
```

`GET /healthz` reports rooms, players and the day's model spend. Logs are JSON lines on
stdout: `room_open`, `room_close`, `llm_call`, `llm_fuse`, `llm_session`.

## Tests and checks

```sh
npm test             # 59 tests: engine, rules and room, LLM and prompt injection, server over real sockets
npm run sim          # balance: does reading personality beat ignoring it? (300 nights a table: --nights 300)
```

Browser checks (need a local Chrome; they are not part of `npm test`):

```sh
cd e2e && npm install
npm run night        # a full night through the real UI on two phones and the projector, with screenshots
npm run flows        # promise kept, cellar door, bouncer, dropped socket resuming, next night
npm run sync         # eight phones: how far behind is everyone else's picture of you?
npm run timing       # cold load to walking, under throttled network and CPU
```

## Layout

```
server/engine/   the OCEAN trait engine: update rule, appraisal table, sampler, presets, interview
server/game/     rules (every trait-gated outcome, as pure functions), the room, the cast
server/llm/      provider adapter, prompt template, sanitizer and output guard, stock lines, prices
server/app.js    HTTP routes, WebSocket protocol, room registry wiring
shared/map.js    the tavern: one module used by the server (collision) and the client (drawing)
client/          the static bundle: three pages, ES modules, no build step
client/vendor/   Phaser 3.90.0 and Pixelify Sans, copied from node_modules by `npm run vendor`
tools/           asset builder (Python + Pillow) and the vendoring script
sim/             balance simulation: persona bots playing the real room
test/, e2e/      tests and browser checks
```

Changing art: edit `tools/build_assets.py`, run `npm run assets`. Changing the Phaser or font
version: edit `package.json`, `npm install`, `npm run vendor`, and update the file name in
`client/js/world.js` and the two `<link rel="preload">` tags.

## Credits

Tavern art: "Tiny Dungeon" and "Tiny Town" by [Kenney](https://www.kenney.nl), CC0.
Type: Pixelify Sans, SIL Open Font License. Engine: [Phaser 3](https://phaser.io), MIT.
