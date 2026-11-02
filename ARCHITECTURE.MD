# Architecture

## Goal

This repository runs a **MikroTik RouterOS** router in Docker (the upstream
`docker-routeros` image). This fork adds a home lab around it: a Kubernetes cluster whose
**MetalLB** load balancer announces Service IPs to that router over **BGP**, the way a
bare-metal cluster announces them to a physical top-of-rack router.

What the lab lets you practise:

- RouterOS 7 BGP: instances, connections, route filters, ECMP (multipath), BFD
- MetalLB in BGP mode (FRR-K8s backend): `IPAddressPool`, `BGPPeer`, `BGPAdvertisement`, `BFDProfile`
- Configuring a router as code (Ansible) and by hand (RouterOS CLI)
- Following a packet from a client, through the router, to a pod

```mermaid
flowchart LR
    client["Client"]
    router["RouterOS<br/>AS 64512"]
    subgraph k8s["Kubernetes (kind)"]
        metallb["MetalLB speaker<br/>AS 64513"]
        svc["Service type LoadBalancer<br/>10.250.0.10"]
        pod["nginx pod"]
    end

    metallb -- "BGP: 10.250.0.10/32 is reachable via me" --> router
    client -- "HTTP to 10.250.0.10" --> router
    router -- "forward to the announcing node" --> svc
    svc --> pod
```

Everything runs on one Apple Silicon Mac inside Docker Desktop, so there is no extra
hardware.

## Topology

### Containers and networks

```mermaid
flowchart TB
    subgraph mac["Mac (Apple Silicon)"]
        you["You: browser, Winbox, ssh,<br/>Ansible, kubectl, Claude Code + MCP"]

        subgraph dd["Docker Desktop Linux VM (arm64, 4 CPU / 6 GB)"]
            subgraph netdef["Docker network: docker-routeros_default"]
                rtr_up["router container<br/>uplink interface"]
            end

            subgraph netkind["Docker network: kind (172.18.0.0/16)"]
                rtr_lan["router container<br/>bridged interface<br/>172.18.0.250"]
                worker["routeros-lab-worker<br/>172.18.0.2"]
                cp["routeros-lab-control-plane<br/>172.18.0.3"]
            end
        end
    end

    you -- "127.0.0.1:8080 / 2222 / 8291 / 8728<br/>(published ports)" --> rtr_up
    you -- "127.0.0.1 API server<br/>(kind-routeros-lab context)" --> cp
    worker -- "BGP session, TCP 179" --> rtr_lan
    cp -. "same L2 subnet, no BGP" .- rtr_lan
```

The router container joins **two** Docker networks (`compose.yml`):

| Network | Purpose | Inside RouterOS |
| --- | --- | --- |
| `docker-routeros_default` | Uplink. Carries the ports published to the Mac (`gw_priority: 1` keeps it the default route) | `ether1`, `10.0.2.15/24` behind QEMU's NAT |
| `kind` (external) | Shared with the Kubernetes nodes, so the router and the nodes are on one LAN | `ether2`, `172.18.0.250/16` |

### Inside the router container

RouterOS is not a container image; it is a VM (RouterOS CHR, x86_64) that QEMU runs
**inside** the container. On Apple Silicon QEMU emulates the x86 CPU (no KVM), which is
why the router is slow to boot and is kept small (1 vCPU, 256 MB).

```mermaid
flowchart LR
    subgraph container["routeros container (Alpine)"]
        up["uplink eth<br/>(docker-routeros_default)"]
        slirp["QEMU user-mode NAT<br/>port forwards 22, 23, 80, 443,<br/>8291, 8728, 8729, ..."]
        lan["kind eth<br/>IP removed, keeps Docker MAC"]
        br["Linux bridge qemubr1"]
        tap["TAP device"]
        dhcp["udhcpd<br/>offers 172.18.0.250"]

        subgraph vm["QEMU VM: RouterOS CHR 7.x"]
            e1["ether1<br/>10.0.2.15"]
            e2["ether2<br/>172.18.0.250<br/>MAC from nic_mac"]
        end
    end

    up --> slirp --> e1
    lan --- br --- tap --- e2
    dhcp -.-> br
```

