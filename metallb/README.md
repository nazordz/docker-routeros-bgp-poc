# MetalLB: installation for the RouterOS BGP lab

MetalLB gives `LoadBalancer` Services an IP from `10.250.0.10-10.250.0.50` and announces
each IP to the RouterOS router (`172.18.0.250`, AS 64512) over BGP. It runs in
**FRR-K8s mode**, the Helm chart's default, which is also what makes BFD possible.

| File | Purpose |
| --- | --- |
| `values.yaml` | Helm values: pins FRR-K8s mode, keeps the control-plane node out of announcements |
| `bgp.yaml` | `IPAddressPool`, `BGPPeer` (the router) and `BGPAdvertisement` |
| `bfd.yaml` | Optional `BFDProfile` for fast failure detection |
| `demo.yaml` | nginx Deployment + `LoadBalancer` Service to test with |

Run every command from the **repository root**. Each one names the lab context
(`kind-routeros-lab`) explicitly, because the kubeconfig also holds real clusters.

## Prerequisites

- The kind cluster is running (`kind create cluster --config kind/cluster.yaml`):

  ```bash
  kubectl --context kind-routeros-lab get nodes
  ```

- `helm` and `kubectl` are installed.

The router does not have to be ready yet. MetalLB keeps retrying the BGP session until
the router answers, after `docker compose up -d` and the Ansible playbook
(`ansible-routeros-bgp/`).

## 1. Add the Helm repository

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update metallb
helm search repo metallb/metallb --versions | head -5
```

This lab is tested with chart **0.16.1** (MetalLB v0.16.1).

## 2. Install MetalLB

```bash
helm --kube-context kind-routeros-lab install metallb metallb/metallb \
  --version 0.16.1 \
  --namespace metallb-system --create-namespace \
  --values metallb/values.yaml \
  --wait --timeout 5m
```

`--wait` returns once the controller and the per-node pods are ready. The chart also
installs MetalLB's CRDs (`BGPPeer`, `IPAddressPool`, `BFDProfile`, ...) and their admission
webhooks.

## 3. Check the installation

```bash
kubectl --context kind-routeros-lab get pods -n metallb-system -o wide
```

Expected, all `Running`:

| Pod | Count | Role |
| --- | --- | --- |
| `metallb-controller-...` | 1 | Assigns IPs from the pool; serves the admission webhook |
| `metallb-speaker-...` | 1 per node (2) | Decides which node announces which IP |
| `metallb-frr-k8s-...` | 1 per node (2) | Runs FRR, which holds the actual BGP (and BFD) session to the router |
| `metallb-frr-k8s-statuscleaner-...` | 1 | Cleans up status objects of removed nodes |

The `metallb-frr-k8s` pods prove that FRR-K8s mode is active. In the deprecated FRR mode
there would be an `frr` container inside the speaker pods instead; in native mode there is
no FRR at all.

```bash
kubectl --context kind-routeros-lab get crd | grep metallb.io
```

## 4. Apply the BGP configuration

```bash
kubectl --context kind-routeros-lab apply -f metallb/bgp.yaml
kubectl --context kind-routeros-lab get ipaddresspools,bgppeers,bgpadvertisements -n metallb-system
```

If this fails with `failed calling webhook`, the controller's webhook is not serving yet.
Wait a few seconds and run the `apply` again.

What `bgp.yaml` sets up:

| Resource | Setting |
| --- | --- |
| `IPAddressPool lab-pool` | `10.250.0.10-10.250.0.50`, outside the kind subnet, so traffic must be routed via the router |
| `BGPPeer routeros` | `myASN: 64513`, `peerASN: 64512` (different ASNs: eBGP), `peerAddress: 172.18.0.250`; only worker nodes peer (`nodeSelectors`) |
| `BGPAdvertisement lab-adv` | Announces IPs from `lab-pool` to `routeros` as `/32` routes |

These values must match the router side in `ansible-routeros-bgp/group_vars/routeros.yml`.

## 5. Check the BGP session

Once the router is configured, the session is visible from the cluster side:

```bash
# Session state per node and peer (FRR-K8s)
kubectl --context kind-routeros-lab get bgpsessionstates -n metallb-system -o wide

