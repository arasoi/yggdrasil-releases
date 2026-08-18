# Architecture

> **Yggdrasil** — the world tree. Binaries: `yggd` (control plane), `ygg-agent` (node agent).
> Game definitions are called **seeds**.
>
> Update this document as the system evolves.

## Overview

Yggdrasil manages and launches containerised game servers across multiple physical hosts. It has
two deployable components:

- **Control plane (`yggd`)** — one instance. Owns all persistent state, serves the
  web UI and JSON API, and is the single place a human interacts with the system.
- **Agent (`ygg-agent`)** — one per physical host. Translates control plane intent into
  Docker operations and reports what is actually happening back upstream.

The control plane is authoritative for **intent** (what should be running, where, with what
configuration); each agent is authoritative for **reality** (what is actually running now).

Two properties follow from that split and constrain everything below:

1. **A control plane outage never stops a game server.** Agents keep supervising.
2. **An agent restart never stops a game server.** Docker owns the container lifecycle;
   the agent is a controller that can be replaced underneath running workloads.

```
              Internet                          LAN
                 │
      Cloudflare Zero Trust tunnel
                 │
   Browser ──────┴─────────┐
                           ▼
                ┌───────────────────────┐
                │   Control plane       │   Go single binary
                │   yggd                │   HTTP+UI, gRPC server,
                │                       │   SQLite, scheduler, jobs
                └───────────┬───────────┘
                            │
             persistent mTLS gRPC bidi stream
             (the AGENT always dials out, over LAN)
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │  ygg-agent   │  │  ygg-agent   │  │  ygg-agent   │
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
          │ Docker API      │                 │
   ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐
   │ pods / vols  │  │ pods / vols  │  │ pods / vols  │
   └──────────────┘  └──────────────┘  └──────────────┘
      host: odin        host: thor        host: loki
```

## Network topology

Agents connect to the control plane over the LAN. The Cloudflare Zero Trust tunnel fronts
the web UI only — it is not in the agent control path, which keeps end-to-end mTLS intact
(Cloudflare terminates TLS at its edge, so an agent connecting through a public hostname
could not present a client certificate to the origin). See ADR-011.

Two consequences for the UI: WebSocket works through the tunnel and carries both console
output (ADR-032) and live state updates (ADR-058) on two separate sockets, and any long-lived
HTTP stream needs a keepalive roughly every 20 seconds to survive idle timeouts — both sockets
ping every 15.

Remote agents outside the LAN are not supported in v1. When they are needed, the intended
path is Cloudflare private network routing or a WireGuard overlay — both transparent at the
TCP level, so the transport design does not change.

## Entity model

Grouping game servers together has three independent axes, and conflating them produces a
model where every operation must branch on what kind of group it is dealing with. They are
kept separate and compose freely (ADR-017, ADR-018):

| Axis | Concept | Solves | Example |
|------|---------|--------|---------|
| Disk | **Install** | N servers, one copy of the game files | Five ARK maps, one 50 GB install |
| Game state | **Cluster** | N servers sharing a volume and cluster identity | ARK character transfer between maps |
| Composition | **Pod** | One server built from N containers | Dune Awakening: game + database + broker |

```
Seed  (data: how to install, run, and configure a game)
    │
    ├──► Install  (materialised game files on a node, refcounted)
    │        │
    │        ├──► Server A  ─┐
    │        ├──► Server B  ─┼──► Cluster  (shared volume + cluster ID)
    │        └──► Server C  ─┘
    │
    └──► Server D  (no shared install — the image contains everything)
             │
             └──► pod: [ game (primary), postgres, rabbitmq ]
```

**Node** — a host running an agent. Reports capacity, CPU architecture, kernel release,
Docker version, and agent version, all at every handshake rather than once at enrollment.
May carry an operator-assigned port range (ADR-048); unset means every port on it allocates
from the global default range instead. It also carries **where it can be reached** — the peer
address observed when its agent last dialled in, plus an optional operator override for when
that is not where players connect (ADR-065). `/nodes/{id}` shows all of it and is where the
override, the port range, agent update and removal live.

**Install** — game files materialised once on a node and mounted into every server that
references it. Refcounted; cannot be deleted while referenced. Optional: seeds whose
container image already contains the game have no separate install.

**Server** — the managed unit. What you start, stop, back up, and open a console on.
**Every server is a pod of one or more containers**, with exactly one marked primary.
Single-container servers are pods of one — there is no special case (ADR-017).

**Cluster** — a named group of servers sharing a cluster volume and a cluster ID. Servers
in a cluster remain independently startable; the cluster is shared state, not shared
lifecycle.

**Allocation** — a reserved `(ip, port, protocol)` tuple on a node. First-class rows, since
game servers are port-hungry and split across TCP and UDP. Only containers that declare
ports get them; sidecars are usually internal-only. A seed's ports are either independently
allocated — always the next free port in the node's range, never the seed's declared default,
since a node's range is the operator's statement of which ports work on that host (ADR-061) —
or derived from another port on the same container by a fixed offset, which Valheim's Steam
query port, hardcoded to `game_port + 1` with no way to configure it separately, is what
exists for (ADR-048). A base port and its offset siblings — including a same-number sibling on
a different protocol, how a seed asks for one number on both TCP and UDP (Vintage Story) —
allocate as one atomic group (`store.AllocatePortGroup`, ADR-083): a base candidate is accepted
only once every sibling's derived number is confirmed free too, so a collision moves the whole
group to the next candidate rather than stranding one member with the other already claimed.

The `ip` in that tuple is the **bind** address — always `0.0.0.0` — and part of the row's
unique key. It is not a connect address, and was shown as one until ADR-065: a server's page
read `0.0.0.0:30055/udp`, which is correct about binding and useless to a player. What an
operator hands out is the *node's* address joined to the allocated port, resolved at render
time; a node whose address is not known yet shows the port alone rather than a host nobody
can reach.

**A container always publishes a port to itself**: host and container port are the same
number, because the game binds whatever the seed's command told it to bind and that value is
the allocated port. A seed whose image starts the server itself, and so never reads the seed's
command, has to be handed the port through env instead — Bedrock's `SERVER_PORT`. Getting this
wrong is silent: the server starts, logs nothing wrong, and refuses connections (ADR-061).

**Job** — a long-running operation with streamed progress: install, install update, backup,
restore, agent update. One mechanism rather than four (ADR-021).

**An install stuck by a control-plane restart self-heals**, up to a small bounded number of
attempts, and stays repairable by hand past that (ADR-106). `ReconcileInterruptedInstalls`
(ADR-021, ADR-101) still runs once at startup, but rather than marking an interrupted install
`failed` outright, it leaves one with at least one server still waiting on it ready to retry —
resuming is sound because SteamCMD's own `app_update ... validate` reconciles a partial download
in place (ADR-091) rather than re-fetching from zero. Nothing is dispatched at that call site,
since no node has connected yet at that point in `cmd/yggd`'s startup; the retry itself is
dispatched once the install's node reconnects. An install genuinely reported failed by its
agent — a real error, not an interruption — is never touched by self-heal at all, only by the
**Retry** action on `/installs` and on a crashed server's own page, which carries no attempt cap.

## Storage layout

```
/var/lib/yggdrasil/installs/<install-id>/                # game files, mounted read-only, refcounted
/var/lib/yggdrasil/servers/<server-id>/                  # per-server writable state (the primary's)
/var/lib/yggdrasil/servers/<server-id>/volumes/<role>/   # a sidecar's own named volumes (ADR-019)
/var/lib/yggdrasil/clusters/<cluster-id>/                # shared cluster volume
/var/lib/yggdrasil/backups/
/var/lib/yggdrasil/agent/                                # agent binary, certs, bootstrap state
```

An install is mounted **read-only** into every server that references it. Paths the game
must write to are bind-mounted over the read-only layer from the server's own directory:

```
/var/lib/yggdrasil/installs/ark-asa-1/                  → /game                  (ro)
/var/lib/yggdrasil/servers/<uuid>/saved/                → /game/ShooterGame/Saved (rw)
/var/lib/yggdrasil/servers/<uuid>/config/               → /game/.../Config        (rw)
/var/lib/yggdrasil/clusters/<uuid>/                     → /cluster                (rw)
```

Docker handles nested bind mounts natively, so this needs no overlayfs and no privileged
mounts. Seeds declare which subpaths are per-server writable.

Some games offer native equivalents — ARK's `-AltSaveDirectoryName` separates saves within
one install without any mount trickery. Seeds can use either mechanism; the bind-mount
path is the general fallback that works for games offering nothing.

**Installs are never backed up.** They are reproducible from the seed, and excluding
them keeps backups proportional to actual game state rather than to disk footprint (ADR-023).

**Every directory above has an owner that removes it** (ADR-088). Until then none of them did:
`Runtime.Destroy` removes containers and the pod network and never a host directory, there was no
route to delete an install at all, and nothing anywhere removed a backup archive — so a fleet with
a scheduled backup (ADR-045) accumulated them for as long as it had been running.

One agent-side command, `ReclaimData`, frees a server directory, an install directory or a backup
archive. It is shaped exactly like `DeleteClusterVolume` (ADR-066) and for the same reason: the
file sandbox is rooted at one *server's* directory (ADR-033) and none of these sit inside such a
root, so it gets its own narrow message whose id is parsed as an id of the matching kind before it
can name a path. Deleting a server frees its directory and its archives, a move frees the source
node's copy, deleting an install frees its files once nothing references it, and the scheduler
prunes archives past `backups.retention_days`.

**Which install has an owner depends on whether it is shared** (ADR-096). A seed declaring
`shared: false` gets a new install per server that nothing else can ever reference, so its
lifetime is that server's and it is removed with it — on delete, on a move (the install does not
travel, so the source's copy is immediately unreferenced), and on a failed creation. A *shared*
install at refcount zero is deliberately kept: it is the cached depot the next server from that
seed reuses, and removing it would turn deleting one of five ARK maps into a 50 GB re-download.
That one stays the operator's, on `/installs`, where its server count already reads zero. The
distinction was missed when the list above was drawn up, and a per-server install stranded 19 GB
on every ordinary delete.

**Reclamation is best-effort and nothing depends on it.** A pre-protocol-4 agent ignores the
command, so every caller logs a failure and carries on — refusing to delete a server because its
disk could not be freed would leave an operator with a row they cannot remove. The cost is an
orphaned directory an operator can still remove by hand.

Ordering differs by path, deliberately. A server's archives are found by listing rows its own
deletion then cascades away, so those are reclaimed *before* the row. An install's id and node are
already known, and its row delete is what refuses while a server still references it — so there the
row goes first, because reclaiming before a refusal wipes an install that is still mounted into
running containers.

An agent also sweeps runtime leftovers at startup, after adoption: an exited one-shot installer
(ADR-057) and a pod network with nothing attached. Deliberately nothing else — it never removes a
container merely because the control plane did not mention it, since ADR-031's reasoning about
acting on incomplete information applies with full force to deletion.

The block above is a node's layout, under the *agent's* `data_dir`. The *control plane's* own
`data_dir` is a separate tree:

```
/var/lib/yggdrasil/pki/ca.{crt,key}          # mTLS certificate authority (ADR-028)
/var/lib/yggdrasil/pki/signing.key           # binary-signing key, a distinct trust role (ADR-040)
/var/lib/yggdrasil/agent-binaries/<sha256>/  # signed agent binaries, content-addressed (ADR-040)
/var/lib/yggdrasil/binary-archive/<channel>  # one saved yggd build per release channel (ADR-056)
/var/lib/yggdrasil/yggd.db                   # SQLite (ADR-005)
```

## Execution model

All game servers run in containers (ADR-010). The `Runtime` interface is retained as a seam
with one implementation, which targets the Docker Engine API rather than Docker specifically —
any daemon speaking that API works, and the conformance suite is what defines "works"
(ADR-030):

```go
// Runtime executes and supervises pods on the local host.
// Implementations must be safe for concurrent use across different server IDs.
type Runtime interface {
    Provision(ctx context.Context, spec PodSpec) error
    Start(ctx context.Context, id ServerID) error
    Stop(ctx context.Context, id ServerID, spec StopSpec) error
    Kill(ctx context.Context, id ServerID) error
    Attach(ctx context.Context, id ServerID, role string) (Console, error)
    Stats(ctx context.Context, id ServerID) (map[string]Stats, error)
    Status(ctx context.Context, id ServerID) (PodStatus, error)
    Destroy(ctx context.Context, id ServerID) error
    Adopt(ctx context.Context) ([]AdoptedPod, error)  // rebuild state from labels
}
```

`PodSpec` carries one or more `ContainerSpec` entries, their dependency edges, mounts, and
allocations. Console attach and stats are role-addressed because a pod has more than one
container to look at.

**Graceful stop is game-specific.** Most game servers want a console command (`stop`,
`quit`, `exit`) rather than a signal, and losing that distinction corrupts world saves.
`StopSpec` carries `{ConsoleCommand, Timeout, ThenSignal}`: write the command to the primary
container's stdin, wait, escalate to SIGTERM, then SIGKILL.

## Pods

Every server is a pod. Most are pods of one; the mechanics below apply uniformly.

**Networking.** Each pod gets a private bridge network named `ygg-pod-<server-id>`, with
each container aliased by its role. The game reaches its database at `postgres:5432`. Only
containers declaring ports publish to the host. A shared network namespace was considered and
rejected — it prevents restarting a sidecar without the primary and turns sidecar port
collisions into an operator problem (ADR-019).

**Start order.** Containers declare `depends_on` with a `started` or `healthy` condition, plus
an optional healthcheck. The agent starts in topological order, waiting on health gates, and
fails the pod with a clear reason on timeout rather than letting the game crash-loop against
an unready database.

**Stop order is the reverse**, and gets the primary first: graceful-stop the game so it
flushes its writes, *then* tear down the database. The opposite order loses saves.

**Sidecar failure.** A crashed sidecar is restarted with backoff. Exceeding the crash-loop
threshold puts the pod in `degraded` — the primary may still be running, but the server is
not healthy, and that is operationally distinct from both `running` and `crashed`.

**Sidecars are observable, not independently controllable** (ADR-022). Their logs and
resource stats surface in the UI behind a container selector on the console view; they have
no independent start/stop. The pod owns the lifecycle, so the dependency graph cannot be
violated by hand.

## Clusters

A cluster gives its members a shared volume and a cluster ID. The seed declares how the
game consumes both — for ARK, a `-clusterid=` argument and a `-ClusterDirOverride=` path
pointing at the mounted shared volume.

Cluster members are peers: independently startable, each with its own console, allocations,
configuration, and backups. Fan-out operations ("restart every map") are a convenience over
the members, not a lifecycle the cluster owns.

