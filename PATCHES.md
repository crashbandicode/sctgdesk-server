# crashbandicode patches (WS peer presence + login error body + state persistence)

This fork carries local patches on top of `sctg-development/sctgdesk-server`
so the server works correctly behind a reverse proxy / Cloudflare where clients
connect over **WebSocket only** (no UDP), and so runtime state actually persists.

> No secrets live in this repo. The server keypair (`id_ed25519`), database
> (`db_v2.sqlite3`), `.env`, and the `-k` license key are all supplied at run
> time and are covered by `.gitignore`. Never commit them.

## What changed

### 1. `src/rendezvous_server.rs` — WebSocket peer presence
Upstream only refreshes a peer's online status (`last_reg_time`) from
`register_peer`, which modern clients send over **UDP**. A WebSocket-only client
(web client, or native client with "Use WebSocket" + Cloudflare) sends
`register_pk` once and then never re-registers over UDP, so after ~30s the
server marks it **OFFLINE** and incoming connects fail with
`PunchHoleResponse{failure = OFFLINE}`.

The patch:
- Adds a `ws_map: Arc<Mutex<HashMap<SocketAddr, Sink>>>` registry of live
  WebSocket peer sinks.
- Refreshes `last_reg_time` on every `register_pk` for WebSocket peers
  (`if changed || ws`), keeping them online.
- Stores each ws peer's live sink and delivers `PunchHole` signaling directly
  down that sink (ws clients have no reachable UDP address).
- Cleans up `ws_map` when the ws connection closes.

### 2. `src/main.rs` — JSON body for `/api/login` auth failures
A failed login returns `401` with an **empty body**. The RustDesk Flutter client
then crashes with *"Null check operator used on a null value"* instead of showing
"wrong password", because it parses the (missing) `error` field. A Rocket
response fairing (`LoginErrorJson`) rewrites empty `401`s on `/api/login` into
`{"error":"Wrong username or password."}`.

### 3. `Dockerfile-integration` — `WORKDIR` so state persists
The final image set **no `WORKDIR`**, so the container CWD was `/`. hbbs opens its
database and keypair via relative paths (`./db_v2.sqlite3`, `./id_ed25519`), which
therefore landed in `/` — the container's **ephemeral layer**, not the mounted
`/usr/local/share/sctgdesk` volume. Every `docker compose ... --force-recreate`
gave a fresh layer, so:
- the admin password **reverted to the `Hello,world!` seed** on each recreate,
- the modern address book (`ab`/`ab_peer` tables) and registered peers were lost,
- the server kept generating a new `id_ed25519` (worked around earlier with `-k`).

Fix: `WORKDIR /usr/local/share/sctgdesk` (the documented volume mount point) so all
relative-path state is written to the persistent volume. The deployment compose
also sets `working_dir: /usr/local/share/sctgdesk` on `hbbs`/`hbbr` as a belt-and-
suspenders override that works even with an unpatched image (no rebuild needed).

Note: the **legacy** `/api/ab` blob endpoint is in-memory only and never lands in
`ab_legacy` upstream (separate sctgdesk bug), but modern desktop/Android clients
use the new `/api/ab/peer/*` API which writes to `ab_peer` directly and now
persists correctly.

## Build

```bash
cd sctgdesk-server
DOCKER_BUILDKIT=1 docker build -f Dockerfile-integration -t sctgdesk-server:patched .
```

## Deploy (compose)

Point the `hbbs` and `hbbr` services at `sctgdesk-server:patched`. **Pin the
server key** explicitly, otherwise the server generates a fresh random key on
start (it looks in rustdesk's config path, not the mounted data dir) and every
client gets `LICENSE_MISMATCH` ("Key mismatch"):

```yaml
# Use the 64-byte base64 SECRET from your data dir's id_ed25519 (NOT the .pub).
command: ["hbbs", "-r", "<relay-host>:21117", "-k", "<id_ed25519-secret-base64>"]
# hbbr:
command: ["hbbr", "-k", "<id_ed25519-secret-base64>"]
```

`-k <secret>` makes both binaries derive the license key (the public key) from
the secret and, for `hbbs`, sign with the matching secret key.

## Update from upstream

```bash
git fetch upstream
git rebase upstream/tcpserver-master-build   # replay these patches on top
# resolve conflicts in src/rendezvous_server.rs / src/main.rs if any
# then rebuild + redeploy as above
```