# FRR's own view, on the worker node
FRR_POD=$(kubectl --context kind-routeros-lab get pods -n metallb-system \
  -l app.kubernetes.io/component=frr-k8s \
  --field-selector spec.nodeName=routeros-lab-worker -o name)
kubectl --context kind-routeros-lab exec -n metallb-system "$FRR_POD" -c frr -- vtysh -c "show bgp summary"
```

`show bgp summary` should list `172.18.0.250` with a prefix count instead of `Active` or
`Connect`. On the router: `/routing/bgp/session/print`.

## 6. Test with a LoadBalancer Service

```bash
kubectl --context kind-routeros-lab apply -f metallb/demo.yaml
kubectl --context kind-routeros-lab get svc web     # EXTERNAL-IP: 10.250.0.10
kubectl --context kind-routeros-lab get servicebgpstatuses -A
```

`servicebgpstatuses` shows which node announces the Service IP to which peer. On the
router:

```routeros
/ip/route/print where bgp
/tool/fetch url=http://10.250.0.10 output=user
```

The Mac cannot reach `10.250.0.x` directly, because Docker Desktop does not route into its
networks. Test from the router, or from a container on the `kind` network (see
`NOTES.md`, step 6).

## 7. Optional: BFD

Both sides need it: MetalLB and the router.

```bash
kubectl --context kind-routeros-lab apply -f metallb/bfd.yaml
```

Then uncomment `bfdProfile: lab-bfd` in `bgp.yaml` and apply it again:

```bash
kubectl --context kind-routeros-lab apply -f metallb/bgp.yaml
```

On the router side, set `bfd_enabled: true` in `ansible-routeros-bgp/group_vars/routeros.yml`
and run the playbook. Check it with:

```bash
kubectl --context kind-routeros-lab exec -n metallb-system "$FRR_POD" -c frr -- vtysh -c "show bfd peers"
```

## Upgrade

Read the [release notes](https://metallb.io/release-notes/) first, then:

```bash
helm repo update metallb
helm --kube-context kind-routeros-lab upgrade metallb metallb/metallb \
  --version <new-version> \
  --namespace metallb-system \
  --values metallb/values.yaml \
  --wait --timeout 5m
```

The `BGPPeer`, pool and advertisement in `bgp.yaml` are kept across upgrades.

## Uninstall

```bash
kubectl --context kind-routeros-lab delete -f metallb/demo.yaml --ignore-not-found
kubectl --context kind-routeros-lab delete -f metallb/bgp.yaml --ignore-not-found
helm --kube-context kind-routeros-lab uninstall metallb --namespace metallb-system --wait
kubectl --context kind-routeros-lab delete namespace metallb-system
```

Delete the custom resources before the chart, while their CRDs still exist. Deleting the
whole kind cluster (`kind delete cluster --name routeros-lab`) also removes everything.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `helm install` times out | Pods stuck `Pending` or `ImagePullBackOff`: `kubectl --context kind-routeros-lab describe pods -n metallb-system`. Usually Docker Desktop is low on memory |
| `apply -f bgp.yaml`: `no matches for kind "BGPPeer"` | MetalLB (and its CRDs) is not installed yet: step 2 |
| `apply -f bgp.yaml`: `failed calling webhook` | Controller not ready yet; wait a few seconds and apply again |
| `EXTERNAL-IP` stays `<pending>` | `bgp.yaml` not applied, or the pool is exhausted: `kubectl --context kind-routeros-lab describe svc web` |
| `show bgp summary`: peer in `Active` / `Connect` | Router not reachable or not configured: router container on the `kind` network, playbook applied, `peerAddress` and ASNs match |
| No session from the worker | `nodeSelectors` in `bgp.yaml`, and the speaker logs: `kubectl --context kind-routeros-lab logs -n metallb-system -l app.kubernetes.io/component=speaker` |
| `bfdProfile` rejected | MetalLB is in native mode: `values.yaml` was not used at install time |
