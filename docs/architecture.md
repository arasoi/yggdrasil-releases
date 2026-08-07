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
exists for (ADR-048).

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

Clusters are a first-class `clusters` table with a globally-unique name (ADR-037), created or
joined from the server-from-seed form: a seed that declares `cluster.supported` shows an
optional cluster name field, where a new name creates a cluster bound to the chosen node, an
existing name on that node joins it, and an existing name bound to a different node is
rejected with a message naming the right one.

`/clusters` lists every cluster with its node and members, and is where a cluster is managed
(ADR-066): rename it, take a member out, or delete it. **Joining is still only possible through
server creation** — this page can empty a cluster but not fill one.

Every operation there is guarded by something the schema does not enforce, because
`servers.cluster_id` has no foreign key: deleting a cluster is refused while it still has
members (the row would otherwise vanish underneath every one of them, still mounting its
volume), and while a cluster backup or restore is in flight. Removing a member clears the
column *and* rebuilds that server's pod, since the `/cluster` mount and `-clusterid=` argument
are fixed into the container at create time and the column alone changes nothing the game sees.

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
`internal/integration/agent_update_test.go`. What is not yet proven is N-1 compatibility
against a *real* older protocol version, since none exists yet (`Protocol == MinProtocol == 1`
today): the mechanism is tested against a hypothetical version range instead
(`internal/integration/n1_compat_test.go`), the same honesty the ARK and Dune Awakening entries
below already apply to their own exit criteria.

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
container exits or the agent shuts down. Log-rule scanning for readiness and crash detection
is not implemented yet — it arrives with seeds in phase 4, alongside the ring buffer that
would back it if backlog-on-attach alone proves insufficient once games with chattier startup
logs are in scope.

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

**Moving a server to another node** is its own action on that page, and a Job rather than an
edit (ADR-063). The source node archives the server's writable state and uploads it; the
control plane stages it; the target downloads, restores, and the server is repointed,
reallocated, provisioned and — if it was running — started again. The **install does not
travel**: it is reproducible from the seed (the same reason ADR-023 excludes it from backups),
so the target resolves its own and the source's refcount drops. Ports are reallocated from the
target's range, so players need the new numbers. A clustered server cannot move on its own,
since a cluster's volume is node-local (ADR-020). The archive travels over two RPCs of its
own rather than the persistent stream, exactly as an agent binary does (ADR-041); backups
themselves remain node-local (ADR-042).

**Every node has a page** at `/nodes/{id}` (ADR-065), for the same reason every server does:
a node was a table row, so the capacity its agent reports at every handshake — cores, memory,
disk — was collected, persisted, and shown nowhere. It carries the host's facts, its agent and
protocol versions, its storage paths, what it holds (servers, clusters, installs), and the two
settings that belong to a host rather than to a server: its port range and its address. The
Nodes list keeps the port-range form as a shortcut, so that form now renders errors back to
whichever page submitted it.

**The servers list is grouped by node**, one collapsible group per host, with that host's
clusters nested inside it and its unclustered servers below them. "What is where" is the
question that page is most often opened to answer, and a flat list answered it only by
reading a Node column on every row — so that column is gone, since the group heading now
says it once. A collapsed group still carries its host's summary: running count, anything
needing attention, and whether the agent is connected. Only nodes with servers become
groups. Creating a server is one **New server** button that opens a chooser — from a seed,
or from a container image — and lands on the page for whichever was picked (`/servers/new-from-seed`
or `/servers/new-from-image`); the image form used to sit permanently expanded at the bottom
of the list, which put a form nobody was filling in below every server they were looking at.
See ADR-054's amendment.

Run state is carried by colour as well as text: a badge plus a stripe on the row's leading
edge, both chosen by one Go method so nothing can disagree about what state a server is in.
`degraded` is coloured apart from `crashed`, since ADR-019 makes them operationally distinct;
idle is muted, so a healthy fleet reads calm and only trouble draws the eye — the wall-mounted
case the stylesheet is written for. Colour stays the only swappable axis (Frost / Grove /
Ember), and semantic colour sits outside the palettes entirely.

