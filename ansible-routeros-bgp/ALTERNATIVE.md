# Alternative: configure BGP by hand (imperative)

These are the RouterOS commands that `configure-router.yml` runs, written out so you can
paste them yourself. Use them to learn what the playbook does, or when Ansible is not
available.

| | Playbook | These commands |
| --- | --- | --- |
| Style | Declarative: describes the end state | Imperative: "add this, then that" |
| Re-running | Safe; updates objects in place, reports `changed=0` | Running `add` twice creates duplicates or fails |
| Checks | Preflight stops on a wrong interface or old RouterOS | You check by hand (step 1) |

Values are the defaults from `group_vars/routeros.yml`:

| Setting | Value |
| --- | --- |
| LAN interface / address | `ether2` / `172.18.0.250/16` |
| BGP instance | `lab`, AS `64512`, router ID `172.18.0.250`, multipath `2` |
| BGP connection | `metallb`, eBGP, listens for AS `64513` from `172.18.0.0/16` |
| Accepted prefixes | inside `10.250.0.0/24` (filter chain `metallb-in`) |

The comments are the same ones the playbook uses (`managed by ansible: <key>`). That way you
can configure by hand now and run the playbook later: it finds these objects and updates
them instead of adding duplicates.

## Where to run them

Any RouterOS terminal works:

- WebFig at `http://127.0.0.1:8080` → **Terminal** (top right)
- SSH: `ssh -p 2222 admin@127.0.0.1`
- Winbox at `127.0.0.1:8291` → **New Terminal**

The router must already be on the kind network (`compose.yml` with the `kind` network,
`NET_ADMIN` and `/dev/net/tun`). Otherwise `ether2` does not exist.

## 1. Check the router (read-only)

Same as the playbook's preflight.

```routeros
# RouterOS version
:put [/system/resource/get version]

# The bridged interface must exist. Its MAC must match routeros_data/nic_mac on the Mac
/interface/ethernet/print where name=ether2

# The BGP instance menu must exist (newer RouterOS 7): look for "instance" in the list
/console/inspect request=child path=routing,bgp
```

Compare the MAC on the Mac side with:

```bash
cat routeros_data/nic_mac
```

RouterOS shows it in upper case; that is fine.

## 2. LAN address on ether2

```routeros
/ip/address/add address=172.18.0.250/16 interface=ether2 \
    comment="managed by ansible: lan-address"
```

The kind nodes (`172.18.0.2`, `172.18.0.3`) should now answer:

```routeros
/ping 172.18.0.2 count=3
```

## 3. Route filter: accept only the MetalLB pool

Anything outside `10.250.0.0/24` is dropped. The default action of a filter chain is drop.

```routeros
/routing/filter/rule/add chain=metallb-in \
    rule="if (dst in 10.250.0.0/24) { accept }" \
    comment="managed by ansible: metallb-in"
```

## 4. BGP instance

```routeros
/routing/bgp/instance/add name=lab as=64512 router-id=172.18.0.250 multipath=2 \
    comment="managed by ansible: bgp-instance"
```

`multipath=2` installs up to two equal routes (ECMP), one per announcing node. Set it to the
number of worker nodes.

## 5. BGP connection for MetalLB

```routeros
/routing/bgp/connection/add name=metallb instance=lab \
    local.role=ebgp local.address=172.18.0.250 \
    remote.address=172.18.0.0/16 remote.as=64513 \
    listen=yes connect=no input.filter=metallb-in use-bfd=no \
    comment="managed by ansible: bgp-connection"
```

| Part | Meaning |
| --- | --- |
| `instance=lab` | Use the AS and router ID from step 4 |
| `local.role=ebgp` | Different AS on each side. For iBGP use `ibgp` and the same AS on both sides |
| `remote.address=172.18.0.0/16` + `listen=yes connect=no` | Accept a session from any node in the kind subnet; MetalLB opens the connection |
| `remote.as=64513` | Only accept peers that say they are AS 64513 |
| `input.filter=metallb-in` | Run received routes through the filter from step 3 |

## 6. Optional: BFD

Only with MetalLB in FRR or FRR-K8s mode and a `BFDProfile` on the `BGPPeer`
(`metallb/bfd.yaml`).

```routeros
/routing/bfd/configuration/add interfaces=ether2 min-rx=1s min-tx=1s multiplier=3 \
    comment="managed by ansible: bfd"
/routing/bgp/connection/set [find name=metallb] use-bfd=yes
```

To turn it off again:

```routeros
/routing/bgp/connection/set [find name=metallb] use-bfd=no
/routing/bfd/configuration/remove [find comment="managed by ansible: bfd"]
```

## 7. Verify

Same as `--tags verify`. Sessions appear once MetalLB is installed and `metallb/bgp.yaml`
is applied.

```routeros
# One established (E) session, from the worker node
/routing/bgp/session/print
# 10.250.0.10/32 via the worker, once a LoadBalancer Service exists
/ip/route/print where bgp
# Only with BFD enabled
/routing/bfd/session/print
```

With `metallb/demo.yaml` applied:

```routeros
/tool/fetch url=http://10.250.0.10 output=user
```

If a session does not come up:

```routeros
/log/print where topics~"bgp"
```

## Change a value later

Use `set` on the existing object instead of `add` again. For example, three worker nodes:

```routeros
/routing/bgp/instance/set [find name=lab] multipath=3
```

## Remove everything

Reverse order, because the connection refers to the instance and the filter chain:

```routeros
/routing/bgp/connection/remove [find comment~"managed by ansible"]
/routing/bgp/instance/remove [find comment~"managed by ansible"]
/routing/filter/rule/remove [find comment~"managed by ansible"]
/routing/bfd/configuration/remove [find comment~"managed by ansible"]
/ip/address/remove [find comment~"managed by ansible"]
```

## All in one paste

Steps 2-5 as a single block, for a freshly reset router:

```routeros
/ip/address/add address=172.18.0.250/16 interface=ether2 comment="managed by ansible: lan-address"
/routing/filter/rule/add chain=metallb-in rule="if (dst in 10.250.0.0/24) { accept }" comment="managed by ansible: metallb-in"
/routing/bgp/instance/add name=lab as=64512 router-id=172.18.0.250 multipath=2 comment="managed by ansible: bgp-instance"
/routing/bgp/connection/add name=metallb instance=lab local.role=ebgp local.address=172.18.0.250 remote.address=172.18.0.0/16 remote.as=64513 listen=yes connect=no input.filter=metallb-in use-bfd=no comment="managed by ansible: bgp-connection"
```
