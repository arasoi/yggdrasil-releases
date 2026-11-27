> Mirrored automatically from the (private) source repository's
> `docs/installation.md`. The "Build from source" section below
> only works if you have access to that repository — everyone else
> should use "Download", which is the normal path.

# Installing Yggdrasil

A from-scratch guide to standing up the control plane (`yggd`) and enrolling
node agents (`ygg-agent`) on blank systems. For what these two things are and
how they relate, see [architecture.md](architecture.md); this document is
purely mechanical — what to run, in what order, on which machine.

There is no `.deb` or container image, but there is nothing to build either:
CI produces a Linux binary for both `yggd` and `ygg-agent` on every push to
each of the project's three branches that changes them
(`.github/workflows/release.yml` — documentation never triggers a release,
and only the binaries a push actually affected are rebuilt, so each binary's
version moves when its own code does, ADR-093/ADR-111), and grabbing one from
the public release repository's GitHub Releases page is the normal path.
Building from source is only needed if you are contributing changes.

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
unit). Run it with no arguments and it asks:

```bash
curl -fsSL https://github.com/arasoi/yggdrasil-releases/releases/download/release-latest/install.sh -o install.sh
sudo bash install.sh
```

It asks what this host should run (control plane or node), which release
channel to follow, and — for a control plane — which addresses to bind and
whether to watch that channel for its own updates. For a node it asks for
one thing: the `ygg-agent enroll ...` command from the control plane's Nodes
page. It never asks where the control plane is, because that command already
carries the address and `ygg-agent enroll` records it in `agent.yaml` itself.

Every question has a flag, and a flag is never asked about — so the same
script serves an unattended run:

```bash
sudo bash install.sh --role control-plane --channel main --http-addr 127.0.0.1:8080

sudo bash install.sh --role node --channel main \
  --enroll-cmd "ygg-agent enroll --control-addr yggd.lan:8443 --token <token> --ca-fingerprint <fingerprint>"
```

Prompts read from `/dev/tty` rather than stdin, so piping the script
(`curl ... | sudo bash`) stays interactive instead of consuming itself as
input. `--non-interactive` turns prompting off and takes defaults for
anything not passed.