**Those badges, stripes and counts move on their own.** State changes are pushed to every open
tab and the page re-reads itself (ADR-058, "Live state" above) — which is what makes the
wall-mounted case actually work, rather than showing whatever was true when someone last
pressed reload.

## Seeds

A seed is YAML, not code, and now describes install strategy, pod composition, and
cluster support. Sketch:

```yaml
id: ark-survival-ascended
schema: 1

install:
  shared: true              # one install, many servers
  method: steamcmd
  app_id: 2430930
  mount: { path: /game, mode: ro }

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
    image: yggdrasil/ark-asa:latest
    command: "...?Map={{.Vars.map}}?..."
    ports:
      - { name: game,  protocol: udp, default: 7777 }
      - { name: query, protocol: udp, default: 27015 }
      - { name: rcon,  protocol: tcp, default: 27020 }

variables:
  - { name: map, default: TheIsland, editable: true }

logs:
  ready: "Server has completed startup"
  crash: "Fatal error"

stop:
  command: "DoExit"
  timeout: 120s
  then: SIGTERM

backup:
  include: [saved, config]
```

ARK's three ports are all independently allocated — each gets its own preferred default and is
scanned from the node's port range if that default is taken. Some games hardcode a relationship
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

**ARK** exercises shared installs and clusters. **Dune Awakening** exercises multi-container
pods. Between them they cover all three axes.

**Architecture.** Nodes report their architecture and seeds declare what they support;
invalid placement is rejected with a clear error rather than an opaque exec-format crash-loop.
Placement scheduling is deferred until a mixed fleet exists (ADR-016).

### What the bundled seeds cover

The bundled seeds deliberately sit at opposite ends of ADR-018's install model, and all run
end-to-end from the UI with no game-specific Go code:

- **Paper** has a non-shared `install` (`method: download`) and a `config.files` block that
  renders `eula.txt` and `server.properties` from real template files in its bundle directory
  (`ConfigFile.SourcePath`, ADR-049) — the licence-gate row in the table above.
  `internal/agent/install` fetches a single file or an archive (`zip`/`targz`) and unpacks it
  with the same zip-slip protection `internal/agent/files` applies to operator-supplied paths
  (ADR-033), applied here to archive-supplied ones. Rendered config files are pushed with the
  *existing* phase 3 `FileWrite` RPC rather than a new mechanism — writing a file into a
  server's own directory is exactly what that RPC already does.
- **Bedrock** has no `install` block at all: its image downloads the official server itself at
  container start, the case ADR-018 already calls out as needing no separate install.
- **ARK Survival Ascended** (`seeds/library/ark-survival-ascended/seed.yaml`) exercises
  ADR-018 and ADR-020: a shared SteamCMD install mounted read-only at `/game`, per-server
  writable paths for saves and config, and a `/cluster` mount with the cluster id and
  directory override passed to the primary container. It is the bundled example for the
  one-install-many-maps workflow phase 5 is building toward. Its primary container runs on
  `images/base-steamcmd-proton` and invokes the game through `umu-run`, plus a fourth writable
  path (`compatdata`) for the Wine prefix that runs under — see phase 5's status below for what
  is and is not yet proven about that path.
- **Valheim** (`seeds/library/valheim/seed.yaml`) is the SteamCMD install path proven against a real
  game end-to-end, not a synthetic stand-in: it has an official Linux dedicated server binary
  (unlike ARK ASA), so no Proton wrapper is needed, and a real run installed, provisioned,
  started, and matched its `logs.ready` rule against genuine stdout. It is also the first seed
  to use an offset-derived port (`query`, `game_port + 1`, ADR-048) — Valheim's Steam query
  port hardcoded with no independent flag, which is what that ADR's schema addition exists for.

One real-world wrinkle worth recording: PaperMC's download API changed shape between when the
sketch above was written and phase 4's implementation — the current API (`fill.papermc.io/v3`)
hands back a content-addressed URL per build rather than one a version and build number alone
can template. The Paper seed's `install.url` is therefore itself a single editable variable
(the resolved download URL) rather than a templated `{version}/{build}` path — bumping the
Minecraft version an operator gets is editing that one field, still a data change rather than
a Go one (ADR-007), just not the exact shape originally sketched.

