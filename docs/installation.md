> Mirrored automatically from the (private) source repository's
> `docs/installation.md` at build time. The "Build from source"
> section below only works if you have access to that repository —
> everyone else should use "Download", which is the normal path.

# Installing Yggdrasil

A from-scratch guide to standing up the control plane (`yggd`) and enrolling
node agents (`ygg-agent`) on blank systems. For what these two things are and
how they relate, see [architecture.md](architecture.md); this document is
purely mechanical — what to run, in what order, on which machine.

There is no `.deb` or container image, but there is nothing to build either:
CI produces a Linux binary for both `yggd` and `ygg-agent` on every push to
each of the project's three branches (`.github/workflows/release.yml`), so
grabbing one from the public release repository's GitHub Releases page is the
normal path. Building from source is only needed if you are contributing
changes.

## Layout

Two roles, on however many machines:

- **One control plane host** runs `yggd`. It needs nothing but a filesystem
  to hold its SQLite database — no container runtime, no GPU, nothing
  game-related. A small always-on machine (or a VM) is enough.
- **One or more node hosts** each run `ygg-agent` plus a container runtime.
  These are the machines that actually run game servers, so they want real
  CPU, RAM, and disk.

The control plane and a node can be the same machine — nothing stops running
`yggd` and `ygg-agent` side by side on one box — but they are described
separately below since most homelab setups split them.

## Quick install

`hack/install.sh` automates everything in "1. Get the binaries" through
"3. Enroll a node" below for a single role in one run (ADR-043) — Linux
only, and it needs to run as root (it creates a system user and a systemd
unit):

```bash
# Control plane
curl -fsSL https://github.com/arasoi/yggdrasil-releases/releases/download/release-latest/install.sh -o install.sh
sudo bash install.sh --role control-plane

# Node (paste the enroll command from the control plane's Nodes page,
# or omit --enroll-cmd and the script will prompt for it)
sudo bash install.sh --role node --control-addr yggd.lan:8443 \
  --enroll-cmd "ygg-agent enroll --control-addr yggd.lan:8443 --token <token> --ca-fingerprint <fingerprint>"
```