**Clusters are node-local in v1**, since the shared volume is a local directory. The volume
is nonetheless modeled as a storage backend (`local` now, `nfs` or similar later) with the
cluster's node binding nullable, so cross-node clustering becomes a new backend rather than a
schema change (ADR-020).

Clusters are a first-class `clusters` table with a globally-unique name (ADR-037) and a
recorded `seed_id` naming the game they are for (ADR-102) — the fact everything else about
clusters turned out to need. **A cluster is created from `/clusters`, on its own**, before any
server joins it: a name, a seed filtered to those declaring `cluster.supported`, and a node.
Creating one writes only the row; the shared volume is a directory `ensureMountSources` creates
when the first member's containers are actually built (ADR-018), and the page says so rather
than implying the volume already exists. A server joins from a dropdown listing every cluster
for the seed it is being created from, each option naming its node — a mismatch is refused with
a message naming the right node — and an existing server joins or leaves from the cluster's own
page or from `/clusters`, both offering a select of *eligible* servers: same seed, same node,
not already in a cluster.

`/clusters` lists every cluster with its node and members; `/clusters/{id}` is the cluster
itself (ADR-080) and is where one is managed (ADR-066): rename it, add or remove a member, back
it up, set its schedule, or delete it. It needs no new query beyond the eligible-member list —
the same data the list already gathers for every cluster, gathered for one — and shows its
members as the same art cards the fleet uses, since a cluster's members are peers with their own
pages. Its overflow menu also reaches `/clusters/{id}/settings` and `/clusters/{id}/files`
(ADR-105) — its values and its shared volume, the two things that exist at the cluster level
rather than on any one member.

Every operation there is guarded by something the schema does not enforce, because
`servers.cluster_id` has no foreign key: deleting a cluster is refused while it still has
members (the row would otherwise vanish underneath every one of them, still mounting its
volume), and while a cluster backup or restore is in flight. Removing a member clears the
column *and* rebuilds that server's pod, since the `/cluster` mount and `-clusterid=` argument
are fixed into the container at create time and the column alone changes nothing the game sees.

**A cluster also carries `variables` and `settings`, the same shape a server's own stored
values take, that its members inherit** (ADR-103). Nothing is inherited unless the cluster
deliberately manages a name — its two columns start empty, and a name neither the cluster nor
a member manages falls through to the seed's ordinary default exactly as it always has. One
function, `storedFor`, composes the two layers (a server's own values winning name for name)
and every place that used to build a server's resolved variables and settings directly — the
pod spec, config files, the connect block, a backup's hooks, an install update, settings
import's comparison — now routes through it, so there is one place the composition can be
gotten right rather than six.

`/clusters/{id}/settings` is where an operator sets what the cluster manages; a member's own
Settings page shows every overridable control with a `manage_<name>` checkbox beside it —
checked means this server's own value wins and is written to its own row, unchecked means the
name is left out of that row entirely and the server keeps following the cluster. The identical
mechanism serves both ends: a cluster's own settings page is the same rendering with nothing
above it to inherit from, so deciding what the cluster manages at all uses the same toggle a
member later uses to claim a name back. Saving a cluster's values rebuilds every member
sequentially — the same shape a seed rollout already uses (`handleRebuildSeedServers`) and for
the same reason: five maps rebuilding on one node at once turns a routine change into a
coordinated outage — and an offline node or a failed rebuild is reported by name rather than
silently dropped.

**Joining a cluster at creation starts from full inheritance, not a snapshot of whatever the
creation form happened to show.** The creation form has no per-control toggle of its own — adding
one to every field on a form that can already run to dozens of controls would trade a default
nobody wants for a decision on every field — so `clusterCreationOverrides` decides by comparison
instead: a resolved value equal to what the server would inherit (the cluster's own value for a
name it manages, the seed's default otherwise) is left off the new row entirely, and only a value
that genuinely diverges — ARK's `map`, chosen differently per member — is stored. A server created
into a cluster therefore opens its Settings page with nothing pinned to itself except what
actually needed to be.

**The shared volume is deleted only if explicitly asked for**, as a separate checkbox on the
delete, since it is a different loss from the row — for ARK it is every character ever
transferred between those maps. It travels as its own `DeleteClusterVolume` command rather than
a file operation: the file sandbox is rooted at one server's directory and a cluster volume
lives outside every such root (ADR-033). Moving a cluster between nodes is still not possible.

## Zero-downtime updates

Updating Yggdrasil must never stop a running game server. This is a first-class requirement and
constrains the agent's design more than anything else.

### The agent holds no authoritative local state

Every container is labelled at creation:

| Label | Purpose |
|-------|---------|
| `ygg.managed` | Marks the container as Yggdrasil's, so nothing else is ever touched |
| `ygg.server.id` | Server UUID — the join key back to the control plane |
| `ygg.node.id` | Guards against a misconfigured agent adopting foreign containers |
| `ygg.pod.role` | `primary`, or the sidecar's role name |
| `ygg.install.id` | Referenced install, if any |
| `ygg.cluster.id` | Cluster membership, if any |
| `ygg.seed.id` / `ygg.seed.version` | What produced this container |
| `ygg.schema` | Label schema version, so future agents can migrate old containers |

On startup the agent enumerates labelled containers and rebuilds its world from them —
running pods, their member containers, states, and allocations. There is no local database
and no state file whose loss matters (ADR-012).

Install directories are *data*, not state: the control plane stays authoritative for "install
X exists on node Y at version Z," and any manifest on disk is an integrity check, never
authority.

Console scrollback is rebuilt from Docker's own container log buffer on each attach rather
than persisted by the agent; nothing here is a record the agent is responsible for keeping.

### Restart authority sits in exactly one place

Containers are created with Docker restart policy `no`, and the agent drives all restarts.
The agent itself runs under systemd with `Restart=always`. Splitting authority between
Docker's policy and the agent's crash-loop detection produces servers where neither component
agrees on whether they should be up (ADR-013).

### Agent updates are pushed from the UI

Built in phase 7 (ADR-040, ADR-041). On `/updates`'s Agents section (there is no separate
`/agent-binaries` page — ADR-056 folded it in), an operator either fetches a `ygg-agent` build
straight from the public releases repo (`arasoi/yggdrasil-releases`) for a chosen channel and
architecture, or uploads a custom/patched build directly (ADR-055) — either way, the control
plane signs the digest with its own key — separate from the mTLS CA, since signing artifacts and
issuing node identity are different trust roles (ADR-040) — and stores the bytes
content-addressed by sha256. A fetched binary is verified against that release's published
checksum before signing, the same checksum-only trust `yggd`'s own self-update already uses
(ADR-044); CI never holds a signing key either way. The same page's Nodes table (and, as a
shortcut, each row of the Nodes page) shows an "Update agent" button once a registered binary
for that node's architecture is genuinely ahead of what it runs — ordered by
`version.Compare`, not merely different, so a registered binary that is *behind* a node is
never offered as an update (ADR-056).

Clicking it dispatches `UpdateAgentStart` over the node's existing stream; the agent fetches
the bytes itself over a dedicated `DownloadAgentBinary` RPC on the same authenticated
connection (ADR-041 — not a separate HTTP endpoint, since only the gRPC listener is guaranteed
reachable from a node host), verifies the digest and the control plane's signature against the
public key pinned at enrollment, writes the new binary alongside the current one, and
atomically renames it into place. It then drains — stops accepting new lifecycle commands,
waits for whatever is already queued to finish, closes its stream cleanly — and exits zero.
systemd's `Restart=always` brings the process back up; it *reconnects* with its existing
certificate (no re-enrollment: identity persists on disk across the restart) and adopts its
pods from container labels exactly as any other restart does (ADR-012). Containers are
untouched throughout; console viewers see a brief reconnect. The previous binary stays on disk
at `<path>.previous` for manual rollback. A job with no successful terminal message of its own
(the process that would send one exits first) resolves when the control plane sees the node
reconnect reporting the new version, or fails after 5 minutes with no such reconnect.

Proven end-to-end against a real control plane and a real agent process, including the failure
path (a tampered checksum is rejected before anything is installed) — see
`internal/integration/agent_update_test.go`. N-1 compatibility now has a **real** range to mean
something against, which it did not when this was written: ADR-077's install-steps work took the
negotiated protocol to 2 while `MinProtocol` stayed 1, so a protocol 1 agent genuinely exists as
a case rather than a hypothesis — it ignores `InstallStart.steps`/`.image` and reads the older
method/url/archive/filename/app_id fields a current control plane keeps writing for one release.
That pairing is now proven against **real older binaries**, not only against constructed ranges.
`hack/n1-check.sh` walks `version.go`'s own history for the last commit that shipped each
protocol below the current one, builds an agent from each in a git worktree with that commit's
own `VERSION`, and pairs all of them with a current control plane at once. Every protocol from
the floor up negotiates its own version rather than the newest, and every one is flagged as
behind in the UI.

`internal/integration/n1_compat_test.go` keeps its constructed ranges and is not superseded by
this: it can pose a peer *newer* than this binary, and one too old to overlap at all, neither of
which any checkout can produce. What it cannot do is prove that an agent somebody actually
shipped still works, which is the half the script covers.

### `yggd` watches its own release channel and can self-update