A seed's containers may name their primary container's role anything readable ("game", as
above) — but on the wire it is always normalised to the same `primary` role every other path
already uses (docs/conventions.md's container conventions table), since the console page and
allocation lookups have no way to discover a differently-named one.

Install jobs (ADR-021) stream progress from agent to control plane over dedicated
`InstallStart`/`InstallProgress` messages, correlated by `job_id` rather than the request's
`command_id` — installing can take anywhere from seconds to minutes, and nothing about it
should block the stream the way a lifecycle command's reply does. Provisioning that has to
wait on one is finished when the install job reports success, by the same handler that records
it (`hub.InstallReconciler`, ADR-059), and again on page load as a safety net (ADR-035) — still
no background worker for it either way. A node's storage paths, needed to build a seed-driven
pod's bind mounts, are reported at handshake rather than assumed by convention (ADR-034).

The page-load path is not redundant: it is what covers a server whose *node* was offline at the
moment its install finished, so the event-driven path had nowhere to dispatch to.

### Seed bundles, authoring UI, and Steam integration

ADR-049's directory-per-seed layout is built. A seed is now `seeds/library/<id>/` (bundled) or
`<seeds-dir>/<id>/` (operator), each holding a `seed.yaml` manifest plus, optionally, real
config-file templates under `configs/` — Paper's `eula.txt`/`server.properties` moved out of
inline YAML string blocks into `configs/eula.txt.tmpl`/`configs/server.properties.tmpl`,
referenced from the manifest by `ConfigFile.SourcePath` rather than `ConfigFile.Template`.
`internal/seed.LoadDir`/`LoadFS` resolve `SourcePath` against the bundle directory into
`Template` before validation runs, so `validateTemplates`'s load-time dry-render needed no
changes — exactly the seam ADR-049 planned. The schema bumped to 2 accordingly; a schema-1 seed
(single flat YAML file, inline templates only) is no longer accepted. `seeds/bundled.go`'s
`go:embed` targets a `library/` subtree rather than seeds/'s own directory, since embedding
`seeds/*` directly would have swept `bundled.go` itself in as if it were a seed — the exact
problem ADR-049 flagged as unresolved when it was written.

ADR-050's seed authoring UI is built for the common single-container case: `/seeds` lists every
seed (bundled and operator, badged by source), `/seeds/new` and `/seeds/{id}/edit` write straight
into the operator's `--seeds-dir` as a real bundle directory, and saving triggers an in-process
reload (`internal/control/web`'s `reloadSeeds`, guarded by a mutex around `s.seeds`) — a seed
created or edited through the UI shows up in "New server from seed" immediately, no restart.
Editing a *bundled* seed is supported by materialising an operator override with the same id on
first save, resolving the open question ADR-050 originally left unanswered. The guided form
covers identity, install (none/download/steamcmd), one writable path, the primary container,
ports, env, variables, logs, stop, and backup-include; anything it can't represent — a `Config`
block, a `Cluster`, more than one container, or an offset-derived port (`OffsetFrom`, ADR-048) —
disables the guided section entirely rather than risking a save that silently drops it, falling
back to the raw-YAML `<details>` section every seed (guided-eligible or not) also has, validated
through the same `internal/seed.Parse` a bundled seed loads through. `Seed`'s YAML tags gained
`omitempty` throughout so a guided save's `yaml.Marshal` output reads like something a human
would hand-write — the first version, before that fix, wrote `install: null`, `env: {}`, and a
spurious empty `stop:` block for every guided-created seed, caught by actually creating one
through a live browser session rather than assumed correct from the code alone.

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

A seed is data, but until ADR-060 the only way a fix to a bundled one reached an operator was a
new `yggd` release — so either `VERSION` climbed because a YAML file changed, or the shipped
seeds drifted behind the repository. Seeds now have their own release channel.

`.github/workflows/seeds.yml` is triggered by a **path filter over `seeds/`**, runs
`cmd/ygg-seedpack`, and publishes to release tags of its own — `seeds-develop-latest`,
`seeds-qa-latest`, `seeds-release-latest` — floating per channel the way ADR-038's binary
channels do, in the same public repository. Separate tags are the point rather than tidiness:
sharing release.yml's would re-couple exactly what this separates. Each seed carries its own
`version:`, and the packer refuses one `version.Compare` cannot order, since publishing that
produces a seed nobody could ever be told to update. Every bundle is loaded through
`internal/seed` before packing, so a seed `yggd` could not load fails in CI rather than on an
operator's control plane.

Seeds therefore load in **three layers** (`seed.Merge`, in order):

| Layer | Where | Why it is at this level |
|-------|-------|-------------------------|
| bundled | embedded in the binary (`go:embed`) | a fresh install has seeds with no network at all |
| catalog | `<data_dir>/seeds/<id>/` | downloaded, updated on its own schedule |
| operator | `--seeds-dir/<id>/` | hand-authored, so a catalog update can never overwrite it |

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

## Container image library

Every bundled seed's container image is either official (Paper's `eclipse-temurin`) or a
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

Network rx/tx are charted as Docker/Podman report them — **cumulative bytes since container
start**, not a computed rate — so the network graph shows total transfer rising over the
window rather than throughput. Computing a true rate would mean reasoning about counter resets
across restarts and gaps in sampling; deferred rather than built silently; see ADR-046.

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
| 6 | **In progress** | Multi-container pods: per-pod networks, dependency ordering, health gates, reverse-order shutdown, `degraded`, sidecar log/stat views | **Dune Awakening**: game + database + broker start in order and stop safely |
| 7 | **In progress** | Signed agent binary distribution, UI-triggered update, N-1 compatibility testing | Update every agent from the UI while servers stay up |
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
ARK-specific belongs baked into one) and invokes `umu-run` against ARK ASA's documented Windows
binary path.

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

