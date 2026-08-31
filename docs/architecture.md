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
**Free disk is the exception, refreshed on every heartbeat** (ADR-134), unlike cores and memory
which only change at handshake — it moves while the node stays connected, and a figure only
collected at handshake would describe stale state. `NodeCapacity` was already sent on every
heartbeat and discarded on all of them, so this needed no new poller. A node below
`nodes.disk_low_percent` (default 10) is flagged on its own page, the node list, and the fleet's
node group; an agent too old to report free space reads as unknown rather than full, since the two
must not be confused (conventions.md's node-capacity rules cover the unknown-vs-zero enforcement in
full). Nothing is refused or scheduled on it — an install that will not fit still fails the way it
did, this only makes the reason visible, which matters because low disk surfaces from SteamCMD as
an error naming neither the disk nor the space (ADR-089).
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
exists for (ADR-048). A base port and its offset siblings — including a same-number sibling on a
different protocol (Vintage Story's one number on both TCP and UDP) — allocate as one atomic group
(`store.AllocatePortGroup`, ADR-083), so a collision retries the whole group rather than stranding
one member with the other already claimed. An offset must be zero or positive, so a derived port's
number is never below its own base's — zero is the identity case a same-number sibling needs, and
anything negative would put the "base" above its own "derived" port (ADR-109). See conventions.md's
ports-and-allocations rules for the mechanism.

**A port number, once allocated, is claimed on every protocol, not only the one holding it**
(ADR-109) — two servers cannot land on the same number just because one asked on tcp and the
other on udp, and neither can one server's own two ports. The only exception is the
atomically-inserted same-number sibling group above; an operator's own pinned port is held to the
same node-range rule as a seed's declared default, since only an offset-derived port may
legitimately sit outside it. See conventions.md's ports-and-allocations rules for the enforcement.

**A node's port conflicts can be found and repaired from the Allocations page.** A conflict —
a number two unrelated servers both ended up holding, which the rule above no longer permits a
fresh allocation to create — can still exist from before the rule did, or from an operator's
own pin, or a race this project has since closed. `/allocations` groups every reservation by
node, with that node's clusters nested inside it the same way the fleet page groups server
cards (ADR-054's amendment); a node with a conflict shows which port numbers and offers
"Remap conflicting ports", which releases and reallocates only the losing side of each
conflict — the earliest-created holder of a number keeps it — and rebuilds those servers'
pods, sequentially, so the new numbers actually reach the running containers.

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

**A port may declare a fixed `container_port`, for the one case neither of those covers**
(ADR-150): an off-the-shelf image with no seed-controllable way to change what it listens on at
all, not through the command and not through an env var. Adminer's own standalone server always
binds 8080 regardless of what host port it is published under — the case a database-admin addon
(ADR-147) needs this for. `container_port` publishes the allocated host port to that fixed
number instead of to itself; unset, the ordinary rule above still applies. Found live: a real
FiveM server's Adminer addon showed as `running` and reachable per the container list, yet every
connection was refused, because the allocated host port was being published to itself and
nothing inside the container was listening there — the same silent failure shape ADR-061
describes, one layer further down the stack, where an image gives the seed no lever to pull at
all rather than the wrong one.

**Job** — a long-running operation with streamed progress: install, install update, backup,
restore, server move, agent update. One mechanism rather than six (ADR-021).

**An install stuck by a control-plane restart self-heals**, up to a bounded number of attempts,
and stays repairable by hand past that (ADR-106). `ReconcileInterruptedInstalls` (ADR-021,
ADR-101) leaves any install with at least one server still waiting on it ready to retry rather
than marking it failed outright — resuming is sound because SteamCMD's own
`app_update ... validate` reconciles a partial download in place (ADR-091). See conventions.md's
install-lifecycle rules for how the retry cap works and how dispatch (on the install's node
reconnecting, via `RetryStuckInstalls`) divides from this reconciliation. An install genuinely
reported failed by its agent — a real error, not an interruption — is never touched by self-heal
at all, only by the uncapped **Retry** action on `/installs` and on a crashed server's own page.

**An install `validate` calls healthy can still be wrong, and `/installs` offers a way out**
(ADR-142). Retry and self-heal both lean on SteamCMD's own `app_update ... validate` reconciling
a depot in place rather than re-fetching it (ADR-091), which is the right default and is not
infallible — a depot corrupted in a way `validate` does not catch reads as fine forever. **Force
clean** re-dispatches the same install with `InstallStart.force_clean` set, which tells
`Installer.Install` to wipe the directory before running any step even for a method that would
otherwise reconcile in place. Unlike Retry, it carries no state gate: it exists precisely for an
install the state machine still calls `Ready`, and is refused only while a job is genuinely
active on it. Deleting the install and letting it be recreated — ADR-091's own suggested
escape hatch — was never actually reachable for this case, since `DeleteInstall` refuses
unconditionally while any server still references it (ADR-018); Force clean is what that
suggestion needed and did not have until now.

**An install job's completion provisions before it restarts anyone waiting on it** (ADR-126).
`handleInstallProgress`'s success case reconciles (provisions whatever needs it) before restarting
whoever ADR-106's `AddJobServerRestart` recorded — order that matters when a Rebuild triggered the
job, since `reprovisionServer` destroys a server's pod *before* dispatching the install its new
seed needs, leaving it with zero containers by the time the job succeeds. Restarting first sent a
bare Start racing ahead of the Provision that would have created something to start, failing
silently with `ErrPodNotFound` and leaving the server `offline` with `Desired` stuck at `running`
**permanently** — the reconcile that ran afterward made it look already-settled to the next page
load, so nothing else would have corrected it. Provisioning first closes the race. See
conventions.md's job-lifecycle rules for the full incident account, including why the fix is a
no-op for install-update and restore.

**A server left `Desired: running` survives a host reboot in the database, but nothing brought
it back until now** (ADR-013, ADR-140). ADR-013 promised this from the day restart authority
moved to the agent — "a host reboot brings servers back through ordinary desired-state
reconciliation rather than a second, separate mechanism" — and no code path ever did it. Docker's
restart policy is `no`, so a container stopped by anything other than the agent's own graceful
path (a host reboot chief among them) stays stopped; a freshly-adopted agent assumes desired
matches whatever it finds running (ADR-012), so nothing on either side ever asked the control
plane to restart it. `Hub.applySnapshot` is the one point every server's `Observed` column for a
reconnecting node holds its fresh, post-reconnect value at once, so `ReconcileRunState` runs
there: for every server whose `Desired` is running and whose freshly-recorded `Observed` is
`offline` or `crashed`, it dispatches the same Start an operator's own button click would.
Deliberately narrower than the drift condition `serverView.Drifting()` already renders
per-server — `installing`, `starting`, `stopping` and `degraded` are all excluded, since each
already has its own owner (an install job, an in-flight command, or the agent's own crash-loop
backoff for a sidecar) that an unconditional Start would race against rather than help. Wired the
same way `RetryStuckInstalls` is (`hub.DesiredStateReconciler`, satisfied by `*web.Server`, set
once from `cmd/yggd`): a hub with none simply does not reconcile run state, and a failure here is
logged and left for an operator to retry by hand, since a server left down is still visible as
drift on its own page.

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

**Every directory has an owner that removes it** (ADR-088) — before this, none did, so a fleet
with a scheduled backup (ADR-045) accumulated archives indefinitely and installs had no deletion
route at all. One agent-side command, `ReclaimData`, frees a server directory, an install
directory, or a backup archive, shaped like `DeleteClusterVolume` (ADR-066) since none of these
sit inside the per-server file sandbox (ADR-033). Deleting a server frees its directory and its
archives, a move frees the source node's copy, deleting an install frees its files once nothing
references it, and the scheduler prunes archives past `backups.retention_days`.

**Which install has an owner depends on whether it is shared** (ADR-096; conventions.md's
ownership rule, backed by `resolveInstall`/`releaseInstall`, states the same distinction). A seed
declaring `shared: false` gets a new install per server that nothing else can ever reference, so
its lifetime is that server's and it is removed with it — on delete, on a move (the install does
not travel, so the source's copy is immediately unreferenced), and on a failed creation. A
*shared* install at refcount zero is deliberately kept as the cached depot the next server from
that seed reuses, staying on `/installs` with its server count reading zero rather than being
removed. The distinction was missed when the list above was drawn up, and a per-server install
stranded 19 GB on every ordinary delete before it was added.

Reclamation is best-effort: a pre-protocol-4 agent ignores `ReclaimData`, so a caller logs the
failure and still deletes the row rather than refusing — the cost is an orphaned directory an
operator can still remove by hand. Ordering follows whichever check can still refuse: a server's
archives are listed and reclaimed before the row that would cascade them away, while an install's
row goes first, since reclaiming before a refused delete would wipe a still-mounted install. The
agent's own startup sweep, run after adoption, is equally narrow — only an exited one-shot
installer (ADR-057) and an unattached pod network, never a container the control plane simply
didn't mention (ADR-031). See conventions.md's ownership-and-reclamation rules for the full
statement.

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
    Start(ctx context.Context, spec PodSpec) error          // dependency-ordered, health-gated
    Stop(ctx context.Context, spec PodSpec, stop StopSpec) error
    Kill(ctx context.Context, id ServerID) error
    StartContainer(ctx context.Context, id ServerID, role string) error
    Attach(ctx context.Context, id ServerID, role string) (Console, error)
    Exec(ctx context.Context, id ServerID, role string, cmd []string) ([]byte, int, error)
    RunOnce(ctx context.Context, spec RunOnceSpec, out io.Writer) (int, error)
    Stats(ctx context.Context, id ServerID) (map[string]Stats, error)
    Status(ctx context.Context, id ServerID) (PodStatus, error)
    Destroy(ctx context.Context, id ServerID) error
    Adopt(ctx context.Context) ([]PodStatus, error)         // rebuild state from labels (ADR-012)
    Events(ctx context.Context) (<-chan Event, error)       // exit watching beats polling
    ReapOrphans(ctx context.Context) (OrphanReport, error)  // startup leftover sweep (ADR-088)
}
```

`Start` and `Stop` take the full `PodSpec` rather than an id, because after an agent restart
container labels carry identity only, never configuration (ADR-012) — the dependency graph
must arrive fresh on every call. The three widenings past the original lifecycle set each
carry their own ADR: `Exec` (ADR-042), `RunOnce` (ADR-057), `ReapOrphans` (ADR-088).

`PodSpec` carries one or more `ContainerSpec` entries, their dependency edges, mounts, and
allocations. Console attach and stats are role-addressed because a pod has more than one
container to look at.

**Graceful stop is game-specific.** Most game servers want a console command (`stop`,
`quit`, `exit`) rather than a signal, and losing that distinction corrupts world saves.
`StopSpec` carries `{ConsoleCommand, Timeout, ThenSignal}`: write the command to the primary
container's stdin, wait, escalate to SIGTERM, then SIGKILL.

**The `StopSpec` rides on the stop command, exactly like the `PodSpec` above, and the agent keeps
no copy of it** (ADR-129) — a cached copy here once silently overrode the real spec (populated
from `runtime.DefaultStopSpec()`), so every seed's graceful-stop command went unused in favor of a
bare SIGTERM until the cache was removed. See conventions.md's rule against caching what a
per-command message already carries.

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

**A setting the seed marks `cluster_default` is managed from the moment the cluster exists**
(ADR-112), rather than starting empty and waiting on an operator to open `/clusters/{id}/settings`
and check boxes by hand. `handleCreateCluster` seeds the cluster's own `settings` column, right
after writing the row, with every `cluster_default` setting's own declared default — the exact
values `/clusters/{id}/settings` would otherwise need an operator to enter one by one before a
single member could inherit anything. This is deliberately narrow: it exists at cluster
*creation* only, never retroactively applied to a cluster made before a seed gained the field, and
it covers `settings` only — a *variable* is far more often the thing meant to differ per member
(ARK's own `map`), so there is no equivalent for `variables`. Everything downstream is unchanged:
a member created afterward inherits exactly as `storedFor` already composes, and an operator can
still override any one control per server through the same `manage_<name>` toggle described below.

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
| `ygg.pod.primary` | Boolean; marks the primary container, read independently of `ygg.pod.role` at adoption |
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
method/url/archive/filename/app_id fields a current control plane still writes alongside the steps.
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

**A release's binaries do not necessarily share one version** (ADR-111): a push rebuilds only
the binaries it actually affected, and the rest are carried forward byte-identical at the
version they were last built at — so a yggd-only change no longer restamps the agent binary,
and nodes stop reading "Rebuild ready" for agent code nobody touched. The manifest carries a
per-binary `binaries` map; `channel.version` is the yggd binary's version. Each consumer reads
its own binary's entry: yggd's self-update compares the running build against `binaries.yggd`,
and a fetched agent binary is registered under `binaries.ygg-agent` — the version the node
running it will actually report back, which the agent-update job's completion check depends on
matching. Documentation changes trigger no release at all; the two mirrored docs are synced by
their own workflow (`docs-sync.yml`).

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

**A stale exit event must be recognised, not applied** (ADR-114): a Restart's back-to-back
Stop-then-Start (`server_lifecycle.go`) can relaunch the same container before the Stop's own exit
event is delivered on the runtime's independent async event stream, so `handleExit` now checks
whether the runtime says that role is running *right now* before acting on the event — the same
principle `console.runInstance` uses for a stale attachment (ADR-100). Confirmed live on two real
ARK servers that stayed genuinely running throughout a Restart while yggd reported them offline on
a stale "exited with code 143".

## Data flow

**Control operations** — browser → `yggd` HTTP API → hub → agent stream → Docker. The API
returns as soon as the command is accepted; completion arrives asynchronously as a state
change event pushed to the browser over WebSocket.

**Long-running operations** (install, install update, backup, restore, server move, agent
update) become Jobs. The agent streams progress up the control channel; the control plane
persists job state and relays to any watching browser. A job survives the operator closing their tab and, where
the underlying work permits, an agent reconnect (ADR-021).

**A job that stopped servers to do its work restores their desired state when it fails, however
the failure arrives** (ADR-129). Install-update and restore both stop every server they need out
of the way, set `Desired` to stopped, and record a restart plan. A failure caught synchronously
always put all of that back; a failure reported later by the agent — which is the ordinary case —
did not, so servers stopped by a failed update sat at `Desired = stopped`, which is precisely what
a deliberately stopped server looks like: no drift shown, no attention flag, and a subsequent
Retry that builds its plan from `Desired == running` found nothing to restart, leaving them down
even when it succeeded. Both paths now restore the intent. Whether the server is also *started*
again is decided per job type rather than uniformly — install-update yes, a rebuild's own install
no (its pod was destroyed before dispatch, ADR-126), a failed restore no (the disk may be
half-written, and that is a worse thing to bring a game up on than nothing).

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

**It attaches to every started server, not only ones whose seeds declare a rule** (ADR-097) —
otherwise a rule-less seed's history would survive only until the first viewer, since the daemon
replays a container's opening burst exactly once. See conventions.md's "Reattaching console
output across restarts" for the two-path (`pump` vs blocking `replay`) mechanism; the cost is one
attachment per running server, cheap because the scanner itself still returns immediately for a
seed with no rules.

**A console session is bound to the container run it attached to, not to the server and role
alone** (ADR-100) — a session outliving the container it attached to answers for a run that no
longer exists, and `Watch` finding it then attaches to nothing at all. That was fixed three times
in eight days, each catching one more way a run ends: a destroyed pod (ADR-095), a crash-loop
restart (ADR-097), then the four transitions *up* that replace nothing and must not close a live
attachment (ADR-098). See conventions.md's "Reattaching console output across restarts" for the
container-id-plus-start-time mechanism that replaced enumeration entirely, and why both halves of
that identity matter.

**A session can narrate before it has anything real to attach to** (ADR-118). Everything above
covers a *started* server; before that — the pod network being created, an image checked or
pulled, a container created, a health-gated dependency being waited on — nothing was shown at
all, so an operator watching a server come up for the first time saw a blank terminal followed by
an abrupt burst of the game's own boot log. `PodSpec.Narrate`, set by `ProvisionServer`'s handler
before any container exists, calls `console.Manager.Narrate` at each of `runtime/docker`'s major
Provision and Start steps; `Narrate` gets-or-creates a session marked `noteOnly` — the same
lock-guarded path `attach` already serialises session creation through, not a second one — and
appends a dim, ANSI-tagged line (`[boot] ...`) to its scrollback. A viewer's `Attach` arriving
while a session is still `noteOnly` binds and replays without waiting on the real attach, which
might be a health-gated dependency away; the real `Watch`, once the primary container actually
exists, reuses the same session — narration and any bound viewer both carry over — and clears
`noteOnly`. Nothing here is seed-authorable: it is infra-level and automatic for every seed, the
same way the container-lifecycle events it narrates are.

**Only one call may ever claim a note-only session's real attach** (ADR-124). The Start path fires
two `Watch` calls for the same server+role close enough together to matter: `startLogScan`, called
directly after the Start command's own `supervisor.Start` returns, and `rewatchOnRestart`, called
from the stream's state-change listener reacting to that same `Start`. Before ADR-124, both could
see a session still `noteOnly`, both would decide they owned the transition to a real attach, and
both would reach `runtime.Attach` and `close(sess.ready)` — the second close panics. `session`
records the claim (`attaching`) under the same lock the `noteOnly` check itself runs, before the
slow `runtime.Attach` call and well before `noteOnly` is actually cleared; a second call landing in
that window now waits on `sess.ready` like an ordinary existing session instead of opening a second
real attach of its own — the same single-opener guarantee ADR-100 already gives every other path
into `attach`.

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

SteamCMD's progress line needed four independent, layered fixes to reach the UI, each found live
against a real install.

**A line boundary is `\r` as well as `\n`.** SteamCMD redraws its progress line in place with a
bare `\r` and no `\n` until it moves to the next state. `logscan.Scanner.Feed` held such a line as
an ever-growing unterminated fragment, eventually dropped at `maxLineBytes` and never matched;
`internal/agent/install`'s own sibling line assembler for a SteamCMD install job's stdout — a
second, independent implementation of the same chunk-to-lines problem, needed because an install
runs before any server exists for `logscan` to key by — held it forever with no bound at all. Both
now close a line on a bare `\r` exactly as `\n` does, tracking one bit of carried state (`afterCR`)
so an ordinary CRLF pair split across two reads still collapses to one line rather than producing a
spurious empty one.

**Recognising a redrawn line is not the same as receiving it.** `RunOnce`
(`internal/agent/runtime/docker`) created the install container with no TTY, and SteamCMD decides
its buffering via `isatty()`: line-buffered against a real terminal, fully block-buffered (glibc's
multi-KiB default) against anything else — so a redrawn progress line sat inside SteamCMD's own
process buffer, never reaching the container's stdout at all, until the buffer filled or the
process exited. `RunOnce` now allocates a TTY. That also changes how the container's log stream is
framed: a non-TTY container's stdout/stderr are multiplexed with per-chunk headers (`stdcopy.StdCopy`)
so the daemon can tell them apart, while a TTY container's two streams already share one pty with
no such framing — reading a TTY stream with `stdcopy.StdCopy` regardless would corrupt the first
bytes read.

**Even with a TTY, Docker's own `ContainerLogs` follow stream independently withholds a partial,
non-newline-terminated write until a real `\n` arrives or the container exits** — a second,
independent buffering layer in the runtime's own log collector, upstream of anything `RunOnce`'s
`out` writer reads. The fix runs SteamCMD through a shell pipeline rather than execing it directly —
`{ steamcmd ...; echo $? >file; } 2>&1 | tr '\r' '\n'; exit "$(cat file)"` — so every redraw becomes
a real newline-terminated line before it reaches the log collector. Since `tr` itself always exits
0, the real exit code is captured to a file inside the group and re-raised afterward rather than
relying on bash's `pipefail`, which `base-linux`'s `/bin/sh` (Debian's `dash`) does not have.
Building a shell string rather than an argv means every argument is shell-quoted (`shellQuote`, in
`internal/agent/install`) so a seed-supplied value cannot break out of its own argument — asserted
by a real round-trip through `sh -c`, not just by inspection of the escaping.

**The delivery mechanism above works; the filter reading it did not.** SteamCMD's first progress
line for a phase arrives clean (`Update state (0x61) downloading, ...`), but every redraw after it
carries a leading SGR reset the first line never had (`\x1b[0m Update state ...`) — invisible to
`strings.TrimSpace` (ESC is not whitespace), so `strings.HasPrefix(line, "Update state")` matched
line one and silently rejected every line after it, forever. `steamcmdProgress` now strips one or
more leading SGR sequences (`^(\x1b\[[0-9;]*m)+`) before matching, verified against lines captured
live.

**Readiness and crash detection are both log-driven where a seed asks for it** (ADR-077). A
`crash: {mode: log}` rule exists for a failure exit codes cannot see — ARK hanging indefinitely
inside Crashpad with the process alive and nothing exiting (ADR-047) — and a match kills the pod,
so the ordinary exit-and-restart path handles it as the single route to `crashed`. `mode: none` is
the default. See conventions.md's "Player lists and crash detection" section for the safety
mechanics and why a crash pattern's bar is higher than a readiness one.

**A seed can also recognise well-known events** — `logs.events`, `join` and `leave` — which
maintain a player list on the server's page and a count on the fleet list (observed state only;
see the "Player lists and crash detection" section of `docs/conventions.md` for why it is
discarded on stop and shown only when a seed declares the rule).

**A join rule may optionally also capture a Steam ID** (ADR-119), the same way it already
captures the player's name. Where present, `internal/control/steam.GetPlayerSummaries`
resolves it to a cached persona name and avatar (`internal/control/web/steam_player_cache.go`, a
24-hour TTL — long enough that routine page views never re-hit Steam, since this is read on every
render rather than fired by an explicit operator action the way the seed-authoring UI's own
1-hour Steam caches are) through `steam.api_key`, the same `KindSecret` setting the authoring UI's
Steam lookups already use. `enrichPlayers` (`internal/control/web/players_enrich.go`) is where a
raw `server_players` row becomes a `playerView` with an avatar; it is best-effort throughout — no
key configured, or a failed lookup, degrades to the plain player list this feature always showed,
never an error — and is called **once per page**, batching every steamid the page is about to show
into a single lookup rather than one per server. This is still Steam only: Epic and Xbox both need
per-title developer credentials this project does not have, and Xbox needs a separate auth chain
from the OIDC sign-in ADR-115 already built. A live view only, too — no history beyond what
`server_players` already held, and no kick/ban/control actions, just visibility.

**A `leave` rule may correlate by a per-session id instead of a shared name** (ADR-127). Matching
a leave against a join by `(server_id, player)` assumes the game prints the same name at both ends
— true for most seeds, and false for Valheim's own PlayFab-era build, whose disconnect line
(`Player connection lost server "...", now N player(s)`) carries no name at all. `join`'s optional
`id` group (alongside `steamid`) captures a stable per-session identifier instead — Valheim's own
ZDO owner number, present on the name-bearing join line and again on every disconnect-time cleanup
line that follows a real leave — and a `leave` rule with no `player` of its own matches by that
`id` instead. `leave` needs one of `player` or `id`, never neither; a seed that never declares
`id` sees no behavior change; the mechanism is generic to any future game with the same split.

**`/players`** is the fleet-wide counterpart to a server's own player list: every online player,
grouped by node and then cluster/standalone exactly like `/allocations` — the same bulk-load,
build-lookup-maps, nest-and-sort shape, live via an unscoped `data-live="players-list"` region.
A server's own Overview and Console tabs, and this page's own per-server groups, all render
through the same `playerCard`/`playerRow` template partials, so there is one row markup rather
than three hand-copied ones.

**The same page also carries a manual lookup** (ADR-120): a search box accepting a SteamID64, a
`steamcommunity.com` profile or custom-URL link, or a bare custom URL name, independent of whether
that person has ever been captured by a `join` rule at all. `internal/control/steam.ParseSteamIdentifier`
decides which shape input is, `ResolveVanityURL` turns a custom name into a SteamID64, and the
result is cross-referenced against the same fleet-wide online list the page already built, naming
every server it finds a match on rather than picking one. Both Steam calls are cache-backed — the
resolution through its own `steamVanityCache`, the profile fetch through the same `steamPlayerCache`
ADR-119 already built — which is what makes it safe for the search to sit in the URL through an
ordinary live-triggered re-render: nothing here differs from a server's own player card re-rendering
on every fleet event, except that a search result's own "who's online now" answer only updates when
the operator submits it again, the same one-shot-until-resubmitted contract `/installs`' state
filter already uses.

**`/jobs`** is the fleet-wide list every long-running operation lacked until now (ADR-140's own
finding, from the top-down review): install, install update, backup, restore, server move and
agent update are all one `Job` (ADR-021), but each type's status used to be visible only on the
page of whatever it targeted — an install's own row on `/installs`, a backup's own line on a
server's Backups tab, an agent update's own row on `/updates` or `/nodes`. A server move never got
a status display anywhere at all. `ListJobs` already returned every job fleet-wide, newest first,
with nothing new needed in the store; the page is a flat table (deliberately not grouped by node
the way `/allocations`/`/players` are, since an operator scanning "what's running or what just
failed" cares about the target more than the node) with two independent server-side filters — type
and state, `/installs`' own select-and-submit shape rather than `/seeds`' client-side chips, since
stacking two chip rows for two independent dimensions is worse than two plain `<select>`s in one
form. Capped to the most recent 200 past the filter, since jobs accumulate over the fleet's
lifetime rather than with its size the way every other list page's row count does — a job's full
history still lives on its own target's page via `ListJobsByTarget`. Live via the same unscoped
`data-live="jobs-list"` region every other fleet list page uses (ADR-058), needing no new event
kind since `EventJob` already existed with nothing rendering it fleet-wide. What a job acted on is
resolved fresh per render rather than stored, tolerating a deleted target by falling back to its
raw id rather than erroring the whole page — an agent-update job is the one special case, since its
own `TargetType`/`TargetID` name the binary being installed rather than the node being updated,
which travels as `NodeID` instead and is what an operator actually wants to see.

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
breadcrumbs, a Monaco editor for files under 1 MiB (vendored the same no-build-step way as the
console's xterm.js, ADR-117), and an upload form, no JS framework needed (ADR-006). The
`<textarea>` Monaco replaces is still what the surrounding form submits — Monaco is a view over
it, not a different field — so the page works, just without highlighting, if the vendored script
fails to load.

A seed may put paths out of reach with `server.file_denylist`, for the files it regenerates
itself — see conventions.md's "Files and sandboxing" for what it does and does not guard (it is
not the security boundary; the sandbox is) and why a denied entry is omitted rather than refused
(ADR-033, ADR-092).

A seed may also mark a config file `importable`, which lets an operator read it back off the node
and adopt the values the seed recognises onto that server's stored settings — the migration path
for a server somebody configured by hand, or a file edited before the seed had a setting for that
key. Only values that differ are adopted, and a file that could not be read whole contributes
nothing, which is the same judgement patching makes in the other direction.

**An upload can be extracted instead of written literally** (ADR-146). The upload form's "Extract
as archive (.zip)" checkbox sends `FileWrite.extract` (protocol 12); `internal/agent/files.Root
.ExtractZip` unpacks the bytes into the target directory using the same `internal/archive.ExtractZip`
and zip-slip guard `internal/agent/install` already trusts for a downloaded archive, denylist-enforced
the same way an ordinary upload is — never the `managed` exemption above, which is for a seed's own
declared rewrite, not an operator's. A decompressed-size cap (`maxExtractedBytes`, 512 MiB) is checked
against the archive's declared entry sizes before anything is written, since nothing upstream bounds
the *unpacked* size of an upload, only the compressed bytes crossing the wire. This exists for a mod
or resource pack that is genuinely many files in nested folders — a real distribution shape this
project had no seed exercising until FiveM's own resource ecosystem needed it.

**A cluster's shared volume gets the same six file operations**, at `/clusters/{id}/files` — the
existing wire messages widened with an alternate `cluster_id` field, mutually exclusive with
`server_id`, rather than a second set of them, since a cluster volume sits outside any server's
own sandbox root (ADR-033) — the same reason `DeleteClusterVolume` needed its own command
(ADR-066, ADR-105). See conventions.md for the dispatch mechanism and why no `file_denylist`
applies there.

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
screen tying them to the server they belonged to. A seed declaring a container UI (ADR-147) adds a
sixth, conditional tab for that addon's own web interface, proxied rather than linked to directly.
Overview shows the pod's containers (every
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
(ADR-085) — a vertical tab rail beside one panel per group once a seed declares four or more
(Vintage Story's 136 controls across 18 groups is the case that motivated it: 3.2 screens of
headings as a stack versus 1.3 screens as tabs), with a single filter spanning variables and
settings, since an operator recognizes a control's name rather than which block declared it.
Every panel stays in the form regardless of which tab is showing, so a hidden control still
submits — conventions.md's Grouped-controls and Filtering-controls rows cover the tab threshold,
the `show_if`/validation interaction, and why a hidden panel must never be dropped.

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
artwork, its facts and its player count. How many fill a row is the grid's decision rather than a
constant — one column at 420px, three at 1440, seven at 2560 — because the count depends on the
viewport and the control plane cannot know it (conventions.md's Art-cards row has the
auto-fill/max-height mechanics). The band is height-capped so a wide card does not become mostly
artwork. This is what finally consumes the artwork ADR-077 and ADR-079 built the whole path for;
before it, a banner appeared on exactly one page as a 120px inline image. Most seeds ship none, so
the card falls back to a striped placeholder carrying the seed's own glyph, and that is the path
built first.

**A card carries its own start/restart/stop, not only a link to the server's page** (ADR-084's
amendment — conventions.md's Art-cards row explains why the stretched-anchor layering was
needed). Restart is Stop immediately followed by Start with desired state held at running
throughout — the same back-to-back sequencing a reprovision already uses for Stop immediately
followed by Destroy — and a button disables itself when its node is disconnected rather than
dispatching a command guaranteed to come back undelivered. A server's own detail page carries the
identical three buttons and, until ADR-138, was the one place that lacked the guard — fixed there
by copying the card's own condition and rationale verbatim, leaving Kill and Delete ungated on
both surfaces.

**A cluster is one card, not one per member.** Its members are reached through it, at
`/clusters/{id}` — the page ADR-066 built rename, remove-member and delete without, putting all
three in a table cell because there was nowhere else. Its card carries a stacked-sheet edge, a
segmented run bar with one segment per member, and aggregate counts.

Run state is carried by colour as well as text — a badge plus a stripe on the row's leading edge,
both chosen by one Go method so nothing can disagree about what state a server is in
(conventions.md's State and Colour rows carry the mechanics). `degraded` is coloured apart from
`crashed`, since ADR-019 makes them operationally distinct; idle is muted, so a healthy fleet
reads calm and only trouble draws the eye — the wall-mounted case the stylesheet is written for.
Colour stays the only swappable axis (Frost / Grove / Ember), and semantic colour sits outside the
palettes entirely.

**A server stuck the wrong way round from what it was asked for now reads as needing attention,
not as deliberately stopped** (ADR-143). `NeedsAttention()` used to check only `crashed` and
`degraded`; `Desired` played no part in it, so a server a failed job left stopped was
indistinguishable on the fleet page from one an operator stopped on purpose — ADR-130 named this
gap directly and left it for its own change. `Stuck()` is deliberately narrower than the
`Drifting()` predicate a server's own detail page already showed: `Installing`, `Starting` and
`Stopping` all mean a job or a command already owns the next transition — a rebuild's own install
leaves `Desired` at `running` throughout (ADR-062), so the raw `Drifting()` predicate would have
flagged every ordinary rebuild as a problem for its whole duration. What is left after excluding
those is exactly the two states nothing else owns fixing: `Desired=running`/`Observed=offline`
(a failed job's own aftermath) and its mirror, a Stop that silently failed to take effect. A fleet
card carries a small `should be {{.Desired}}` badge under the name for exactly this case, and it
rolls into every existing "N needs attention" count — the page tile, a node group's summary, and a
cluster's own worst-member colouring — with no other change, since all three already loop over
`NeedsAttention()`.

**A seed contributes one more colour, and only a hue.** `internal/control/branding.Accent` takes
the dominant hue of a seed's own artwork and clamps lightness and chroma to fixed constants, so a
game's colour can touch the chrome — conventions.md's Game-accent row names exactly where (an
edge, a run bar, an active tab underline) — without ever competing with the product accent or
with semantic state colour. It is **derived, never stored**: the bytes already answer it, and a
persisted copy would be a second writer able to drift from the image beside it (ADR-071).
Memoised per seed, forgotten on reload, and empty for a seed with no artwork or no usable hue —
which the stylesheet's own `--game: var(--accent)` default covers with no branch in any template.
`branding.accent`, the field a seed *declares*, stays reserved and applied by nothing (ADR-077 §14).

**A server's and a cluster's page open with an art band**: the same banner, blurred on a layer
of its own behind a fixed scrim, with the tab strip at the top and the identity block, endpoints
and lifecycle controls on it.

**Headline numbers are stat tiles** rather than a line of body text: the fleet's counts, and the
live CPU/memory/network readings on a server and in the console's rail — the same markup reads
as a row on one page and a stack on the other (conventions.md's Stat-tiles row has the grid/swap
mechanics).

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
alongside the steps, so a protocol 1 agent installs exactly as it did; that work
took the negotiated protocol to 2 (ADR-014's N-1 window, which until then had no real history to
mean anything against), and later additive bumps have since taken it to 12 — the log viewer
(ADR-082), `ReclaimData` (ADR-088), managed config writes (ADR-092), cluster file operations
(ADR-105), the destroy confirmation (ADR-107), a per-step SteamCMD depot-bitness override
(ADR-121), a per-step SteamCMD depot-platform-OS override (ADR-123), a node's own free-disk
figure on every heartbeat (ADR-134), the install force-clean escape hatch (ADR-142), and the
Files page's zip-extract option (ADR-146).
`internal/configfile` is the shared key-patching implementation behind both
the `patch` step and a config file managed `patch`, so a key path means the same thing at
install time and at provision time.

**Variables and settings are separate blocks**, both embedding one `Control` type so the UI can
render either kind with no new component (`varfields.html`): a variable shapes how the server is
*built* — reaching a command, an env value, an install step, or a mount — so changing one rebuilds
the pod, while a setting is written into the game's own configuration and declares its destination
(a config file key or an env var). Settings also carry a tri-state absent/empty/set, letting a
game whose own image rewrites its config be told to leave a key alone — something schema 2 could
not express. `seed.ResolveDestinations` separately resolves each setting's destination once per
rebuild, so the pod builder and the config writer read from one result and can never disagree
about whether a setting is present. See conventions.md's "Variables and settings" section for the
exact value semantics, including how `servers.settings` stores only the operator's own overrides,
never the seed's defaults.

**A config file declares how it is managed**: `always` rewrites it wholesale (the default, and
schema 2's only behaviour), `once` writes it only if absent, and `patch` sets just the keys the
seed and its settings name, through `internal/configfile`. Patching is the only thing on the
provisioning path that reads from a node, and a read that fails or truncates degrades to writing
the declared keys alone rather than failing the rebuild — conventions.md adds the corollary that
such a read is never merged into, since writing it back would delete whatever it didn't reach.

**Readiness is a declared mode** — `immediate` (the default, unchanged from what every seed did
before, so a seed asking for nothing else gets no `ReadySpec` at all), `log`, `port`, or
`healthcheck` — replacing schema 2's `logs.ready` substring, which four bundled seeds declared and
nothing ever matched: making readiness depend on a match risked stranding a healthy server in
`starting` forever (ADR-067). `mode: port` needs no game-specific knowledge, which makes it the
right first choice for a game added blind. conventions.md's Readiness section covers the
mode-by-mode mechanics in full, including which restarts re-establish it and why `logscan` stays
the one line assembler for both readiness and log-value extraction.

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
shipped with the seed and provisioned only if the operator enables it; conventions.md's
Ports-and-allocations and Addons entries cover how `enabledContainers` keeps a disabled addon
invisible to allocation and to the pod spec. Ports are released on the next rebuild rather than at
the moment of the save, so the invariant holds for a server whose earlier save failed halfway.
Validation carries the rest: an addon cannot be primary, nothing required may depend on one, and
no template may reference one's port.

**A container's own web UI can get its own tab, proxied rather than linked to directly**
(`containers[].ui`, ADR-147) — the case a database admin tool beside a bundled database exists
for. At most one container per seed may declare it, so a server's tab strip never has to choose
among several; the tab appears only while that container is currently enabled. `GET
/servers/{id}/addon` is the ordinary tab chrome around an iframe, and
`/servers/{id}/addon/proxy/...` is where `yggd` itself reverse-proxies to the container's node and
allocated port, resolved fresh on every request. Proxying rather than linking directly to the
node's address is deliberate: a direct link breaks in an iframe the moment the operator reaches the
UI through the Cloudflare tunnel (mixed content, HTTPS parent page and bare HTTP target), where a
same-origin proxy does not. The proxy strips an addon's own `X-Frame-Options` and any CSP
`frame-ancestors` directive from its response (ADR-151) — a real backend refusing to be framed by
default (Adminer's own `X-Frame-Options: deny`) would otherwise defeat the one thing this route
exists for, since being embedded in this page's iframe is not incidental here, it is the point.
A published port whose image cannot be told what to bind to at all may also need
`container_port` (ADR-150) to be reachable in the first place — Adminer's own standalone server
always listens on a fixed 8080 regardless of the host port it is allocated.

**A port declares what it is for** (`kind: game|query|rcon|web|voice|other`), and a seed may
declare a **connect** block — a URI a client understands, an address to copy, or both. It is
rendered per request rather than stored, because it templates over the node's address, which can
change and can legitimately be unknown (ADR-065); with no address known it renders nothing at
all, since a bare port reads as incomplete and a URI with an empty host does not.

**A command can reach the node's address directly, with no port joined to it** (ADR-128).
`TemplateData.NodeAddress` is the same address the connect block and every allocation's endpoint
already resolve at render time (ADR-065) — `Endpoint` joins a port onto it, `NodeAddress` is the
bare host, for a flag that needs one on its own rather than "host:port". ARK Survival Ascended is
the first consumer: `-PublicIPForEpic={{.NodeAddress}}`, which tells Epic Online Services what
address to hand a connecting client, wrapped in `{{if .NodeAddress}}...{{end}}` so it renders to
nothing rather than an empty flag while the node's address is not known yet — degrading exactly
as a connect block already does, not a second rule for the same ambiguity. Every render site that
already builds `Endpoint` from a real node address builds `NodeAddress` from the same call, so the
two can never name different hosts.

**A seed can carry its own migrations.** Stored values are keyed by name, so a bare rename
silently discards what every operator chose — ADR-074 declined to rename Valheim's
`crossplay_flag` for exactly that reason, since losing the stored value would have turned
crossplay back on wherever it had been turned off. `renamed_from` and a `migrations:` block
(`rename`/`drop`/`rewrite`/`promote`) carry values forward instead; conventions.md covers the
idempotence rule that lets nothing be recorded per server.

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

That repository publishes a growing set of seeds across three channels — including ARK Survival
Ascended, ARK Survival Evolved, Minecraft Java (Paper), Minecraft Bedrock, Valheim, Vintage Story,
Palworld and Empyrion at various stages of promotion — exercising both ends of ADR-018's install
model between them: a shared SteamCMD install mounted read-only with per-server writable overlays
and cluster support at one end, and an image that installs the game itself with no install block
at all at the other. Phase 5's exit criterion is stated against one of them and is tracked in the
phase table below, not here.

**The exact roster and per-channel counts are deliberately not stated here**, since this
paragraph has already gone stale twice trying to keep one current — the catalog is a separate
repository on its own release cadence (ADR-081) that no test in this one can see. Check `/seeds`
on a running control plane, or the seeds repository's own channel releases, for what is actually
published right now.

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
`seed.Seed`, and `Encode` is its inverse, existing so that every seed in the fixture corpus
round-trips unchanged and does so idempotently — the mechanical answer to a failure this codebase
has recorded four times: a seventh variable, a `Required` flag, an offset relationship, and a
`logs.values` block, each found only after it shipped, by someone noticing a value had vanished.
Two further tests join the halves: the rendered page must emit every field name the decoder reads,
and each row prototype's path shapes must match a rendered row's, so a row added in the browser
submits the same field set as one from the server.

Decoding is index-driven, not count-driven, with indices collected from what was actually
submitted, so a row deleted in the browser leaves a gap and nothing downstream cares (the same
discipline conventions.md states for this exact code, ADR-079). Each discriminated block (an
install step's `op`, a setting's destination, a control's `type`) reads only the fields its
selected case accepts, because the form renders them all and hides the rest, and `Validate`
rejects a field that does not belong.

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
`<kind>.<detected ext>`, never as what the upload was called (conventions.md's Seed branding rule,
ADR-079) — so a file named `icon.png` whose contents aren't actually a PNG is refused rather than
served as one from this origin. SVG stays refused on this path even so, because a published
bundle's images are served from *other* operators' origins, where a hand-placed SVG in your own
bundle is a purely local decision about your own control plane. Nothing is written until every
fetch has succeeded, so a save can never leave a manifest naming an image that was never
retrieved.

`guidedEditable`'s allowlist stays at full coverage, so it keeps guarding not today's gaps but the
*next* field added to `seed.Seed`: one nobody has taught the form about is unrepresentable by
default and falls back to the raw-YAML pane — the same discipline conventions.md describes for
`guidedSeedFields`/`unrepresentableSeedFields`. That pane is now a co-equal path rather than a
fallback, and both routes converge on the same `internal/seed.Parse` and `Validate` every seed
loads through.

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
where each appears rather than by pixel size — reached from the Art tab, with each control
associated to the outer form via `form="seedform"` rather than nested inside it (conventions.md's
two-forms-cannot-nest rule, ADR-079): nesting would mean the dialog's own `<form method="dialog">`
close button falls back to submitting the outer form instead of dismissing the dialog.

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

`publish.ResolveTarget` refuses `arasoi/yggdrasil-releases` as a checked error,
case-insensitively, even when set deliberately, for the narrower reason that a wholesale release
replacement there would take out the distribution itself — not because it is the repository this
control plane reads its catalog from, which stays the ordinary, safe case for whoever maintains
one (conventions.md's Configuration section states the general refusal rule; ADR-081). For an
operator who is not the maintainer, GitHub's own permissions are the boundary, and the token's
Test action asks exactly that before they press anything.

**This path is for an operator with no CI**, or one publishing a catalog for their own fleet from
a seeds directory that is not a git clone at all; the project's own catalog instead leaves
`seeds.publish_repo` **unset**, since CI in the seeds repository is its one writer and an unset
target renders the Publish button inert ("Set a publishing repository in Settings first") rather
than risking a second writer.

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
anything, unlike an agent binary that executes with Docker-socket access on every node.
`seed_channel` defaults to `main` (ADR-087, since the catalog is the only way seeds arrive at
all); setting it to empty is still respected and means no outbound call, with the page offering
a channel picker instead that persists the choice to `yggd.yaml`.

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
hold files the running binary never shipped, so a later validation tightening can invalidate a
bundle that was fine when it was installed, and refusing to start over it would crash-loop the
control plane with no UI left through which to remove it (the same `seed.LoadDirTolerant` rule
conventions.md states). Startup and the in-process reload both use it, logging each skipped
bundle at error level and falling through to the layer beneath. `LoadFS` stays strict, and is what
the base image library and the seed fixture corpus load through: those are compiled in or checked
in, so a failure there is a build defect rather than a file an operator installed.

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
  SteamCMD needs, plus SteamCMD itself, pre-warmed at build time. Also installs
  `libcurl3t64-gnutls`: some Steam-distributed dedicated servers' own binaries link against
  curl's legacy GnuTLS-flavoured SONAME rather than the OpenSSL-flavoured build Debian installs
  by default, found live when Team Fortress 2's `replay_srv.so` refused to load without it
  (ADR-047's 2026-08-27 amendment) — the two builds share a SONAME but are ABI-incompatible, so
  only installing the real package fixes it. Its `entrypoint.sh` also stages SteamCMD's own
  native `steamclient.so` into `$HOME/.steam/sdk<32|64>` unconditionally on every container
  start, before the privilege-drop decision below — replacing a fix three seeds (Garry's Mod,
  Team Fortress 2, Palworld) used to duplicate by hand in their own launch command (ADR-125; see
  conventions.md's container-preparation rule).
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

`.github/workflows/images.yml` publishes all five to GHCR
(`ghcr.io/<owner>/base-linux`, `.../base-steamcmd`, `.../base-steamcmd-proton`, `.../base-java`,
`.../base-dotnet`) under the same floating per-branch channel tags
(`develop-latest`/`qa-latest`/`release-latest`) the binary release workflow uses (ADR-038), plus
an immutable `:<short-sha>` tag for anyone who wants to pin rather than track a channel. A push
to `develop`/`qa`/`main` rebuilds only the images it actually touched, plus everything downstream
of them (ADR-093's amendment); a child pins its parent's same-run sha tag when the parent rebuilt
in that run, and the parent's channel tag otherwise — with `cancel-in-progress` closing the
window a floating tag could move in between. `base-linux`, `base-java` and `base-dotnet` are
published as multi-arch (amd64 + arm64) manifest lists, each architecture built on its own
native runner; `base-steamcmd` and `base-steamcmd-proton` are amd64-only and always will be —
SteamCMD is a 32-bit x86 binary with no arm build, and GE-Proton is x86 Wine (ADR-094).

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
pruned back to the `stats.retention_days` setting (default 7 days, ADR-078) on every tick, so
lowering it frees space within a minute. A server with nothing live to report is
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

Every operator-facing string goes through `internal/i18n`, keyed by its own English source text
rather than an invented identifier, with a generated-and-checked catalogue guarded against silent
drift — see conventions.md's "Translation" section for the full rule and its guarantees (ADR-086).

Templates are parsed **once per shipped language**, because a `FuncMap` is fixed at parse time
and `T` must close over the language it renders in; `render()` then selects a pre-parsed set and
does no per-request template work. Language resolves as the operator's stored preference
(`ui.language`, a runtime setting), then the browser's `Accept-Language`, then English — read per
request, so a change takes effect on the next page load with nothing to invalidate. `<html lang>`
declares whichever language actually rendered.

Formatting follows the locale too — German writes `64,0 MiB` — while unit symbols and
seed-schema literals stay English by design (see conventions.md).

**English is the only catalogue that ships.** The machinery is complete and exercised — a test
injects a second catalogue and drives a real request through `render()`, asserting it renders in
that language, declares it, and falls back to English for a key it lacks — but what is absent is
translated content, which nobody here can verify: a machine translation of ~1,040 strings would
put unreviewable text on destructive actions.

One limit is carried forward rather than read as done. The ~150 template fragments that were
sentences split by inline markup — each half a sentence that could not be translated alone — are
done: `TH` carries a message with its markup, extraction is at zero untranslated fragments, and the
coverage report that counted them down is a gate rather than a report.

**Seed content is done too, not merely specified as this section once said.** A seed's own labels
and descriptions are the majority of what an operator reads on a settings page, and ADR-086's
design is built: `internal/seed/translate.go`'s `Locale` carries a seed's own name, description,
group headings, control labels/descriptions/option text, container names and the connect label,
one file per language at `<seed-id>/locales/<lang>.yaml`, keyed by the declared control or
container name rather than by source text — a seed author may not write in English, so keying by
source text would privilege the authored language the way the app's own catalogue does not need
to. `LoadFS`/`LoadDir` attach it after `finalize`, so a locale keyed by a control the manifest does
not declare is a lint warning (`lintLocales`) rather than a load failure, and a translation missing
an entry falls back to the seed's own authored text — the identical "partial is usable, never
blank" rule the app's own catalogue follows. `Seed.Localized` is applied wherever an operator
actually reads a seed's prose — server settings, cluster settings, the seed-creation flow — rather
than inside seed loading itself, so a caller that only needs the structure pays nothing.

What is still missing is authoring, not the mechanism: neither the seed editor nor `ygg-seed` has
any locale-aware tooling, so a translator hand-writes `locales/<lang>.yaml` directly into the
bundle. That is a real gap, and the honest one to carry forward here in this document's place.

## Operator settings

Configuration splits on one question — is it needed before there is a database to read?
`yggd.yaml` keeps what is (listen addresses, `data_dir`, the database path, `seeds_dir`),
immutable after startup; everything else is a runtime setting in the `settings` table, adjustable
from `/settings` while yggd runs. See conventions.md's "Configuration versus settings" for the
resolution order, secret-handling, and most-repeated-bug rules (ADR-078).

The page itself is a tab rail beside one panel per group with a single search box over every
setting, reusing `static/varform.js` unchanged from the seed-variables form (ADR-085); Sign-in's
ten settings additionally sub-divide into per-provider boxes (Shared, Microsoft, Apple, Discord)
decided in the web layer from each key's own prefix (ADR-116). A setting an operator has actually
set gets a small dot and a heavier label, so the page can be scanned for what is configured rather
than read row by row. Release channels and a node's own settings (address, port range) are
surfaced on `/settings` with a link but stay owned by their own pages, since switching one is an
action rather than a stored value (ADR-056).

The table is key/value; what a key *means* lives in `internal/control/settings` as a registry of
typed `Definition`s. Nine ship, each with a live consumer: `log.level` (the process's own
`slog.LevelVar`, so debug can be switched on and off without a restart), `stats.retention_days`
(read by `telemetry.Collector` on every prune), `backups.retention_days` (read by the scheduler's
archive prune the same way, ADR-088), `nodes.disk_low_percent` (read through `Capacity.DiskLow`,
ADR-134), `ui.language` (ADR-086), `steam.api_key`, `seeds.catalog_repo` (ADR-081), and the pair
that lets this control plane publish its own seeds, `seeds.publish_repo` and
`seeds.publish_token` (ADR-079) — the first credential here that writes outside this deployment,
which is why its target has no default and the one repository it must never name is refused
outright.

`store.Open` narrows the database file to `0600`, since it holds password hashes, session
tokens, bootstrap tokens and now credentials. Defence in depth rather than the boundary: the
containing directory is already `0750`, which is why a `chmod` failure is non-fatal.

## Security model

Scope is a homelab: a small set of trusted people, not untrusted tenants.

- **Human auth** — a local admin account (argon2id, session cookies) plus optional federated
  sign-in through Microsoft, Apple, or Discord (ADR-115). Federated login is strictly additive:
  the local account always keeps working, so a misconfigured or unreachable identity provider
  can never lock every operator out. A new account is never created just because someone can
  authenticate to one of those providers — an admin must first create a one-time join code at
  `/users`, naming a role but not a person; whoever redeems that link, through whichever
  provider they choose, is who the account belongs to (ADR-115's amendment). `api_tokens`
  remains dead schema, unused since the first migration.
- **RBAC** — three global, fleet-wide roles (`Viewer` < `Operator` < `Admin`), enforced by
  `requireRole` alongside the existing `requireAuth`. Admin covers settings, security, updates,
  node management, catalog/publish actions, and user management; Operator covers fleet mutation
  (server and cluster lifecycle, files, console, backups) and anything that can reveal a secret
  even as a plain read (a settings form's current values, a config file's contents); everything
  else needs only an authenticated session. Scoping a role to specific servers or nodes rather
  than the whole fleet is a deliberately reserved seam, not built.
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

## Delivery history

This project began as an eight-phase delivery plan (0 through 8, below) that took it from
nothing to a working fleet manager. That plan is finished, and phase-based tracking ends with
it: everything built since — federated sign-in, i18n, the settings registry, cluster
inheritance, the players page, disk monitoring, force-clean install, desired-state
reconciliation, the fleet-wide jobs page, and every other capability this document describes
above — is ongoing fleet-operations feature work with no phase to attach to, and belongs to no
row in a table like this one. `docs/decisions.md` is the actual current record of it, entry by
entry, in the order it happened; `grep -n '^### ADR' docs/decisions.md` finds anything by title.
What follows is kept as history, not as a status board — the one exception being phase 5, whose
original scope is still genuinely unfinished.

| # | What it built |
|---|---|
| 0 | Repo, docs, proto contract, config, DB schema, both binaries build and start |
| 1 | Enrollment, mTLS, persistent stream, protocol version negotiation, node registry with architecture, node list UI |
| 2 | `Runtime` interface + Docker impl, `PodSpec` (single-container), labelling, adoption, supervisor state machine, start/stop/kill |
| 3 | Console streaming (xterm.js) with stdin and multi-viewer fan-out; file browser, editor, and upload, sandboxed per server (ADR-033) |
| 4 | Seed schema, Job entity, install pipeline with progress, config templating, allocations UI |
| 5 | Shared installs, refcounting, read-only mounts with writable overlays, clusters, orchestrated install updates — **the one row below still open** |
| 6 | Multi-container pods: per-pod networks, dependency ordering, health gates, reverse-order shutdown, `degraded`, sidecar log/stat views |
| 7 | Signed agent binary distribution, UI-triggered update, N-1 compatibility testing |
| 8 | Manual and scheduled backup and restore with per-container hooks, cluster volume backups, and resource graphs |

Phases 0, 1, 3 and 4 each closed on the first pass and need no further account. The rest are
worth a paragraph each, either for a design decision that still shapes the codebase or because
the phase's own scope was never fully closed out.

**Phase 2 modeled a server as a pod from the start**, even though every pod had exactly one
container until phase 6. A server that owns a *set* of containers with one primary cost almost
nothing to build there and would have been an invasive refactor to retrofit later, since console
attach, stats, adoption, and the state machine all change shape. Its own exit criterion —
`kill -9` the agent while a server runs, and confirm the restarted agent adopts it with correct
state — was deliberately brutal, because agent statelessness is the property the entire
zero-downtime-update story rests on: unproven under load this early, it would not have held once
phase 7 needed it.

**Phase 5's mechanism is built and verified, but not yet against ARK by name — this is the one
piece of the original eight-phase scope still open.** Shared installs,
refcounting, read-only mounts with writable overlays, cluster creation and joining, and the
orchestrated install-update job are all implemented and proven — against a real control plane, a
real `ygg-agent`, and real Podman — using a synthetic seed shaped like `ark-survival-ascended.yaml`
(one shared install, five map servers, one cluster), since ARK Survival Ascended has no official
Linux server and needs a SteamCMD-plus-Proton wrapper. That testing found and fixed a real bug:
`Provision` was not publishing a state change for a freshly provisioned but not-yet-started server,
so such a server never reported its containers upstream and stayed stuck on "installing" forever.

`images/base-steamcmd-proton` (see "Container image library" above) is built and proven to run
Wine/Proton for real, and **a real ARK Survival Ascended server now boots** through it: the primary
container's `ygg-proton` wrapper invokes GE-Proton directly rather than through `umu-launcher` — the
`umu-launcher` path stalls indefinitely inside Sentry/Crashpad's own startup sequence, and its only
known workaround (renaming the game's own crash-handler binary out of the way) requires mutating the
read-only-mounted install, which ADR-018's model forbids. Getting a real depot running via raw Proton
needed several further fixes, fully recorded in ADR-047's amendments: GE-Proton needs a newer glibc
than Debian `bookworm` ships, which is why `base-linux` and everything built on it moved to `trixie`;
a per-server writable directory must be owned/writable by the container's user or Unreal silently
redirects its log to Wine's stubbed Windows event log and dies with nothing to read; `/etc/machine-id`
must exist; and the actual indefinite hang on this path was ARK's own bundled `steamclient64.dll`
winning over the system one on application-directory precedence and blocking forever waiting for a
Steam client that was never there. With those fixed, a real depot boots to "has successfully
started!" in about 18 seconds with the game port bound and RCON listening, install mounted read-only
with per-server writable overlays exactly as ADR-018 specifies.

The exit criterion nonetheless **stays open**: the Steam subsystem reports `FAILED` (expected
harmless, since ASA lists through Epic Online Services, but unconfirmed by a real client connecting);
this proves one server booting, not phase 5's five-maps-one-install-with-character-transfer workflow;
and it was driven with Podman directly, not yet through `yggd` and `ygg-agent` end to end.

**Phase 6 is done.** Its exit criterion no longer names Dune Awakening specifically, since that
game has no public dedicated-server image and naming it would make the criterion hostage to a
third party's release decision rather than to anything in this repository. It instead describes
what a multi-container pod must do — dependency-ordered start, health gates, reverse-order
shutdown with the primary first, sidecar crash-loop restart into `degraded`, and per-container
console/stats — proven by a full-stack test (real control plane, real `ygg-agent`, real Podman, no
fakes) and a live browser session against a genuinely running multi-container pod, both run
against a synthetic game+database+broker seed, which is the *right* target rather than a stand-in:
what is being proven is the pod mechanism, and a synthetic seed exercises its corners more
deliberately than any one real game would. A real Dune Awakening seed remains wanted and is
tracked as catalog work in `arasoi/yggdrasil-seeds` (ADR-087), not as a phase gate here. The
honest limit, carried forward rather than read as covered: no seed in the published catalog
declares more than one container, so the multi-container path has never run against a game an
operator actually plays.

**Phase 7's mechanism is built and verified end-to-end, including live in the browser**: sign
and register a binary, click Update on a connected node, and watch a real agent process
download, verify, install, drain, exit, and — restarted the way systemd would — reconnect
running the new version with the job resolving automatically. See "Agent updates are pushed
from the UI" above for the full sequence and ADR-040/ADR-041 for the design.

**The N-1 half is closed too, and the criterion is met in full.** It could not have been when it
was written: `Protocol == MinProtocol == 1`, so there was no older peer in existence to build,
and the negotiation could only be exercised against constructed ranges. As of this writing the
protocol is `11` (`internal/shared/version.Protocol`) with the floor still at `1`, so ten older
peers genuinely exist — a count that only grows as the protocol does, which is exactly why
`hack/n1-check.sh` reads `version.go` itself rather than a number written here: it walks the
file's own history and builds a peer from every protocol between the floor and the current one,
so the *script* cannot go stale as the protocol moves. This sentence can, and already has three
times — read "the protocol is `11`" as accurate as of this edit, `version.go` as the actual source
of truth, and `hack/n1-check.sh`'s own output as what to trust over either. The verified run below
was made at protocol 5, when four older peers existed (0.11.0 at
protocol 1, 0.39.0 at 2, 0.40.1 at 3, 0.44.0 at 4), and ran the whole criterion against them —
end to end on a real control plane with real Podman:

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

**Phase 8 is done.** Its own criterion originally named Dune Awakening too, for the same reason
phase 6's did, and was corrected on the same reasoning: naming a game with no public
dedicated-server image made a backup phase's completion hostage to an unrelated dependency. What
it actually demonstrated is a real control plane, a real `ygg-agent`, and real Podman backing up a
multi-container pod (primary plus a database sidecar with a volume), running a pre-hook standing
in for `pg_dump` via `Runtime.Exec`, destroying the sidecar's live data, restoring it, and bringing
the whole pod back up running — against the same synthetic game+database+broker seed phase 6 uses.
A real Dune Awakening seed remains catalog work, not a gate on either phase.

Nothing is outstanding from the original scope. What the table's row bundled in but ADR-042
deferred as its own follow-on work rather than shoehorning into that round — a cron-like
backup scheduler, and resource-usage graphs — are both built: the scheduler as an
interval-based one reusing the manual backup path (ADR-045, "Scheduled backups" above), and
the graphs as a bounded-ring-table collector and server-rendered SVG (ADR-046, "Resource
graphs" above).

One limit is worth carrying forward rather than reading "Done" as covering it. Backups are
node-local: an archive never leaves the node it was taken on and a restore reads it back there,
so there is no browser download and no off-node retention (ADR-042). (The other limit this
paragraph used to carry — the network series being a cumulative byte counter — was closed by
ADR-075, which differences it into a per-second rate; see "Resource graphs" above.)