`yggd` has no control plane above it, so the agent's signed-fleet-update mechanism above does
not fit it — there is nothing to hold a separate signing key and push a verified binary down.
Instead, an operator opts a running instance into update awareness by setting
`update_channel: develop|qa|main` (empty and disabled by default). The `/updates` page checks
every channel's GitHub release — cached in memory for an hour, checked on page load rather than
by a background poller (the same on-demand philosophy the "Seed-driven provisioning is
reconciled on page load" section below already uses) — and lists all three with the version each
publishes, for both binaries, marking the one this instance watches. The page says when it last
looked and carries a **Check now** button that drops the cache, since an hour is exactly the
wrong TTL in the moment after publishing a release; every other timestamp on the page is a
release's own publication time, which reads like freshness while saying nothing about how old
the answer is. An **Update now** button
appears on the watched channel once what it publishes is actually ahead of the running build
(ADR-056): a higher version number, or the same version number from a different commit (a
rebuild, the ordinary case on `develop` where `VERSION` is bumped by hand per ADR-039). A
channel that has fallen behind reads "Behind" and offers nothing. Only the watched channel can
install, since doing so replaces this running process.

**Switching channels is reversible.** Before moving, the binary this host is running is copied
into a per-channel archive under `<data_dir>/binary-archive/`, filed under the channel it
actually came from — which after a switch and before the update that follows it is *not* the
configured channel, so the archive tracks the running binary's origin separately in
`running.json`. Switch back and the page offers to restore that exact build from disk: no
download, and no dependency on that release still being published. Restoring saves the outgoing
build the same way, so the move works in both directions. An ordinary update archives the binary
it replaces too, making it a rollback point. Only the watched channel can install or restore,
since either replaces this running process; the switch itself is written back to `yggd.yaml`
(`config.SetServerUpdateChannel`) so it survives a restart rather than silently reverting.
See ADR-056.

A channel's published version comes from that release's own `releases.json` manifest, fetched
over HTTPS *without* a checksum — release.yml generates `SHA256SUMS` from `release/` before
writing the manifest into `public-assets/`, so it is not in that file and never has been. That
is correct for what it is: a version label shown in the UI and recorded alongside a registered
agent binary, never executed. Every binary is still verified against the published checksum
before anything is installed or signed.

Clicking it downloads the matching binary and the release's `SHA256SUMS`, verifies the checksum,
and atomically replaces the running binary (`internal/control/selfupdate`, the same
stage-then-rename shape as the agent's own `internal/agent/selfupdate`). There is deliberately no
second signing key here (ADR-044): the trust level is HTTPS plus a published checksum, exactly
what an operator was already told to check by hand for a manual update. Once installed, `yggd`
sends itself `SIGTERM` — reusing the graceful-drain shutdown path it already runs for a normal
signal — but only when `INVOCATION_ID` is set in the environment, systemd's marker that
`Restart=always` will actually bring the process back up. Restarting `yggd` never stops a game
server either way, the same "control plane outage never stops a game server" property the
overview above already guarantees; the only visible effect is the UI being briefly unreachable.

### Version skew is permanent, not transient

- The handshake negotiates a protocol version; the control plane supports N-1.
- Protobuf changes are additive only. Field tags are never renumbered or reused.
- Migrations are backward-compatible for one release, since agents reconnect while the
  control plane is mid-restart.
- The UI shows each node's agent version and flags nodes running N-1.

See ADR-014.

## Control channel

The agent always dials out over a persistent mTLS gRPC bidirectional stream. Commands flow
down; events, telemetry, job progress, and console frames flow up. Adding a node is a single
pasted command with no firewall configuration (ADR-002).

**Enrollment.** An operator generates a short-lived, single-use bootstrap token in the UI.
The agent presents it once and receives a client certificate signed by the control plane's
internal CA. The enrollment response records architecture, Docker version, and agent version.

**Multiplexing.** Console sessions and job progress share the one stream, framed in an
envelope carrying a stream ID. File operations share it too but are addressed by command_id
instead: a directory listing or a file's contents is a single bounded request/reply, not an
ongoing session, so the hub's `Call` blocks for the matching reply rather than opening a
sub-stream (ADR-033). One TCP connection per node either way.

**Reconnection.** Exponential backoff with jitter. On reconnect the agent sends a full state
snapshot built from container labels. While a node is disconnected its servers show as
`unknown` rather than `offline` — they are almost certainly still running.

## Credential rotation

Neither the mTLS CA nor the binary-signing key had a rotation story until ADR-052: the CA
package's own comment used to call rotation "disruptive," and closer inspection turned up why —
`Enroll` always mints a brand-new node ID, and `servers.node_id` cascades on delete, so
re-enrolling an already-enrolled node loses every one of its server records while the real
containers, labelled with the old node ID, become permanently un-adoptable (ADR-012's own
adoption rule forbids claiming them). ADR-052 solves the actual blocker — renewing a node's
credentials without losing its identity — as a primitive, then builds both rotations on it.

**`RenewIdentity`** is a unary RPC on `AgentService`, authenticated the same way
`DownloadAgentBinary` already is: the caller's *current* verified client certificate is the only
source of its node ID (`peerNodeID`), so the request carries no claimed identity to trust. It
always refreshes all three trust anchors together — node certificate, CA certificate(s), signing
public key — regardless of which rotation asked for it, which is what lets CA rotation and
signing-key rotation share one mechanism instead of two. The control plane pushes
`RenewIdentityStart` down a node's persistent stream (mirroring `UpdateAgentStart`'s shape); the
agent generates a fresh keypair, calls `RenewIdentity` on its own short-lived connection
(mirroring `DownloadAgentBinary`'s own out-of-band dial), saves the result, and reconnects with
the new credentials — a plain reconnect, not the drain-and-exit a full binary update needs, since
nothing about the running process itself changes.

**CA rotation** is a dual-trust transition, not an instant cutover, because the CA authenticates
both directions of the mTLS handshake:

1. *Begin* generates a new CA and trusts it alongside the current one for verifying incoming node
   certificates, without touching what the control plane's own server certificate presents.
   Already-connected agents, and even a fresh reconnect, are completely undisturbed.
2. Renewal is dispatched to connected nodes (on demand, no poller — the same "reconcile when
   asked" judgment ADR-035 and ADR-044 already apply elsewhere). A node that renews gets a
   certificate signed by the **pending** CA — not the current one, deliberately, since a
   certificate the outgoing CA signed would stop verifying the instant finalize narrows the trust
   pool — and a `ca_certificate` containing **both** CAs concatenated as PEM, so it can keep
   verifying the still-old-CA-signed server certificate right up until finalize actually flips it.
3. *Finalize* promotes the pending CA to current and narrows the pool back to one CA. This takes
   effect on a live `yggd` process with no restart: the server's TLS config is resolved fresh on
   every handshake via `GetConfigForClient` rather than built once, which is also what makes step
   1's dual trust take effect immediately with no second mechanism needed.
4. A node that never renews before finalize is cut off there — its certificate no longer verifies
   against the narrowed pool, and the server's newly-presented certificate no longer matches what
   it has pinned locally either. Recovery is the pre-existing `ygg-agent enroll --force` path,
   with the identity-loss consequence that already implies. *Cancel* is the mirror image: a node
   that already renewed under the pending CA needs one more renewal afterward, since its
   certificate was signed by a CA the pool no longer trusts once cancel drops it.

**Signing-key rotation** is simpler, since the key is never part of the mTLS handshake — it only
matters when a node verifies a downloaded agent binary. Rotating replaces the key immediately (no
transition window), re-signs every already-registered `agent_binaries` row under the new key from
its stored digest alone (no re-upload), and dispatches renewal to every connected node in the
same action — a node holding the old pinned key cannot verify anything signed after the rotation
until it renews.

The `/security` page is where an operator drives both: CA fingerprint and rotation status, a
per-node renewal table built by re-verifying each node's stored certificate against the current
and pending CA (`CA.Generation` — no stored renewal-status column), and the signing key's
fingerprint and one-click rotation. Proven live end to end, including a full CA rotation cycle
through finalize with the renewed node staying connected throughout, and under `go test -race`
for the concurrency the agent's own credential swap introduces. See ADR-052.

## Server state machine

```
                  ┌──────────┐
                  │ offline  │◄──────────────┐
                  └────┬─────┘               │
                       │ start               │ stop complete
                       ▼                     │
                  ┌──────────┐          ┌────┴─────┐
                  │ starting │─────────►│ stopping │
                  └────┬─────┘  stop    └────▲─────┘
                       │ ready rule matched  │ stop
                       ▼                     │
                  ┌──────────┐───────────────┘
                  │ running  │
                  └──┬────┬──┘
     primary exited  │    │  sidecar crash-looping
                     ▼    ▼
              ┌──────────┐ ┌──────────┐
              │ crashed  │ │ degraded │
              └────┬─────┘ └──────────┘
                   │ restart policy ──► starting

  installing ──► offline          unknown = agent disconnected
```

`starting` covers dependency-ordered sidecar startup and health gates before the primary is
even launched. The agent watches the Docker event stream for exits rather than polling.

## Data flow

**Control operations** — browser → `yggd` HTTP API → hub → agent stream → Docker. The API
returns as soon as the command is accepted; completion arrives asynchronously as a state
change event pushed to the browser over WebSocket.

**Long-running operations** (install, install update, backup, restore, agent update) become
Jobs. The agent streams progress up the control channel; the control plane persists job state
and relays to any watching browser. A job survives the operator closing their tab and, where
the underlying work permits, an agent reconnect (ADR-021).

**Live state** — `hub.EventHub`, the console hub's sibling, fans state changes out to every
open browser tab so a page reflects what the fleet is doing without being reloaded (ADR-058).
It publishes from wherever the control plane already learns something moved: a
`ServerStateChanged`, a `StateSnapshot`'s per-pod records, a node connecting or its stream
tearing down, each of the four job-progress handlers, and — for changes originating in the
control plane rather than at an agent — a lifecycle command's recorded intent and a server
created or deleted.

An event carries **identity, never content**: kind, server id, node id. A browser told that
server X changed re-reads the page it is already on through the ordinary handler and swaps the
regions that moved, so there is exactly one renderer (the Go templates) rather than a second
one in JavaScript that could disagree with it. Pages opt in by marking regions
(`<div data-live="servers-list">`), optionally scoped with `data-live-server` /
`data-live-node`; `static/live.js` does the fetch-and-swap, in the same shape `stats.js`
already uses for the live resource panel.

A subscriber that falls behind has its backlog collapsed into a single `resync` rather than
having events dropped, and the same `resync` is sent unprompted on every connect — so a tab
that reconnects after a laptop wakes, or after `yggd` restarts to self-update, catches up
without a second rule on the client. The socket announces itself only when it is *down*
("reconnecting…"), because a page that looks current while no longer updating is the failure
worth interrupting for.

Three things are deliberately left outside live regions: the console page (no state badge, and
its own socket already covers it), the Nodes page's one-time enrollment callout (a live re-read
would swap the operator's single-use command away mid-copy), and the stats/graphs panels, which
own the elements `stats.js` and `graphs.js` poll into.

**Console** (phase 3) — the hub attaches to a server's primary container at most once per
role, regardless of how many browser viewers are watching: the first `Subscribe` sends
`ConsoleAttach` and opens the agent-side attachment, later ones join for free and get the
buffered backlog immediately, and the hub fans live frames out to every viewer over its own
websocket. Backlog on attach comes from Docker's own log buffer (`Attach` requests `Logs`
alongside `Stream`), not from anything the agent persists — consistent with the agent holding
no authoritative state (ADR-012).

The agent-side attachment deliberately outlives any single viewer (ADR-032): closing it
whenever the last viewer disconnects would send the container's stdin an EOF it never asked
for, which for a process that treats EOF as "the caller is done" — a shell is the sharpest
example — means walking away from a console can silently kill the server. Detach unbinds
forwarding rather than closing the connection; the connection itself closes only when the
container exits or the agent shuts down.

**A hub-side console session dies with its node's stream** (ADR-072). Its stream_ids mean
nothing to a restarted agent, so when a node's stream tears down — or is displaced by a
reconnect — every console session on that node is closed and its viewers told why. The
browser's ordinary reconnect loop is the recovery path: it resubscribes, which sends a fresh
`ConsoleAttach` once the node is back, and the agent replays its scrollback into the new
session. The terminal resets itself when the first data of a new session arrives (never
eagerly on connect, which would wipe the last readable output while a node is still down), so
a replayed backlog lands in an empty terminal rather than duplicating what was on screen.

The console page itself is a full terminal, not a log tail: xterm.js with a 50k-line
scrollback, ANSI color themed from the same CSS variables as the rest of the UI (so the
terminal and the box it sits in cannot disagree — two different darks there is what once made
the page look like two nested consoles), search with match highlighting (vendored
`@xterm/addon-search`, Ctrl+F), clickable URLs, and refitting driven by a ResizeObserver plus
font readiness rather than a single load-time measurement. Ctrl+C with a selection copies it
instead of sending ETX to the game's stdin — a keystroke that would otherwise shut some
servers down.

**The agent also attaches for itself.** `Watch` opens a server's attachment with no viewer
bound, and every chunk read is passed to an observer whether or not anyone is watching — which
is what makes log-value extraction work at all, since the values worth extracting are printed
during startup, long before an operator opens a console (ADR-067). A viewer arriving later
reuses that same session rather than opening a second one, so this uses the property ADR-032
already relies on rather than adding a second mechanism.

**It attaches to every started server, not only the ones whose seeds declare a rule** (ADR-097).
A container's opening burst is written once, and the daemon replays its log exactly once — when
an attachment is opened — so a server nothing was reading kept its history only until the first
viewer, and reached them through the *droppable* frames path rather than the blocking scrollback
replay beside it. Watching from the start puts every server's history in the scrollback, where
that blocking replay serves it, and stops the console's behaviour depending on a seed-authoring
detail no operator can see. It costs one attachment per running server and nothing in the
scanner, which returns immediately for a server with no rules.

**A session is bound to the container run it attached to** (ADR-100). A session is keyed by
server and role and reused across viewer churn, which is right until the container behind it is
replaced — then it answers for a run that no longer exists and `Watch`, finding it, attaches to
nothing at all. That was fixed three times in eight days, each fix enumerating one more event
that ends a run: a destroyed pod (ADR-095), a crash-loop restart (ADR-097), then the four
transitions *up* that replace nothing and must not close a live attachment (ADR-098).

Enumeration is the wrong shape for it, so the run is measured instead. A session records the
container id and start time it attached to, and `attach` compares that against the run holding
that server and role now; different means stale, and the session is replaced by construction on
whatever event caused it. Both halves of that identity matter — a reprovision gives a new
container, while a crash-loop restart reuses the same one and only moves its start time, which
is exactly the case an id-only check would miss. An unresolvable run reads as "do not know"
rather than "changed", so a daemon hiccup can never be the reason a live attachment is torn
down.

**A pending attach is still called off when the server stops** (ADR-098), and a destroy still
closes a pod's sessions promptly (ADR-095). Neither is load-bearing now — the comparison above
would catch the same thing at the next attach — but both free an attachment sooner than that.

`internal/agent/logscan` assembles complete lines from those chunks — a container write splits
anywhere, so matching raw chunks would miss any value straddling a read — and matches them against
all four kinds of rule a seed can declare: `logs.values`, `logs.events`, a readiness pattern and a
crash pattern. One assembler rather than four, since a second would be a second place for the
straddled-line bug to live. Each match travels up as a `ServerFact`, which the
control plane stores per `(server, name)` and shows on the server's page and its row in the
fleet list. Facts are discarded the moment a server stops: a Valheim join code from a previous
run is not stale but *wrong*, and would send players to a session that no longer exists.

**Readiness and crash detection are both log-driven where a seed asks for it**, each behind a
declared mode with a safe default (ADR-077). Readiness is covered above. A `crash: {mode: log}`
rule exists for the failure exit codes cannot see — ADR-047 records ARK hanging indefinitely
inside Crashpad with the process alive and nothing exiting — and a match **kills the pod**, so the
ordinary exit path does the transition and the restart with the matched line as the reason. One
route to crashed, not two; marking a server crashed while its containers run would be a state the
restart policy could not act on. The signal goes to the supervisor rather than being turned into a
state change by whatever read the line, because whether a line means anything depends on what the
server is doing: a shutdown message during a deliberate stop is not a crash. `mode: none` is the
default, and the bar for adopting a pattern is higher than readiness's — a wrong readiness pattern
delays a server, a wrong crash pattern stops a healthy one, which is why neither bundled seed that
once carried a candidate still declares it as a field.

**A seed can also recognise well-known events** — `logs.events`, `join` and `leave` — which
maintain a player list on the server's page and a count on the fleet list. Observed state and
nothing else: derived from what the game printed, discarded when the server stops for the same
reason its facts are, and depended on by nothing. The count is shown only for a seed that declares
such a rule, since "0 players" on a server nobody is counting is a confident wrong answer.

**Logs** (ADR-082) — a live view of `yggd`'s and `ygg-agent`'s own runtime output, not any
game's. Server output already has a live-viewing mechanism above; the control plane and the
agent's own process logs had none until this, so an operator debugging a node or the control
plane had only whatever they happened to be tailing at the shell. Both binaries tee their
existing `slog` handler into a bounded, line-count-capped `logging.Ring` (2000 records, capped
by count rather than bytes — the unit here is a whole structured record, and a byte cap would
let one attribute-heavy record, a stack trace, evict most of the backlog). Nothing about
existing stdout output changes: the ring is a second destination, not a replacement.

A node's ring reaches the browser the same way console output does — `stream_id`-correlated
`LogWatch`/`LogUnwatch`/`LogBatch`/`LogClosed` messages on the persistent connection, a session
per node in a new `LogHub` structurally mirroring `ConsoleHub`, ending the same way a stale
console session does when the node's stream tears down (ADR-072). It differs from console in
two ways that follow directly from the content being structured rather than raw bytes: lines
are batched on a short coalescing window rather than sent one message per line, and the control
plane's own logs never cross the wire at all — `LogHub` serves them directly from a local
session fed straight off `yggd`'s own ring, sharing the same viewer/backlog/fan-out machinery
rather than a second hub type. Neither leg persists anything: a restart loses the ring's
history, the same way console's own scrollback is lost on restart (ADR-012).

`/logs` shows the control plane's own log stream; a node's page carries the same view in a
Logs card, kept outside that page's live region for the reason `stats-panel`/`graphs-panel`
already are (ADR-058) — a region swap on every unrelated node change would tear down and
reconnect an open websocket for no reason. Both render through `static/logs.js`, a plain
scrolling list rather than xterm.js: these are structured records with a level and a
timestamp, not ANSI terminal bytes, so a terminal emulator is the wrong tool. Server console
output is unaffected by any of this — `internal/agent/console`, `ConsoleHub`, and `console.js`
are untouched.

**Files** (phase 3) — each server gets a sandboxed directory on its node, rooted under the
agent's `servers_dir` and created on first use. `internal/agent/files.Root` resolves every
path relative to that root and rejects anything that would escape it — a lexical `..` is
neutralised before touching the filesystem, and a symlink is caught by resolving through
`filepath.EvalSymlinks` and re-checking containment against the real path (ADR-033). List,
read, write, delete, mkdir, and rename are request/reply, correlated by command_id via the
hub's `Call` rather than multiplexed by stream_id — a directory listing is a single bounded
answer, not an ongoing session. The web UI is plain server-rendered forms: a listing with
breadcrumbs, a textarea editor for files under 1 MiB, and an upload form, no JS framework
needed (ADR-006).

A seed may put paths out of reach with `server.file_denylist`, for the files it regenerates itself:
without it the browser invites an edit the next rebuild silently discards. It is not a security
boundary — the sandbox is, and containment is unconditional — so a denied entry is omitted from a
listing rather than shown and refused, and it is enforced on the agent as well as hidden in the UI
because a node must not depend on the thing sending it commands being correct.

A seed may also mark a config file `importable`, which lets an operator read it back off the node
and adopt the values the seed recognises onto that server's stored settings — the migration path
for a server somebody configured by hand, or a file edited before the seed had a setting for that
key. Only values that differ are adopted, and a file that could not be read whole contributes
nothing, which is the same judgement patching makes in the other direction.

**A cluster's shared volume gets the same six operations**, at `/clusters/{id}/files`
(ADR-105). It cannot reuse a server's sandbox — that is rooted per server (ADR-033) and a
cluster volume sits outside every such root by design, the same reason `DeleteClusterVolume`
(ADR-066) already needed its own command — so the six wire messages carry an alternate
`cluster_id` alongside `server_id`, mutually exclusive, and one agent-side dispatcher
(`filesRoot`) resolves whichever root the request actually names. No `file_denylist` applies
there: that is a seed's own declaration about one server's writable paths, and a cluster's
shared volume has no seed of its own to declare one over it.

**Bulk data** is always proxied through the control plane; direct browser-to-agent transfer
was considered and rejected (ADR-011). Uploads and downloads go through the same JSON-free
HTTP handlers as the rest of the UI, bounded to 64 MiB per upload.

## Operator UI shape

The UI is a fixed left nav rail — grouped Fleet / Library / Infrastructure / System, so the
destination count can keep growing — beside an inset content pane with a slim context bar
(ADR-054). It replaced a horizontal tab bar that nine destinations had already outgrown.

**Every server has a page** at `/servers/{id}`: a breadcrumb, a tab strip shared by Overview,
Console, Files, Backups and Settings, and lifecycle controls that stay visible across all five. Before
this, those were three unrelated top-level pages reached from a table row, with nothing on
screen tying them to the server they belonged to. Overview shows the pod's containers (every
server is a pod, ADR-017), its allocations joined to the seed declaration that produced them
so an offset-derived port reads as derived rather than as an unexplained number (ADR-048),
what the server is built from, and its backup position — reusing the console page's existing
`stats.js`/`graphs.js` panels and endpoints rather than a second implementation.

**Settings** edits a server's display name, its seed's editable variables, and its
independently-allocated ports (ADR-062). Applying any of them destroys and re-provisions the
pod, because a changed `PodSpec` is otherwise silently ignored — `Provision` returns early on
`ErrPodExists` and `Start` never reads the spec for what the containers should be. World data
survives, since `Destroy` removes containers and the pod network but never a host directory,
and every writable path is a bind mount from the server's own directory. A port field left
blank takes the next free port in the node's range; a derived port is shown but not editable.

**A seed's controls are tabbed once there are enough of them, and the page is searchable**
(ADR-085). A block of four or more groups renders as a vertical tab rail beside one panel per
group — Vintage Story's 136 controls in 18 groups are 3.2 screens of headings as a stack and 1.3
screens as tabs — and the seed's `collapsed` declaration decides which tab opens rather than
whether a fieldset starts shut. Above them sits **one** filter for the whole page, matching a
control's name, label, description or group: the variable/setting split is the seed author's and
not the operator's, who knows the name of what they are looking for and not which block declared
it. Every panel stays in the form whichever tab is showing, since a setting that failed to submit
would read as *absent* — a third state meaning "leave the game's own config alone" — rather than
as unchanged. A field a `show_if` rule hides is never surfaced by a match, and a control the
browser refuses to submit has its tab and the filter cleared before the browser reports it, since
constraint validation runs on hidden controls and would otherwise block Save with nothing on
screen to point at.

**Moving a server to another node** is its own action on that page, and a Job rather than an
edit (ADR-063). The source node archives the server's writable state and uploads it; the
control plane stages it; the target downloads, restores, and the server is repointed,
reallocated, provisioned and — if it was running — started again. The **install does not
travel**: it is reproducible from the seed (the same reason ADR-023 excludes it from backups),
so the target resolves its own and the source's refcount drops. Ports are reallocated from the
target's range, so players need the new numbers. A clustered server cannot move on its own,
since a cluster's volume is node-local (ADR-020). The archive travels over two RPCs of its
own rather than the persistent stream, exactly as an agent binary does (ADR-041); backups
themselves remain node-local (ADR-042), which is why a move reclaims the source's *directory*
(ADR-088) and deliberately not its archives — those did not travel, and removing them would
destroy the server's backup history as a side effect of moving it.

**Every node has a page** at `/nodes/{id}` (ADR-065), for the same reason every server does:
a node was a table row, so the capacity its agent reports at every handshake — cores, memory,
disk — was collected, persisted, and shown nowhere. It carries the host's facts, its agent and
protocol versions, its storage paths, what it holds (servers, clusters, installs), and the two
settings that belong to a host rather than to a server: its port range and its address. The
Nodes list keeps the port-range form as a shortcut, so that form now renders errors back to
whichever page submitted it.

**The servers list is grouped by node**, one collapsible group per host. "What is where" is the
question that page is most often opened to answer, and a flat list answered it only by
reading a Node column on every row — so that column is gone, since the group heading now
says it once, alongside that host's running count, anything needing attention, whether the
agent is connected, and the capacity its agent reports at every handshake. Only nodes with
servers become groups. Creating a server is one **New server** button that opens a chooser — from a seed,
or from a container image — and lands on the page for whichever was picked (`/servers/new-from-seed`
or `/servers/new-from-image`); the image form used to sit permanently expanded at the bottom
of the list, which put a form nobody was filling in below every server they were looking at.
See ADR-054's amendment.

**Each group is a grid of art cards** (ADR-080): a seed's own banner, its state as a chip on the
artwork, its facts and its player count. Every entry is a card, and how many fill a row is the
grid's decision rather than a constant — one column at 420px, three at 1440, seven at 2560 —
because the number that fills a row depends on the viewport and the control plane cannot know
it. The band is height-capped so a wide card does not become mostly artwork. This is what
finally consumes the artwork ADR-077 and ADR-079 built the whole path for; before it, a banner
appeared on exactly one page as a 120px inline image. Most seeds ship none, so the card falls
back to a striped placeholder carrying the seed's own glyph, and that is the path built first.

**A card carries its own start/restart/stop, not only a link to the server's page** (ADR-084's
amendment). The whole-card-is-the-link property survives as a stretched `<a>` layered under the
buttons rather than being the card element itself, since a `<form>` cannot nest inside an `<a>`.
Restart is Stop immediately followed by Start with desired state held at running throughout — the
same back-to-back sequencing a reprovision already uses for Stop immediately followed by Destroy —
and a button disables itself when its node is disconnected rather than dispatching a command
guaranteed to come back undelivered.

