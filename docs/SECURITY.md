# Security

This mod can send chat messages and run commands **as you**. Anyone who can
reach its port can talk and issue commands with your account in whatever world
or server you are on. Treat the port like a remote control for your player.

## Defaults

| Setting | Default | Why |
|---|---|---|
| `host` | `127.0.0.1` | Only this machine can connect. |
| `port` | `0` (automatic) | The OS picks a free port per start; nothing assumes a fixed one. |
| `authEnabled` | `false` | Safe *only* because the default bind is loopback. |
| `token` | empty | — |

## In-game commands

`/commandapi ...` runs with your privileges by construction: it is typed at
your keyboard, answered inside your client, and never sent as a Minecraft
server command. No token is needed (or accepted) there. The same caution as
hand-editing the file applies — `/commandapi host 0.0.0.0` exposes you, and the command tells you so
unless auth is on.

The API is never exposed to the network unless you change `host` yourself.

## If you expose it

Binding to `0.0.0.0` or a LAN address (for a container, another machine, or a
stream setup) means anyone who can route to that port controls your player.
Turn authentication on at the same time:

```json
{
  "host": "0.0.0.0",
  "token": "a-long-random-string",
  "authEnabled": true
}
```

The mod prints a prominent warning on startup if it is bound past loopback with
authentication disabled. It does not refuse to start — that is your call — but
it will not do it quietly.

Generate a token with something like `openssl rand -hex 32`. Never reuse a
password.

## Audit notes

What was reviewed and what the code does:

| Area | Behaviour |
|---|---|
| Network binding | Bound explicitly to `config.host`; loopback default. Never binds `0.0.0.0` implicitly. |
| Bearer parsing | Requires the exact `Bearer ` prefix; a malformed or absent header is a 401, never a bypass. |
| Token comparison | Length check, then a comparison that does not exit early for equal-length tokens. |
| Auth bypass | Skipped when `authEnabled` is false **or** the token is empty. An empty token leaves the API open even if `authEnabled` is true. |
| Token logging | Never logged. `ApiConfig.toString()` reports only whether a token is configured (there is a test for this). |
| Request size | Capped at 64 KiB while reading; a larger body is a 413, and the read stops rather than buffering it all. |
| Batch size | At most 32 messages per request. |
| Message length | At most 256 Java UTF-16 code units per message. |
| Malformed JSON | Caught and answered with 400; parse failures never reach the game. |
| HTTP methods | `/api/chat` and `/api/execute` are POST only; `/api/status` is GET only. Wrong-method responses carry an `Allow` header. |
| Unknown paths | JSON 404 from a catch-all handler instead of an empty response. |
| Uncaught exceptions | Handlers convert unexpected failures to JSON `500` when the client connection is still open. |
| Disconnected clients | A broken pipe while writing is logged, not rethrown. |
| Concurrency | Handlers are stateless; the only shared state is the config and the bridge. The config reference is swapped only while the server is stopped for a restart, and every service method is synchronized. |
| Client thread | Minecraft status reads and sends are scheduled onto the client thread with a 5 s timeout (see below). |
| Shutdown | `stop()` is idempotent and releases the port. A JVM shutdown hook stops the HTTP server and removes the address file if the client exits first. Worker threads are daemons. |

## Thread safety

HTTP handlers run on worker threads. Minecraft state is accessed on the
client thread, so `ClientThreadBridge` schedules status reads and sends onto
`Minecraft.getInstance()` (which is itself an `Executor`) and waits up to five
seconds for the result. On timeout the request answers with an error instead of
blocking forever, and a rejected schedule (client shutting down) is reported
rather than thrown.
The bridge re-reads the player and connection for each send after reaching the
client thread, so a reconnect does not reuse a cached connection.

## Reporting

Found something? Open an issue at
<https://github.com/nglmercer/commandclient/issues>. Do not include your token.
