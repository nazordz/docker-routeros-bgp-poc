# Ansible: configure RouterOS for MetalLB BGP

This playbook turns the RouterOS lab router from `../compose.yml` into the BGP peer for
MetalLB in a local kind cluster. It connects over SSH (`127.0.0.1:2222`) and sets up:

| Object | Default |
| --- | --- |
| IP address on the interface bridged to the kind network | `172.19.0.250/16` on `ether2` |
| Route filter: accept only the MetalLB pool | chain `metallb-in`, accepts `10.250.0.0/24` |
| BGP instance | `lab`, AS `64512`, router ID `172.19.0.250`, multipath `2` |
| BGP connection: wait for any kind node to connect | `metallb`, eBGP with AS `64513`, listens on `172.19.0.0/16` |
| BFD (optional) | off |

See `../NOTES.md` for what BGP, BFD and FRR are.

It is safe to run repeatedly. Every object it creates has a comment starting with
`managed by ansible:`. That comment is how the playbook finds the object again: a second
run updates it in place and reports `ok` (unchanged) instead of creating a duplicate.

## Files

| File | Purpose |
| --- | --- |
| `configure-router.yml` | The playbook: preflight checks, configuration, verification |
| `tasks/ensure.yml` | Creates, updates or removes one RouterOS object |
| `group_vars/routeros.yml` | **The settings you edit**: addresses, AS numbers, BFD |
| `inventory.yml` | How to reach the router; credentials from the vault, or from the environment |
| `host_vars/routeros-lab.yml` | **Encrypted** router username and password (Ansible Vault, you create it in step 4) |
| `ansible.cfg` | Points at the inventory and sets longer timeouts for the emulated router |
| `requirements.yml` | Ansible collections needed |

## Before you run it

### 1. The router must be on the kind network