**A cluster is one card, not one per member.** Its members are reached through it, at
`/clusters/{id}` — the page ADR-066 built rename, remove-member and delete without, putting all
three in a table cell because there was nowhere else. Its card carries a stacked-sheet edge, a
segmented run bar with one segment per member, and aggregate counts.

Run state is carried by colour as well as text: a badge plus a stripe on the row's leading
edge, both chosen by one Go method so nothing can disagree about what state a server is in.
`degraded` is coloured apart from `crashed`, since ADR-019 makes them operationally distinct;
idle is muted, so a healthy fleet reads calm and only trouble draws the eye — the wall-mounted
case the stylesheet is written for. Colour stays the only swappable axis (Frost / Grove /
Ember), and semantic colour sits outside the palettes entirely.

**A seed contributes one more colour, and only a hue.** `internal/control/branding.Accent`
takes the dominant hue of a seed's own artwork and clamps lightness and chroma to fixed
constants, so a game's colour can mark its own card's edge, its run bar and its page's active
tab without ever competing with the product accent or with semantic state colour. It is
**derived, never stored**: the bytes already answer it, and a persisted copy would be a second
writer able to drift from the image beside it (ADR-071). Memoised per seed, forgotten on
reload, and empty for a seed with no artwork or no usable hue — which the stylesheet's own
`--game: var(--accent)` default covers with no branch in any template. `branding.accent`, the
field a seed *declares*, stays reserved and applied by nothing (ADR-077 §14).

**A server's and a cluster's page open with an art band**: the same banner, blurred on a layer
of its own behind a fixed scrim, with the tab strip at the top and the identity block, endpoints
and lifecycle controls on it. The scrim is never tuned per seed — a seed can ship any artwork,
and the contrast guarantee has to hold for all of them.

**Headline numbers are stat tiles** rather than a line of body text: the fleet's counts, and the
live CPU/memory/network readings on a server and in the console's rail. The live fragment
carries its own grid, since `stats.js` replaces that element's contents wholesale and a grid
declared outside it would not survive the swap; it auto-fits, so one piece of markup is a row on
one page and a stack on the other.

**Those badges, stripes, cards and counts move on their own.** State changes are pushed to every open
tab and the page re-reads itself (ADR-058, "Live state" above) — which is what makes the
wall-mounted case actually work, rather than showing whatever was true when someone last
pressed reload. The art band is deliberately outside every live region, for the same reason the
stats and graphs panels are: a swap on every event would restart the image load each time.

## Seeds

A seed is YAML, not code, and describes install strategy, pod composition, game
configuration, and cluster support. **The format is specified in
[seed-spec.md](seed-spec.md)** (schema 3, ADR-077); `internal/seed/schema.json` is the
generated machine-readable version and [seed-fields.md](seed-fields.md) the generated field
list. `internal/seed.Validate` is normative — the cross-field rules that matter most (exactly
one primary container, a derived port naming a non-derived sibling, a template that must
dry-render) cannot be expressed in a schema, so both artifacts are generated from the Go types
with a test that fails when the committed copies drift.

Sketch:

```yaml
id: ark-survival-ascended
schema: 3
version: "4.0.0"

install:
  shared: true              # one install, many servers
  mount: { path: /game, mode: ro }
  image: { base: steamcmd } # required only because a steamcmd step needs a container
  steps:
    - op: steamcmd
      app_id: 2430930

server:
  writable_paths:           # bind-mounted over the read-only install
    - { from: saved,  to: /game/ShooterGame/Saved }
    - { from: config, to: /game/ShooterGame/Saved/Config }

cluster:
  supported: true
  mount: /cluster
  args: ["-clusterid={{.ClusterID}}", "-ClusterDirOverride=/cluster"]

containers:
  - role: game
    primary: true
    image: { base: steamcmd-proton }   # or a verbatim `ref:` for a third-party image
    command: "...?Map={{.Vars.map}}?..."
    ports:
      - { name: game,  protocol: udp, default: 7777,  kind: game }
      - { name: query, protocol: udp, default: 27015, kind: query }
      - { name: rcon,  protocol: tcp, default: 27020, kind: rcon }

variables:                  # shape how the server is built; changing one rebuilds the pod
  - { name: map, default: TheIsland, editable: true }

settings:                   # the game's own configuration, each declaring where it lands
  - name: max_players
    type: number
    min: 1
    max: 200
    step: 1
    default: "70"
    to: { file: GameUserSettings.ini, key: "/Script/Engine.GameSession/MaxPlayers" }

ready:                      # a declared mode, not a bare substring
  mode: log
  pattern: "has successfully started!"
  timeout: 900s

logs:
  values:                   # extracted and shown (ADR-067)
    - { name: join_code, label: "Join code", fleet: true, pattern: "join code ([0-9]+)" }
  events:                   # named groups, fixed vocabulary; maintains a player list
    - { event: join,  pattern: '(?P<player>\S+) joined the game' }
    - { event: leave, pattern: '(?P<player>\S+) left the game' }

stop:
  command: "DoExit"
  timeout: 120s
  then: SIGTERM

backup:
  include: [saved, config]
```

**An install is an ordered list of typed operations**, not a method enum. Schema 2 had
`method: download|steamcmd` plus five conditionally-required siblings, which could express
exactly two installs — a game needing fetch-then-extract-then-chmod had none. Arbitrary shell
was rejected (it would make a seed code, and no form can generate it), so the ops are a fixed
vocabulary: `download`, `extract`, `steamcmd`, `mkdir`, `copy`, `move`, `chmod`, `write`,
`patch`, plus three reserved lifecycle ops for a game that must boot once to write its own
config. `install.image` is required exactly when a step needs a container and forbidden
otherwise, and a shared install may not reference per-server data at all — it is mounted
read-only by every server referencing it.

The list travels to the agent on `InstallStart` and `internal/agent/install` executes it. Two
things deliberately do **not** travel, for the same reason: a step's `if` is evaluated by the
control plane and a false step is simply not sent, and a symbolic base image is resolved by the
control plane — so the agent still needs no access to a seed's variables or to the release
channel (ADR-012). The old `method`/`url`/`archive`/`filename`/`app_id` fields are still written
alongside the steps for one release, so a protocol 1 agent installs exactly as it did; that work
took the negotiated protocol to 2 (ADR-014's N-1 window, which until then had no real history to
mean anything against), and ADR-082's log viewer has since taken it to 3 the same additive way.
`internal/configfile` is the shared key-patching implementation behind both
the `patch` step and a config file managed `patch`, so a key path means the same thing at
install time and at provision time.

**Variables and settings are separate blocks.** A variable reaches a command, an env value, an
install step or a mount, so changing one rebuilds the pod. A setting is written into the game's
own configuration and *declares its destination* — a config file key or an env var — so a game
with forty-four documented keys needs no template line per key. Both embed one `Control` type,
which is what makes the split free in the UI: `varfields.html` renders either kind. Settings
also carry a tri-state absent/empty/set, so a game whose image owns and rewrites its own config
file can be told to leave a key alone, which schema 2 could not express.

`seed.ResolveDestinations` works out where every setting goes, once per rebuild, and both the pod
builder and the config writer read from it — so a container's environment and a config file
cannot disagree about whether a setting is present. A server's settings are stored in their own
`servers.settings` column, holding the operator's own choices and never the seed's defaults:
filling defaults in on write would collapse the tri-state permanently on first save.

