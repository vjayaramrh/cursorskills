---
name: devstack-deploy
description: >-
  Step 1 of DevStack then Sylva RKE2 management then OKD workload.
  Deploys a fresh OpenStack DevStack cloud with stack.sh as user stack.
  Prompts for hypervisor IP, SSH user, and SSH private-key path (never
  paste the key). Default follow-on is RKE2 + OKD: run the RAM/vCPU/disk
  check (user may skip if they only want DevStack). SR-IOV on the OKD
  worker is optional intent only. Use when the user asks to install,
  stack, bootstrap, or deploy DevStack on a hypervisor (including
  192.168.39.53 / sylva-53 / lab53). Do not use for teardown (see
  devstack-teardown) or for Sylva bootstrap (see rke2-management-deploy).
  Does not configure host SR-IOV, q-sriov-agt, or PHYSICAL_DEVICE_MAPPINGS
  unless the user explicitly asks. On stable/2026.1, local.conf must pin
  the OVS mechanism driver and LIBVIRT_CPU_MODE=host-model before the
  first stack.sh. Do not rerun stack.sh while devstack@ units are active.
---

# DevStack deploy

Intended path: **this skill** → `rke2-management-deploy` → `okd-workload-deploy`. Stop after Gate A. Do not run `bootstrap.sh` here.

## Hard rules

- Run as user **`stack`**, `DEST=/opt/stack`. Never `bootstrap.sh` / Sylva as `stack`.
- **One `stack.sh`.** `pgrep -af '[b]ash ./stack.sh'` before start. Shells whose parent is that PID are helpers. Do not start a second copy.
- **Do not run `stack.sh` over live `devstack@` units.** `stack.sh` runs `recreate_database neutron` and will not restart an already-active `devstack@neutron-api`. The API then serves an empty VXLAN pool and network create fails with `No project network is available for allocation`. If any `devstack@` unit is active, run `./unstack.sh` as `stack`, confirm none are active, then one `stack.sh`. Confirm with the user before unstacking a cloud that already passes Gate A. A failed `stack.sh` from this same deploy is not a live cloud: unstack and stack once. Do not `clean.sh` unless they asked for a full wipe (`clean.sh` deletes `/opt/stack/data`, including the venv, so the next `stack.sh` is slower).
- **Write a correct `local.conf` before the first `stack.sh`.** `stable/2026.1` defaults `Q_ML2_PLUGIN_MECHANISM_DRIVERS` to `ovn` and `LIBVIRT_CPU_MODE` to `custom` / `Nehalem`. Post-config `cpu_mode=host-model` alone leaves `cpu_model` set, and `n-cpu` exits with `The cpu_models option is not required when cpu_mode!=custom`. If an existing `local.conf` contains `q-sriov-agt`, `sriovnicswitch`, `PHYSICAL_DEVICE_MAPPINGS`, or `device_spec` and the user did not ask for host SR-IOV, back it up and replace it with the minimal template. Keep the existing `ADMIN_PASSWORD` in the file. Never print it, including when grepping the file (`ADMIN_PASSWORD=…` is one field, so printing `$1` leaks it). `chmod 600`.
- Nova compute reads **`/etc/nova/nova-cpu.conf`**, not only `nova.conf`. The template sets `LIBVIRT_CPU_MODE=host-model` and `cpu_mode=host-model` in **both** post-config stanzas. If `/etc/nova/nova.conf` or `nova-cpu.conf` already exist, delete any `cpu_model` line before `stack.sh`. `LIBVIRT_CPU_MODE=host-model` does not remove a leftover `cpu_model`.
- Leave **`PUBLIC_INTERFACE` unset**. Do not steal the management NIC.

## Start here (ask before any SSH or `stack.sh`)

Do **not** assume `192.168.39.53`. Ask, wait, then use the answers as `HOST_IP` and whether to run the heavy check.

Ask only what this skill can act on. Do **not** walk CP-then-worker or igb Glance steps here — those belong when RKE2 / OKD are actually applied.