Run `install.sh --help` for every flag (channel, addresses, install
directory). It verifies the download the same way step 1 below does by hand
— HTTPS plus the release's published `SHA256SUMS` (ADR-043); it does not
check the release signature `yggd` checks on later updates (see
[Release signing](#release-signing) to check one by hand). Re-running it
against an already-installed host is a real update path, not just best-effort: it replaces the binary and restarts the service,
but never rewrites a config value already in the file (yours or a previous
run's) — a config option added by a newer release is appended net-new,
commented out exactly as the example ships it, rather than silently never
reaching an already-installed host. A node that is already enrolled is never
re-enrolled automatically, since re-enrolling mints a new node identity and
orphans that node's existing servers (ADR-052) — pass `--force-enroll` if you
genuinely mean to replace it. A fresh control-plane install also provisions
and configures a seeds directory (`seeds_dir`, under `--config-dir`)
automatically, so the seed authoring UI works with no manual edit. Game
definitions themselves are **not** in the binary (ADR-087): `yggd` reads them
from the published catalog, which `seed_channel` tracks and which defaults to
`main`, so the Seeds page lists what is available with no configuration —
installing any of it is one click per seed. A host with no outbound access
gets no seeds this way and needs bundles placed in `seeds_dir` by hand.

To undo an install, `hack/uninstall.sh --role control-plane|node|both`
removes the unit, binary, config, data directory, and system user — see
"Uninstalling" below before running it, since there is no undo.

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
- **Nothing for SteamCMD.** A seed with a `steamcmd` install step (ARK
  Survival Ascended and Valheim, among the catalog's seeds) runs it in a
  container on the node rather than on the host (ADR-057), so there is no
  `steamcmd` binary to install and no `PATH` to arrange. What the node does
  need is to be able to pull `ghcr.io/arasoi/base-steamcmd:release-latest`.
  If that package is not readable from your node — the source repository is
  private, so it may not be — mirror it and point the agent at the mirror
  instead of installing SteamCMD by hand.
- **Direct outbound access for seed downloads.** A seed's `download` step is
  fetched by the agent itself, and it ignores `HTTP_PROXY`/`HTTPS_PROXY`, since
  a proxy would carry the connection past its address check. That check refuses
  the node's loopback, link-local (including cloud metadata), CGNAT and other
  reserved addresses; a mirror on a private LAN address (`10/8`, `172.16/12`,
  `192.168/16`, `fc00::/7`) is fine (ADR-179).

Neither binary needs a Go toolchain, a database server, or anything else
installed to *run* — only the container runtime and systemd above, on node
hosts. Building from source (the alternative to downloading, below) does
need the Go version in `go.mod`'s `go` directive (currently 1.26.5) or newer,
but that is only ever required on whichever machine does the build.

## 1. Get the binaries

### Download (recommended)

Every push to `develop`, `qa`, and `main` that changes the software rebuilds
the binaries it affected and republishes the channel's GitHub Release in the
public release repository — a binary the push did not touch is carried
forward unchanged, keeping the version it was last built at (ADR-111) — see [Release channels](#release-channels) below for which one to use.
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
checkout). `ygg-seed` is in there too, and is the only one you do not need
to run Yggdrasil: it is the seed authoring tool, published because the seeds
repository's CI has to download it rather than build it from private source
(ADR-081). Pass `--pattern` flags instead if you only want specific files.
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

> **Running the tunnel connector or a reverse proxy on another machine? Set
> `http_trusted_proxies`.** yggd only believes `X-Forwarded-Proto`,
> `X-Forwarded-Host` and the client-address header (below) from peers in that
> list, which defaults to loopback —
> right when `cloudflared` runs on the same host as `yggd`, which is the
> ordinary setup. From any other peer those headers are stripped (ADR-180).
> Behind a proxy that is *not* listed, sign-in still works but the session
> cookie is no longer marked Secure, and every form POST and websocket is
> refused with "cross-site POST blocked" because the origin no longer matches.
> `yggd` logs `ignoring X-Forwarded-* headers from a peer outside
> http_trusted_proxies` (at most once every ten minutes) naming the peer
> address to add:
>
> ```yaml
> http_trusted_proxies: ["127.0.0.0/8", "::1/128", "192.168.1.20"]
> ```

The browser's own address — what sign-in lockout counts against, what the audit
log records and what `/profile` shows beside each session — comes from **one**
header a trusted proxy sets, `http_client_ip_header` (`YGG_HTTP_CLIENT_IP_HEADER`):

- `X-Forwarded-For` (the default) — yggd takes its **rightmost** entry, the one
  your proxy appended itself. Right for nginx (`proxy_set_header
  X-Forwarded-For $proxy_add_x_forwarded_for`), Caddy, Traefik, and the
  Cloudflare tunnel, whose edge appends the visitor's address too.
- `CF-Connecting-IP` — what `cloudflared` sets; choose it if the tunnel is the
  only thing in front of yggd and you prefer Cloudflare's own header.
- `X-Real-IP` — only for a proxy that **overwrites** it with the address it saw.

Every other client-address header is removed, even from a trusted proxy: most
proxies pass a browser-sent `X-Real-IP` or `CF-Connecting-IP` through
unchanged, and believing it would let a client pick a new address per request
and walk around the sign-in lockout. Any other value refuses to start.

```yaml
http_client_ip_header: X-Forwarded-For
```

Three more settings harden the UI listener itself. All three are read at
startup from `yggd.yaml` (or `YGG_HTTP_TLS_CERT`, `YGG_HTTP_TLS_KEY`,
`YGG_HTTP_ALLOWED_HOSTS` as a comma-separated list, and
`YGG_HTTP_TRUSTED_PROXIES` likewise), since they shape the listener before
there is a database to read settings from:

- `http_tls_cert` and `http_tls_key` — serve the UI over HTTPS (TLS 1.2 or
  newer). Set both or neither; one alone refuses to start. Use this whenever
  `http_addr` is reachable from the LAN, because over plain HTTP anyone on the
  network can read a sign-in password or the session cookie in transit — `yggd`
  logs a warning at startup when it is listening beyond loopback without TLS.
  The files are re-read when they change, so a renewal (certbot, a cron job)
  takes effect without restarting. Over HTTPS, and through the tunnel, the
  session cookie is marked Secure and responses carry
  `Strict-Transport-Security: max-age=31536000`. HSTS pins the *host name* the
  browser used: if you reach the same name over plain HTTP on your LAN
  (split-horizon DNS), give the LAN its own name or turn TLS on there too.
- `http_allowed_hosts` — the host names the UI answers to, such as your tunnel
  hostname and a LAN name. A request addressed to any other name gets
  `421 Misdirected Request`, which is what stops a malicious web page from
  reaching a loopback-bound `yggd` through DNS rebinding — including the
  unauthenticated first-run `/setup`. IP addresses and `localhost` are always
  accepted, so `curl http://127.0.0.1:8080/healthz` keeps working. Empty, the
  default, accepts any name and logs a warning at startup recommending you set
  it.

### Addon web UIs: a second origin

Some seeds bundle an addon with a web interface of its own — a database admin tool such as
Adminer, a map viewer — and give it a tab on the server's page. Since ADR-182 that tab is served
from a **second listener with an origin of its own**, never from the UI's own address: an addon is
a third-party image running third-party script, and served from the UI's origin anything it ran
could act as you (create accounts, read settings, type into a game console). **Until you configure
an addon origin, addon tabs show a notice saying the feature is off; nothing else changes.**

Two settings, set together (environment: `YGG_ADDON_HTTP_ADDR`, `YGG_ADDON_PUBLIC_URL`):

- `addon_http_addr` — the addon listener's own address, like `http_addr`. Keep it on loopback
  behind a tunnel or proxy, exactly as you would `http_addr`.
- `addon_public_url` — the bare origin browsers use to reach that listener, such as
  `https://addons.ygg.example.com`. No path. yggd puts it in each addon tab's iframe and refuses
  any request to the listener addressed to another host name (421).

```yaml
addon_http_addr: 127.0.0.1:8081
addon_public_url: https://addons.ygg.example.com
```

**Use a distinct host name, not just a distinct port.** Both work, and yggd refuses only a URL that
*is* the UI's own origin. But browsers scope cookies by host and ignore the port, so with
`https://ygg.lan:8081` beside `https://ygg.lan:8080` your browser sends yggd's session cookie to the
addon listener on every request. That is still safe — the addon listener strips it before proxying
and never treats it as signing you in, and every state-changing route and websocket on the UI checks
`Origin`, which an addon's script cannot forge — but a separate host name means the cookie is never
sent there at all. Either way, keep the two **same-site** (a sibling subdomain of the UI's
registrable domain, as above, or the same host): the addon's session cookie is `SameSite=Lax`, and
browsers do not send such a cookie inside a frame from a different site, so an addon on an
unrelated domain would never stay signed in.

**Behind the Cloudflare tunnel this is a second public hostname.** Add it to the tunnel's ingress
rules pointing at `addon_http_addr`, beside the UI's own hostname, and protect it with the same
Access policy:

```yaml
# cloudflared config.yml
ingress:
  - hostname: ygg.example.com
    service: http://127.0.0.1:8080
  - hostname: addons.ygg.example.com
    service: http://127.0.0.1:8081
  - service: http_status:404
```

The addon listener trusts `X-Forwarded-*` from the same `http_trusted_proxies` the UI does.

**TLS.** `addon_tls_cert` and `addon_tls_key` (both or neither) give the addon listener its own
pair; unset, it reuses `http_tls_cert`/`http_tls_key`, so serving both from yggd directly needs a
certificate covering both names (a SAN or wildcard certificate). With TLS on the listener,
`addon_public_url` must be `https://`. Behind the tunnel alone, neither is needed: Cloudflare
terminates TLS at its edge.

What reaching an addon looks like: opening its tab mints a single-use, one-minute ticket; the
addon listener redeems it, checks you still hold the server's "Manage server addons" permission,
and signs you in to that one addon for up to eight hours, re-checking that permission (and that
you have not signed out of yggd) at least every minute. A seed's `ui.login` auto-login still
works: the credential is filled in on the addon origin and never appears on a UI page. yggd
creates `<data_dir>/addon-ticket.key` (0600) the first time an addon origin is configured, and
generates a new one at its next start if you delete it.

#### A host per addon session: for addons that need the root of a host

With a single addon host, every addon is served under a path of its own
(`/a/<server>/<addon>/s/<session>/app/`). That suits an addon that only uses relative links,
such as Adminer, and breaks one that requests absolute paths: FiveM's **txAdmin** panel calls
`/api/...` and `/login`, which land outside its path. A seed marks such a tab `ui.root: true`,
and until the addon origin can give it a host of its own the tab says so instead of showing a
broken page (ADR-218).

Give `addon_public_url` a **wildcard** and every addon session gets its own host,
`<random label>.<domain>`, with the addon at its root:

```yaml
addon_http_addr: 127.0.0.1:8081
addon_public_url: https://*.addons.ygg.example.com
```

Every addon then works this way, whatever its seed says, and each session is an origin of its
own, so no addon's script can reach another's. The label is 26 random characters and is not a
secret; the session itself rides a host-only `HttpOnly` cookie set on that host when the tab's
single-use ticket is redeemed. As before, keep the addon hosts **same-site** with the UI.

What it needs:

- **DNS:** a wildcard record for the domain pointing where the addon listener is reached
  (behind the tunnel, a wildcard CNAME to the tunnel).
- **Tunnel:** a wildcard public hostname for the addon listener, listed after the UI's own
  hostname so the UI's exact match wins:

  ```yaml
  ingress:
    - hostname: ygg.example.com
      service: http://127.0.0.1:8080
    - hostname: "*.addons.ygg.example.com"
      service: http://127.0.0.1:8081
    - service: http_status:404
  ```

- **TLS:** a certificate for the wildcard. Cloudflare's free Universal SSL covers only one level
  (`*.example.com`); a deeper wildcard such as `*.addons.ygg.example.com` needs an Advanced
  Certificate. A first-level wildcard is accepted too (`https://*.example.com` beside a UI at
  `ygg.example.com`): only a 26-character random label is ever served as an addon, so the UI's
  own name never is. Serving TLS from yggd directly needs a certificate for the wildcard name.
- **Access:** an Access application covering the wildcard hostname, with the same policy as the
  UI.

yggd refuses any request to the addon listener whose host is not one label beneath the domain.

`GET /healthz` answers `{"status":"ok","database":"ok"}` to anyone, and adds
the running version, commit and protocol only for a signed-in browser
session (ADR-180) — a monitor needs the first, and only an operator needs the
second.

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
RestartSec=250ms
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
# or, with http_tls_cert set: curl https://<name in your certificate>:8080/healthz
```

Then open the UI (directly at `http://<host>:8080` if you exposed it that
way, or through whatever proxy you put in front of it) and create the
administrator account — the first visit to any page redirects here
automatically.

First-run setup needs a **one-time setup token** (ADR-181), so whoever reaches
the listener first cannot simply claim the fleet. While no account exists,
yggd prints the token at every start, as a ready path, to its stdout and its
log:

```bash
journalctl -u yggd | grep setup
# ... /setup?token=Zk3...  <- open this path on the control plane
```

Open that path (`http://<host>:8080/setup?token=...`), or paste the token into
the setup page. The token changes each time yggd starts — use the most recent
one — stops working the moment the account is created, and ten wrong guesses
from one address lock that address out of setup for fifteen minutes.
`hack/install.sh` prints the link for you when it finishes a fresh install —
`https://` once `http_tls_cert` is set — taken from that start's own output, so
re-running it to upgrade an install whose administrator already exists prints
no setup link at all rather than a stale one from the first start.

Secret settings (sign-in client secrets, the Apple key, the Steam key, the seed publish token) are
encrypted in the database under `<data_dir>/pki/settings.key`, created on first start. Back it up
with the rest of `pki/`: a database restored without it keeps everything else but reads those
credentials as unusable until you enter them again on the Settings page. `pki/device.key`, also
created on first start, seals the cookie a browser gets after signing in with a password, which
keeps that browser able to sign in while someone is guessing passwords against the account from
elsewhere; losing it only means each browser is treated as new until it next signs in.

Sessions sign out after 12 hours unused and 30 days regardless; both are
**Sign-in** settings (`auth.session_idle_hours`, `auth.session_max_days`).
Sensitive actions — changing who can do what, credentials, rotations,
enrollment tokens, updates, deleting a server or cluster — ask you to confirm
your password (or a linked sign-in provider) if you have not done so in the
last ten minutes.

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

Copy the example config first, so enrollment has a file to record the
control plane's address in:

```bash
sudo cp agent.example.yaml /etc/yggdrasil/agent.yaml
sudo chown yggdrasil:yggdrasil /etc/yggdrasil/agent.yaml
```

Then paste the enrollment command, run as that user:

```bash
sudo -u yggdrasil ygg-agent enroll --control-addr yggd.lan:8443 --token <token> --ca-fingerprint <fingerprint>
```

This writes a client certificate under `/var/lib/yggdrasil/agent/certs/`,
records `control_addr: yggd.lan:8443` in `/etc/yggdrasil/agent.yaml` (add
`--config <path>` if yours lives elsewhere), and exits — it does not start
the agent. You do not need to set `control_addr` by hand: the enrollment
command is the one place that address is already given to you, and repeating
it in the config was a step that only existed to be got wrong. Everything
else about which servers to run arrives from the control plane on connect
(ADR-012) — this file only needs to say where to keep its own state.

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
--now ygg-agent`, or `hack/uninstall.sh --role node` for a full removal, see
below) if you are decommissioning the machine entirely.

## Uninstalling

`hack/uninstall.sh --role control-plane|node|both` reverses what
`hack/install.sh` (or the manual steps above) set up on a host: it stops and
disables the systemd unit(s), and removes the binary, config file, data
directory, and the `yggdrasil` system user.

```bash
sudo hack/uninstall.sh --role node          # this node only
sudo hack/uninstall.sh --role control-plane # this control plane only
sudo hack/uninstall.sh --role both          # a combined host, entirely
```

It is published alongside `install.sh` on every release, so a host set up
with the `curl` one-liner does not need a source checkout to undo it:

```bash
curl -fsSL https://github.com/arasoi/yggdrasil-releases/releases/download/release-latest/uninstall.sh -o uninstall.sh
sudo bash uninstall.sh --role node
```

There is no undo. Removing a control plane deletes its SQLite database and
its `pki/` — the mTLS CA and the binary-signing key — so every enrolled node
must re-enroll against whatever replaces it. Removing a node deletes that
node's agent directory (`<data-dir>/agent`: its certificates and identity); its
installs, server state, and local backups are left on disk and are removed only
by deleting them from the control plane first or by hand. Back
up `--data-dir` (`/var/lib/yggdrasil` by default) first if you might want
any of it later. The script asks for confirmation before deleting anything
unless `-y`/`--yes` is given; `--keep-data` leaves the data directory alone
and `--keep-user` leaves the system user in place. `--help` lists every flag.

Your own seeds under `/etc/yggdrasil/seeds` are treated as data: removed
with a control plane unless `--keep-data` is given. Seeds installed from the
catalog live under the data directory and go with it for the same reason;
reinstalling them is a click each on the Seeds page.

On a combined host running both roles, uninstalling one leaves the other's
data, binary, config, unit, and the shared system user untouched. The two
share `--data-dir` there: control-plane state sits at the top of it, a
node's own state under `<data-dir>/agent`, and — by default — both binaries
under `<data-dir>/bin`, so only the slice belonging to the role being
removed is deleted.

This script only removes what is on the host it runs on. It does not revoke
a node from the control plane's registry (delete it from the **Nodes** page
for that, per "Removing a node" above), and it does not un-publish anything
from GitHub Releases or GHCR.

## Updating

If you installed with `hack/install.sh`, re-running the same command for
your role updates the binary and restarts the service without touching your
existing config (see "Quick install" above). The manual paths below are the
same thing done by hand.

### Updating past the seed unbundling: install your seeds first

Seeds are no longer built into `yggd` (ADR-087). They are downloaded from the
catalog, and the unbundling shipped in the late 0.43.0 builds on `develop` —
so 0.44.0 and later everywhere, plus any 0.43.0 build published after it, is a
control plane with no seeds of its own.

**Your servers keep running.** Agents supervise them independently of the
control plane, so nothing goes down. But a seed-driven server rebuilds its pod
specification from its seed on *every start and stop*, so until the matching
seed is installed those actions fail with "seed ... is no longer available",
and so does anything that rebuilds a pod: the Settings page, Rebuild, an
install update, removing a cluster member, and moving a server.

Nothing is lost, and recovering takes one click per seed:

1. Update `yggd` as below.
2. Open **Seeds**. The catalog is on by default and points at this project's
   own, so the list is already populated.
3. Install each seed your servers were built from — the ids are unchanged, so
   `minecraft-java-paper` is still `minecraft-java-paper`.

Lifecycle control returns as soon as each seed is installed; no restart is
needed. Doing this *before* you update avoids the window entirely, since an
installed seed sits in the data directory rather than in the binary and
survives the upgrade.

Two things worth knowing while you are there. The catalog's seeds are newer
than the ones earlier builds bundled, so the first start after installing
rebuilds that server's pod from a newer definition -- seed migrations carry
your stored settings across, which is what they exist for, but the change
happens on that start rather than when you choose it. And a control plane with
no outbound network gets no seeds this way at all: place the bundles in
`seeds_dir` by hand instead.

### Updating past the addon origin change: addon tabs need configuring

From this release, an addon's web UI (a seed's database admin or map tab) is no longer served
from the UI's own address (ADR-182). This is a deliberate breaking change: after updating, every
addon tab shows a notice saying addon web UIs are turned off, and any bookmark to a
`/servers/<id>/addon/<role>/proxy/...` address returns 404. **Game servers, their addon containers
and their data are untouched;** only how the browser reaches the addon's page changed.

To get addon tabs back:

1. Pick a host name for the addon origin, same-site with the UI — `addons.ygg.example.com` beside
   `ygg.example.com`, say (see "Addon web UIs: a second origin" above for why a host name beats a
   port).
2. Add `addon_http_addr` and `addon_public_url` to `yggd.yaml`.
3. Behind the Cloudflare tunnel, add that host name to the tunnel's ingress pointing at
   `addon_http_addr`, under the same Access policy; on a LAN, give it a DNS record (and a
   certificate covering it, if `yggd` serves TLS itself).
4. Restart `yggd`, then reopen the addon tab.

A seed's addon auto-login (`ui.login`) keeps working with no change to the seed.

### `yggd`

Set `update_channel` (config key or `YGG_UPDATE_CHANNEL`) to `develop`,
`qa`, or `main` — whichever channel you downloaded from — and the
**Updates** page lists all three channels with the version each currently
publishes, marking the one you watch. An **Update now** button appears on
that channel when what it publishes is actually ahead of what's running
(ADR-044): a higher version number reads **Update ready**, the same version
number built from a different commit reads **Rebuild ready** (the ordinary
case on `develop`, where `VERSION` is bumped by hand per ADR-039), and a
channel that has fallen behind reads **Behind** and offers nothing — so
following `main` while running a `develop` build never invites a downgrade.
Clicking it downloads the build,
verifies it against the release's published `SHA256SUMS` (the same check
you'd run by hand below, just automated) — and, once a release signing key is
compiled in, checks the publisher signature on that `SHA256SUMS` first (see
[Release signing](#release-signing)) — swaps it into place, and restarts
`yggd`. This requires `Restart=always` in the
systemd unit (the example above already has it); without it, the binary
still gets installed but nothing brings the process back up, and the page
says so.

**Moving an existing install to another channel from the command line** is
the same script, with the channel named explicitly:

```bash
sudo bash install.sh --role control-plane --channel develop
```

That downloads and verifies the develop build, saves the build you were
running into the per-channel archive (so the Updates page can offer to
restore it), rewrites `update_channel` to match, and restarts. Naming the
channel is what makes it a switch — re-running with no `--channel` follows
whatever the config already says, so an ordinary update never moves a host
between channels by accident.

Both binaries report what they are with `--version`, which is what the script
reads to label the saved build:

```bash
/var/lib/yggdrasil/bin/yggd --version
```

**Switching channels from the page is safe to undo.** Each non-watched channel
has a **Switch to** button. Before moving, the build you are running is copied
into a per-channel archive (`<data_dir>/binary-archive/`), filed under the
channel it came from — so switching `main` → `qa` to try something, then
switching back, offers a **Restore** button that reinstalls your exact `main`
build from disk. No download, and it does not matter whether that release is
still published. Restoring saves the outgoing build the same way, so you can
bounce between channels freely. An ordinary update archives the build it
replaces too, so that is a rollback point as well. The switch is written back
to `yggd.yaml` for you, so it survives a restart.

**Rolling back to a build older than 0.64 needs your secret settings
re-entered.** From 0.64 secret settings are stored encrypted (`enc:v1:...`,
under `pki/settings.key`). An older build does not know that format and reads
the stored ciphertext as the literal value, so sign-in providers, the Steam key
and the seed publish token stop working on it until you enter them again on its
Settings page. Everything else — accounts, sessions, servers — keeps working
across the rollback, and moving forward again re-encrypts whatever was
re-entered.

`update_channel` is unset by default — this is opt-in, and the page's off state
has buttons to start watching a channel (and a link to stop again) rather than
requiring a config edit. Leave it unset and update manually instead:

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

Signed, UI-triggered updates (ADR-015, ADR-040, ADR-041) are built, and live
on the same **Updates** page. Its **Agents** section lists the same three
channels with the `ygg-agent` version each publishes, and a per-architecture
button to **fetch** that build straight from the public releases repo; below
it, an **upload** form takes one you built yourself
(`make build-agent-linux`) for a custom or patched build, or for a control
plane with no outbound access to GitHub (ADR-055). Either way the control
plane verifies it — a fetched binary against that release's published
checksum, the same trust level `yggd`'s own self-update already uses — signs
it with its own key, and lists it.

The **Nodes** table on that same page then shows each node's running version
against the newest registered binary for its architecture, with an **Update
agent** button wherever the registered one is genuinely ahead (the same
newer/rebuild/behind rule the control plane section uses). Clicking it tells
that node to download, verify, install, and restart itself. Progress and the
eventual result (or failure reason) show inline. The Nodes page carries the
same button on each row as a shortcut. Nothing else on that host is touched —
Docker owns the container lifecycle, not the agent (ADR-012), so a restarting
agent never stops a game server.

The control plane's own signing key never leaves it (ADR-040): a node pins
that key, and this control plane signs every binary it registers, fetched or
uploaded. A fetched binary is checked first against the release's
`SHA256SUMS` and — once a release signing key is compiled in — against the
publisher signature on that `SHA256SUMS` and on `releases.json` (see
[Release signing](#release-signing)), so the control plane only ever vouches
for bytes the project's own CI built. A fetched binary's version is
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

## Release signing

A release carries `SHA256SUMS.sig` and `releases.json.sig`: ed25519 signatures,
made in CI, over the release's checksum manifest and its version manifest
(ADR-183). `yggd` checks them before a self-update or an agent-binary fetch,
against public keys compiled into the binary
(`internal/shared/releasesig/keys.go`). Without them, an update is trusted
because it matches a `SHA256SUMS` published in the same release — so anyone
able to publish a release could ship a matching binary and checksum together.

**Until a key is compiled in, nothing is checked**: updates work exactly as
before, and `/updates` and the startup log both say signatures are not
checked. Once one is, `yggd` refuses an update or a fetched agent binary whose
signature is missing or invalid and says which file and why — and the release
workflow refuses to publish without a signature, so a signed channel cannot
quietly go unsigned.

### Creating the key (once)

This is done by whoever maintains the source repository's CI. The private key
only ever exists on your machine and in the Actions secret; never commit it.

1. Generate a keypair from a checkout of the source repository:

   ```bash
   go run ./hack/releasesign keygen
   ```

   It prints a public key (a quoted base64 string) and a private key (an
   unquoted base64 string). Nothing is written to disk. Keep an offline copy
   of the private key somewhere safe — losing it means rotating to a new one.

2. Store the private key as an Actions secret on the **source** repository
   (the one whose `release.yml` publishes, not the public release repository),
   **before** step 3:

   ```bash
   gh secret set YGG_RELEASE_SIGNING_KEY --repo arasoi/yggdrasil
   # paste the private key at the prompt
   ```

   From then on releases are signed, though nothing checks them yet: the
   workflow logs a warning saying so.

3. Add the public key to `trustedKeys` in
   `internal/shared/releasesig/keys.go`, exactly as `keygen` printed it:

   ```go
   var trustedKeys = []string{
       "BASE64PUBLICKEY=",
   }
   ```

   Commit it on a feature branch and merge it the ordinary way. From the
   commit that adds it onward, the release workflow fails rather than publish
   unsigned, and every `yggd` built from it enforces signatures.

4. Update each control plane once. The update that *installs* the first
   enforcing build is itself checked by the build being replaced, which has
   no key, so it is trusted on the checksum alone, as before; every update
   after it is checked. To check that first one by hand, download the release
   and run, from a checkout at the commit that added the key:

   ```bash
   go run ./hack/releasesign verify -tag release-latest SHA256SUMS releases.json
   ```

   (substituting the tag of the channel you downloaded), which checks
   `SHA256SUMS.sig` and `releases.json.sig` beside those files against the
   compiled-in keys — then `sha256sum -c SHA256SUMS --ignore-missing` as usual.

### Rotating the key

`keys.go` may list several public keys, and the secret may hold several
private keys separated by whitespace; a release is then signed once per key,
one line each in the `.sig` file, and accepted by any build trusting any one
of them.

1. `go run ./hack/releasesign keygen` for the new pair.
2. Set the secret to *both* private keys, old then new, separated by a
   newline: `gh secret set YGG_RELEASE_SIGNING_KEY --repo arasoi/yggdrasil`
   and paste both.
3. Add the new public key to `trustedKeys` alongside the old one, merge, and
   update every control plane.
4. Once every control plane in use runs a build that trusts the new key,
   remove the old public key from `trustedKeys` and the old private key from
   the secret.

If a private key leaks, do the same but remove the leaked public key in the
same commit that adds the new one, and set the secret to the new key alone;
a control plane still on a build that trusts only the leaked key needs one
update checked by hand (step 4 above) to cross over.