**A config file declares how it is managed**: `always` rewrites it wholesale (the default, and
schema 2's only behaviour), `once` writes it only if absent, and `patch` sets just the keys the
seed and its settings name, through `internal/configfile`. Patching is the only thing on the
provisioning path that reads from a node, and a read that fails or truncates degrades to writing
the declared keys alone rather than failing the rebuild.

**Readiness is a declared mode** — `immediate` (the default, and what every seed did before),
`log`, `port`, or `healthcheck` — and the supervisor honours it. Schema 2's `logs.ready` was a
substring four bundled seeds declared and nothing ever matched, because making readiness depend on
a match risked stranding a healthy server in `starting` forever (ADR-067). The mode is what makes
it safe: opting in is deliberate, and a timeout reports the server running anyway *with a reason*
rather than leaving it in limbo. `mode: port` needs no game-specific knowledge, which makes it the
right first choice for a game added blind.

`immediate` is the same call at the same point it always was, and the control plane sends no
`ReadySpec` at all for a seed that asks for nothing else — so such a seed's pod spec is unchanged.
A log pattern is matched by `internal/agent/logscan`, which already assembles lines out of chunks
for log-value extraction, so there is one line assembler rather than two. Only the paths that
restart the *primary* re-establish readiness; bringing a crashed sidecar back does not, since the
pod already satisfied its rule and a startup line will not be printed again.

**Container images may be named symbolically.** `image: {base: steamcmd-proton}` resolves
through the control plane to the right registry, owner and channel at provision time; a
verbatim `ref:` still works for any third-party image. This exists because ADR-073 recorded a
real base-image fix that could not reach a node: a seed pinning `release-latest` is stuck on a
tag built only from `main`.

Each base carries a descriptor at `images/<name>/base.yaml`, embedded by `images/library.go`,
recording what it provides, which architectures it is published for, how it runs, and which
environment variables it honours. The registry name comes from that descriptor rather than from
assembling `"base-" + name`, so a base whose directory and seed-facing name diverge still
resolves to something that exists. Only the descriptors are embedded — a Dockerfile stays data a
human or CI builds, the same line ADR-049 draws for a seed's own `image/` directory. An unknown
base is a warning at load and an error at provision, never a load failure, since adding an image
would otherwise make an older binary reject a newer seed.

**A seed can carry its own branding**, served from `GET /seeds/{id}/assets/{kind}` where kind is
`icon`, `logo` or `banner`. The request names a *kind* and the seed says which file that is, so
no request-supplied path reaches the filesystem at all — a stronger guarantee than sandboxing
one, and a real difference, since a bundle can also hold an install script or a Dockerfile.
Bytes come from whichever layer the effective copy of the seed came from, highest first, matching
`seed.Merge`. Responses carry a restrictive CSP and `nosniff`: an SVG is a document that can
carry script and this one is served from the control plane's own origin. A server's page leads
with the banner (or the logo, when it declares no banner); `branding.accent` is reserved and
applied by nothing, because the accent tokens travel as a set and a seed supplying one of the
three can make its own page unreadable.

**An addon is an optional container** — `containers[].optional`, a map renderer or an admin UI —
shipped with the seed and provisioned only if the operator enables it. All three port paths take
their container list from one `enabledContainers` function, so a disabled addon is invisible to
allocation at creation, to `ensureContainerPorts` on rebuild, and to the pod spec by
construction; the alternatives are an allocation nothing publishes, which reads as drift on every
settings save, or a container port taken from an allocation that does not exist. Ports are
released on the next rebuild rather than at the moment of the save, so the invariant holds for a
server whose earlier save failed halfway. Validation carries the rest: an addon cannot be
primary, nothing required may depend on one, and no template may reference one's port.

**A port declares what it is for** (`kind: game|query|rcon|web|voice|other`), and a seed may
declare a **connect** block — a URI a client understands, an address to copy, or both. It is
rendered per request rather than stored, because it templates over the node's address, which can
change and can legitimately be unknown (ADR-065); with no address known it renders nothing at
all, since a bare port reads as incomplete and a URI with an empty host does not.

**A seed can carry its own migrations.** Stored values are keyed by name, so renaming a variable
silently discards what every operator chose — ADR-074 declined to rename Valheim's
`crossplay_flag` for exactly that reason, since losing the stored value would have turned
crossplay back on wherever it had been turned off. `renamed_from` and a `migrations:` block
(`rename`/`drop`/`rewrite`/`promote`) carry values forward, with **nothing recorded per server**:
every operation is idempotent by construction, and validation enforces that rather than trusting
an author to be careful.

**A schema 2 document still loads.** It is up-converted before validation, so a catalog bundle an
operator already installed keeps working and there is exactly one shape in memory. The converter
deliberately drops `logs.ready`/`logs.crash` rather than translating them: nothing consumed them,
so dropping changes no behaviour, while translating would make readiness depend on a substring
nobody verified.

ARK's three ports are all independently allocated — each scanned from the node's port range,
which since ADR-061 is the whole candidate pool: a `default:` documents the game's conventional
port and the authoring form's preview value, and is never offered as an allocation preference.
Some games hardcode a relationship
between two ports instead: Valheim's Steam query port is always `game_port + 1` inside the
binary, with no flag to set it separately. `offset_from`/`offset` express that (ADR-048) —
mutually exclusive with `default`, since a derived port has no preferred value of its own:

```yaml
ports:
  - { name: game,  protocol: udp, default: 2456 }
  - { name: query, protocol: udp, offset_from: game, offset: 1 }
```

A multi-container seed adds dependencies and healthchecks:

```yaml
containers:
  - role: postgres
    image: postgres:16
    healthcheck: { command: "pg_isready -U game", interval: 5s, retries: 20 }
    volumes: [{ from: pgdata, to: /var/lib/postgresql/data }]
    backup: { pre: "pg_dump -U game -Fc game > /backup/game.dump" }

  - role: game
    primary: true
    image: ...
    depends_on: [{ role: postgres, condition: healthy }]
```

### The v1 test targets

**Minecraft Java and Bedrock** exercise the seed schema on single-container servers:

| | Java (Paper) | Bedrock (BDS) |
|---|---|---|
| Primary port | TCP 25565 | **UDP 19132** |
| Licence gate | `eula.txt` written before first boot | none |
| Memory | `-Xmx` **and** the container limit | container limit only |
| Architectures | `amd64`, `arm64` | **`amd64` only** |

**ARK** exercises shared installs and clusters. **Dune Awakening** is what motivated
multi-container pods, and is named here as the case the axis was designed for rather than as a
target: it has no public dedicated-server image, so the pod mechanism is proven against a
synthetic game+database+broker seed instead (see phase 6 below). Between them the three axes are
covered.

**Architecture.** Nodes report their architecture and seeds declare what they support;
invalid placement is rejected with a clear error rather than an opaque exec-format crash-loop.
Placement scheduling is deferred until a mixed fleet exists (ADR-016).

### What the seed catalog covers

The seeds themselves live in `arasoi/yggdrasil-seeds` (ADR-081, ADR-087), which is where they are
authored, versioned and published, and where what each one covers is documented next to it. This
document describes the *format* and the machinery; the catalog describes the games.

That repository publishes **six seeds on `main` and seven on `develop`** at the time of writing —
ARK Survival Ascended, Minecraft Java (Paper), Minecraft Bedrock, Valheim, Vintage Story and
Palworld, with Empyrion still on the edge channel. They exercise both ends of ADR-018's install
model between them: a shared SteamCMD install mounted read-only with per-server writable overlays
and cluster support at one end, and an image that installs the game itself with no install block
at all at the other. Phase 5's exit criterion is stated against one of them and is tracked in the
phase table below, not here.

**A count here is a snapshot and will drift**, which it already did once — this paragraph said
"five seeds" for as long as it took two more to be published, and nothing could have caught that,
because the catalog is a separate repository on its own release cadence (ADR-081) and no test in
this one can see it. Read the number as "roughly this many, as of the last edit"; the channel
itself is authoritative, and the per-channel split is worth stating because promotion is a merge
there too, so edge is routinely ahead.

**What this repository keeps is a fixture corpus**, `internal/seed/seedtest`, six bundles written
for coverage rather than for play. It exists because the embedded set was never only a shipping
mechanism: eighteen test files swept it, and `seedform`'s round-trip sweep in particular is the
mechanical guard against the drop-on-save bug this codebase has recorded four times. Deleting the
seeds without replacing the corpus would have deleted that guard.

The corpus is deliberately not a copy of what shipped — a vendored copy nobody updates is the
same second-writer drift ADR-087 removed — so each fixture names the corner it owns:

| Fixture | Owns |
|---|---|
| `shared-install-cluster` | a shared refcounted install, read-only mount with writable overlays, cluster args, a steamcmd step, ini settings behind `manage: patch` and `importable` |
| `derived-port-logs` | an offset-derived port, `logs.values` and `logs.events`, `ready: {mode: log}`, `renamed_from` and a rename migration |
| `env-settings` | settings with `env` destinations, the absent/empty/set tri-state, `promote` migrations, and no install block |
| `file-settings` | config files managed `once` and `always`, `source_path` templates, `show_if`, and the RCON-needs-a-password template guard |
| `many-groups` | enough groups to cross `varSection`'s tab threshold, every control type, and a JSON config where numbers and booleans must render unquoted |
| `addons-branding` | an optional addon container, branding files with real image bytes, a `connect` block, a multi-container pod with health gates, volumes and backup hooks |

A test that needs something none of these declares should add a fixture rather than reach for a
published seed: a fixture states the property it is there for, where a shipped game states only
what that game happens to need.

### Seed bundles, authoring UI, and Steam integration

ADR-049's directory-per-seed layout is built. A seed is a directory — `<data_dir>/seeds/<id>/`
(catalog) or `<seeds-dir>/<id>/` (operator) — each holding a `seed.yaml` manifest plus,
optionally, real config-file templates under `configs/`, referenced from the manifest by
`ConfigFile.SourcePath` rather than `ConfigFile.Template`. `internal/seed.LoadDir`/`LoadFS`
resolve `SourcePath` against the bundle directory into `Template` before validation runs, so
`validateTemplates`'s load-time dry-render needed no changes — exactly the seam ADR-049 planned.
The schema bumped to 2 accordingly; a schema-1 seed (single flat YAML file, inline templates
only) is no longer accepted.

The seed authoring UI covers the **whole** of schema 3 (ADR-079). `/seeds` lists every seed
(catalog and operator, badged by source), `/seeds/new` and `/seeds/{id}/edit` write
straight into the operator's `--seeds-dir` as a real bundle directory, and saving triggers an
in-process reload (`internal/control/web`'s `reloadSeeds`, guarded by a mutex around `s.seeds`) —
a seed created or edited through the UI shows up in "New server from seed" immediately, no
restart. Editing a *catalog* seed materialises an operator copy with the same id on first save,
resolving the open question ADR-050 originally left unanswered — and taking that seed out of the
catalog's reach, since an update replaces a bundle wholesale and the operator layer sits above it.

The form has real fields for every block: identity and branding, the install's ordered steps (all
nine executable ops), writable paths and the file denylist, cluster support, containers with their
ports, env, volumes, dependencies, healthcheck and backup hooks, variables and settings sharing
one control renderer, groups, config files, readiness, crash detection, stop, log values and
events, connect, backup, and migrations. **Rows repeat** — `static/seedform.js` clones a
`<template>` prototype and renumbers a placeholder token — which is what retired the fixed row
counts the old form padded to, along with the three `guidedEditable` checks whose only job was to
refuse a seed that exceeded one. Three of the four recorded drop-on-save bugs were that pattern
failing.

`internal/control/web/seedform` owns the form's shape: `Decode` turns `url.Values` into a
`seed.Seed`, and `Encode` is its inverse, existing so that **every seed in the fixture corpus
round-trips unchanged** and does so idempotently. That is the mechanical answer to a failure this codebase has
recorded four times — a seventh variable, a `Required` flag, an offset relationship, a
`logs.values` block — each found only after it shipped, by someone noticing a value had vanished.
Two further tests join the halves: the rendered page must emit every field name the decoder reads,
and each row prototype's path shapes must match a rendered row's, so a row added in the browser
submits the same field set as one from the server.

Decoding is **index-driven, not count-driven** — `containers.0.ports.1.name`, with the indices
collected from what was actually submitted — so a row deleted in the browser leaves a gap and
nothing downstream cares. Each discriminated block (an install step's `op`, a setting's
destination, a control's `type`) reads only the fields its selected case accepts, because the form
renders them all and hides the rest, and `Validate` rejects a field that does not belong.

**Branding images are uploaded, fetched from a URL, or taken from Steam**, and land in the seed's
own bundle directory (`internal/control/branding`) — so publishing the seed carries the artwork to
everyone who installs it, which is what makes this worth storing rather than hotlinking. Steam is
not a third mechanism: it resolves an app id to an image and then takes the URL path. The app id is
its own field rather than the install step's, because they are often different — Valheim's
dedicated-server tool has no store page while its base game does, and the same holds for ARK
(2430930 versus 2399830).

**Which image is the operator's choice, not the app id's.** Each slot offers a gallery of every
piece of artwork Steam actually has, from two sources that are each incomplete alone: the Store
API's own fields (header, the capsules, the page background) and the well-known per-app CDN paths
it does not return — the portrait capsule, the wide hero, and the transparent `logo.png`, which are
the three that fit a seed's own icon/logo/banner slots at all. Taking the header for every slot,
which is what this did before, put a 460×215 banner in the icon and the logo.

Those CDN paths are a convention rather than a contract, which ADR-051 treated as disqualifying
when it rejected SteamDB. What makes them acceptable here is that **nothing depends on one
existing**: every candidate is probed with a HEAD before it is offered, so a Valve change shows
fewer choices rather than breaking the picker or the save — and an app with no store page, which
is the ordinary case for a dedicated-server tool app, correctly yields an empty gallery that says
why. The gallery is ordered by which slot an image suits and never filtered, since the shape is a
suggestion and an operator who wants the hero art as an icon knows something the heuristic does
not. With nothing picked the header is still taken, so a save that never opened the gallery — or
one made with JavaScript disabled — behaves exactly as it did before.

The bytes decide the filename: a raster is decoded with the standard library and stored as
`<kind>.<detected ext>`, never as what the upload was called, so a file named `icon.png` whose
contents are not a PNG is refused rather than served as one from this origin. **SVG is refused on
this path and still served when hand-placed** — an operator putting one in their own bundle has
made a local decision about their own control plane, whereas a published bundle's images are served
from *other* operators' origins. Nothing is written until every fetch has succeeded, so a save
cannot leave a manifest naming an image that was never retrieved.

`guidedEditable` and its allowlist stay, at full coverage. What they guard is not today's gaps but
the *next* field added to `seed.Seed`: one that nobody has taught the form about is unrepresentable
by default, so a seed using it falls back to the raw-YAML pane rather than being saved without it.
That pane is now a co-equal path rather than a fallback, and both routes converge on the same
`internal/seed.Parse` and `Validate` every seed loads through.

**The editor is organised into five tabs — Identity, Art, Runtime, Networking, Install — beside
a validation rail**, a visual pass over the same form above rather than a new one (ADR-079's
amendment). Every tab's fields stay in the one `<form>` regardless of which is showing —
`static/seedtabs.js` toggles `hidden` client-side, so a hidden tab's inputs still submit and the
page is every field, unfiltered, with the script absent. The rail runs the same `Validate` a save
already gates on (at most one error, since `Validate` fails fast at its first problem) plus every
`seed.Lint` warning, and a tab's own count badge is the same two checks re-grouped by which tab
each one's field name points at — approximate, since `Validate`'s error text is a sentence with a
topic prefix and not a structured path, but a hint costs nothing a save does not already need.
Runtime carries a **resolved command preview**: the primary container's command rendered against
every declared variable, setting and port default (`seed.MergeStored` plus the same
preview-before-allocation rendering server creation already does), so an author sees what a fresh
server would run without creating one. The Install tab's own footer is deliberately **Validated**,
never *Ran* — nothing here executes a step against a node, only the schema and template checks
`Validate` already runs.

Repeated rows — containers, ports, variables, settings, install steps — render as `.entry` cards
with a header (name, type, required/secret chips) over a collapsible body, and an install step
additionally carries a numbered spine. Markup only: every row's `name=` attributes are byte-for-byte
what they were, which is what lets `seed_form_template_test.go`'s round-trip checks stay the proof
that a save cannot silently drop a field, unchanged by any of this. Branding moved into a dialog —
still the same three bundle-relative slots (icon, logo, banner) schema 3 actually has, named by
where each appears rather than by pixel size — reached from the Art tab and associated with the
outer form via `form="seedform"` on each control, since a `<dialog>` cannot itself be nested inside
the form whose own close button already uses `<form method="dialog">` (two `<form>` elements cannot
nest; the parser drops the inner one, and the close button would otherwise fall back to submitting
the outer form instead of dismissing the dialog).

**`/seeds/new` is a chooser before it is a form**: from a Steam app id (looks up the same
`internal/control/steam` lookup the guided form's own "Fetch from Steam" button already calls, and
seeds an `install.steps` steamcmd op from it), duplicating an existing seed (install, containers,
variables and settings carry over; branding does not, since the bytes live under the source seed's
own id and copying only the filenames would point at images that do not exist for the new one), a
base-image template, pasting YAML or JSON directly, or starting blank. Every path still lands on
the one guided form and the one save handler — the chooser only decides what `seed.Seed` that form
starts from.

**What this deliberately does not build**: a persisted draft record and an explicit Publish action
separate from Save. The original design sketch offered a reduced version precisely for this case —
keep saving straight through, one action, no autosave — and that is what shipped: `Save` still
writes the operator bundle immediately, exactly as it already did. Building a draft table, an
autosave endpoint, and a reconciliation story between an unpublished draft and the seed a server was
actually built from is real, separable scope, and the honest reduction was taken deliberately rather
than half-built. Live, as-you-type validation is the same call: the rail reflects the seed as it was
last loaded or last submitted, not a live re-check on every keystroke, which is consistent with every
other server-rendered form in this codebase and adds no new failure mode to reason about.

### Seeds can be published, not only downloaded

The catalog was one-way until ADR-079: `yggd` read `seeds.json` from a floating channel tag and
nothing could push one, so a seed authored on one control plane reached another only by copying
files. `internal/control/publish` is the outbound half, reusing `internal/seedpack` unchanged — so
a seed published from the UI and one packed by CI are byte-identical artifacts, and the version
rule that makes a catalog orderable is enforced by the same code in both. **Only the operator
layer is published**, never the merged set: republishing the catalog seeds an operator never
wrote, under their own channel, would mean a later catalog update silently changing what they
were shipping.

**The catalog has a repository of its own** (`catalog.DefaultRepo`, `arasoi/yggdrasil-seeds`),
separate from the one publishing binaries and container images. Publishing replaces a release
wholesale — the tag floats, so the existing release is deleted and recreated — so a repository
that is also a publish target must hold nothing else worth losing. That separation is what makes
a catalog publishable from a running control plane at all rather than only by CI (ADR-081), and
everything below follows from it.

`publish.ResolveTarget` still refuses `arasoi/yggdrasil-releases` as a checked error,
case-insensitively, and refuses it even when set deliberately — but for the narrower reason that
it is the one place where a wholesale release replacement takes out the distribution itself. It
is deliberately **not** "the repository this control plane reads its catalog from": that is the
ordinary case for whoever maintains a catalog. For an operator who is not the maintainer, GitHub's
own permissions are the boundary, and the token's Test action asks exactly that before they press
anything.

**This path is for an operator with no CI**, or one publishing a catalog for their own fleet from
a seeds directory that is not a git clone at all. Where CI publishes — as it does for the
project's own catalog (ADR-081's amendment) — it is the one writer, and `seeds.publish_repo` is
left **unset**, which renders the Publish button with "Set a publishing repository in Settings
first" and makes it incapable of becoming a second writer. Inert by configuration rather than by
removal.

Packing runs first and completely, before anything on the remote is touched. `seedpack` loads
every bundle through the strict loader and rejects a version `version.Compare` cannot order, so a
seed that would produce an uninstallable catalog fails while the existing release is still intact
— deleting it and *then* discovering the bundles are unpublishable would leave the channel empty,
which is worse than not having published.

`seeds.publish_repo` has **no default**, so publishing is off until an operator names a target, and
`seeds.publish_token` is a `KindSecret` (ADR-078) whose value cannot reach page data by
construction. The token carries a Test action that asks GitHub both questions worth asking — is
this token accepted, and can it write to that repository — because a valid token for the wrong
account authenticates fine and is exactly the misconfiguration that otherwise surfaces as a failed
publish much later. This is the first credential `yggd` holds that lets it write somewhere outside
itself; every other one is inbound-facing or read-only.

ADR-051's Steam integration is built: a "Fetch from Steam" button next to the SteamCMD app id
field (`internal/control/steam`, called from `internal/control/web/steam_lookup.go`) queries two
real sources — Steam's public, unauthenticated Store API for a name, description, and header
image (used to prefill the Icon field, since it is already a plain `http(s)://` URL), and a local
`steamcmd +app_info_print` for whether a Linux depot exists at all, the same authoritative check
ADR-047's own development used by hand for Valheim's app id. Results are cached in memory per app
id for an hour and the lookup is entirely on-demand — clicking the button, never a background
poller. SteamDB is a plain search-page deep link next to the button, exactly as ADR-051 decided:
nothing in `yggd` parses or scrapes it. The two sources degrade independently: an app id with no
ordinary store page (confirmed live against Valheim's real dedicated-server app id, 896660, which
genuinely has none) still gets its Linux-depot answer, and a control plane host with no SteamCMD
installed still gets the Store API's polish, with the UI saying plainly which half it couldn't
check rather than guessing.

Not yet built: the optional icon image, seed-specific Dockerfile, and bundled sidecar "addons"
a bundle directory can *structurally* hold (ADR-049/ADR-050) have no consumer yet — nothing reads
or renders them, and the authoring UI's icon field is a plain text input (now prefillable from
Steam's header image, but still no file upload). Exporting a locally-authored seed as a shareable
archive (ADR-049) is unbuilt too, though the half it was missing — packing a directory — now
exists as `internal/archive.CreateTarGz`. None of this changes seeds' filesystem-only,
load-once-at-startup model (ADR-007) — a save is still a plain file write, reloaded the same way
startup already loads every seed.

### Seeds are downloadable, and versioned independently of `yggd`

A seed is data, but until ADR-060 the only way a fix to an embedded one reached an operator was a
new `yggd` release — so either `VERSION` climbed because a YAML file changed, or the shipped
seeds drifted behind the repository. Seeds now have their own release channel.

A catalog lives on release tags of its own — `seeds-develop-latest`, `seeds-qa-latest`,
`seeds-release-latest` — floating per channel the way ADR-038's binary channels do, in a
repository of its own (`arasoi/yggdrasil-seeds`, ADR-081), which carries the same three branches
and publishes each one's channel on push. Separate tags *and* a separate repository are both the
point rather than tidiness: sharing release.yml's tags would re-couple exactly what this
separates, and sharing its repository would mean a catalog publish replacing a release that
carries binaries. Each seed carries its own `version:`, and the packer refuses one
`version.Compare` cannot order, since publishing that produces a seed nobody could ever be told
to update.

**A release tag has exactly one writer**, since two producers feeding one floating tag would
clobber each other on every push. For the project's own catalog that writer is CI **in the seeds
repository** (`arasoi/yggdrasil-seeds`), where the branch *is* the channel: pushing to `develop`
publishes edge, and merging into `qa` or `main` promotes it. That mapping is the reason it lives
there rather than behind a control plane's Publish button — `internal/control/publish` takes its
channel from the `seed_channel` setting, which has nothing to do with what is checked out, so
branch and channel could disagree silently and promotion by merge would mean nothing. Publishing
to its own releases also needs only `GITHUB_TOKEN`, so no personal access token exists to manage.

*This* repository runs no seed workflow at all. It had one — `.github/workflows/seeds.yml`, which
linted and packed `seeds/library` and uploaded nothing — and ADR-087 deleted it along with the
embedded set it was checking: with no seeds here, the seeds repository's own CI is the only
checker, which is this section's one-writer rule applied to checking as well as publishing.
`make seeds-check` and `make seed-catalog` survive as author tools and now require `SEEDS=` naming
a checkout. `release.yml` still publishes
`ygg-seed-linux-<arch>`, because the seeds repository holds seeds rather than Go source and has to
download the packer — checksum-verified against `SHA256SUMS`, since building it there would need a
read credential for a private repository inside a public one's CI (ADR-081's amendment).

