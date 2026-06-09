# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

Independent services deployed across **VM-A** (frontend tier) and **VM-B** (broker / TCP / DB tier). Each has its own `docker-compose.yml`, `.env`, and is built/run independently — there is no top-level compose:

- `gateway/` — simulated streetlight gateway (Python, paho-mqtt). Subscribes to `streetlight/server_to_gateway`, publishes to `streetlight/gateway_to_server`, periodically emits `DRD_10`. TCP path is **peer dual-channel** (model B): gateway **listens** on `GATEWAY_LISTEN_PORT` (default 15566) for the server to dial in with `ROLE manager\n` (downstream), and **dials** the server's `UPSTREAM_PORT` (default 15567) with `ROLE gateway\n` (upstream).
- `socket_server/` (VM-B, "TCP Management") — TCP server + outbound dialer:
  - **listen** 15567 (UPSTREAM, gateway dials in for reports)
  - **listen** 19100 (HTTP `/api/commands` — subscriber forwards here)
  - **dial** every entry in `GATEWAYS=host:port,...` (downstream — server pushes commands to gateway listening port)
  Writes `command_logs` and upserts `runtime` to MongoDB; also forwards upstream lines to subscriber `/api/messages` for the Socket.IO UI. The "round" ports 5566/5567/9100 are intentionally **left free for a future intercept proxy** (same pattern as MQTT 1883/11883).
- `mongodb/` (VM-B) — MongoDB compose with auth enabled. Stores `command_logs` (append-only) and `runtime` (per-MAC upsert). Mongo container's host port maps to **17017** (real); the well-known **27017** is reserved for a future intercept proxy that forwards to 17017. All clients (subscriber + socket_server) point at `<VM-B>:27017` so the proxy sees every read/write.
- `subscriber/` — Flask + Flask-SocketIO. **Single user-facing API surface** for the entire system. Hosts:
  - **HMI / Socket.IO UI** at `/` (login at `/login`)
  - `POST /api/commands` — third-party / frontend SPA HTTP entry point; writes `command_logs(stage=accepted)` and forwards to socket_server `:19100/api/commands`
  - `POST /api/messages` — receives upstream pushes from socket_server (gateway reports) and emits to UI via Socket.IO
  - `GET /api/runtime` and `GET /api/command_logs` — read MongoDB
  - **Honeypot bait**: `/.env`, `/env`, `/.env.bak`, `/config.env`, `/admin/.env`, `/.env.production` all return the literal `subscriber.env` content with deliberate credential leak; access logged to `${HONEYPOT_LOGS_DIR}/envleak.log`
  Connects to MQTT (sub_client port 11883, pub_client port 1883) and to MongoDB on 27017.
- `mosquitto/` — Eclipse Mosquitto broker config + compose.

`ARCHITECTURE.md` describes the dual-channel TCP topology and MongoDB schema. Read it before changing the wire protocol or relay flow.

## Communication modes (the central design decision)

Each component reads `COMM_MODE` from its env file and behaves differently. **Gateway, subscriber, and socket_server must be configured consistently**:

| Mode | Gateway → ? | ? → Subscriber UI | When |
|---|---|---|---|
| `mqtt` | publishes to Mosquitto | subscriber's sub_client receives from Mosquitto | default; broker-mediated |
| `socket` | TCP client to `socket_server` dual ports (15566 read / 15567 write) | (legacy direct-to-gateway socket — deprecated under new VM-A/VM-B topology) | no broker |
| `api` (subscriber) / `socket` (gateway) | gateway → socket_server upstream (15567) → subscriber `/api/messages` | subscriber UI sends commands via HTTP `:19100/api/commands` to TCP Management | three-tier with HTTP fanout + MongoDB |
| `both` | publishes to Mosquitto and TCP to `socket_server` (dual port) | subscriber receives MQTT and `/api/messages`; UI events are de-duplicated | redundant MQTT + TCP path |

Gateway and subscriber both support `COMM_MODE=both` (MQTT + TCP simultaneously).

Wire format is a single hex string. Over MQTT it's the raw payload (no JSON wrapping). Over TCP, gateway **listens** on 15566 for the server to dial in with `ROLE manager\n` (downstream commands, gateway reads only) and **dials** server 15567 with `ROLE gateway\n` (upstream reports, gateway writes only). Server pushes commands to gateways via internal HTTP `:19100/api/commands` from subscriber. socket_server forwards upstream hex to subscriber as `{"data": "<hex>", "source": "gateway" | "api"}`. (Ports 5566/5567/9100 are reserved for an intercept proxy, mirroring the MQTT 1883/11883 split.)

## Critical invariants