**Phase 6's mechanism is built and verified, but not yet against Dune Awakening by name.**
Dependency-ordered start, health gates, reverse-order shutdown, sidecar crash-loop restart,
and the console role selector and stats panel are all implemented and proven — including a
full-stack test (real control plane, real `ygg-agent`, real Podman, no fakes) and a live
browser session against a genuinely running multi-container pod — using a synthetic
game+database+broker seed rather than the real thing, since Dune Awakening has no public
dedicated-server image to build a real seed against yet. The exit criterion stays open until
one does.

**Phase 7's mechanism is built and verified end-to-end, including live in the browser**: sign
and register a binary, click Update on a connected node, and watch a real agent process
download, verify, install, drain, exit, and — restarted the way systemd would — reconnect
running the new version with the job resolving automatically. See "Agent updates are pushed
from the UI" above for the full sequence and ADR-040/ADR-041 for the design. What is not yet
provable is the exit criterion's N-1 half against a *real* older protocol version, since one
does not exist yet (`Protocol == MinProtocol == 1`) — tested against a hypothetical version
range instead, the same kind of stand-in phases 5 and 6 use for a real ARK image and a real
Dune Awakening image respectively. The exit criterion stays open until a genuine version bump
gives N-1 something real to mean.

**Phase 8 is done, against a criterion that was reworded to what the phase actually proves.**
It originally read "restore a Dune pod including its database", and that sentence was met
literally: a real control plane, a real `ygg-agent`, and real Podman back up a multi-container
pod (primary + database sidecar with a volume), run a pre-hook standing in for `pg_dump` via
`Runtime.Exec`, destroy the sidecar's live data, restore it, and bring the whole pod back up
running. But it was met against the same synthetic game+database+broker seed phase 6 uses, for
the same reason — there is no public Dune Awakening image to build a real seed against — so
naming that game in a backup phase's criterion made this phase's completion hostage to a
dependency that has nothing to do with backups. The criterion now describes the capability
that was demonstrated, and **Dune Awakening by name remains phase 6's open item**, where the
dependency actually belongs.

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