**`cmd/ygg-seed` is the tool the format exists for.** It scaffolds a bundle that already
validates, loads one exactly as `yggd` does, lints it, migrates a schema 2 manifest to schema 3
in place (on a `yaml.Node` tree, so comments and key order survive), packs a catalog, and imports
a Pelican or Pterodactyl egg best-effort — reporting everything it could not carry rather than
dropping it, since an egg's install is a bash script and a seed's is a fixed vocabulary of typed
operations. Nothing in it reimplements the format: a seed it accepts is a seed `yggd` accepts,
which is the property that makes it worth having at all. `ygg-seedpack` was a thin alias for its
`pack` subcommand and is gone — see ADR-060's 2026-08-12 amendment.

Seeds therefore load in **two layers** (`seed.Merge`, in order):

| Layer | Where | Why it is at this level |
|-------|-------|-------------------------|
| catalog | `<data_dir>/seeds/<id>/` | downloaded, updated on its own schedule |
| operator | `--seeds-dir/<id>/` | hand-authored, so a catalog update can never overwrite it |

There was a third beneath these — the set embedded in the binary — until ADR-087 removed it.
The catalog repository (ADR-081) is where seeds are authored and published, so an embedded copy
of the same content was a second producer of it that could only drift, and CLAUDE.md carried a
standing rule to keep the two in step by hand. **A seed now reaches a control plane through the
catalog or through `--seeds-dir`, and through nothing else.**

The property that cost is ADR-060's own reason for embedding: a fresh install no longer has seeds
with no network at all. `seed_channel` therefore defaults to `main` rather than to empty, so the
catalog is *listed* without being configured first — installing any of it is still one click per
seed, and setting the channel to empty still means no outbound call. An air-gapped control plane
installs bundles by hand into either directory, which is what the operator layer has always been
for. Both the startup log and the Seeds page say when there are no seeds and why, rather than
showing an unexplained empty list.

Downloaded bundles deliberately do **not** live in `--seeds-dir`: an update replaces a bundle
wholesale, and that operation must be structurally incapable of reaching hand-written content.
Each carries a `.ygg-source.json` recording channel, version, and digest — inside the bundle
rather than in a database, so a seed and its provenance cannot be separated and nothing can fall
out of sync with the filesystem. A bundle with no marker is one an operator placed by hand, and
the catalog neither claims nor deletes it.

`/seeds` lists what the catalog publishes beside what is installed, ordered by `version.Compare`
so a channel that has fallen behind reads "older" rather than offering a downgrade. Install and
update are the same action, and both end in the same in-process `reloadSeeds` a save through the
authoring UI already runs — a downloaded seed is usable without restarting `yggd`. An install is
staged, validated through the ordinary loader, and only then swapped into place, so a malformed
entry fails at download with the working copy untouched.

Trust is HTTPS plus the published `SHA256SUMS` (ADR-044's level, reused rather than
reimplemented), not ADR-040's signing key: a seed is parsed and validated before it can replace
anything, unlike an agent binary that executes with Docker-socket access on every node. The
catalog is opt-in — `seed_channel` defaults to empty, and with it unset the page makes no
outbound call at all, offering a channel picker instead that persists the choice to `yggd.yaml`.

**Where it is read from is a setting**, `seeds.catalog_repo`, defaulting to the project's own
catalog. That is the operator-added catalog path ADR-060 deferred, arriving as a repository name
rather than a free-form URL — so the trust level is unchanged (same tag layout, same published
`SHA256SUMS`, same parse-and-validate before anything is replaced) while a fork, a mirror, or a
catalog an operator publishes for their own fleet becomes a value rather than a build. It is
total: a stored value that somehow arrived empty falls back to the default rather than failing
the page an operator would use to correct it. The in-memory index cache is keyed by repository
*and* channel, since a channel-only key would answer with the previous repository's listing for
up to an hour after a switch.

**A bundle on disk that will not load is skipped, not fatal.** The catalog and operator layers
are files the running binary did not ship, so closing a validation gap invalidates bundles that
were perfectly acceptable when they were installed — and refusing to start then crash-loops the
control plane over a file it could ignore, with no UI left through which to remove it. Startup
and the in-process reload both use `seed.LoadDirTolerant`, logging each skipped bundle at error
level and falling through to the layer beneath. `LoadFS` stays strict, and is what the base image
library and the seed fixture corpus load through: those are compiled in or checked in, so a
failure there is a build defect rather than a file an operator installed.

## Container image library