- **TCP socket is peer dual-channel** (model B): gateway **listens** on 15566 for downstream (server dials in with `ROLE manager\n`), gateway **dials** server 15567 for upstream (`ROLE gateway\n`). Downstream is read-only on the gateway side; upstream is write-only on the gateway side. Heartbeat: gateway sends empty newline on upstream every 30s; server sends empty newline on each downstream socket every 30s. Don't try to send replies on the wrong direction — the receiver-side log will print "略過" but that's wasted bandwidth.
- **subscriber is the only user-facing API host.** Earlier iterations had a separate `web_api/` FastAPI service on a dedicated VM-A; that has been collapsed into subscriber `/api/commands`. The Bridge port (15568) and BridgeHandler/BridgeClient code have been removed accordingly. If you need a programmatic frontend (React/Vue/etc.), point it at `subscriber:8080/api/commands`.
- **MQTT path is untouched.** Any MQTT-related code in `gateway/gateway.py` (on_connect/on_message/MQTT publish in periodic_report) and `subscriber/app.py` (sub_client/pub_client/setup_mqtt/on_sub_message) was deliberately left alone in the dual-channel migration. Don't merge or rewrite the two paths.
- **`runtime` upserts only happen in `socket_server.py`'s upstream handler.** VM-A FastAPI may write `command_logs(stage=accepted)` on inbound POST per the diagram, but `runtime` is never written from VM-A — that keeps MAC-keyed state consistent.
- **Cross-machine MongoDB exposure is intentional but firewalled.** Per the architecture diagram VM-A directly reads VM-B's MongoDB. For honeypot / cross-cloud deployment, restrict 27017 to VM-A's source IP (Security Group / GCP firewall), and enable MongoDB auth. Do not bind 27017 to `0.0.0.0` without source-IP filtering.
- **MongoClient must set `socketTimeoutMS`, `maxIdleTimeMS`, `connectTimeoutMS`.** Cross-VM TCP through NAT (AWS / VPC peering / Tailscale) gets idle-killed silently around 350s. Pymongo defaults are `socketTimeoutMS=None` and `maxIdleTimeMS=None`, leaving stale sockets in the pool that hang on next use → endpoints like `/api/command_logs` (slower than `/api/runtime`) eventually fail with "Failed to fetch" after the app's been up a while. Both `subscriber/app.py` and `socket_server/socket_server.py` set `socketTimeoutMS=5000` (fail-fast + auto retry on a fresh socket), `maxIdleTimeMS=30000` (recycle before NAT idle), `connectTimeoutMS=3000`. Don't remove them.
- **`_BridgeClient.send()` must invalidate the cached socket on I/O failure.** Subscriber's bridge to socket_server keeps a long-lived TCP socket; the maintenance thread polls `getsockopt(SO_ERROR)` every 5 seconds. **`SO_ERROR` does NOT report graceful FIN closes** (socket_server restart, proxy restart, cloud LB idle-kill) — only RST or async I/O failures set it. The only signal application code gets is when `sendall()` raises `BrokenPipe`, `readline()` raises OSError, or `readline()` returns `""` (EOF). When any of these happen, `send()` calls `_invalidate(sock)` to null out `self._sock` / `self._reader` and `shutdown+close` the dead socket; the maintenance thread's next 5-second poll then sees `self._sock is None` and `break`s into the reconnect path. Without this, every subsequent `send()` keeps hitting the same dead socket — symptom is unbounded `[BRIDGE] 寫入失敗: Broken pipe` log spam with no `已連線` recovery message ever appearing. Don't add any return path in `send()` that doesn't either succeed or call `_invalidate(sock)`.
- **Don't wrap mongo writes in an application-level lock.** `MongoClient` is already thread-safe (each op borrows a socket from the pool independently). A single `self._lock` covering both `insert_command_log` and `upsert_runtime` was previously here — when one op hit a stale TCP socket and hung, the lock blocked every other writer for the duration (until kernel TCP retransmission gave up, ~5 minutes), turning a single hang into a system-wide outage. Don't add it back; rely on `socketTimeoutMS=5000` to bound any individual hang and `maxIdleTimeMS=30000` to recycle pooled sockets before NAT kills them.
- **Mongo connection is lazy + self-healing — don't refactor back to single eager `_init_mongo` at module load.** Both subscriber and socket_server use a "ensure-on-use" pattern: `_ensure_mongo()` (subscriber) / `MongoStore._ensure()` (socket_server) check the connection on every read/write; if it's down they try to reconnect, but no faster than once every 5 seconds (so a flapping mongo doesn't make every request hang for 3s of handshake). On write failures both call a `_drop()`-style helper to clear the cached client so the next op forces a reconnect. The reason: previously a single `_init_mongo()` ran at module import, so if mongo was momentarily unavailable when subscriber started (mongo container still booting), `mongo_db` stayed `None` for the entire lifetime of the process and only `docker restart subscriber` recovered it. The lazy-ensure + 5s backoff keeps service up across mongo restarts / network blips without operator intervention.
- **Honeypot bait routes**: `subscriber/app.py` exposes `/.env`, `/env`, `/.env.bak`, `/config.env`, `/admin/.env`, `/.env.production` — all return the contents of `subscriber.env` verbatim, deliberately leaking the MongoDB credentials, Flask `SECRET_KEY`, broker host, and `ADMIN_USER`/`ADMIN_PASS`. Each hit is logged to `${HONEYPOT_LOGS_DIR}/envleak.log` with timestamp, peer IP (respects `X-Forwarded-For`), and user-agent. Attackers who find this can connect to MongoDB on 27017 with the leaked creds and dump `command_logs` / `runtime`. **This is the entire point of the deployment** — do not "fix" the leak; it is the bait. The real production path should rotate creds away from this hardcoded set.
- **Two streetlight_data.json files** exist (`gateway/` and `subscriber/`) and **must stay in sync**. They contain `streetlights`, `cmd_names`, `commands`, `command_response_map`, `drd10_data`. Gateway uses the response map to fake replies; subscriber uses `commands`/`cmd_names` to render the UI and parse incoming hex. `socket_server/` reads `cmd_names` for log decoration if the file is mounted in.
- **Subscriber uses two separate MQTT clients on different ports.** `sub_client` connects to `BROKER_SUB_PORT` (default 11883, direct to Mosquitto) for subscribing only. `pub_client` connects to `BROKER_PUB_PORT` (default 1883, via the honeypot proxy) for publishing only. This split is intentional — the honeypot proxy logs HMI commands. Don't merge them.
- **Don't double-emit to Socket.IO.** In MQTT mode, `handle_send_command` only calls `pub_client.publish` and **does not** emit to the UI directly; the message comes back via the broker through `sub_client.on_message` and is emitted there. In `socket` and `api` modes the UI emit happens elsewhere (socket mode emits locally; api mode relies on socket_server pushing back via `/api/messages`). In `both` mode, MQTT and API callbacks can return the same message, so `_emit_mqtt_to_socketio` de-duplicates short-lived identical topic/data events. `subscriber/test_no_duplicate.py` guards this.
- **Socket.IO async mode defaults to `threading`.** Eventlet + paho `loop_forever` deadlocked the UI in the past. Only switch via `SOCKETIO_ASYNC_MODE=eventlet` if you know what you're doing.
- **Socket.IO `connect` handler must not return False on missing session.** Reverse proxies / Docker can drop the Flask session on the first `/socket.io` poll; rejecting connect leaves the UI permanently "connecting." Authorization is tracked per-sid in `_socket_authorized_sids` instead.
- **Hex parsing positions are fixed.** `hex_data[4:6]` is the command byte (mapped via `cmd_names`); `hex_data[6:16]` is the MAC, except group commands where it starts with `4081`. Changing these breaks UI classification.

## Common commands

Each service is built and run from its own directory. PowerShell on Windows:

```powershell
# Run a service (from inside gateway/, socket_server/, subscriber/, or mosquitto/)
docker compose up -d --build
docker compose logs -f
docker compose down

# Run subscriber locally without Docker (for UI iteration)
cd subscriber
pip install -r requirements.txt
python app.py   # serves on :5000

# Run the duplicate-emit regression test
cd subscriber
python test_no_duplicate.py
```

There is no lint config, no pytest config, and no top-level test runner. `test_no_duplicate.py` is run directly as a script and `sys.exit(1)`s on failure.

## Environment variables that matter

Each service has its own `.env` checked in with `CHANGEME_*` placeholders for IPs/DNS. When deploying:

- `gateway/gateway.env`: `COMM_MODE`, `BROKER_HOST`, `SOCKET_HOST` (VM-B, upstream target), `UPSTREAM_PORT=15567`, `GATEWAY_LISTEN_HOST=0.0.0.0`, `GATEWAY_LISTEN_PORT=15566`, `GATEWAY_REPORT_INTERVAL`.
- `socket_server/socket_server.env`: `UPSTREAM_PORT=15567`, `HTTP_PORT=19100`, **`GATEWAYS=host:port,...`** (downstream dial targets), `SUBSCRIBER_API_URL`, `MONGO_URI`, `MONGO_DB`, optional `API_TOKEN`/`SUBSCRIBER_API_TOKEN`.
- `mongodb/mongodb.env`: `MONGO_DB`, `MONGO_PORT=17017` (real host port), `MONGO_DATA_DIR`, `MONGO_USER`, `MONGO_PASS`. Both subscriber and socket_server connect to `<VM-B>:27017` (proxy port) — the docker DNS shortcut `mongo:27017` is no longer used so the proxy can side-channel both clients' traffic.
- `subscriber/subscriber.env`: `COMM_MODE`, `BROKER_HOST`, `BROKER_SUB_PORT=11883`, `BROKER_PUB_PORT=1883`, `SOCKET_SERVER_HOST`, `SOCKET_SERVER_HTTP_PORT=19100` (TCP-out goes via HTTP `/api/commands`), `MONGO_URI` (with `streetlight_admin` auth), `MONGO_DB`, `ADMIN_USER`/`ADMIN_PASS` (deliberately weak admin/admin), `SECRET_KEY`. **This file is intentionally exposed at `/.env` for honeypot purposes.**

Subscriber publishes its UI on host port `${SUBSCRIBER_HTTP_PORT:-8080}` (mapped to container 5000) — the 8080 default is for the ICS-Honeypot ProxyManager which fronts it on :80.
