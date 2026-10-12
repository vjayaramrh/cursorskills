---
name: rke2-management-teardown
description: >-
  Tears down a Sylva CAPO RKE2 management cluster (OpenStack VM, not
  RKE2 on the metal). Prompts for hypervisor IP, SSH user, SSH
  private-key path (never paste the key). Lists management cluster
  names and asks which to delete. Refuses if workload clusters still
  exist unless the user explicitly accepts orphaning. Use when the
  user asks to delete, destroy, or tear down the RKE2 management
  cluster. Do not use for OKD workload teardown, stack.sh, or
  DevStack unstack.
---

# RKE2 management teardown

Inverse of **rke2-management-deploy**. Delete one Sylva CAPO RKE2
*management* cluster (OpenStack VM, not RKE2 on the metal). This is
**not** OKD workload teardown and **not** DevStack unstack. Never
`bootstrap.sh` / `stack.sh` / `unstack.sh`.

## Hard rules

- Prompt hypervisor IP, SSH user, key **path** (skip path if ssh-agent/config works). Never paste private keys or passwords. SSH as documented (`Host lab53` / `Host lab53-stack` / then `sudo -u sylva`). `KUBECONFIG` = `management-cluster-kubeconfig` while it still works (typically `/home/sylva/sylva-core-cabpoa/management-cluster-kubeconfig`). Never `stack.sh` / `unstack.sh`.
- List clusters: `kubectl get cluster -A`. Identify management (typically `sylva-system` / `lab53-rke2`, 1 CP, md0=0) vs workload (`okd-capo` / `okd-openstack` etc.). Also `openstack server list` for matching VMs.
- Show a table: namespace, name, role (management vs workload), CP/worker, phase, Nova servers.
- If any workload clusters exist: do **not** delete management. Say they must run **okd-workload-teardown** first. Only continue if they explicitly say to orphan/abandon those workloads.
- Ask `Which RKE2 management cluster should be deleted?` Wait. Confirm the name.
- After pivot, CAPI lives on the management VM — do **not** rely on `kubectl delete cluster` against itself as the only path.
- Do **not** unstack DevStack. Do **not** delete `/opt/stack/devstack/local.conf`. Keep `environment-values` unless they asked.
- Default: do **not** delete ports with `DO-NOT-DELETE` in the name.

### Can the RKE2 management cluster be deleted without deleting workload clusters?

**No, not safely.** After Sylva pivot, CAPI/CAPO/Flux run ON the RKE2 management VM. Workload clusters (OKD) are Cluster CRs in namespaces on that API; CAPO there owns the OpenStack servers. If you destroy management first:
- Workload VMs keep running in Nova with no controller (orphans): no scale, no delete via CAPI, no kubeconfig refresh.
- Heat stacks, FIPs, VIP ports remain.
- You cannot use **okd-workload-teardown** afterward because `management-cluster-kubeconfig` is dead.

Required order: all workload clusters gone first (skill **okd-workload-teardown**), then management.

The only exception is if the user explicitly wants to abandon workloads and delete Nova servers/Heat by hand; that is not "keep the workload clusters". If `kubectl get cluster -A` still shows a non-management Cluster (e.g. `okd-openstack` in `okd-capo`), STOP and tell them to tear those down first unless they explicitly accept orphaning.

## Start here (ask before SSH)

1. `What IP is the DevStack hypervisor?`
2. `What SSH user?` and `What path is the SSH private key?` (skip path if agent/`~/.ssh/config` works). Never paste the key.

```bash
ssh -o BatchMode=yes -o IdentitiesOnly=yes -i <key_path> <ssh_user>@<HOST_IP> '...'
```

If they skipped the path, omit `-i` and `-o IdentitiesOnly=yes`. If SSH fails, ask again — do not try passwords in chat.

## List clusters and wait

As `sylva` while the management API still works:

```bash
sudo -u sylva -H bash -lc '
export KUBECONFIG=/home/sylva/sylva-core-cabpoa/management-cluster-kubeconfig
kubectl get cluster -A
kubectl get machinedeployment -A
'
```

If that kubeconfig is missing, find `management-cluster-kubeconfig` under `/home/sylva`. As `stack` (openrc), match Nova servers:

```bash
openstack server list
openstack stack list
```

Show a table: namespace, name, role (management vs workload), CP/worker, phase, Nova servers.

If any non-management `Cluster` exists: **STOP**. They must run **okd-workload-teardown** first. Continue only if they explicitly accept orphaning/abandoning those workloads.

Ask: `Which RKE2 management cluster should be deleted?` Wait. If several, they must pick. If none, stop. Confirm the chosen name. If they name a workload cluster, refuse and point to **okd-workload-teardown**.

## Teardown order (chosen management cluster only)

After pivot, CAPI/CAPO/Flux run on the management VM. You cannot rely on `kubectl delete cluster` against itself as the only path.

1. Confirm workloads gone (or explicit orphan).
2. Note Nova server id, Heat stack name, VIP ports named `DO-NOT-DELETE`.
3. Delete the OpenStack server for the RKE2 CP (as `stack`, openrc).
4. Delete leftover Heat stack for that management cluster if still present (`openstack stack delete`, or leftover HeatStack CR if the API is still up — usually the API dies with the VM).
5. Grep Nova/Heat/ports for the cluster name. **Never** delete ports with `DO-NOT-DELETE` in the name unless Heat already removed them and they are leftover **and** the user asked; default: do not delete `DO-NOT-DELETE` ports.
6. Kind `sylva` on the metal if still present (usually gone after pivot): `kind delete cluster` only if it is leftover bootstrap, not if they still need it.

```bash
# as stack
cd /opt/stack/devstack && source openrc admin admin
openstack server list | grep <name>
openstack stack list | grep <name>
openstack port list | grep <name>
openstack server delete <rke2-cp-server-id>
# if leftover Heat for this management cluster:
# openstack stack delete <stack>
```

```bash
# leftover Kind bootstrap only — not if they still need it
kind get clusters
# kind delete cluster --name sylva
```

Do **not** unstack DevStack. Do **not** delete `/opt/stack/devstack/local.conf`. Keep `environment-values` unless they asked.

If they explicitly orphaned workloads, they must delete those Nova servers/Heat by hand; this skill does not keep those clusters.

## Gate (torn down)

- No RKE2 CP server.
- `kubectl` to the old `management-cluster-kubeconfig` fails.
- DevStack hypervisor still up.