| Prompt | Ask at DevStack deploy | Ask at RKE2 / OKD deploy |
|---|---|---|
| Hypervisor IP | **Yes** (`HOST_IP` in `local.conf`) | **Yes** again (new chat; may be the same IP) |
| SSH user + key **path** to that host | **Yes** (to reach the box). Never paste the private key | **Yes** again if a new chat |
| RKE2 + OKD later? + RAM/vCPU/disk check | **Yes** (abort before hours of `stack.sh`) | Repeat a short capacity glance if the cloud already exists |
| SR-IOV/VT10 on the OKD worker? | Optional **intent only** (`yes` / `no` / skip) | **Yes — this is the real decision** |
| CPs first (`md0=0`), then worker (`md0=1`)? | No | **Yes**, when applying the OKD workload |

1. **Where?** Ask all three; wait. Do not guess `stack` as the SSH login (`stack` is who runs `stack.sh` on the box). This is the key to the **hypervisor**, not an OpenStack VM keypair.
   - `What IP should DevStack be deployed on?`
   - `What SSH user should be used to reach that host?`
   - `What path is the SSH private key on this machine?` (e.g. `~/.ssh/id_ed25519_lab53`). If they already have `ssh-agent` or a `~/.ssh/config` `Host` entry, they may skip the path.
2. **What next?** Default for this skill’s owner is **yes**. Ask: `Will this cloud later run an RKE2 management cluster and an OKD workload cluster?` (`yes` / `no`). If they only want DevStack, they say **no** and skip the capacity check.
3. **Only if 2 is yes — optional SR-IOV intent:** `In some cases OKD needs SR-IOV (VT10 / simulated PF on the worker). Do you want that later?` (`yes` / `no` / skip). Record it. The real CP-then-worker confirmation is **okd-workload-deploy**, not here. Do **not** change `local.conf` for this.

Never print, commit, or echo the private key. Use `-i` only:

```bash
ssh -o BatchMode=yes -o IdentitiesOnly=yes -i <key_path> <ssh_user>@<HOST_IP> '...'
```

If they skipped the path, omit `-i` and `-o IdentitiesOnly=yes` so agent/config can work. If SSH fails, ask again for user/path — do not try passwords in chat.

If **2 is no**: skip the RKE2+OKD pre-req check and skip question 3. Still require Ubuntu 24.04, `/dev/kvm`, and user `stack` with passwordless sudo on that IP. Proceed to deploy.

If **2 is yes**: SSH to that IP and **run** the check below. Show measured vs required. A miss is a warning, not a hard fail — ask **continue** or **stop**. Only stop if they say stop.

This skill **only** runs `stack.sh`. It does **not** bootstrap Sylva, apply OKD, or scale MachineDeployments.

If question 3 is **yes**: still do **not** add `q-sriov-agt`, `PHYSICAL_DEVICE_MAPPINGS`, or `[pci] device_spec`. VT10 here is emulated igb on the OKD **worker**, configured when that cluster is applied.

## Preconditions (every deploy)

- Target is the IP the user just gave. Clone and `stack.sh` run **there**, not on the agent’s laptop unless that IP is local.
- Ubuntu 24.04 hypervisor, `/dev/kvm`, user `stack` with passwordless sudo.
- Clone: `https://opendev.org/openstack/devstack.git` → `/opt/stack/devstack`, branch **`stable/2026.1`** unless the user names another.
- `local.conf` path: `/opt/stack/devstack/local.conf`. Set `HOST_IP` to the IP they gave.

## RKE2 + OKD pre-req check (only if they said yes)

Remote, as the SSH user on `HOST_IP`:

```bash
ssh -o BatchMode=yes -o IdentitiesOnly=yes -i <key_path> <ssh_user>@<HOST_IP> 'bash -s' <<'EOF'
set -e
echo "OS $(. /etc/os-release; echo $PRETTY_NAME)"
test -e /dev/kvm && echo kvm_ok || echo kvm_MISSING
nproc
awk '/MemTotal/ {printf "RAM_GiB %.0f\n", $2/1024/1024}' /proc/meminfo
df -BG / | awk 'NR==2 {print "disk_free_GiB", $4}'
id stack >/dev/null 2>&1 && echo stack_user_ok || echo stack_user_MISSING
command -v qemu-system-x86_64 >/dev/null && qemu-system-x86_64 --version | head -1 || echo qemu_not_installed_yet
libvirtd --version 2>/dev/null || echo libvirt_not_installed_yet
EOF
```

