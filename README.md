# Yggdrasil

**Yggdrasil** manages and launches containerised game servers across multiple
physical hosts. One control plane (`yggd`) owns all persistent state and
serves a web UI; one agent (`ygg-agent`) runs per host and translates control
plane intent into Docker (or Podman) operations.

- A control plane outage never stops a running game server.
- An agent restart never stops a running game server.

This repository holds only the **public release assets** — prebuilt Linux
binaries, checksums, an install script, and a machine-readable release
manifest — published automatically from every push to the source
repository's three permanent branches. There is no source code here; see
**[arasoi/yggdrasil](https://github.com/arasoi/yggdrasil)** for that,
including full architecture and setup documentation
([`docs/architecture.md`](https://github.com/arasoi/yggdrasil/blob/main/docs/architecture.md),
[`docs/installation.md`](https://github.com/arasoi/yggdrasil/blob/main/docs/installation.md)).

## Release channels

Every push to `develop`, `qa`, or `main` in the source repository rebuilds
both binaries and publishes (or overwrites) one release here. Each channel
tag is a **moving target** — pushing to that branch replaces the release
rather than creating a new one, so a tag today is not necessarily the same
bytes as that tag yesterday.

| Branch | Release tag | Meaning |
|--------|-------------|---------|
| `develop` | [`develop-latest`](../../releases/tag/develop-latest) | Newest merged work. Moves often, least tested. Use for trying new features early or contributing. |
| `qa` | [`qa-latest`](../../releases/tag/qa-latest) | Promoted from `develop` for testing before `main`. A reasonable middle ground if `develop` feels too unstable. |
| `main` | [`release-latest`](../../releases/tag/release-latest) | Production-ready. Also marked "Latest" on this repo's Releases page. **Use this unless you have a reason not to.** |

Each release includes:

| Asset | Purpose |
|-------|---------|
| `yggd-linux-amd64`, `yggd-linux-arm64` | Control plane binary |
| `ygg-agent-linux-amd64`, `ygg-agent-linux-arm64` | Node agent binary |
| `SHA256SUMS` | Checksums for every asset above, for verifying your download |
| `yggd.example.yaml`, `agent.example.yaml` | Config templates for steps 2 and 3 below |
| `install.sh` | Automated installer (see "Quick install") |
| `releases.json` | Machine-readable manifest: channel, version, and every asset URL — for scripting or an auto-update client |

`yggd` and `ygg-agent` log their actual version on startup as
`<version>-<short commit>` regardless of which channel they came from — two
binaries built from the same commit report the same version even if
downloaded from different release tags.

## Prerequisites

Production targets **Linux** for both binaries.

- **Control plane host** — needs nothing but a filesystem for its SQLite
  database. No container runtime required.
- **Node host(s)** — a container runtime speaking the Docker Engine API
  (Docker Engine or Podman's compatibility endpoint), and systemd to
  supervise and restart the agent if it ever exits. `steamcmd` on `PATH` is
  only needed if you use a seed whose install method requires it (e.g. ARK
  Survival Ascended, Valheim).

## Quick install

`install.sh` automates enrollment for either role in one run — download it
and pick your channel:

```bash
# Control plane
curl -fsSL https://github.com/arasoi/yggdrasil-releases/releases/download/release-latest/install.sh -o install.sh
sudo bash install.sh --role control-plane

# Node (paste the enroll command shown on the control plane's Nodes page,
# or omit --enroll-cmd and the script will prompt for it)
sudo bash install.sh --role node --control-addr yggd.lan:8443 \
  --enroll-cmd "ygg-agent enroll --control-addr yggd.lan:8443 --token <token> --ca-fingerprint <fingerprint>"
```

Substitute `develop-latest` or `qa-latest` for `release-latest` in the URL
above to install from a different channel — pass `--channel develop` (or
`qa`) to `install.sh` as well, so re-running it later to pick up updates
tracks the same channel. Run `install.sh --help` for every flag.

The script verifies its download the same way a manual download does below
— HTTPS plus this release's published `SHA256SUMS`, no separate signing key
— and is safe to re-run to pick up a newer build, though it will not
reconcile a config file you've since hand-edited beyond the fields it
originally set.

## Manual install

If you'd rather not run a script, or need to customize beyond its flags,
grab the binaries directly:

```bash
# Pick a channel: develop-latest, qa-latest, or release-latest
gh release download release-latest --repo arasoi/yggdrasil-releases --clobber
sha256sum -c SHA256SUMS --ignore-missing
chmod +x yggd-linux-amd64 ygg-agent-linux-amd64
```

Or without `gh`, from this repo's Releases page directly. From here, follow
the source repository's
**[step-by-step installation guide](https://github.com/arasoi/yggdrasil/blob/main/docs/installation.md)**
for setting up the control plane, enrolling a node, and updating later —
this README only covers getting the right binary; that document covers
everything after.

## Updating

- **`yggd`** can watch its own release channel and self-update from the
  UI's Updates page (set `update_channel` in its config to `develop`, `qa`,
  or `main`), or be replaced manually — restarting it never stops a running
  game server.
- **`ygg-agent`** updates are pushed from the control plane's UI once you
  upload a binary for it to sign, or can be replaced manually the same way.
  Restarting the agent never stops a running game server either — Docker
  owns the container lifecycle, not the agent.

See the source repo's
[`docs/installation.md`](https://github.com/arasoi/yggdrasil/blob/main/docs/installation.md#updating)
for the full detail on both paths.

## Source

All code, architecture docs, and issue tracking live at
**[arasoi/yggdrasil](https://github.com/arasoi/yggdrasil)**. This repository
is generated content only — its `README.md`, `install.sh`, and
`releases.json` are overwritten by
[`.github/workflows/release.yml`](https://github.com/arasoi/yggdrasil/blob/main/.github/workflows/release.yml)
in the source repository on every push, except this file itself, which is
maintained by hand.
