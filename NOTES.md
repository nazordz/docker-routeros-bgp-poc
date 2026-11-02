# Personal Notes

## 1. BGP (Border Gateway Protocol)

BGP is the routing protocol routers use to tell each other which networks (prefixes) they
can reach. It is what holds the internet together between providers, and it is also the
standard way for a Kubernetes cluster to tell a physical router "send traffic for this IP to me".

### How it works

- Two routers (peers) open a TCP session on **port 179** and exchange route announcements:
  "prefix `10.250.0.10/32` is reachable via next hop `172.19.0.3`".
- Each side belongs to an **AS** (Autonomous System), identified by an AS number (ASN).
  Private ASNs for labs: `64512`–`65534`.
- Routes are chosen by attributes (AS path length, local preference, MED, ...), not by
  link speed. BGP is about policy: what you accept and what you announce.
- Keepalives detect a dead peer. The default hold time is ~90s (RouterOS: 180s), which is
  slow; that is the problem BFD solves (section 2).

### eBGP vs iBGP

| | eBGP | iBGP |
| --- | --- | --- |
| Peers are in | Different ASes | The same AS |
| Typical use | Between organizations, or router ↔ Kubernetes | Inside one network |
| Gotcha | None for directly connected peers | Routes learned from one iBGP peer are not passed to another without a route reflector or full mesh |

For MetalLB, eBGP is the simple choice: router AS `64512`, MetalLB AS `64513`.

### In this lab (RouterOS ↔ MetalLB)

- MetalLB runs a speaker on each node. When a `LoadBalancer` Service gets IP `10.250.0.10`,
  every node that can serve it announces `10.250.0.10/32` with itself as next hop.
- RouterOS installs that route. With `multipath`, it keeps one route per node (ECMP) and
  spreads traffic across them.
- If a node dies, its announcement disappears and RouterOS stops sending traffic to it,
  but only once the BGP session times out.

```routeros
# Sessions and their state (E = established)
/routing/bgp/session print
# Routes learned from MetalLB
/ip/route print where bgp
```

## 2. BFD (Bidirectional Forwarding Detection)

BFD is a lightweight "are you still there?" protocol between two directly connected
routers. It does not carry routes; it only answers one question very fast: is the path to
the neighbor still working?

### Why it exists

BGP's own keepalives take 90-180 seconds to declare a peer dead. During that time, traffic
is still sent to a node that is gone. BFD exchanges tiny UDP packets (port 3784 for
single-hop) every few hundred milliseconds, so a failure is detected in around a second.
When BFD says the neighbor is down, BGP tears the session down immediately and withdraws
its routes.

### Key timers

- **min-tx / transmitInterval**: how often I want to send BFD packets.
- **min-rx / receiveInterval**: how often I can handle receiving them.
- **multiplier / detectMultiplier**: how many packets can be missed before the neighbor is
  declared down.
- Detection time ≈ negotiated interval × multiplier. For example, 300ms × 3 = ~0.9s.

### In this lab

- MetalLB supports BFD only in its FRR-based modes (FRR-K8s, the default backend, or the
  deprecated FRR mode), not in native mode. You create a `BFDProfile` and reference it from
  the `BGPPeer` with `bfdProfile: <name>`.
- RouterOS: allow BFD on the interface with `/routing/bfd/configuration` (sessions that no
  entry matches are not allowed), then set `use-bfd=yes` on the BGP connection.
- This router runs under QEMU emulation on one vCPU, so very short timers may cause false
  "down" events (flapping). Start relaxed (for example 1000ms × 3) and tighten later.

```routeros
/routing/bfd/configuration add interfaces=ether2 min-rx=1s min-tx=1s multiplier=3
/routing/bgp/connection set metallb use-bfd=yes
/routing/bfd/session print
```

```yaml
apiVersion: metallb.io/v1beta1
kind: BFDProfile
metadata:
  name: lab-bfd
  namespace: metallb-system
spec:
  receiveInterval: 1000
  transmitInterval: 1000
  detectMultiplier: 3
# then add `bfdProfile: lab-bfd` to the BGPPeer spec
```

## 3. FRRouting (Free Range Routing)

FRRouting (FRR) is an open-source routing software suite for Linux. It turns a normal Linux
machine or container into a router that speaks BGP, OSPF, IS-IS, BFD, PIM and more. It is
a fork of Quagga, and it is used by many network vendors, cloud platforms and Kubernetes
networking projects.

### How it is built