Required if they plan RKE2 (1 CP) + OKD (3 CP + 1 worker), all 16 vCPU / 32 GiB / 160 GB:

| Check | Pass |
|---|---|
| Guest RAM | host **≥ 192 GiB** (5 × 32 GiB guests + host/QEMU) |
| vCPU | 80 vCPUs scheduled; overcommit OK if host has ~20–40 cores |
| Disk | **≥ ~1 TB** free (`/` or Nova’s volume path) |
| KVM | `/dev/kvm` exists |
| OS | Ubuntu 24.04 |
| Cannot | two 32 GiB workers at once |

QEMU ≥ 8.0 / libvirt ≥ 9.3 are **informational** until they need emulated igb; do not fail the check on missing QEMU before `stack.sh` (DevStack will install it). If question 3 is yes, also show `grep -E 'intel_iommu|amd_iommu|iommu=pt' /proc/cmdline` (needed later for vIOMMU; do not fail stacking if unset).

Report:

```text
Target: <HOST_IP>
Plan: RKE2 management + OKD workload
Host: <OS>, <nproc> CPUs, <RAM_GiB> GiB RAM, <disk> free, kvm=<ok|MISSING>
RKE2+OKD capacity: PASS | SHORT (<which>)
```

If SHORT: say NoValidHost / DiskPressure / OOM is likely later. Ask **continue** or **stop**.

## Minimal local.conf (plain DevStack)

```ini
[[local|localrc]]
ADMIN_PASSWORD=<set-me>
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

DEST=/opt/stack
HOST_IP=<HOST_IP>
SERVICE_HOST=$HOST_IP
IP_VERSION=4
LOGFILE=$DEST/logs/stack.sh.log
LOGDAYS=7
VERBOSE=True

disable_service ovn-controller ovn-northd q-ovn-agent q-ovn-metadata-agent tempest
enable_service q-agt q-dhcp q-l3 q-meta
enable_plugin neutron https://opendev.org/openstack/neutron.git stable/2026.1
enable_plugin heat https://opendev.org/openstack/heat.git stable/2026.1
enable_service h-eng h-api h-api-cfn

Q_AGENT=openvswitch
Q_PLUGIN=ml2
# 2026.1 defaults this to ovn. Leave it unset and Neutron loads OVN while q-agt is OVS.
Q_ML2_PLUGIN_MECHANISM_DRIVERS=openvswitch
Q_ML2_PLUGIN_TYPE_DRIVERS=flat,vlan,vxlan
Q_ML2_TENANT_NETWORK_TYPE=vxlan
# stackrc defaults to custom/Nehalem. Post-config cpu_mode=host-model alone
# leaves cpu_model set, and Nova 2026.1 then refuses to start n-cpu.
LIBVIRT_CPU_MODE=host-model
VOLUME_BACKING_FILE_SIZE=100G

[[post-config|$NOVA_CONF]]
[DEFAULT]
rpc_response_timeout = 300

[[post-config|$NOVA_CPU_CONF]]
[libvirt]
cpu_mode=host-model
[DEFAULT]
rpc_response_timeout = 300
```

Do **not** add `q-sriov-agt`, `sriovnicswitch`, `[pci] device_spec`, or `PHYSICAL_DEVICE_MAPPINGS` unless the user asked for host SR-IOV.

## Before `stack.sh`

Do this once, then launch. Do not discover these by failing `stack.sh` and retrying.

1. `pgrep -af '[b]ash ./stack.sh'`. If one is running, monitor it.
2. `systemctl list-units 'devstack@*' --state=active`. If any are active, unstack (hard rules), then continue.
3. Back up `local.conf` to `local.conf.bak-<UTC stamp>` and write the template. Required before the first start: `Q_ML2_PLUGIN_MECHANISM_DRIVERS=openvswitch`, `LIBVIRT_CPU_MODE=host-model`, both `cpu_mode=host-model` stanzas, no host SR-IOV lines unless they asked. `chmod 600`.
4. If `/etc/nova/nova.conf` or `nova-cpu.conf` exist, delete `cpu_model` lines.

Tell them the expected wait, then start **one** `stack.sh` as `stack`:

