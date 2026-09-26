# Runtime verification

Compiling proves the adapter matches the Minecraft API it was built against.
It does **not** prove the mod loads, that the HTTP server starts, or that a
message actually reaches the chat. Those need the game.

This project therefore tracks two separate things, and never lets one imply the
other:

| Level | Meaning | Recorded in |
|---|---|
| Build verified | The target compiles and produces a JAR | build manifest / CI |
| Runtime verified | Minecraft was launched and the API exercised | `runtimeVerified` in `versions.json` |

`runtimeVerified` is `false` for every target until a human runs the checklist
below and records the result. The generated tables show `—` for anything not
verified, never a tick.

## Why this is not fully automated

Running the client in CI needs a real display, a Mojang account for anything
past the main menu, and a game session per Minecraft version. A headless
GitHub runner cannot log in to a server or spawn a player, so an automated
"the chat message arrived" check is not possible on hosted runners.

What *is* automated: every target compiles, its metadata is inspected inside
the JAR, and the whole HTTP layer is covered by unit tests against a fake
bridge (`common/src/test/.../HttpServerManagerTest`), including status codes,
authentication, limits and the missing-player path.

## Per-family checklist

Run these representative versions to check each adapter and build-family
boundary. Record `runtimeVerified: true` only for the exact Minecraft versions
actually launched and exercised.

Representative versions:

| Adapter family | Verify with |
|---|---|---|
| `legacy-chat` | 1.16.5 |
| `signed-chat` | 1.19.2 |
| `network-chat` (legacy build) | 1.21.11 |
| `network-chat` (modern build) | 26.3 |

The last row matters: the same adapter source is compiled by two different
build families, so both need a runtime check.

### Steps

The port is automatic by default, so resolve it once per session from the
address file (paths below use your launcher's `config/` dir):

```bash
export PORT=$(python3 -c "import json; print(json.load(open('config/commandapi-address.json'))['port'])")
```

1. **Install** the matching JAR plus Fabric Loader into a clean profile. Do not
   install Fabric API — the mod must work without it.
2. **Start Minecraft.** The log shows `[CommandAPI] Listening on http://127.0.0.1:<port>`
   and `config/commandapi-address.json` contains the same address. Failure here
   usually means the entrypoint or `fabric.mod.json` is wrong.
3. **Before joining a world**, check the offline path:
   ```bash
   curl -s http://127.0.0.1:$PORT/api/status            # in_world: false
   curl -s -o /dev/null -w '%{http_code}\n' \
        -X POST http://127.0.0.1:$PORT/api/chat -d '{"text":"hi"}'   # 503
   ```
4. **Join a world or server**, then:
   ```bash
   curl -s http://127.0.0.1:$PORT/api/status            # in_world: true, player_name set
   ```
5. **Send a chat message** and confirm it appears in the chat as your player:
   ```bash
   curl -s -X POST http://127.0.0.1:$PORT/api/chat -d '{"text":"runtime check"}'
   ```
6. **Send a command** and confirm it executes:
   ```bash
   curl -s -X POST http://127.0.0.1:$PORT/api/chat -d '{"text":"/time set day"}'
   ```
7. **Send a batch** and confirm ordering:
   ```bash
   curl -s -X POST http://127.0.0.1:$PORT/api/chat -d '{"messages":["one","two"]}'
   ```
8. **In-game commands**: type `/commanda` and confirm Tab suggests
   `/commandapi`; type `/commandapi ` and confirm its subcommands appear.
   Then type `/commandapi status` and `/commandapi restart`
   (answered locally, nothing reaches the server). Then change to a known free
   port and confirm the address file and `GET /api/status` match. Try a port
   held by another process and confirm the old config, address file, and API
   are restored. Finally use `/commandapi port 0` and confirm a fresh bound
   address. This exercises the generation's mixin and rollback path.
9. **Disconnect** back to the main menu, repeat step 3 (must be 503 again, not a
   crash), then **reconnect** and repeat step 5. This catches adapters that
   cache a stale player or connection.
10. **Close Minecraft** and confirm the port is free and the address file is gone:
   ```bash
   ss -ltn | grep $PORT || echo "port released"
   ls config/commandapi-address.json || echo "address file removed"
   ```

### Recording the result

Set `runtimeVerified` to `true` for the versions covered, add a note with the
date and what was tested, then regenerate the tables:

```bash
python3 scripts/generate-version-table.py
```

Only set it for versions you actually ran. If an adapter family is verified on
one version, say so in the note rather than silently ticking the others.