- Each protocol runs as its own daemon: `bgpd`, `ospfd`, `bfdd`, and so on.
- `zebra` sits in the middle: it collects routes from the protocol daemons and installs the
  winners into the Linux kernel routing table.
- `vtysh` is the shell for all of them, with a Cisco-like CLI (`show ip bgp summary`,
  `show bfd peers`, `configure terminal`). The configuration is stored in `frr.conf`.

### Why it matters for MetalLB

MetalLB has different backends for speaking BGP:

| Mode | What speaks BGP | BFD | IPv6 |
| --- | --- | --- | --- |
| Native | MetalLB's own Go BGP code | No | No |
| FRR (deprecated) | FRR inside the speaker pod, configured by MetalLB | Yes | Yes |
| FRR-K8s (default) | FRR managed by a separate `frr-k8s` component with its own API | Yes | Yes |

So when you configure a `BGPPeer` or `BFDProfile` in MetalLB, FRR is what actually opens
the session to RouterOS. To debug the cluster side, run `vtysh` inside the FRR container:

```bash
kubectl --context kind-routeros-lab -n metallb-system get pods
kubectl --context kind-routeros-lab -n metallb-system exec -it <frr-pod> -c frr -- vtysh -c "show bgp summary"
kubectl --context kind-routeros-lab -n metallb-system exec -it <frr-pod> -c frr -- vtysh -c "show bfd peers"
```

In FRR-K8s mode the FRR pods may be in a separate namespace (for example `frr-k8s-system`),
depending on how MetalLB was installed. Check `kubectl get pods -A | grep frr`.

### FRR vs RouterOS

Both are routing stacks that implement the same standard protocols, which is why they can
peer with each other. RouterOS is MikroTik's own operating system with its own CLI; FRR is
software you run on Linux. In this lab, RouterOS plays the "physical" upstream router and
FRR (inside MetalLB) plays the Kubernetes side.

## 4. Lab setup: RouterOS + kind + MetalLB BGP

### Topology

```text
Docker network "kind" (check the subnet in step 2; 172.19.0.0/16 assumed below)
 ├── routeros-lab-control-plane   172.19.0.x
 ├── routeros-lab-worker          172.19.0.y   ← MetalLB (AS 64513) peers from here
 └── routeros (ether2)            172.19.0.250 ← RouterOS (AS 64512)

RouterOS ether1 = QEMU NAT (10.0.2.15), only for management from the Mac:
WebFig 127.0.0.1:8080, SSH 127.0.0.1:2222, Winbox 127.0.0.1:8291

LoadBalancer pool: 10.250.0.10-10.250.0.50, announced as /32 routes via BGP
```

| Piece | File |
| --- | --- |
| Router container | `compose.yml` |
| Kubernetes cluster (1 control-plane + 1 worker) | `kind/cluster.yaml` |
| Router config (IP, filter, BGP, optional BFD) | `ansible-routeros-bgp/` (see its `README.md`) |
| MetalLB config | `metallb/bgp.yaml`, `metallb/bfd.yaml` |
| Test app | `metallb/demo.yaml` |

### Step 0: give Docker Desktop enough resources

Settings → Resources: at least **4 CPUs and 6 GB RAM**. The default here was 2 CPUs and
~1.9 GB, which is not enough for two kind nodes plus the router.

Every `kubectl`/`helm` command below names the lab context explicitly. The kubeconfig
also holds real clusters (EKS, GKE, DigitalOcean), and the current context is not the lab.

### Step 1: create the kind cluster

```bash
brew install kind
kind create cluster --config kind/cluster.yaml
kubectl --context kind-routeros-lab get nodes
```

### Step 2: check the kind network subnet

```bash
docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}'
```

If it is not `172.19.0.0/16`, update every place that uses it:

- `compose.yml`: `ipv4_address` (step 3)
- `ansible-routeros-bgp/group_vars/routeros.yml`: `routeros_lan_address` and `metallb_peer_range`
- `metallb/bgp.yaml`: `peerAddress`

### Step 3: attach the router to the kind network

Add to the `routeros` service in `compose.yml` (keep `command`, `cpus`, `mem_limit`,
ports and volume as they are):

```yaml
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun]
    networks:
      default:
        gw_priority: 1               # keeps the published management ports working
      kind:
        ipv4_address: 172.19.0.250   # becomes ether2 in RouterOS

networks:
  kind:
    external: true
```

```bash
docker compose up -d
```

This recreates the container, which **resets RouterOS** (back to the image version, empty
`admin` password, no config). Afterwards:

1. Wait 2-4 minutes for boot (`docker compose logs -f routeros`).
2. Log in to WebFig at `http://127.0.0.1:8080` as `admin` with no password and set the password.
3. Forget the old SSH host key: `ssh-keygen -R '[127.0.0.1]:2222'`.

### Step 4: configure the router with Ansible

The playbook lives in `ansible-routeros-bgp/`, a uv project on Python 3.14
(`ansible-core` and `paramiko` are its dependencies in `pyproject.toml`).

```bash
cd ansible-routeros-bgp
export MIKROTIK_PASSWORD='your-routeros-password'

uv sync                                                      # once: creates .venv (Python 3.14)
uv run ansible-galaxy collection install -r requirements.yml # once
uv run ansible-playbook configure-router.yml --tags preflight   # read-only: connection, version, ether2, MAC
uv run ansible-playbook configure-router.yml                    # apply
uv run ansible-playbook configure-router.yml                    # again: should report changed=0
cd ..
```

Or `source .venv/bin/activate` once per shell and drop the `uv run` prefix.

The playbook puts `172.19.0.250/16` on `ether2`, adds a route filter that only accepts
`10.250.0.0/24`, creates BGP instance `lab` (AS 64512, multipath 2) and a connection that
listens for MetalLB (AS 64513) from the whole kind subnet.

### Step 5: install and configure MetalLB

```bash
helm --kube-context kind-routeros-lab repo add metallb https://metallb.github.io/metallb
helm --kube-context kind-routeros-lab install metallb metallb/metallb \
  -n metallb-system --create-namespace --wait
kubectl --context kind-routeros-lab apply -f metallb/bgp.yaml
```

The Helm chart installs MetalLB in FRR-K8s mode by default (needed for BFD). `bgp.yaml`
must be applied after the chart, because it uses MetalLB's CRDs.

### Step 6: test

```bash
kubectl --context kind-routeros-lab apply -f metallb/demo.yaml
kubectl --context kind-routeros-lab get svc web    # EXTERNAL-IP, e.g. 10.250.0.10
cd ansible-routeros-bgp && uv run ansible-playbook configure-router.yml --tags verify && cd ..
```

Expected on the router:

- `/routing/bgp/session print`: one established session, from the worker node.
- `/ip/route print where bgp`: `10.250.0.10/32` with the worker's IP as gateway.
- `/tool fetch url=http://10.250.0.10 output=user`: the nginx welcome page.

The Mac cannot reach `10.250.0.x` directly (Docker Desktop does not route into its
networks). To test like a real client, route a throwaway container through the router:

```bash
docker run --rm -it --network kind --cap-add NET_ADMIN nicolaka/netshoot \
  sh -c "ip route add 10.250.0.0/24 via 172.19.0.250 && curl -s http://10.250.0.10"
```

### Step 7 (optional): enable BFD

1. `kubectl --context kind-routeros-lab apply -f metallb/bfd.yaml`
2. In `metallb/bgp.yaml`, uncomment `bfdProfile: lab-bfd`, then apply `bgp.yaml` again.
3. In `ansible-routeros-bgp/group_vars/routeros.yml`, set `bfd_enabled: true`, then run
   `uv run ansible-playbook configure-router.yml` in that folder.
4. Check on the router: `/routing/bfd/session print`.

### Troubleshooting

| Symptom | Check |
| --- | --- |
| Playbook: `ether2 does not exist` | Router is still single-network; do step 3 |
| Playbook: `host key mismatch` | Router was recreated: `ssh-keygen -R '[127.0.0.1]:2222'` |
| Playbook/MCP: authentication failed | Router was recreated and has an empty password again (step 3) |
| No BGP session | AS numbers and `peerAddress` match the router; speaker logs: `kubectl --context kind-routeros-lab -n metallb-system logs -l component=speaker`; router: `/log print where topics~"bgp"` |
| Session up, no routes | Service has an EXTERNAL-IP? Pool inside `10.250.0.0/24` (the router filter drops anything else)? |
| `EXTERNAL-IP` stays `<pending>` | `bgp.yaml` not applied, or applied before MetalLB was ready |
| Containers restarting / exit 137 | Out of memory: raise Docker Desktop memory (step 0) |

### Tear down

```bash
kubectl --context kind-routeros-lab delete -f metallb/demo.yaml
kind delete cluster --name routeros-lab
docker compose down          # removes the router container (and its RouterOS config)
```

Before `docker compose up -d` again, remove the `kind` network from `compose.yml` (or
recreate the cluster first), because the external network no longer exists.