| Host state | Expect |
|---|---|
| Git tree, apt packages, and `/opt/stack/data` venv already present (restack after `unstack.sh`) | about **7–10 minutes** (lab53: 432 seconds) |
| After `clean.sh` on a host that already has apt packages and the git tree (venv deleted; pip cache may remain) | still about **7–10 minutes** on lab53 (497 seconds). A host that still has to install apt packages or clone devstack takes longer |

```bash
sudo -u stack -H bash -lc 'cd /opt/stack/devstack && ./stack.sh'
```

Log: `/opt/stack/logs/stack.sh.log`. Poll `stack.sh.log.*.summary.*`. Success is `stack.sh completed in N seconds`. That number is only that process, not time spent on earlier failed attempts.

## If `stack.sh` exits before that line

Do not start another `stack.sh` while any `devstack@` unit is active. Fix, unstack, then one new run.

| Log | Fix before the next run |
|---|---|
| Neutron did not start, mechanism driver `ovn` | `Q_ML2_PLUGIN_MECHANISM_DRIVERS=openvswitch` was missing |
| `cpu_models option is not required when cpu_mode!=custom` | `LIBVIRT_CPU_MODE=host-model` and delete leftover `cpu_model` |
| `No project network is available for allocation` | Neutron API outlived `recreate_database neutron`. Unstack until no `devstack@` unit is active |

If Nova starts but cells/Horizon fail:

```bash
sudo nova-manage cell_v2 discover_hosts --verbose
```

## Gate

Do not print the token.

```bash
sudo -u stack -H bash -lc 'cd /opt/stack/devstack && source openrc admin admin && openstack token issue >/dev/null && echo token_ok && openstack hypervisor list'
grep -E "cpu_mode|cpu_model" /etc/nova/nova-cpu.conf
grep "^mechanism_drivers" /etc/neutron/plugins/ml2/ml2_conf.ini
systemctl is-active devstack@n-cpu.service
```

Expect `token_ok`, hypervisor `up`, `cpu_mode = host-model` and **no** `cpu_model` line, `mechanism_drivers = openvswitch`, and `n-cpu` `active`. Horizon: `http://<HOST_IP>/dashboard`.

If question 2 was **yes**, as `stack` after openrc. Skip create when the flavor already exists:

```bash
openstack flavor show m1.xxlarge-160 || openstack flavor create --ram 32768 --disk 160 --vcpus 16 --public m1.xxlarge-160
openstack quota set --cores 120 --ram 180000 --instances 20 admin
```

If question 2 was **yes**, also give the `private` subnet public DNS. DevStack’s dnsmasq does not forward, and an empty `dns_nameservers` makes the RKE2 installer’s `curl -sfL https://get.rke2.io` fail with no output. The node then never gets `rke2-server.service`.

```bash
openstack subnet set private-subnet --dns-nameserver 8.8.8.8 --dns-nameserver 1.1.1.1
```

Raise Glance unified limits before Sylva image sync. The default `image_size_total` / `image_stage_total` is **2000 MiB**. The RKE2 image is about 15 GiB, so usage is already over that limit. The later SCOS upload (~16 GiB raw) is rejected with HTTP 413, which Apache shows to the sync pod as **502 Bad Gateway**.

```bash
openstack registered limit set --service glance --default-limit 100000 --region RegionOne image_size_total
openstack registered limit set --service glance --default-limit 100000 --region RegionOne image_stage_total
```

Raise the Cinder volume count. The default is **10 volumes**. The management cluster’s PVCs plus the image-sync scratch volume (about 18 GiB, one per attempt) hit that cap, and the next sync pod stays Pending with `VolumeLimitExceeded`. Gigabytes can stay at 1000.

```bash
openstack quota set --volumes 40 --snapshots 40 admin
openstack quota set --secgroups 100 --secgroup-rules 1000 admin
```

The default security-group quota is **10**. The RKE2 cluster plus the OKD cluster’s managed groups exceed that, and `OpenStackCluster` stays `SecurityGroupsReady=False` with Neutron `OverQuota` on `security_group`.

If question 3 was **yes**, also:

```bash
openstack flavor create --ram 32768 --disk 160 --vcpus 16 --public m1.xxlarge-160-igb
openstack flavor set m1.xxlarge-160-igb --property hw:viommu_model=intel
```

Do **not** extra-spec `m1.xxlarge-160`. Next skill: **rke2-management-deploy**.
