# oxlify

oxlify is a per-node game server daemon written from scratch in Crystal. It is a
drop-in replacement for the panel's Node.js daemon: it speaks the same HTTP and
WebSocket contract, drives Docker over the Unix socket, and keeps its data in the
same wings-style directory layout (with `oxlify` substituted for `pterodactyl`).

Nothing is shared with Pterodactyl or Wings; the daemon uses only the Crystal
standard library (no shards, no Kemal).

## Requirements

- Linux (the daemon targets a single node, run as root so it can manage Docker
  and confine SFTP sessions with `chroot`)
- Crystal `>= 1.9`
- Docker Engine reachable at `/var/run/docker.sock` (see [Installing Docker](#installing-docker))
- Optional: `libssh` with server support (`libssh-dev`) to build the SFTP helper

## Installing Docker

oxlify drives Docker over the daemon socket, so Docker Engine must be installed
and running before the daemon starts. oxlify does not manage Docker's lifecycle;
it only talks to the socket.

### Debian / Ubuntu

```sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu ${UBUNTU_CODENAME:-$VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
```

On Debian replace `ubuntu` with `debian` in the two download URLs.

Ubuntu derivatives (Linux Mint, Pop!_OS, etc.) report their own
`VERSION_CODENAME` (for example `zena`), which is not published by the Docker
repository. The `${UBUNTU_CODENAME:-$VERSION_CODENAME}` fallback above selects
the underlying Ubuntu codename (`noble`) instead.

### Convenience script

```sh
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
```

### Verify

```sh
sudo docker info
ls -la /var/run/docker.sock
curl --unix-socket /var/run/docker.sock http://localhost/version
```

### Non-standard socket paths

Rootless Docker, snap and Docker Desktop expose the socket somewhere else (for
example `/run/user/<uid>/docker.sock` or
`/var/snap/docker/common/run/docker.sock`). Point oxlify at it with
`docker.socket` in the configuration file or the `DOCKER_SOCKET` environment
variable.

### Troubleshooting

```
cannot reach docker socket /var/run/docker.sock: connect: No such file or directory
```

This means Docker is not installed, not running, or the socket lives elsewhere.
Check `systemctl status docker` and `ls -la /var/run/docker.sock`, then restart
the daemon with `sudo systemctl restart oxlify`.

```
Err:8 https://download.docker.com/linux/ubuntu zena Release
  404  Not Found
```

`apt-get update` fails because the codename in
`/etc/apt/sources.list.d/docker.list` is not published by Docker. On Ubuntu
derivatives use the underlying Ubuntu codename (`UBUNTU_CODENAME`, e.g.
`noble`) as shown above, or install via the convenience script.

## Installation

Docker Engine (above) and Crystal `>= 1.9` (<https://crystal-lang.org/install/>)
are required.

```sh
sudo ./deploy/install.sh              # build + install binaries, config and unit
sudo editor /etc/oxlify/config.json   # set "token" to the node's daemon token
sudo systemctl enable --now oxlify    # start the daemon
sudo systemctl status oxlify
```

`deploy/install.sh` runs `make release`, builds the SFTP helper when a C compiler
is available, installs the binaries into `/usr/local/bin`, seeds
`/etc/oxlify/config.json`, creates the data directories and installs
`deploy/oxlify.service`.

The daemon reads `/etc/oxlify/config.json` by default; use `--config <path>` or
the `OXLIFY_CONFIG` environment variable to select another file.

## Filesystem layout

```
/etc/oxlify/config.json                     daemon configuration
/var/lib/oxlify/
  volumes/<uuid>/                           server data (mounted at /home/container)
  backups/<uuid>/                           reserved for structural parity
  archives/<uuid>/                          reserved for structural parity
  servers/<uuid>.json                       oxlify metadata (image, limits, sftp)
  tmp/                                      scratch space (uploads, extracts)
  sftp/users.db                             SFTP credentials (base64, see below)
  sftp/ssh_host_key                         SSH host key (generated on first use)
/var/log/oxlify/oxlify.log                  daemon log
```

The `volumes`, `backups`, `archives` and log locations mirror the panel's
expectations. `servers`, `tmp` and `sftp` are additions required by the daemon.

## Building

```sh
make release        # builds bin/oxlify (Crystal)
make sftp           # builds bin/oxlify-sftp (C, requires libssh-dev)
sudo make install   # installs binaries, config and data directories
```

`deploy/install.sh` performs the same steps, builds the SFTP helper when a C
compiler is available, and installs `deploy/oxlify.service`.

To run from source during development:

```sh
crystal run src/oxlify.cr -- --config config/config.example.json
```

## Configuration

`/etc/oxlify/config.json`:

```json
{
  "uuid": "00000000-0000-0000-0000-000000000000",
  "token": "change-me-to-the-daemon-token",
  "host": "0.0.0.0",
  "port": 8080,
  "data_dir": "/var/lib/oxlify",
  "log_dir": "/var/log/oxlify",
  "log_level": "info",
  "ssl": { "enabled": false, "cert": "/etc/oxlify/cert.pem", "key": "/etc/oxlify/key.pem" },
  "docker": { "socket": "/var/run/docker.sock", "network": "pylox-servers", "network_mode": "pylox-servers" },
  "sftp": { "enabled": false, "host": "0.0.0.0", "port": 2022, "helper": "oxlify-sftp" },
  "limits": { "disk_check_interval": 30, "stats_sample_interval": 2, "disk_usage_ttl": 30, "stop_timeout": 30, "startup_timeout": 300, "console_buffer": 4096, "console_throttle_ms": 50 }
}
```

Command line flags override the file (`--config`, `--token`, `--host`, `--port`,
`--data`, `--log-dir`, `--log-level`), and environment variables override both
(`OXLIFY_TOKEN`, `OXLIFY_HOST`, `OXLIFY_PORT`, `OXLIFY_DATA`,
`OXLIFY_LOG_LEVEL`, `OXLIFY_SFTP_PORT`, `DOCKER_SOCKET`, `DOCKER_NETWORK`).

When `ssl.enabled` is true the daemon serves HTTPS using `ssl.cert`/`ssl.key`;
otherwise it serves plain HTTP.

## HTTP API

Every request except `GET /` and `GET /health` must send the header
`x-daemon-token: <token>`. Errors are returned as `{"error": "..."}` with an
optional `"code"` field.

| Method | Path | Description |
| --- | --- | --- |
| GET | `/` | liveness probe |
| GET | `/health` | liveness probe |
| GET | `/api/info` | host totals: `total_memory`, `cpus`, `total_disk` |
| POST | `/api/servers` | create/ensure a server (idempotent) |
| GET | `/api/servers/:uuid/status` | `status`, `startedAt`, `containerId` |
| GET | `/api/servers/:uuid/stats` | `cpu`, `memory`, `memory_limit`, `disk`, `network_rx`, `network_tx` |
| POST | `/api/servers/:uuid/install` | run the install container |
| POST | `/api/servers/:uuid/start` | start |
| POST | `/api/servers/:uuid/restart` | restart |
| POST | `/api/servers/:uuid/stop` | graceful stop |
| POST | `/api/servers/:uuid/kill` | force kill |
| POST | `/api/servers/:uuid/limits` | update `memory`, `cpu`, `disk` |
| POST | `/api/servers/:uuid/exec` | run a command, returns `{"output": "..."}` |
| POST | `/api/servers/:uuid/sftp/password` | set the SFTP password |
| DELETE | `/api/servers/:uuid?wipe=1` | delete the server (and data with `wipe`) |
| GET | `/api/servers/:uuid/files?path=` | list a directory |
| GET | `/api/servers/:uuid/files/read?path=` | read a file |
| POST | `/api/servers/:uuid/files/write` | write a file |
| POST | `/api/servers/:uuid/files/mkdir` | create a directory |
| POST | `/api/servers/:uuid/files/delete` | delete a path |
| POST | `/api/servers/:uuid/files/upload?path=` | upload raw bytes |
| POST | `/api/servers/:uuid/files/rename` | rename within a directory |
| POST | `/api/servers/:uuid/files/move` | move into a directory |
| POST | `/api/servers/:uuid/files/copy` | copy |
| POST | `/api/servers/:uuid/files/chmod` | change mode |
| POST | `/api/servers/:uuid/files/compress` | create a zip |
| POST | `/api/servers/:uuid/files/extract` | extract an archive |
| GET | `/api/servers/:uuid/files/download?path=` | download (zips directories) |
| GET | `/api/servers/:uuid/backup` | stream a zip of the server volume |
| POST | `/api/servers/:uuid/restore` | restore from a zip body |

`POST /api/servers` is idempotent — the panel calls it before every start. The
container is only recreated when the image or limits change.

### Graceful stop

`POST /api/servers/:uuid/stop` mirrors Wings. If the server defines a stop
command (the egg's `config.stop`, which the panel sends as `stopCommand`), the
daemon writes it to the container console — for example `stop` for a Minecraft
server — and waits up to `stop_timeout` seconds for the container to exit. Only
then does it fall back to the Docker stop signal (SIGTERM, then SIGKILL after
10s). When the server has no stop command, the daemon goes straight to the
Docker stop with a 30s grace. A stop command that starts with `^` is treated as
a signal (`^C` becomes `SIGINT`). `restart` runs the same graceful stop before
starting the container again, while `kill` always sends `SIGKILL` immediately.

### Status mapping

| Docker state | Reported |
| --- | --- |
| `running` | `running` (or `starting`, see below) |
| `restarting`, `paused` | `starting` |
| anything else | `offline` |

A server that was never created reports `not_created` (the panel maps this to
`offline`). `startedAt` is formatted as `%Y-%m-%dT%H:%M:%SZ`.

### Starting state

Docker reports a container as `running` the instant it is created, which says
nothing about whether the game inside it has finished booting. Like Wings, the
daemon therefore derives an extra `starting` state from the egg: the panel sends
the egg's startup line (`config.startup.done`, e.g. `)! For help, type `) as
`startupDone`, and after a start the daemon reports `starting` until that line
appears in the console, then flips to `running`.

The line is matched as a regular expression when it compiles and as a plain
substring otherwise (many eggs contain unbalanced parentheses that are not valid
regexes). Detection runs in the background, so it does not depend on a panel
console being open; it gives up after `startup_timeout` seconds and assumes the
server is up. A server with no `startupDone` goes straight to `running`, and any
stop/kill/delete clears the `starting` state immediately.

### Disk limits

The daemon measures the server volume size. If it exceeds the configured limit it
refuses to start the server with HTTP `409` and body
`{"error": "...", "code": "disk_limit_blocked", "used_mb": N, "limit_mb": N}`.

## WebSocket console

```
ws(s)://<host>:<port>/ws?token=<token>&server=<uuid>
```

Daemon to panel:

```json
{"event": "console", "data": "..."}
{"event": "status", "status": "running"}
{"event": "error", "message": "..."}
```

Panel to daemon:

```json
{"action": "start" | "stop" | "restart" | "kill" | "status"}
{"action": "command", "command": "say hello"}
```

Console output is buffered and throttled (`console_buffer`,
`console_throttle_ms`) before it is fanned out to WebSocket clients. Status is
emitted only when it actually changes (plus once to each new client), so the
console is not spammed with repeated `status` frames.

### Resource stats

`GET /api/servers/:uuid/stats` answers from an in-memory cache that a background
sampler refreshes every `stats_sample_interval` seconds, and it also carries the
live `status`/`startedAt` fields. This keeps the endpoint instant: Docker's
non-streaming `stats` call itself takes about a second (it waits for two
collection cycles), so calling it on every request would add that delay to the
panel's dashboard, console polling and page loads. Disk usage is recomputed at
most once per `disk_usage_ttl` seconds.

## SFTP

SFTP is enabled by default and served by the small C helper `oxlify-sftp`, which
embeds a libssh SFTP server. The Crystal daemon spawns it with:

```
oxlify-sftp --host 0.0.0.0 --port 2022 \
            --users /var/lib/oxlify/sftp/users.db \
            --host-key /var/lib/oxlify/sftp/ssh_host_key
```

- Credentials live in `<data>/sftp/users.db`, one user per line:
  `base64(username) base64(password) base64(root)`. The daemon rewrites this
  file whenever a server is created/deleted or a password changes, and the helper
  reloads it for each new session.
- After password authentication the helper `chroot`s into the server's volume
  directory, so every path is confined to that server.
- An SSH host key (`ssh_host_key`) is generated automatically on first start
  (Ed25519, falling back to RSA).
- The helper requires root (for `chroot`) and should be built against a libssh
  that was compiled with server support.

### Build SFTP helper

The Makefile builds the helper separately. On a Linux build host (the VM where
you run oxlify), build and install it with `libssh` headers present:

```sh
sudo apt update
sudo apt install -y build-essential libssh-dev pkg-config
make release
make sftp
sudo make install
```

## Service

```sh
sudo systemctl enable --now oxlify
sudo journalctl -u oxlify -f
```

The unit creates `/var/lib/oxlify`, `/etc/oxlify` and `/var/log/oxlify` via
systemd's `StateDirectory`, `ConfigurationDirectory` and `LogsDirectory`.

### Verify SFTP is listening

After enabling/restarting the daemon, ensure SFTP is bound:

```sh
sudo ss -ltnp | grep 2022
# Should show LISTEN on 0.0.0.0:2022 or *:2022
sudo journalctl -u oxlify -n 40 --no-pager
```

If nothing is listening, ensure the helper exists and SFTP is enabled in
`/etc/oxlify/config.json` (`"sftp": { "enabled": true, ... }`).

### Firewall

The panel talks to the daemon over TCP, so the listening port (8080 by default)
must be reachable from the panel host. If a firewall is enabled, connections
are dropped and the panel appears to hang for a couple of seconds on every
request instead of failing fast.

```sh
sudo ufw allow 8080/tcp        # or the port set in config.json
sudo ufw allow 2022/tcp        # SFTP (enabled by default)
sudo ufw status verbose
```

Verify from the panel host:

```sh
curl http://<node-ip>:8080/health
```

A `Connection refused` means the daemon is not running (or the port is wrong);
a timeout usually means a firewall or a wrong address.

## Testing

```sh
make test     # crystal spec
```

The specs are self-contained and cover path confinement and the zip archive
codec (including archive-slip rejection). The Docker, SFTP and HTTP layers are
exercised against a real node.