The entrypoint (`scripts/entrypoint.sh`) builds this at start-up:

1. Finds the uplink: the interface with the IPv4 default route. It stays untouched, so the
   published ports keep working through QEMU's NAT.
2. Takes the other interface (the `kind` one), removes its IP and puts it into the bridge
   `qemubr1`. It keeps its Docker-assigned MAC. This needs `cap_add: NET_ADMIN`.
3. Starts QEMU with a second NIC on a TAP device attached to that bridge. The guest NIC
   (`ether2`) gets the MAC stored in `routeros_data/nic_mac`. This needs
   `devices: /dev/net/tun`.
4. Runs a one-lease DHCP server that offers the container's own kind address to the guest.

The result: the address Docker assigned on the kind network (`172.18.0.250`) now belongs
to RouterOS's `ether2`, which sits on the same L2 segment as the kind nodes.

The bridge port and the guest must have **different** MACs. The upstream image's entrypoint
gives the bridge port the guest's MAC; the bridge then treats frames for the guest as its
own and drops them, so ARP works but no unicast traffic (BGP, SSH) reaches RouterOS.
`compose.yml` therefore mounts this repository's fixed `scripts/entrypoint.sh` over the
image's copy.

## Addressing and numbers

| Item | Value | Defined in |
| --- | --- | --- |
| kind network | `172.18.0.0/16`, gateway `172.18.0.1` | Docker (chosen when the cluster is created) |
| Router on kind | `172.18.0.250` (`ether2`) | `compose.yml`, `ansible-routeros-bgp/group_vars/routeros.yml` |
| kind worker / control-plane | `172.18.0.2` / `172.18.0.3` | Docker (assigned by kind) |
| Router management | `10.0.2.15` (`ether1`, QEMU NAT) | QEMU defaults |
| LoadBalancer pool | `10.250.0.10-10.250.0.50` | `metallb/bgp.yaml` |
| Prefixes the router accepts | inside `10.250.0.0/24` | `group_vars/routeros.yml` (`metallb_pool_prefix`) |
| Pod / Service subnets | `10.244.0.0/16` / `10.96.0.0/16` | `kind/cluster.yaml` |
| Router AS | `64512` | `group_vars/routeros.yml`, `metallb/bgp.yaml` (`peerASN`) |
| MetalLB AS | `64513` (eBGP) | `metallb/bgp.yaml` (`myASN`), `group_vars/routeros.yml` |

The pool is deliberately **outside** the kind subnet. If it were inside, clients would
reach the IPs directly over L2 (ARP) and BGP would play no part.

### Ports published to the Mac

All bound to `127.0.0.1` only.

| Mac | Router service |
| --- | --- |
| `127.0.0.1:8080` | WebFig (HTTP 80) |
| `127.0.0.1:2222` | SSH (Ansible, MCP server, your terminal) |
| `127.0.0.1:2223` | Telnet |
| `127.0.0.1:8291` | Winbox |
| `127.0.0.1:8728` / `8729` | RouterOS API / API-SSL |

## How BGP works here

### Session and route announcement

The router only listens (`listen=yes connect=no`) for any address in the kind subnet.
MetalLB's FRR (FRR-K8s mode) on the worker node opens the session. The `BGPPeer`'s
`nodeSelectors` keep the control-plane node from peering, because kind excludes it from
announcing LoadBalancer IPs.