A published seed's container image is typically either official (Paper's `eclipse-temurin`) or a
well-known community one (Bedrock's `itzg/minecraft-bedrock-server`) — nothing a seed author
writing an image from scratch could start from. `images/` (ADR-047) is a small, deliberately
generic base image library filling that gap:

- **`images/base-linux`** — Debian `trixie-slim` (moved from `bookworm-slim`, ADR-047's third
  amendment — GE-Proton needs a newer glibc than bookworm ships) with `tini` as `ENTRYPOINT`
  (correct signal forwarding: StopSpec's `SIGTERM` escalation only reaches the game if the
  container's PID 1 actually forwards it, which a bare shell does not), `ca-certificates`,
  `tzdata`, and a properly generated `en_US.UTF-8` locale.
- **`images/base-steamcmd`** — `FROM` `base-linux` (a `BASE_IMAGE` build arg, so a local build
  never needs a registry round trip), adding the `i386` architecture and 32-bit libraries
  SteamCMD needs, plus SteamCMD itself, pre-warmed at build time.
- **`images/base-steamcmd-proton`** — `FROM` `base-steamcmd`, adding a pinned GE-Proton runtime
  and `umu-launcher` (ADR-047's amendment), for a Steam-distributed game whose dedicated server
  has no native Linux build at all and ships only a Windows binary. ARK Survival Ascended is the
  first consumer.
- **`images/base-java`** — `FROM` `base-linux`, adding a pinned Eclipse Temurin JRE (ADR-047's
  second amendment) — the same distribution Paper's own `eclipse-temurin` image is built from,
  for the next JVM-based seed that isn't Paper and so has no runtime image of its own to start
  from.
- **`images/base-dotnet`** — `FROM` `base-linux`, adding a pinned .NET runtime (ADR-047's second
  amendment), for a dedicated server that ships as a .NET application.

`base-steamcmd` has two consumers, and since ADR-057 the first is the more important. It is what
`internal/agent/install`'s `steamcmd` method runs to populate a shared install's read-only mount
— in a container, via `Runtime.RunOnce`, rather than on the node host as it did originally
(ADR-018's mechanism is unchanged; only where SteamCMD executes moved). It is also available to
a seed's runtime **container** wanting SteamCMD for itself — the pattern several community Steam
dedicated-server images use to validate or self-update at container start — which is what
`base-steamcmd-proton` inherits it for.
`base-steamcmd-proton` closes the gap `base-steamcmd`'s own entry used to name here: what a
genuine ARK image needs (phase 5's status below) now has both the SteamCMD half and the Proton
half built and verified against a real Podman host — the ARK image itself is still unbuilt.

None of the five bundle a game. The library's own design rule (ADR-047's amendment) is
environment only — runtime, its OS-level dependencies, and whatever launches it — with the game
installed into that environment afterward, the same read-only-install-plus-writable-overlay
split ADR-018 already uses for every other seed. Two independent axes, not one taxonomy: how a
game is *distributed*, and what language runtime (if any) it needs.

| Axis | Image | Covers |
|------|-------|--------|
| Distribution | `base-linux` | Anything with its own native Linux binary (a `download`-method install, or a game whose image contains everything, like Bedrock's) |
| Distribution | `base-steamcmd` | Any Steam-distributed game with a native Linux build |
| Distribution | `base-steamcmd-proton` | Any Steam-distributed game with **no** native Linux build |
| Runtime | `base-java` | A JVM-based server with no runtime image of its own |
| Runtime | `base-dotnet` | A .NET-based server |

Paper still pulls `eclipse-temurin` directly rather than `base-java` — that was already working
and proven, and switching it over for consistency alone would be change for its own sake with no
seed-visible benefit. `base-java`/`base-dotnet` exist for the *next* JVM- or .NET-based seed,
which is the concrete case that prompted building them (ADR-047's own bar for adding to this
library: build it when something needs it, not speculatively).

`.github/workflows/images.yml` builds and publishes all five to GHCR
(`ghcr.io/<owner>/base-linux`, `.../base-steamcmd`, `.../base-steamcmd-proton`, `.../base-java`,
`.../base-dotnet`) on every push to `develop`/`qa`/`main` that touches `images/**`, under the
same floating per-branch channel tags (`develop-latest`/`qa-latest`/`release-latest`) the binary
release workflow uses (ADR-038), plus an immutable `:<short-sha>` tag for anyone who wants to pin
rather than track a channel. `base-steamcmd-proton` pins its own `BASE_IMAGE` build arg to the
same-run `base-steamcmd` sha tag, for the identical reason `base-steamcmd` already pins
`base-linux`'s; `base-java` and `base-dotnet` each pin `base-linux`'s sha tag directly, the same
way `base-steamcmd` does.

## Backups

Backups cover per-server writable state, cluster volumes, and sidecar data — never installs.
Phase 8 built manual backup and restore, triggered from the UI, and resource graphs
("Resource graphs" below).

### Scheduled backups

A server or cluster may have at most one recurring backup schedule (ADR-045): an interval
preset (hourly, every 6h, every 12h, daily, or weekly — not cron syntax, which homelab scale
doesn't need) set from the same backups page a manual "Back up now" lives on.
`internal/control/scheduler` checks once a minute for schedules whose `next_run_at` has passed
and fires them through `internal/control/web.Server`'s `TriggerServerBackup`/
`TriggerClusterBackup` — the exact job-creation and dispatch path a manual click already uses,
not a second implementation of it.
A fired schedule's `next_run_at` always advances from the moment it fires, never from the missed
time, so a schedule blocked by an offline node or an in-flight job retries on the next tick
rather than hammering every tick forever, and downtime never produces a backlog of catch-up
backups.

Tarring a live database directory produces a corrupt database directory, so containers may
declare `pre` and `post` backup hooks (`pg_dump` and cleanup) that run inside the container
before the archive is taken, via `Runtime.Exec` (ADR-042, amending ADR-010). A hook whose
container is not running is skipped rather than failing the backup; one that runs and exits
non-zero aborts it. Cluster volumes are backed up once per cluster, never once per member
(ADR-023) — a cluster backup is its own action from the cluster's own UI control, not something
folded into any member's.

The archive itself is a single `.tar.gz` per backup, written to the agent's `backups_dir`
(node-local — restores read it back on the same node, no transfer involved in this phase) with
top-level members `writable/<from>/...` (the seed's `server.writable_paths` opted into
`backup.include`), `volumes/<role>/<from>/...` (sidecar volumes), and `cluster/...` (a cluster's
shared volume, present only in a cluster backup). Restore extracts the matching members back;
if the server was running, it is stopped first and restarted afterward, reusing install-update's
stop-before/restart-after bookkeeping (ADR-042) scoped to one server instead of a fleet.

## Resource graphs

The console page's live stats panel (`internal/control/web.handleServerStats`, polled every 5s
by `stats.js`) shows one reading at a time; it never showed history. `internal/control/telemetry`
closes that gap without adding a time-series database or a JS charting library (ADR-046).

A `Collector`, shaped exactly like `internal/control/scheduler`'s ticker, samples every server
whose observed state is `running` or `degraded` once a minute through the same `hub.Hub.Stats`
method the live panel already calls, and writes one row per container role into the
`stats_samples` table — a bounded ring table (ADR-005's amendment on high-frequency telemetry)
pruned back to a 7-day retention window on every tick. A server with nothing live to report is
skipped rather than logged as a failure, the same judgment `stats.js`'s poller already makes
for one missed live sample.

The console page's graphs panel (`graphs-panel`, alongside `stats-panel`) fetches a rendered
fragment from `/servers/{id}/stats/history` every 60s via `graphs.js`, mirroring `stats.js`'s
own fetch-and-swap shape. `internal/control/web/graph.go`'s `renderGraph` turns
`[]store.StatsSample` into a small inline `<svg>` polyline per metric — CPU, memory, and
network rx/tx — scaled to whichever series peaks highest in the window, with no client-side
computation and no vendored library (ADR-006). A fixed set of window presets (1h, 6h, 24h, 7d)
is offered as plain buttons rather than a free-form range picker, the same homelab-scale call
ADR-045 made for backup intervals.

Network rx/tx are **charted as a per-second rate** (ADR-075), not as the cumulative counters
Docker and Podman actually report — a raw counter only ever rises, so its slope carried all the
meaning and the graph answered little beyond "has there been traffic". Differencing needs the
two cases ADR-046 deferred it for, and both are handled: a counter that goes backwards means the
container restarted, so the line breaks rather than plotting a negative dip, and a gap wider
than a few sample intervals breaks it too rather than interpolating across an agent outage.

The x axis starts at the **first real sample** rather than at the requested window's start, with
both ends labelled in clock time — five minutes of history inside a 7d window previously drew in
the rightmost 0.05% of the width. Windows longer than the plot is wide are downsampled to a peak
envelope (each bucket keeps its maximum, never its mean, so a spike survives), and the four
series carry palette-independent colours of their own for the same reason semantic state colour
does: "in" and "out" have to stay tellable apart whichever accent is active.

## The UI is translatable

Every operator-facing string goes through `internal/i18n` (ADR-086), keyed by its own **English
source text** rather than an invented identifier. A template reads as English prose
(`{{T `Back up now`}}`), and a message with no translation falls back to its key — which is
already correct English. So a partially translated language is usable, and there is no state in
which a page shows a raw identifier or a blank.

Templates are parsed **once per shipped language**, because a `FuncMap` is fixed at parse time
and `T` must close over the language it renders in; `render()` then selects a pre-parsed set and
does no per-request template work. Language resolves as the operator's stored preference
(`ui.language`, a runtime setting), then the browser's `Accept-Language`, then English — read per
request, so a change takes effect on the next page load with nothing to invalidate. `<html lang>`
declares whichever language actually rendered.

**Plural forms come from CLDR via `x/text`, not from the catalogue**, so a language distinguishing
one/few/many selects correctly the moment its file is added, with no code change. That is what
makes adding a language a data change: drop a locale file beside `en.json` and it appears in the
setting, the matcher and every page. Formatting follows the locale too — German writes
`64,0 MiB` — while unit symbols and seed-schema literals (`patch`, `tcp`, `amd64`) deliberately
stay English, since translating either would make it wrong.

The English catalogue is **generated from the source by the same test that checks it**, the
discipline `internal/seed/schema.json` already uses. Three guards close the silent failures: a
used key missing from the catalogue, an orphaned entry (what an edited English string leaves
behind), and a coverage ceiling that may only come down.

**English is the only catalogue that ships.** The machinery is complete and exercised; what is
absent is translated content, which nobody here can verify — and a machine translation of ~790
strings would put unreviewable text on destructive actions. Because that leaves every
language-selection path unexercised by construction, a test injects a second catalogue and drives
a real request through `render()`, asserting it renders in that language, declares it, and falls
back to English for a key it lacks.

One limit is carried forward rather than read as done. The ~150 template fragments that were
sentences split by inline markup — each half a sentence that could not be translated alone — are
done: `TH` carries a message with its markup, extraction is at zero untranslated fragments, and the
coverage report that counted them down is a gate rather than a report. What remains is that
**seed content is specified but not built** — a seed's own labels and descriptions are the
majority of what an operator reads on a settings page, and ADR-086 records the design: per-language files inside the bundle
(`<seed-id>/locales/<lang>.yaml`), keyed by the declared control name rather than by source text
(a seed author may not write in English), with a stale entry as a lint warning rather than a load
error and missing entries falling back to the seed's own text.

## Operator settings

Configuration splits on one question: **is it needed before there is a database to read?**

`yggd.yaml` keeps what is — listen addresses, `data_dir`, the database path, `seeds_dir` —
resolved once at startup into an immutable struct, and changing one means restarting the
process that read it. Everything else is a **runtime setting** in the `settings` table,
adjustable from `/settings` while yggd runs (ADR-078).

Resolution is **database → yggd.yaml → registry default**, and the page says which is in
effect, so a value an operator put in the file is never silently overruled without
explanation. Clearing an override is a delete rather than a write of the empty string: those
are different states, and "give me the default back" is the first one.

The table is key/value; what a key *means* lives in `internal/control/settings` as a registry
of typed `Definition`s. So adding a setting is a Go declaration plus a consumer — no
migration, no template edit, no new form field, because the page renders whatever the registry
declares. Six ship, each with a live consumer: `log.level` (the process's own
`slog.LevelVar`, so debug can be switched on and off without a restart), `stats.retention_days`
(read by `telemetry.Collector` on every prune, so lowering it frees space within a minute),
`steam.api_key`, `seeds.catalog_repo` (where the Seeds page downloads from, ADR-081), and the
pair that lets this control plane publish its own seeds — `seeds.publish_repo` and
`seeds.publish_token` (ADR-079). That last one is the first credential here that writes somewhere
outside this deployment, which is why its target has no default and why the one repository it must
never name is refused rather than merely discouraged. Its catalog sibling is the mirror image: a
read is safe, so it has a default and every control plane gets the project's catalog without being
configured.

**A secret's value cannot reach the page**, as a property of the types rather than a rule the
template is trusted to follow: `Manager.Resolve` — what the page reads — reports only whether
one is *configured*, and `Manager.Value` — what a consumer reads — returns the real thing. An
empty submission means "leave the stored one alone", so removing one is its own Clear action.

**Every setting must name itself in `web.applySetting`**, whose default case is an error. A
setting an operator can save that nothing reads is this codebase's most-repeated bug (ADR-067,
ADR-071, ADR-077) and a settings page hides it especially well, because saving appears to
succeed; a test saves every registry key so a new one fails there instead.

The **release channels are shown but not owned**. Switching one archives the running binary and
changes what Update installs, so ADR-056 deliberately keeps that setting on the page performing
the action; `/settings` reports both with a link. A **node's** own settings — its address, its
port range — likewise belong to that node and stay on its page.

`store.Open` narrows the database file to `0600`, since it holds password hashes, session
tokens, bootstrap tokens and now credentials. Defence in depth rather than the boundary: the
containing directory is already `0750`, which is why a `chmod` failure is non-fatal.

## Security model

Scope is a homelab: a single trusted operator, not untrusted tenants.

- **Human auth** — single admin account, argon2id, session cookies. API tokens for scripting.
  A `role` column exists from day one so adding RBAC later is not a painful migration, but no
  RBAC logic exists in v1.
- **Perimeter** — the UI is reachable only over the LAN or through the Cloudflare tunnel,
  which provides external authentication. `yggd` never binds to a public interface.
- **Agent auth** — mTLS with certificates from the control plane's internal CA, separate from
  human auth.
- **Agent binaries** are signed and verified before replacement — the self-update path fetches
  and executes code, so it gets real cryptographic verification, not a checksum alone.
- **File and backup-hook operations** are path-sandboxed with symlink escape checks (ADR-033).
  Backup hooks execute seed-supplied commands inside containers, which is acceptable at the
  homelab trust level and would not be for untrusted tenants.
- **Docker socket access** makes the agent effectively root on its host. Inherent to the
  design and the main thing needing revisiting for untrusted tenants.

## External dependencies

| Dependency | Purpose | Required? |
|-----------|---------|-----------|
| Docker Engine | Container runtime — the only execution path | Yes, per node |
| SQLite | Control plane persistence (pure-Go driver, no cgo) | Yes |
| systemd | Agent supervision and restart-on-update | Yes, per node |
| SteamCMD | Installing Steam-distributed games (ARK, Valheim) | No — runs in a container on the node, not on the host (ADR-057) |
| SteamCMD (control plane host) | Seed authoring UI's Linux-depot check (ADR-051) | Optional |
| Cloudflare Tunnel | External UI access | Optional |

SteamCMD runs in a **container** on the node, not on the node host (ADR-057): the agent calls
`Runtime.RunOnce` with `images/base-steamcmd` and the install directory bind-mounted in, so a
node needs only the container runtime it already needs for every game server. It used to shell
out to a `steamcmd` binary on the host, which made SteamCMD a per-node prerequisite an operator
had to satisfy by hand and failed, where they had not, as a puzzling install-job error rather
than a missing dependency.

The container runs as the agent's own uid:gid, so the depot it writes stays owned by the process
that has to refcount, mount, and eventually delete it. **That makes "runnable as an arbitrary
uid" part of `base-steamcmd`'s contract, not an incidental property** — SteamCMD writes at the
`/opt/steamcmd` root (`steamapps/`, `userdata/`) and self-updates into `linux32/`, so the image
chmods that tree writable at build time. Removing that line does not fail loudly: installs die
with `Missing configuration`, an error naming neither the uid nor the unwritable path, which is
exactly how it went unnoticed once before (ADR-057's amendment). `RunOnceSpec.User` is pinned by
the `Runtime` conformance suite so the uid dimension is exercised rather than assumed.

The image is overridable via the agent's `steamcmd_image`, for an air-gapped node mirroring it
locally, an operator pinning a sha tag instead of tracking a floating channel, or taking a fix
from a channel ahead of the default — the default is `release-latest`, built only from `main`,
so without the override a fix on `develop` cannot reach a node until it is promoted. What a node
does need either way is to be able to *pull* that image: the source repository is private, so the
GHCR package must be readable, or mirrored locally and the override pointed at the mirror.

`images/base-steamcmd` ("Container image library" above) now serves both of its uses: this one,
and a *seed's runtime container* for a game that wants SteamCMD available to validate or
self-update itself at container start. These were separate layers that never met until ADR-057;
the install path is now the image's primary consumer.

No message broker, no external cache, no separate reverse proxy. The control plane is one
binary and one database file.

## Delivery phases

| Phase | Status | Content | Exit criteria |
|-------|--------|---------|---------------|
| 0 | **Done** | Repo, docs, proto contract, config, DB schema, both binaries build and start | `yggd` serves a health endpoint; `ygg-agent` starts and logs |
| 1 | **Done** | Enrollment, mTLS, persistent stream, protocol version negotiation, node registry with architecture, node list UI | A remote agent appears in the UI and survives restarts of both sides |
| 2 | **Done** | `Runtime` interface + Docker impl, `PodSpec` (single-container), labelling, adoption, supervisor state machine, start/stop/kill | `kill -9` the agent while a server runs: the server survives, and the restarted agent adopts it with correct state |
| 3 | **Done** | Console streaming (xterm.js) with stdin and multi-viewer fan-out. File browser, editor, and upload, sandboxed per server (ADR-033) | Play a full session end-to-end from the browser |
| 4 | **Done** | Seed schema, Job entity, install pipeline with progress, config templating, allocations UI | Minecraft **Java and Bedrock** run from seeds alone, with no game-specific Go code |
| 5 | **In progress** | Shared installs, refcounting, read-only mounts with writable overlays, clusters, orchestrated install updates | **ARK**: one install, five maps, character transfer between them, one-button update that stops and restarts exactly what was running |
| 6 | **Done** | Multi-container pods: per-pod networks, dependency ordering, health gates, reverse-order shutdown, `degraded`, sidecar log/stat views | Start a game + database + broker pod in dependency order behind a health gate, crash-loop a sidecar into `degraded` and recover it, read each container's own logs and stats, and stop the pod in reverse order with the primary first |
| 7 | **Done** | Signed agent binary distribution, UI-triggered update, N-1 compatibility testing | Update every agent from the UI while servers stay up — including an agent four protocol versions behind, with its server running throughout (`hack/n1-check.sh`) |
| 8 | **Done** | Manual and scheduled backup and restore with per-container hooks, cluster volume backups, and resource graphs | Back up a multi-container pod, run its `pre` hook, destroy the database sidecar's data, restore it, and bring the pod back up running |

**Phase 2 must model a server as a pod from the start**, even though every pod has exactly one
container until phase 6. A server that owns a *set* of containers with one primary costs
almost nothing to build in phase 2 and is an invasive refactor to retrofit in phase 6, since
console attach, stats, adoption, and the state machine all change shape.

Phase 2's exit criterion is deliberately brutal, because agent statelessness is the property
the entire update story rests on. If it is not proven by killing the agent under load in phase
2, it will not hold in phase 7.

**Phase 5's mechanism is built and verified, but not yet against ARK by name.** Shared installs,
refcounting, read-only mounts with writable overlays, cluster creation and joining, and the
orchestrated install-update job (stop exactly what's running, update, restart exactly what was
stopped) are all implemented and proven — against a real control plane, a real `ygg-agent`, and
real Podman, no fakes — using a synthetic seed shaped identically to `ark-survival-ascended.yaml`
(one shared install, five map servers, one cluster) rather than the real thing, since ARK
Survival Ascended has no official Linux server: a real install needs a SteamCMD-plus-Proton
wrapper. Driving that synthetic seed live also caught a real bug now fixed: `Provision` was not
publishing a state change for a freshly provisioned but not-yet-started server, so a seed-driven
server that is not started immediately after creation never reported its containers upstream and
the "does this server have any containers yet" reconcile gate never cleared, leaving it stuck on
"installing" in the UI forever.

The SteamCMD-plus-Proton wrapper itself is no longer the open half of this: `images/base-
steamcmd-proton` ("Container image library" above) is built, and — unlike the plain "it builds"
bar most of this table otherwise uses — actually proven to run Wine/Proton for real: `umu-run
createprefix` produces a genuine, fully-populated Wine prefix through the real container
entrypoint (ADR-047's amendment has the full account, including two bugs — a wrong env var name
and a root-vs-non-root permission error — that only surfaced by actually running it, not from
reading the Dockerfile). The bundled `ark-survival-ascended` seed now points its primary
container at `base-steamcmd-proton` directly (no separate `yggdrasil/ark-asa` image — nothing
ARK-specific belongs baked into one) and invoked `umu-run` against ARK ASA's documented Windows
binary path — the invocation that later moved behind `ygg-proton`, for the reasons the rest of
this entry works through.

That path is no longer just documented, either: a real `+app_update 2430930` (~12 GB) has
actually been fetched and run against this image. The exe path was exactly right, and the engine
genuinely starts — `ShooterGame.log` shows a real `ARK Version: 92.28` line, not a synthetic
stand-in — once two more real, confirmed bugs were fixed (`PROTON_USE_XALIA=0`, needed by any
Windows game this image runs, not just ARK; `PROTON_USE_WINED3D=1`, ARK-specific, avoiding a
VKD3D-Proton D3D12 crash `-nullrhi` does not prevent). It still does not reach a ready state via
`umu-run`: it stalls indefinitely inside Sentry/Crashpad's startup sequence, a specific,
reproduced, partially-diagnosed problem whose only workaround found so far does not fit
ADR-018's read-only install model.

A known-working reference image (`ghcr.io/parkervcp/steamcmd:proton`, with a documented ARK ASA
track record) reframed this further: it invokes GE-Proton directly, never through `umu-launcher`,
and testing that path exposed and fixed a real, structural gap — GE-Proton needs a newer glibc
than Debian bookworm ships, silently papered over by `umu-launcher`'s own sandboxed runtime.
`images/base-linux` and everything built on it moved to Debian `trixie` as a result (ADR-047's
third amendment has the full account, including two further regressions that move surfaced and
fixed: `umu-launcher`'s vendored Python extension needed the `debian-13` build, and
`base-dotnet`'s apt repo needed a different signing key entirely on Debian 13). Raw Proton then
got substantially further than `umu-run` ever did — past a missing prefix directory, a relative-
path resolution difference, and a native Steamworks bridge library (`lsteamclient.so`) whose
real search path was extracted directly from the compiled binary via `strings`.

**A real ARK Survival Ascended server now boots.** Reading the matching Pelican egg alongside that
image closed the remaining gaps (ADR-047's final amendment has the full account): the per-server
writable directory has to be writable by the container's user or Unreal silently redirects its log
to Wine's stubbed Windows event log and dies with nothing to read; `/etc/machine-id` has to exist;
and — the actual indefinite hang, found with `WINEDEBUG=+loaddll` — ARK's depot ships its own
Windows `steamclient64.dll`, which wins on application-directory precedence and then blocks forever
waiting for a Steam client that is not there. With those fixed, a real depot boots to
`Server: "…" has successfully started!` in about 18 seconds with the game port bound and RCON
listening, **with the install mounted read-only and per-server writable overlays exactly as ADR-018
specifies**. The invocation details now live in `ygg-proton` inside the image, so the seed's command
is a single readable line.

The exit criterion nonetheless **stays open**, for reasons worth stating plainly rather than
rounding off: the Steam subsystem reports `FAILED` (a direct consequence of the `steamclient64`
override — expected to be harmless since ASA lists through Epic Online Services, but no real game
client has connected to confirm it); this proves one server booting, not phase 5's five-maps-one-
install-with-character-transfer workflow; and it was driven with Podman directly, not yet through
`yggd` and `ygg-agent` end to end.

**Phase 6 is done, against a criterion that was reworded to what the phase actually proves** —
the same correction phase 8 already made, for the same reason, and it should have been made at
the same time. It originally read "**Dune Awakening**: game + database + broker start in order and
stop safely", and every word of that except the game's name was met: dependency-ordered start,
health gates, reverse-order shutdown with the primary first, sidecar crash-loop restart into
`degraded`, and the console role selector and stats panel are all implemented and proven by a
full-stack test (real control plane, real `ygg-agent`, real Podman, no fakes) and a live browser
session against a genuinely running multi-container pod.

What was never met is the name. **Dune Awakening has no public dedicated-server image**, so there
is no seed to build against and no amount of work in this repository can produce one — the
criterion was hostage to a third party's release decision rather than to anything here. Phase 8's
row was reworded on exactly that reasoning ("naming that game in a backup phase's criterion made
this phase's completion hostage to a dependency that has nothing to do with backups") and then
pointed at this row as the place the dependency legitimately belonged. That was wrong: the
dependency does not belong anywhere. A multi-container pod is the capability, and Dune Awakening
was only ever the example that motivated it.

So the criterion now describes what a multi-container pod must do, and it is met against the
synthetic game+database+broker seed — which is the *right* target rather than a stand-in, because
what is being proven is the pod mechanism and a synthetic seed exercises its corners more
deliberately than any one game would. **A real Dune Awakening seed remains wanted** and is
tracked as catalog work in `arasoi/yggdrasil-seeds` where every other game lives (ADR-087), not
as a phase gate here.

The honest limit, carried forward rather than read as covered: no seed in the published catalog
declares more than one container, so the multi-container path has never run against a game an
operator actually plays. That is a gap in the *catalog*, and it is the same gap whether this row
says "in progress" or "done".

**Phase 7's mechanism is built and verified end-to-end, including live in the browser**: sign
and register a binary, click Update on a connected node, and watch a real agent process
download, verify, install, drain, exit, and — restarted the way systemd would — reconnect
running the new version with the job resolving automatically. See "Agent updates are pushed
from the UI" above for the full sequence and ADR-040/ADR-041 for the design.

**The N-1 half is closed too, and the criterion is met in full.** It could not have been when it
was written: `Protocol == MinProtocol == 1`, so there was no older peer in existence to build,
and the negotiation could only be exercised against constructed ranges. The protocol is 5 now
with the floor still at 1, so four older peers genuinely exist — and `hack/n1-check.sh` builds
one from each of the commits that shipped them (0.11.0 at protocol 1, 0.39.0 at 2, 0.40.1 at 3,
0.44.0 at 4) and runs the whole criterion against them. Verified end to end on a real control
plane with real Podman:

- all four negotiate their own protocol against a protocol 5 control plane, not the newest, and
  all four are flagged as behind in the UI (ADR-014's own promise, which had never been checked
  against a binary that was really behind);
- a real container runs on the **protocol 1** node, four protocol versions and 37 releases
  behind the control plane driving it;
- that agent updates itself from the UI — downloads over `DownloadAgentBinary`, verifies the
  digest and the signature against the key it pinned at enrollment, installs, keeps the old
  binary at `.previous`, drains and exits zero;
- **the container keeps running through the window when no agent exists at all**, which is the
  half of "while servers stay up" that a same-version update never really tests;
- the restarted agent comes back as 0.48.1 at protocol 5, on the *same node id* — identity
  survived, no re-enrollment (ADR-052's blocker never fires) — and adopts the running server
  from its container labels;
- the `agent_update` job resolves itself on that reconnect, which is the only completion signal
  this job type ever produces (ADR-021's amendment);
- and it is the same container with the same start time, so it was never restarted.

The one thing not proven is a *cross-architecture* update, since both binaries here are built
for the host. That is the same limit every live verification in this document has, and the
architecture is checked before an update is offered rather than at install time.

**Phase 8 is done, against a criterion that was reworded to what the phase actually proves.**
It originally read "restore a Dune pod including its database", and that sentence was met
literally: a real control plane, a real `ygg-agent`, and real Podman back up a multi-container
pod (primary + database sidecar with a volume), run a pre-hook standing in for `pg_dump` via
`Runtime.Exec`, destroy the sidecar's live data, restore it, and bring the whole pod back up
running. But it was met against the same synthetic game+database+broker seed phase 6 uses, for
the same reason — there is no public Dune Awakening image to build a real seed against — so
naming that game in a backup phase's criterion made this phase's completion hostage to a
dependency that has nothing to do with backups. The criterion now describes the capability
that was demonstrated.

This paragraph used to end "**Dune Awakening by name remains phase 6's open item**, where the
dependency actually belongs", and that was half right. Moving the dependency off a backup phase
was correct; parking it on phase 6 was not, because it does not belong on any phase — a game with
no public server image cannot be a gate on work that is finished. Phase 6's row has since been
reworded on the same reasoning, and a real Dune Awakening seed is catalog work rather than a
phase item.

Nothing is outstanding from the original scope. What the table's row bundled in but ADR-042
deferred as its own follow-on work rather than shoehorning into that round — a cron-like
backup scheduler, and resource-usage graphs — are both built: the scheduler as an
interval-based one reusing the manual backup path (ADR-045, "Scheduled backups" above), and
the graphs as a bounded-ring-table collector and server-rendered SVG (ADR-046, "Resource
graphs" above).

Two limits are worth carrying forward rather than reading "Done" as covering them. Backups are
node-local: an archive never leaves the node it was taken on and a restore reads it back there,
so there is no browser download and no off-node retention (ADR-042). And the network series on
the resource graphs is a cumulative byte counter as the container runtime reports it, not a
computed rate (ADR-046).