In single-network mode the router only has `ether1` (QEMU's NAT) and cannot peer with
MetalLB. The preflight check stops with a clear message in that case. Create the kind
cluster first, then attach the router to the `kind` network in `../compose.yml`:

```yaml
    cap_add: [NET_ADMIN]
    devices: [/dev/net/tun]
    networks:
      default:
        gw_priority: 1
      kind:
        ipv4_address: 172.19.0.250   # same as routeros_lan_address

networks:
  kind:
    external: true
```

`docker compose up -d` then recreates the router. **This resets RouterOS**: log in to
WebFig at `http://127.0.0.1:8080` as `admin` with an empty password and set your password
again before running the playbook.

### 2. Match the settings to your kind network

```bash
docker network inspect kind -f '{{range .IPAM.Config}}{{.Subnet}} {{end}}'
```

If the IPv4 subnet is not `172.19.0.0/16`, change these in `group_vars/routeros.yml`
(and the `ipv4_address` in `compose.yml`):

- `routeros_lan_address`, for example `172.20.0.250/16`
- `metallb_peer_range`, for example `172.20.0.0/16`

`metallb_pool_prefix` must contain the addresses in MetalLB's `IPAddressPool`, and must be
outside the kind subnet.

### 3. Install the dependencies

Ansible runs through [uv](https://docs.astral.sh/uv/)'s `uvx`, so nothing is installed into
your system or pyenv Python. `uvx` builds a cached environment with `ansible-core` and
`paramiko` (needed for SSH) on the first run and reuses it afterwards.

Define two short aliases for this shell session:

```bash
alias ap="uvx --from 'ansible-core>=2.16' --with paramiko ansible-playbook"
alias ag="uvx --from 'ansible-core>=2.16' ansible-galaxy"
```

Then install the collections. `ansible.cfg` sets `collections_path = ./collections`, so they
go into `ansible/collections/` (git-ignored), where the `uvx` Ansible finds them:

```bash
cd ansible
ag collection install -r requirements.yml
```

Without the aliases, use the full commands, for example
`uvx --from 'ansible-core>=2.16' --with paramiko ansible-playbook configure-router.yml`.

### 4. Credentials (Ansible Vault)

The router username and password are stored encrypted in `host_vars/routeros-lab.yml` with
`ansible-vault`. The file is unlocked with a **vault password** that you type when Ansible
asks for it, so no secret is ever stored in plain text. This works the same on any OS.

```text
host_vars/routeros-lab.yml  ── encrypted (AES256) with ──▶  vault password
(safe to commit)                                             (typed at the prompt)
```

Run the commands in this folder. The vault password is a new password you choose; it only
protects this file and is not the router password.

**4.1 Create the encrypted file.** `ansible-vault` asks for the new vault password twice, then
opens an editor. `EDITOR="code --wait"` uses VS Code (save and close the tab to finish);
without it, `vi` opens.

```bash
mkdir -p host_vars
EDITOR="code --wait" uv run ansible-vault create host_vars/routeros-lab.yml
```

Put the router login in it, using these exact variable names:

```yaml
vault_mikrotik_username: admin
vault_mikrotik_password: your-routeros-password
```

**4.2 Check it.**

```bash
head -1 host_vars/routeros-lab.yml                   # $ANSIBLE_VAULT;1.1;AES256: encrypted on disk
uv run ansible-vault view host_vars/routeros-lab.yml # asks for the vault password, shows the content
```

**4.3 Run playbooks with the vault password.** Add `--ask-vault-pass` (short: `-J`) to every
`ansible-playbook` command:

```bash
unset MIKROTIK_USERNAME MIKROTIK_PASSWORD   # make sure the vault, not the environment, is used
uv run ansible-playbook configure-router.yml --tags preflight --ask-vault-pass
uv run ansible-playbook configure-router.yml -J
```

**Day-to-day**

| Task | Command |
| --- | --- |
| Change the router password (for example after the router was reset) | `uv run ansible-vault edit host_vars/routeros-lab.yml` |
| See the stored values | `uv run ansible-vault view host_vars/routeros-lab.yml` |
| Change the vault password | `uv run ansible-vault rekey host_vars/routeros-lab.yml` (asks for the current one, then the new one twice) |
| Encrypt a plain file you wrote yourself | `uv run ansible-vault encrypt host_vars/routeros-lab.yml` |
| Turn it back into plain text | `uv run ansible-vault decrypt host_vars/routeros-lab.yml` (don't commit it like that) |

The encrypted `host_vars/routeros-lab.yml` is safe to commit. Commit it only if `head -1`
shows `$ANSIBLE_VAULT`. If you lose the vault password, the file cannot be recovered: delete it
and create a new one (4.1).

**Optional: skip the prompt.** Ansible can read the vault password from a file instead. Keep
that file **outside this repository** and readable only by you, then point Ansible at it for
the current shell:

```bash
mkdir -p ~/.ansible
read -rs VAULT_PW && printf '%s\n' "$VAULT_PW" > ~/.ansible/routeros-lab.vault-pass && unset VAULT_PW
chmod 600 ~/.ansible/routeros-lab.vault-pass
export ANSIBLE_VAULT_PASSWORD_FILE=~/.ansible/routeros-lab.vault-pass
uv run ansible-playbook configure-router.yml      # no -J needed while the variable is set
```

**Without a vault:** if `host_vars/routeros-lab.yml` does not exist, the inventory falls
back to environment variables:

```bash
export MIKROTIK_PASSWORD='your-routeros-password'
export MIKROTIK_USERNAME=admin   # optional, default admin
```

The vault only covers Ansible. The MikroTik MCP server for Claude Code still reads its
credentials from its own environment (`.claude/settings.local.json`).

## Run it

Run everything from this `ansible/` folder, so `ansible.cfg` is picked up. `ap` is the
alias from step 3.

```bash
# 1. Read-only: check the connection, the RouterOS version and the interface
ap configure-router.yml --tags preflight

# 2. Apply the configuration and show the BGP state
ap configure-router.yml

# 3. Run again: everything should be "ok", changed=0
ap configure-router.yml
```

Other useful runs:

```bash
# Only show BGP sessions and routes (read-only)
ap configure-router.yml --tags verify

# Override a setting for one run
ap configure-router.yml -e bgp_multipath=3
```

The preflight checks run on every invocation. They stop the run if:

- RouterOS has no `/routing/bgp/instance` menu (too old a RouterOS 7)
- the LAN interface (`ether2`) does not exist
- the interface's MAC differs from `../routeros_data/nic_mac`, which means it is not the
  NIC bridged to the kind network

## After MetalLB is installed

The router only listens; MetalLB's speakers open the sessions. Once MetalLB is configured
with a `BGPPeer` pointing at `172.19.0.250` (`myASN: 64513`, `peerASN: 64512`) and a
LoadBalancer Service has an IP:

```bash
ap configure-router.yml --tags verify
```

You should see one established session per worker node, and a `/32` route for each
LoadBalancer IP with a worker node as gateway.

## Options

### iBGP instead of eBGP

Set `metallb_as` to the same value as `bgp_local_as` (for example both `64512`). The
playbook then uses `local.role=ibgp`. Use the same AS in MetalLB's `BGPPeer`.

### BFD

Set `bfd_enabled: true`. The playbook adds a `/routing/bfd/configuration` entry for the
LAN interface and sets `use-bfd=yes` on the BGP connection. Setting it back to `false`
removes both again.

MetalLB must run in FRR or FRR-K8s mode, with a `BFDProfile` referenced from the
`BGPPeer`; native mode has no BFD. The default timers (1s × 3) are deliberately relaxed
because the router is emulated; shorter timers can make sessions flap.

## Troubleshooting

| Symptom | Likely cause |
| --- | --- |
| `Authentication failed` | Wrong password in the vault (`ansible-vault edit`), or no vault and `MIKROTIK_PASSWORD` not exported, or the router was recreated and still has an empty password |
| `Attempting to decrypt but no vault secrets found` | `host_vars/routeros-lab.yml` exists, but the command was run without `--ask-vault-pass` / `-J` (step 4.3) |
| `Decryption failed (no vault secrets were found that could decrypt)` | Wrong vault password typed, or `ANSIBLE_VAULT_PASSWORD_FILE` points at a file with a different password |
| `host key mismatch for 127.0.0.1` | The router got a new SSH host key (the container was recreated). Remove the old one with `ssh-keygen -R '[127.0.0.1]:2222'` and run again. `host_key_checking = False` does not prevent this with `ansible.netcommon` 8.x |
| `paramiko is not installed` | Ran plain `ansible-playbook` instead of the `uvx ... --with paramiko` command from step 3 |
| `couldn't resolve module/action 'community.routeros.command'` | Collections missing: run `ag collection install -r requirements.yml` in this folder |
| `Connection timed out` / `command timeout` | Router still booting (2-4 minutes under emulation). Wait, or raise the timeouts in `ansible.cfg` |
| `ether2 does not exist` | Router is in single-network mode; see step 1 |
| A config task fails with `'RESULT=' not in ...` | RouterOS rejected the command. Run with `-v` to see its message, for example `bad parameter` or `invalid value` |
| Session not established | Check `kubectl -n metallb-system logs -l component=speaker`, the AS numbers, and that the router IP matches `peerAddress` |

## Remove the configuration

Everything the playbook created carries the `managed by ansible` comment. In the RouterOS
terminal:

```routeros
/routing/bgp/connection/remove [find comment~"managed by ansible"]
/routing/bgp/instance/remove [find comment~"managed by ansible"]
/routing/filter/rule/remove [find comment~"managed by ansible"]
/routing/bfd/configuration/remove [find comment~"managed by ansible"]
/ip/address/remove [find comment~"managed by ansible"]
```
