---
name: okd-workload-teardown
description: >-
  Tears down a Sylva CAPO OKD workload cluster on an existing RKE2
  management cluster. Lists OKD/workload cluster names, then prompts
  which one to delete. Prompts for hypervisor IP, SSH user, and SSH
  private-key path (never paste the key). Use when the user asks to
  delete, destroy, wipe, or tear down an OKD workload cluster.
  Do not use for stack.sh, RKE2 bootstrap, or DevStack unstack.
---

# OKD workload teardown

Inverse of **okd-workload-deploy**. Delete one Sylva CAPO OKD *workload*
cluster from the RKE2 management cluster. This is **not** DevStack unstack
and **not** RKE2 management teardown. Never `bootstrap.sh` / `stack.sh`.
Run kubectl as **`sylva`**.

## Hard rules

- Prompt hypervisor IP, SSH user, key **path** (skip path if ssh-agent/config works). Never paste private keys or passwords.
- `KUBECONFIG` = `management-cluster-kubeconfig` (RKE2), typically `/home/sylva/sylva-core-cabpoa/management-cluster-kubeconfig`. Run as user **sylva**. Never `bootstrap.sh` / `stack.sh`.
- Discover names. Do **not** assume `okd-openstack` or ns `okd-capo`.
- Do **not** `kubectl delete ns` first. Do **not** delete ports named `DO-NOT-DELETE`.
- Do **not** delete the RKE2 management cluster (`lab53-rke2` / `sylva-system`) or unstack DevStack.
- Keep `values.yaml` unless they asked to delete the env overlay.

## Start here (ask before SSH)

1. `What IP is the DevStack hypervisor?`
2. `What SSH user?` and `What path is the SSH private key?` (skip path if agent/`~/.ssh/config` works). Never paste the key.

```bash
ssh -o BatchMode=yes -o IdentitiesOnly=yes -i <key_path> <ssh_user>@<HOST_IP> '...'
```

If they skipped the path, omit `-i` and `-o IdentitiesOnly=yes`. If SSH fails, ask again — do not try passwords in chat.

## List OKD workload clusters and wait

As `sylva`:

```bash
sudo -u sylva -H bash -lc '
export KUBECONFIG=/home/sylva/sylva-core-cabpoa/management-cluster-kubeconfig
kubectl get cluster -A
kubectl get ns
kubectl get sylvaunitsrelease -A
kubectl get machinedeployment -A
'
```

If that kubeconfig is missing, find `management-cluster-kubeconfig` under `/home/sylva`. Also inspect namespaces matching `okd` / `workload-clusters` / `SylvaUnitsRelease`. If easy, match Nova servers (`openstack server list`) to each cluster name.

Exclude `lab53-rke2` / ns `sylva-system` from the pick list (leave them running).

Show a table: namespace, cluster name, CP/worker desired, phase, OpenStack servers matching the name if easy.

Ask: `Which OKD workload cluster should be deleted?` Wait. If several, they must pick. If none, stop. Confirm the chosen name before any delete. If they name the RKE2 management cluster, refuse.

## Teardown order (chosen cluster only)

Do **not** `kubectl delete ns` first. Do **not** delete ports named `DO-NOT-DELETE`.

1. Suspend Flux in that namespace. `flux` is often not on `sylva`'s `PATH`, and `kubectl patch --all` is invalid. Patch each object:

```bash
sudo -u sylva -H bash -lc '
export KUBECONFIG=<management-cluster-kubeconfig>
for n in $(kubectl get ks -n <ns> -o name); do
  kubectl patch "$n" -n <ns> --type merge -p "{\"spec\":{\"suspend\":true}}"
done
for n in $(kubectl get hr -n <ns> -o name); do
  kubectl patch "$n" -n <ns> --type merge -p "{\"spec\":{\"suspend\":true}}"
done
'
```

2. `kubectl delete cluster <name> -n <ns> --wait=false`

3. Wait until OpenStackMachines/Machines in that ns are gone **and** Nova servers for that cluster name are gone. Leave the RKE2 management VM.

4. If leftover: `kubectl delete heatstack heatstack-capo-cluster-resources -n <ns>`. Wait until `openstack stack list` no longer shows that cluster's stack. Deleting the stack removes its VIP/ingress ports, including names that contain `DO-NOT-DELETE`. Do **not** `openstack port delete` those ports yourself, and do **not** touch the management cluster's stack or its `DO-NOT-DELETE` ports.

5. Then delete the workload namespace: `kubectl delete ns <ns>`. It may stay `Terminating` until the HeatStack is gone.

6. Grep Nova/Heat/ports for leftovers matching the cluster name. Report leftovers; delete only objects clearly named for this cluster. A Rancher `clusters.management.cattle.io` object for this workload (not `local`) should disappear with the Cluster. Do not delete `local`.

```bash
openstack server list | grep <name>
openstack stack list | grep <name>
openstack port list | grep <name>
```

Keep `values.yaml` (the env overlay) unless they asked to delete it.

## Gate (torn down)

- Chosen `Cluster` gone; its workload ns gone.
- No Nova servers / Heat stacks for that name. RKE2 management still up. DevStack still stacked.
