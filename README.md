# Yggdrasil — release binaries

[![Develop version](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Farasoi%2Fyggdrasil-releases%2Fmain%2Fbadges%2Fdevelop-latest-version.json)](https://github.com/arasoi/yggdrasil-releases/releases/tag/develop-latest)
[![Develop updated](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Farasoi%2Fyggdrasil-releases%2Fmain%2Fbadges%2Fdevelop-latest-updated.json)](https://github.com/arasoi/yggdrasil-releases/releases/tag/develop-latest)
[![QA version](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Farasoi%2Fyggdrasil-releases%2Fmain%2Fbadges%2Fqa-latest-version.json)](https://github.com/arasoi/yggdrasil-releases/releases/tag/qa-latest)
[![QA updated](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Farasoi%2Fyggdrasil-releases%2Fmain%2Fbadges%2Fqa-latest-updated.json)](https://github.com/arasoi/yggdrasil-releases/releases/tag/qa-latest)
[![Release version](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Farasoi%2Fyggdrasil-releases%2Fmain%2Fbadges%2Frelease-latest-version.json)](https://github.com/arasoi/yggdrasil-releases/releases/tag/release-latest)
[![Release updated](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Farasoi%2Fyggdrasil-releases%2Fmain%2Fbadges%2Frelease-latest-updated.json)](https://github.com/arasoi/yggdrasil-releases/releases/tag/release-latest)

> The QA and Release badges above only populate once something has actually
> been promoted to the `qa`/`main` branches of the source repository — until
> then they'll show as invalid/"not found". The Develop badges are live now.

**Yggdrasil** (the world tree) manages and launches containerised game servers
across one or more physical hosts. It's two binaries:

- **`yggd`** — the control plane. One instance, owns all persistent state,
  serves the web UI and JSON API, and is the only place you interact with the
  system.
- **`ygg-agent`** — one per host running game servers. Talks to Docker (or a
  Docker-API-compatible daemon such as Podman) to actually run containers, and
  reports back what's really running.

Game definitions ("what to install, how to run it, what ports it needs") are
called **seeds** — plain YAML, not code, so adding a new game is a data
change. A handful of seeds ship out of the box, including Minecraft (Java and
Bedrock), Valheim, ARK: Survival Ascended, and a synthetic multi-container
example.

The control plane never has to be reachable from the internet directly — a
[Cloudflare Zero Trust tunnel][cloudflare-tunnel] (or similar) in front of it
is enough, and agents only ever need to reach the control plane over the LAN.

This repository holds nothing but **published binaries and their supporting
docs** — no source code. It exists because the actual Yggdrasil source
repository is private; this is the public artifact repository a fresh install
downloads from. See [Where this comes from](#where-this-comes-from) below.

[cloudflare-tunnel]: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/

## Release channels

Every push to the source repository's `develop`, `qa`, or `main` branch
rebuilds both binaries and republishes them here, under a **floating tag** for
that channel — the tag is overwritten on every push, so it always points at
that branch's current tip rather than a fixed version:

| Channel | Tag | Stability | Use it if… |
|---|---|---|---|
| **Develop** | [`develop-latest`](https://github.com/arasoi/yggdrasil-releases/releases/tag/develop-latest) | Bleeding edge, prerelease | You want the newest features and are fine with occasional breakage |
| **QA** | [`qa-latest`](https://github.com/arasoi/yggdrasil-releases/releases/tag/qa-latest) | Being validated, prerelease | You want something more settled than `develop` but ahead of a full release |
| **Release** | [`release-latest`](https://github.com/arasoi/yggdrasil-releases/releases/tag/release-latest) | Stable | You're running this for real and want the least churn |

Because the tag floats, `git describe`-style version strings aren't reliable
here — each release's title and the `version.json` badge payload carry the
actual `<VERSION>-<commit>` string that was built, which is the one to quote
if you ever need to report a bug.

## What's in a release

Every release (for every channel) carries the same set of assets:

| Asset | What it is |
|---|---|
| `yggd-linux-amd64`, `yggd-linux-arm64` | The control plane binary |
| `ygg-agent-linux-amd64`, `ygg-agent-linux-arm64` | The node agent binary |
| `SHA256SUMS` | Checksums for every binary in the release — verify before running anything you downloaded |
| `yggd.example.yaml`, `agent.example.yaml` | Example config files to copy and edit |
| `install.sh` | A script that automates the manual steps below (see [Scripted install](#scripted-install)) |
| `releases.json` | A small machine-readable manifest of this release's URLs, for tooling |
| `version.json`, `updated.json` | The badge payloads used above — also fetchable directly if you want to script against them |

Only `linux/amd64` and `linux/arm64` are built. Production Yggdrasil targets
Linux specifically — `ygg-agent` needs systemd (for supervised restarts) and a
container daemon, both Linux-only requirements.

## Prerequisites

- A Linux host (systemd-based) for both `yggd` and every node running
  `ygg-agent`.
- Docker Engine (or a Docker-API-compatible daemon, e.g. Podman) on every node
  that will run game servers.
- Nothing else — both binaries are statically linked, so there's no runtime
  or interpreter to install alongside them.

## Scripted install

The fastest path. `install.sh` automates everything below: downloading and
checksumming the right binary, creating a system user, writing a systemd
unit, and — for a node — running enrollment.

```bash
curl -fsSL https://github.com/arasoi/yggdrasil-releases/releases/download/release-latest/install.sh -o install.sh
chmod +x install.sh

# Control plane:
sudo ./install.sh --role control-plane

# Node (paste the enrollment command the control plane's UI gave you,
# or pass it non-interactively with --enroll-cmd):
sudo ./install.sh --role node
```

Swap `release-latest` for `develop-latest` or `qa-latest` to install from a
different channel. Re-running the script is safe and is how you pick up a
newer build later.

## Manual install

If you'd rather do it by hand, or need to customize something the script
doesn't cover, see [`docs/installation.md`](docs/installation.md) — the full
walkthrough: downloading and verifying a binary, creating a system user,
writing the config file, setting up the systemd unit, and enrolling a node.

For a deeper look at how the pieces fit together (control plane vs. agent,
storage layout, the update model, security model, and so on), see
[`docs/architecture.md`](docs/architecture.md).

## Updating

- **`ygg-agent`** updates itself: once a node is enrolled, the control plane's
  UI shows an "Update" button per node whenever a newer signed binary is
  registered for its architecture. Clicking it never stops a running game
  server — that's a core design property, not an incidental one.
- **`yggd`** can update itself too, if you set an `update_channel` in its
  config pointing at one of the channels above; otherwise, replace the binary
  by hand and restart the service — the same download-and-checksum steps as a
  fresh install, just skipping enrollment.

Either way, `install.sh` re-run against the channel you're tracking will also
pick up the newest build.

## Where this comes from

The Yggdrasil source code lives in a private repository, so it isn't linked
here — a public visitor can't reach it anyway, and a broken link is worse
than no link. Everything a user actually needs is mirrored into *this* public
repository automatically by CI on every push to the source repo's `develop`,
`qa`, or `main` branch:

- The binaries and checksums above.
- [`docs/architecture.md`](docs/architecture.md) and
  [`docs/installation.md`](docs/installation.md), kept in sync with the
  source repository automatically — if you spot something out of date here,
  it'll correct itself on the next push, no action needed on this repo.
- The version/updated badges at the top of this page, generated fresh on
  every build.

If you have access to the source repository, the workflow that produces all
of this is `.github/workflows/release.yml` there.