Run `install.sh --help` for every flag (channel, addresses, install
directory). It verifies the download the same way step 1 below does by hand
— HTTPS plus the release's published `SHA256SUMS`, no separate signing key
(ADR-043). Re-running it against an already-installed host is a real update
path, not just best-effort: it replaces the binary and restarts the service,
but never rewrites a config value already in the file (yours or a previous
run's) — a config option added by a newer release is appended net-new,
commented out exactly as the example ships it, rather than silently never
reaching an already-installed host. A node that is already enrolled is never
re-enrolled automatically, since re-enrolling mints a new node identity and
orphans that node's existing servers (ADR-052) — pass `--force-enroll` if you
genuinely mean to replace it. A fresh control-plane install also provisions
and configures a seeds directory (`seeds_dir`, under `--config-dir`)
automatically, so the seed authoring UI works with no manual edit; bundled
seeds (Minecraft, ARK, Valheim, ...) need no configuration at all, since
they're embedded in the `yggd` binary itself (ADR-049).

The rest of this document is the manual, step-by-step version of the same
process — read it if you're customizing beyond what the script's flags
cover, building from source, or running a non-Linux dev build.

Production targets **Linux** for both binaries: the agent depends on a
container daemon and on systemd for supervision (ADR-013), and the control
plane's default paths (`/var/lib/yggdrasil`, `/etc/yggdrasil`) assume it. The
control plane happens to build and run on Windows and macOS too — useful for
working on it, not a supported way to run it in production.

## Prerequisites

On **each node host**:

- A container runtime speaking the Docker Engine API: Docker Engine or
  Podman's Docker-compatibility endpoint both work (ADR-030). Whatever user
  runs the agent needs permission to use it — being in the `docker` group is
  the usual way.
- systemd, to supervise the agent and restart it (`Restart=always`) if it
  ever exits. The agent does not restart itself.
- **`steamcmd`, on PATH (or pointed at via `YGG_STEAMCMD_BIN`) — only if a
  seed you use declares `install.method: steamcmd`** (ARK Survival Ascended
  and Valheim, among the bundled seeds). This runs directly on the node
  host, not inside a container (ADR-018): the agent shells out to it with
  `os/exec` the same way it would any other local tool. Nothing installs it
  for you — see [SteamCMD's own docs](https://developer.valvesoftware.com/wiki/SteamCMD)
  for your distribution. A seed using only `download`-method installs, or
  none at all, never touches this.

Neither binary needs a Go toolchain, a database server, or anything else
installed to *run* — only the container runtime and systemd above, on node
hosts. Building from source (the alternative to downloading, below) does
need Go 1.24+, but that is only ever required on whichever machine does the
build.

## 1. Get the binaries

### Download (recommended)

Every push to `develop`, `qa`, and `main` rebuilds both binaries and
publishes them as a GitHub Release in the public release repository for that
branch — see [Release channels](#release-channels) below for which one to use.
From that repository's **Releases** page, or with `gh`:

```bash
# Pick one channel: develop-latest, qa-latest, or release-latest
gh release download release-latest --repo arasoi/yggdrasil-releases --clobber
sha256sum -c SHA256SUMS --ignore-missing
chmod +x yggd-linux-amd64 ygg-agent-linux-amd64
```

That downloads every asset in the release: both architectures' binaries,
`SHA256SUMS`, and `yggd.example.yaml`/`agent.example.yaml` (the config
templates steps 2 and 3 below copy from — bundled in the release precisely
so a binary-only download still has them, without needing a full repo
checkout). Pass `--pattern` flags instead if you only want specific files.
Substitute `arm64` for `amd64` on ARM hosts. `SHA256SUMS` covers every
architecture in the release, so `--ignore-missing` skips the ones you did
not download rather than failing on them. The rest of this guide refers to
the binaries as `yggd` and `ygg-agent` — rename them after downloading, or
adjust the paths in the systemd units below to match.

### Build from source

Only needed if you are changing the code. From a checkout of this
repository:

```bash
git clone https://github.com/arasoi/yggdrasil && cd yggdrasil
make build
```

This produces `bin/yggd` and `bin/ygg-agent` for the machine you built on.
To cross-compile the agent for Linux from elsewhere:

```bash
make build-agent-linux
```

This produces `bin/ygg-agent-linux-amd64` and `bin/ygg-agent-linux-arm64`.
There is no equivalent `make` target for `yggd` itself, since it has no
native dependencies to worry about (ADR-005's pure-Go SQLite driver needs no
cgo) — if you need it for Linux from a non-Linux build machine too, a plain
`GOOS=linux GOARCH=amd64 go build -o bin/yggd-linux-amd64 ./cmd/yggd` works
the same way. Without `make` at all, every target here is just a thin
wrapper over `go build`/`go test` — see the Makefile for the exact
invocations (it only adds version ldflags; nothing it does is otherwise
special).

Either way, get `yggd` onto the control plane host and `ygg-agent` onto each
node host — matching that host's architecture — into a service-owned directory
such as `/var/lib/yggdrasil/bin/yggd` and `/var/lib/yggdrasil/bin/ygg-agent`.
The self-update path stages a temporary replacement next to the running binary,
so that directory must be owned by the `yggdrasil` user and be writable by it.
Downloading directly on the target host (`gh release download` or `curl -L`
against the release asset URL) avoids a separate copy step; `scp` from
wherever you built or downloaded works just as well. If you are migrating an
existing install from the old `/usr/local/bin` location, re-running the
bootstrap script updates the service unit to the new path and installs the
binary into the service-owned directory for you.

### Release channels

The public install script and any auto-update client should read the public
release manifest at
`https://github.com/arasoi/yggdrasil-releases/releases/download/release-latest/releases.json`.
It exposes the current channel tag, release URL, and asset URLs for the
published binaries and installer script.

| Branch | Release tag | Meaning |
|--------|-------------|---------|
| `develop` | `develop-latest` | Newest merged work. Moves often, least tested. |
| `qa` | `qa-latest` | Promoted from `develop` for testing before `main`. |
| `main` | `release-latest` (also marked "Latest" on GitHub) | What CLAUDE.md's branch strategy calls "production-ready code only". |

Each tag is a moving target — pushing to that branch overwrites the release
rather than creating a new one — so `release-latest` today is not
necessarily the same bytes as `release-latest` yesterday.

`yggd` and `ygg-agent` both log their version on the very first line at
startup; that string is independent of which release channel a binary came
from — it is `<VERSION file contents>-<short commit>` (e.g. `0.1.0-5861c02`),
computed by `hack/version.sh` (ADR-039) from the repository's `VERSION` file
at the commit that was actually built, not from the floating channel tag
above. Two binaries from the same commit report the same version regardless
of which of the three releases you got them from.

## 2. Set up the control plane

On the control plane host:

```bash
sudo useradd --system --home /var/lib/yggdrasil --shell /usr/sbin/nologin yggdrasil
sudo mkdir -p /var/lib/yggdrasil /var/lib/yggdrasil/bin /etc/yggdrasil
sudo chown -R yggdrasil:yggdrasil /var/lib/yggdrasil
sudo chmod 0755 /var/lib/yggdrasil/bin
```

Copy the example config and edit it:

```bash
sudo cp yggd.example.yaml /etc/yggdrasil/yggd.yaml
```

At minimum, decide:

- `http_addr` — keep this on a private interface (the default,
  `127.0.0.1:8080`, is deliberately not reachable from the network). If you
  are fronting the UI with a reverse proxy or the Cloudflare tunnel
  described in architecture.md's "Network topology", point it at that proxy
  instead of exposing it directly.
- `grpc_addr` — this **does** need to be reachable from your node hosts over
  the LAN (agents dial in to it, ADR-002). The default, `0.0.0.0:8443`, is
  usually right; just make sure nothing blocks port 8443 between your node
  hosts and this machine.

Every config key has a flag and an environment variable equivalent
(`YGG_HTTP_ADDR`, etc. — see the comments in `yggd.example.yaml`), so a
config file is convenient but not required. If you want `yggd` to watch for
and offer to install its own updates (see "Updating → yggd" below), also
set `update_channel` to whichever channel you downloaded from.

Create a systemd unit:

```ini
# /etc/systemd/system/yggd.service
[Unit]
Description=Yggdrasil control plane
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=yggdrasil
Group=yggdrasil
ExecStart=/var/lib/yggdrasil/bin/yggd --config /etc/yggdrasil/yggd.yaml
Restart=always
RestartSec=2
AmbientCapabilities=
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

`Restart=always` rather than `on-failure` matters beyond the usual
robustness argument: it's what lets a self-update (below) actually come
back up after the clean exit it triggers — `on-failure` would leave `yggd`
down until someone started it by hand.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now yggd
sudo systemctl status yggd
```

Check it came up:

```bash
curl http://127.0.0.1:8080/healthz
```

Then open the UI (directly at `http://<host>:8080` if you exposed it that
way, or through whatever proxy you put in front of it) and create the
administrator account — the first visit to any page redirects here
automatically. This account is the only one; see ADR-009 for why v1 has no
multi-user support.

## 3. Enroll a node

Everything from here happens through the UI, on the **Nodes** page: press
**Add a node** to generate a one-time enrollment command. It looks like:

```bash
ygg-agent enroll --control-addr yggd.lan:8443 --token <token> --ca-fingerprint <fingerprint>
```

The token is single-use and expires in an hour; the fingerprint pins the
control plane's certificate authority so a brand-new agent — which has
nothing to trust yet — can verify what it is talking to on its very first
connection (ADR-028). Generate a fresh one for each node rather than reusing
one across hosts.

On the node host, as whatever user will run the agent:

```bash
sudo useradd --system --home /var/lib/yggdrasil --groups docker --shell /usr/sbin/nologin yggdrasil
sudo mkdir -p /var/lib/yggdrasil/agent /etc/yggdrasil
sudo chown -R yggdrasil:yggdrasil /var/lib/yggdrasil
```

(`--groups docker` is what grants access to the Docker socket; use whatever
group your runtime's socket is owned by if it differs, e.g. Podman's rootful
socket setup.)

Paste the enrollment command, run as that user:

```bash
sudo -u yggdrasil ygg-agent enroll --control-addr yggd.lan:8443 --token <token> --ca-fingerprint <fingerprint>
```

This writes a client certificate under `/var/lib/yggdrasil/agent/certs/` and
exits — it does not start the agent. Copy the example config and point it at
your control plane:

```bash
sudo cp agent.example.yaml /etc/yggdrasil/agent.yaml
```

Set `control_addr` in it to the same address you enrolled with. Everything
else about which servers to run arrives from the control plane on connect
(ADR-012) — this file only needs to say where to find it and where to keep
its own state.

Create a systemd unit:

```ini
# /etc/systemd/system/ygg-agent.service
[Unit]
Description=Yggdrasil node agent
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=yggdrasil
Group=yggdrasil
ExecStart=/var/lib/yggdrasil/bin/ygg-agent --config /etc/yggdrasil/agent.yaml
Restart=always
RestartSec=2

[Install]
WantedBy=multi-user.target
```

`Restart=always` here is not a suggestion — the agent owns restart decisions
for the containers it supervises, but relies on systemd to restart *itself*
if it ever exits (ADR-013). Drop `Requires=docker.service`/adjust `After=`
if you are running Podman instead, which is not typically a systemd-managed
daemon in the same way.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now ygg-agent
sudo systemctl status ygg-agent
```

Back in the UI's **Nodes** page, the node should appear and show as
connected within a few seconds, reporting its architecture, Docker version,
and agent version. If it does not:

- `journalctl -u ygg-agent -f` on the node host is the first place to look.
- A permission error talking to the container runtime means the user the
  agent runs as is not actually in the right group — group membership
  changes need a fresh login (or, for a systemd service, just restarting it
  after `usermod` is enough).
- A connection error reaching `grpc_addr` means port 8443 (or whatever you
  configured) is not reachable from the node — check firewalls between the
  two hosts.
- `ygg-agent enroll` failing with a fingerprint mismatch means the value was
  copied wrong, or the control plane's CA was regenerated since the command
  was issued (deleting and recreating `<data_dir>/pki` on the control plane
  would do that) — generate a new enrollment command and retry.

## Removing a node

Deleting a node from the UI revokes it immediately: the hub refuses any
stream from a node not in the registry, even though its certificate remains
cryptographically valid otherwise. It does not touch anything already
running on that host — stop `ygg-agent` there yourself (`systemctl disable
--now ygg-agent`) if you are decommissioning the machine entirely.

## Updating

If you installed with `hack/install.sh`, re-running the same command for
your role updates the binary and restarts the service without touching your
existing config (see "Quick install" above). The manual paths below are the
same thing done by hand.

### `yggd`

Set `update_channel` (config key or `YGG_UPDATE_CHANNEL`) to `develop`,
`qa`, or `main` — whichever channel you downloaded from — and the
**Updates** page shows an **Update now** button whenever that channel has a
newer build than what's running (ADR-044). Clicking it downloads the build,
verifies it against the release's published `SHA256SUMS` (the same check
you'd run by hand below, just automated — no separate signing key), swaps
it into place, and restarts `yggd`. This requires `Restart=always` in the
systemd unit (the example above already has it); without it, the binary
still gets installed but nothing brings the process back up, and the page
says so.

`update_channel` is unset by default — this is opt-in. Leave it unset and
update manually instead:

```bash
sudo systemctl stop yggd
# replace the binary at the same path
sudo systemctl start yggd
```

Restarting `yggd` — by either path — does not stop any game server: the
control plane holds no game state and agents keep supervising while it is
down (the "control plane outage never stops a game server" property in
architecture.md's overview).

### `ygg-agent`

Signed, UI-triggered updates (ADR-015, ADR-040, ADR-041) are built: on the
**Agent Binaries** page, either **fetch** a `ygg-agent` build straight from
the public releases repo for a chosen channel and architecture, or **upload**
one you built yourself (`make build-agent-linux`) for a custom or patched
build (ADR-055). Either way the control plane verifies it — a fetched binary
against that release's published checksum, the same trust level `yggd`'s own
self-update already uses — signs it with its own key, and lists it. Back on
the **Nodes** page, a node running an older version than what's registered for
its architecture shows an **Update** button; clicking it tells that node to
download, verify, install, and restart itself. Progress and the eventual
result (or failure reason) show inline on the node's row. Nothing else on
that host is touched — Docker owns the container lifecycle, not the agent
(ADR-012), so a restarting agent never stops a game server.

There is still no CI-to-control-plane signing pipeline (ADR-040 explains why:
keeping the signing key out of CI entirely is the point) — fetching only
gets you the checksummed bytes; this control plane still does its own local
signing after either fetching or uploading. A fetched binary's version is
read automatically from the release's own manifest, so it always matches
what the binary actually reports. An uploaded binary's version is whatever
you type in the upload form instead — it cannot be read automatically from a
binary that may be cross-compiled for a different OS/architecture than the
one running `yggd` — so make sure it matches what the binary actually reports
on startup, or the update will "succeed" (install correctly) but never
resolve as complete, since completion is driven by the node reconnecting and
reporting that same version string.

Manual updates still work exactly as before, and are the only option for a
node's very first update (before any binary is registered), for an
air-gapped node with no outbound access to GitHub, or for troubleshooting:

```bash
sudo systemctl stop ygg-agent
# replace the binary at the same path
sudo systemctl start ygg-agent
```

Either way, restarting `ygg-agent` does not stop any game server, for the
same ADR-012 reason. Both binaries reconnect/resume on their own; there is
no ordering requirement between updating the control plane and updating any
given node.