```mermaid
sequenceDiagram
    autonumber
    participant K as kubectl
    participant C as MetalLB controller
    participant S as FRR on worker<br/>(AS 64513, 172.18.0.2)
    participant R as RouterOS<br/>(AS 64512, 172.18.0.250)

    S->>R: TCP 179 connect, BGP OPEN (AS 64513)
    R->>R: match connection "metallb":<br/>source in 172.18.0.0/16, AS 64513
    R-->>S: OPEN, KEEPALIVE: session established
    K->>C: apply Service web (type LoadBalancer)
    C->>C: assign 10.250.0.10 from lab-pool
    S->>R: UPDATE 10.250.0.10/32, next hop 172.18.0.2
    R->>R: filter metallb-in: inside 10.250.0.0/24, accept
    R->>R: install route 10.250.0.10/32 via 172.18.0.2
    Note over S,R: Keepalives every ~30-60 s.<br/>Optional BFD probes every 1 s
```

### Traffic path

```mermaid
sequenceDiagram
    autonumber
    participant CL as Client<br/>(container on kind, route via router)
    participant R as RouterOS
    participant W as kind worker 172.18.0.2
    participant P as nginx pod

    CL->>R: HTTP to 10.250.0.10
    R->>R: route lookup: 10.250.0.10/32 via 172.18.0.2 (BGP)
    R->>W: forward packet
    W->>W: kube-proxy: Service IP to pod IP
    W->>P: deliver
    P-->>W: response
    W-->>CL: response, directly on the shared L2 network
```

The return traffic does not pass through the router: the worker and the client are on the
same subnet, so the worker answers directly (asymmetric routing). That is fine for this lab,
because the router has no stateful firewall rules.

The Mac itself cannot reach `10.250.0.x`: Docker Desktop does not route from macOS into its
networks. Test from the router (`/tool/fetch`) or from a container on the `kind` network
that routes `10.250.0.0/24` via `172.18.0.250` (see `NOTES.md`, step 6).

### Failure detection: BGP alone vs BFD

```mermaid
sequenceDiagram
    participant W as Worker node
    participant R as RouterOS

    Note over W,R: Without BFD
    W-xR: worker dies
    R->>R: waits for the BGP hold time (90-180 s)<br/>still sends traffic to the dead node
    R->>R: hold time expires: drop session and routes

    Note over W,R: With BFD (1 s x 3)
    W-xR: worker dies
    R->>R: 3 BFD packets missed (~3 s)
    R->>R: BFD down: BGP drops session and routes at once
```

BFD is off by default. It needs MetalLB's FRR or FRR-K8s mode, `metallb/bfd.yaml`, and
`bfd_enabled: true` in the Ansible vars. The timers are relaxed (1 s × 3) because the
emulated router can miss shorter intervals and flap.

## Who configures what

```mermaid
flowchart LR
    you(("You"))

    subgraph tools["Tools on the Mac"]
        kindcli["kind CLI"]
        compose["docker compose"]
        ansible["Ansible (uv, Python 3.14)<br/>ansible-routeros-bgp/"]
        manual["RouterOS CLI by hand<br/>ALTERNATIVE.md"]
        helm["helm + kubectl<br/>(context kind-routeros-lab)"]
        mcp["Claude Code + mikrotik MCP<br/>(.mcp.json, uvx)"]
    end

    cluster["kind cluster<br/>kind/cluster.yaml"]
    routerc["router container<br/>compose.yml"]
    ros["RouterOS config:<br/>IP, filter, BGP, BFD"]
    mlb["MetalLB:<br/>metallb/*.yaml"]

    you --> kindcli & compose & ansible & manual & helm & mcp
    kindcli -- "creates nodes + kind network" --> cluster
    compose -- "starts router on default + kind" --> routerc
    ansible -- "SSH 127.0.0.1:2222" --> ros
    manual -- "WebFig / SSH / Winbox" --> ros
    mcp -- "SSH 127.0.0.1:2222<br/>inspect, small changes" --> ros
    helm -- "install chart, apply CRs" --> mlb
    mlb -. "runs in" .-> cluster
    ros -. "runs in" .-> routerc
```

Set-up order matters, because each step needs the previous one:

```mermaid
flowchart LR
    s0["0. Docker Desktop<br/>4 CPU / 6 GB"] --> s1["1. kind create cluster"]
    s1 --> s2["2. check kind subnet"]
    s2 --> s3["3. docker compose up -d<br/>reset password in WebFig"]
    s3 --> s4["4. Ansible playbook<br/>(or ALTERNATIVE.md)"]
    s4 --> s5["5. helm install MetalLB<br/>apply metallb/bgp.yaml"]
    s5 --> s6["6. apply demo.yaml<br/>verify on router"]
    s6 -.-> s7["7. optional BFD"]
```

- The `kind` network must exist before the router container can join it (step 1 before 3).
- `ether2` must exist before Ansible can configure it (step 3 before 4).
- MetalLB's CRDs must be installed before `bgp.yaml` can be applied (Helm before `kubectl apply`).

The full commands are in `NOTES.md`, section 4.

## Repository map

| Path | Role |
| --- | --- |
| `Dockerfile`, `scripts/` | Upstream image: Alpine + QEMU + RouterOS CHR, and the entrypoint that builds the bridge |
| `compose.yml` | This lab's router: resource limits, two networks, published ports |
| `docker-compose.dist.yml` | Upstream example compose file |
| `routeros_data/` | Volume: `nic_mac` (bridged NIC MAC) and `disk2.raw` (second disk) |
| `kind/cluster.yaml` | Kubernetes cluster: 1 control-plane + 1 worker |
| `metallb/` | `bgp.yaml` (pool, peer, advertisement), `bfd.yaml` (optional), `demo.yaml` (test app) |
| `ansible-routeros-bgp/` | uv project with the playbook that configures RouterOS; `ALTERNATIVE.md` has the same steps as CLI commands |
| `.mcp.json` | MikroTik MCP server for Claude Code (password comes from the environment) |
| `NOTES.md` | Concepts (BGP, BFD, FRR) and the step-by-step lab guide |

## Design decisions

| Decision | Why |
| --- | --- |
| kind instead of minikube | Nodes share Docker's memory instead of reserving ~1.8 GB each; the cluster is a YAML file in the repo |
| eBGP (64512 ↔ 64513) | MetalLB's documented default; no iBGP route-reflection rules to think about |
| Router listens on the whole kind subnet | New worker nodes peer without router changes |
| Route filter on input | The router accepts only the LoadBalancer pool, nothing else a node might announce |
| Pool outside the kind subnet | Forces traffic through BGP routing instead of L2 |
| FRR-K8s mode (Helm default) | Supports BFD; native mode does not |
| Ansible over SSH with RouterOS scripts | The installed `community.routeros` API modules lacked the RouterOS 7.20+ BGP fields (`instance`, `multipath`, `use-bfd`); CLI commands work on any version |
| Objects tagged `managed by ansible: <key>` | Lets the playbook (and hand-made config from `ALTERNATIVE.md`) be re-applied without duplicates |
| Everything bound to `127.0.0.1` | The router starts with an empty `admin` password |

## Limitations

- **Slow router.** RouterOS is emulated x86 on ARM: 2-4 minute boots, higher CPU than real hardware.
- **1 Mbps per interface.** The free RouterOS CHR licence limits throughput; fine for
  configuration practice, not for performance tests.
- **Router config is not persistent.** The RouterOS system disk lives in the container,
  not in `routeros_data/`. Recreating the container (`docker compose down/up`, compose
  changes, image updates) resets RouterOS: empty password, no config, new SSH host key.
  Re-apply with Ansible or `ALTERNATIVE.md`.
- **No access from macOS to LoadBalancer IPs.** Docker Desktop does not route into its
  networks; test from the router or from a container on the `kind` network.
- **Subnet depends on Docker.** kind's network subnet is picked when the cluster is created.
  If it is not `172.18.0.0/16`, update `compose.yml`, the Ansible vars and `metallb/bgp.yaml`.
