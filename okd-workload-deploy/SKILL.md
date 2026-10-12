---
name: okd-workload-deploy
description: >-
  Step 3 of DevStack then Sylva RKE2 management then OKD workload.
  Applies a Sylva CAPO OKD cluster (3 CP + 1 worker) onto an existing
  RKE2 management cluster.   Prompts for hypervisor IP, SSH user, key
  path, whether this OKD worker needs SR-IOV (VT10 / simulated
  igb PF), and   whether to enable flavour/VT08-VT10 test extras
  (sriov-network-operator). Calico off and OpenShift pod/service
  CIDRs are always required (not part of that question). Walks environment-values section by section (cluster,
  capo, md0, openstack, units, source_templates if present,
  pull-secret present/missing, test extras if requested) and
  waits for edit or next before apply. If SR-IOV, confirm CPs first
  (md0=0) then worker (md0=1).
  Use when the user asks to deploy or apply an OKD workload cluster
  on DevStack/Sylva. Do not use for stack.sh, RKE2 bootstrap, or
  unstack. Never bootstrap.sh or kubectl scale for this cluster.
---

# OKD workload deploy

Intended path: `devstack-deploy` → `rke2-management-deploy` → **this skill**. Apply as **`sylva`**. Do **not** use `apply.sh` or `bootstrap.sh`. Do **not** `kubectl scale` Machines.

## Hard rules

- Require Gate B (RKE2 kubeconfig, API on FIP). If missing, stop and use **rke2-management-deploy**.
- `control_plane_replicas: 3` from t=0. Assisted will not let you add CPs later.
- Drive `md0` via `values.yaml` + `./apply-workload-cluster.sh`, not `kubectl scale`.
- Never paste pull secrets or OpenStack passwords. `values.yaml` / secrets mode 600.
- Remap **this** cloud’s flavor, keypair, `private` / `public` UUIDs. Turn off Ceph/NFS aimed at other labs. Cinder `__DEFAULT__`.
- CPs stay **pc + virtio** on `m1.xxlarge-160`. Never put CPs on the igb flavor or on `sriov-net`.
- **Do not invent overlay units.** Toggle only units that exist on this branch. Never copy empty `unit_templates: []` stubs. If `enabled: false` alone fails Helm schema (`missing info|unit_templates|helmrelease_spec`), **omit that key** rather than stubbing it (`multus-ready` on cabpoa). Calico can be disabled with `enabled: false` only.
- Worker igb SR-IOV and **flavour test extras** are separate questions. Test extras do not copy another lab’s Ceph, NFS, flavors, keypairs, or network UUIDs.
- **cabpoa/OKD always** (even when flavour test extras are **no**):
  - **Disable Calico** and anything that still depends on it. Chart default `cluster.cni.default_provider` is calico; with cabpoa that yields Helm `circular dependency: calico -> calico-apiserver-rbac -> openshift-security-context-constraints -> calico-ready -> calico`. After Calico is off, `traefik` (and `ingress-nginx`) still depend on that CNI unit and fail with `dependency on disabled unit 'calico'`. Set `traefik.enabled: false` and `ingress-nginx.enabled: false`. Omit `multus-ready` (do not stub it).
  - **Do not use the chart CIDR defaults** (`cluster_pods_cidrs: 100.72.0.0/16`, `cluster_services_cidrs: 100.73.0.0/16`). Those match the RKE2 management cluster, so both `kubernetes` Services are `100.73.0.1`. Rancher then stores OKD’s API as `https://100.73.0.1:443`, which from the management cluster is its own API. `cluster-import-check` waits 60s for `AgentDeployed=true` on `clusters.management.cattle.io`, the cattle agent is never installed, and Flux reports a 30s `HealthCheckFailed` on that Job. Always set OpenShift ranges: pods `10.128.0.0/14`, services `172.30.0.0/16`. These are immutable after create; a collided cluster must be recreated.
  - Set `security.restrict_pod_scheduling: false`. The chart default is `true`, which installs `restrict-node-selection`. That policy denies any Pod create that sets `nodeName`, and its binding only skips `kube-system`. OpenShift static-pod namespaces are not skipped, so kubelet cannot register mirror Pods for `etcd`, `kube-apiserver`, and `kube-controller-manager`. The containers run, but the operators stay degraded with `Missing operand`. Leave the flag false on every later apply. Turning it back on without also exempting `openshift-etcd`, `openshift-kube-apiserver`, `openshift-kube-controller-manager`, and `openshift-kube-scheduler` in the policy binding brings the missing operands back. A local edit of that binding file is not used until it is on the branch Flux watches.
  - Set `sylvactl_timing_hints.node_create_time_per_infra.capo: 30`.
  - Do not copy `ceph-csi-cephfs`, `nfs-ganesha`, or `sriov-network-operator` from a reference overlay unless they asked.
- **pullSecret is required.** Do not omit it and do not use `{"auths":{}}` or `""`. Assisted rejects those (`invalid pull secret`, `SpecSynced=BackendError`, machines stay Pending). If they have no Red Hat pull secret, use the documented OKD dummy `{"auths":{"fake":{"auth":"aWQ6cA=="}}}` in `values.yaml` (mode 600). Never print it. A real secret replaces the dummy; it does not remove the field.

## Start here (ask before apply)

1. `What IP is the DevStack hypervisor?` SSH user + private-key **path** (never paste the key). Skip path if agent/config works.
2. **Worker SR-IOV (hardware path):** `Does this OKD workload cluster need SR-IOV support (VT10 / simulated PF on the worker)?` (`yes` / `no`). Some runs need it; many do not. This is igb+q35+vIOMMU on md0, not host `q-sriov-agt`.
3. If worker SR-IOV **yes**, wait for confirm: CPs first (`md0=0`), then worker (`md0=1`) with igb+q35+vIOMMU.

If worker SR-IOV **no**: still 3 CPs from t=0 and `md0=0` until they are Installed, then `md0=1` on **`m1.xxlarge-160`** (virtio). Skip Glance igb properties.

4. **Flavour / VT08-VT10 test extras (in-cluster):** Always ask, even if step 2 was **no**. Wait (`yes` / `no`):

```text
Will this OKD cluster run flavour tests similar to VT08/VT10
(sriov-network-operator on the workload cluster)?
This is independent of worker igb SR-IOV.
Calico off and OpenShift pod/service CIDRs are always applied either way.
Reply yes or no.
```

If **yes**, add the **test-extras** keys below (from `values-reference.yaml` if present, remapped to this cloud). If **no**, omit only those extras. Always include the cabpoa block in Hard rules (CIDRs, Calico/traefik off, timing hint, pull secret).

If **yes**, proposed extras (do **not** copy another lab’s Ceph/NFS/UUIDs/flavor/keypair):

```yaml
_internal:
  static_k8s_version: true
security:
  restrict_pod_scheduling: false
cluster:
  freeze_first_node_files: true
default_storage_class: cinder-default
units:
  sriov-network-operator:
    enabled: true
  sriov-crd:
    depends_on:
      multus: false
```

Always include this cabpoa block (extras **yes** or **no**):

```yaml
security:
  restrict_pod_scheduling: false
cluster:
  cluster_pods_cidrs:
    - 10.128.0.0/14
  cluster_services_cidrs:
    - 172.30.0.0/16
sylvactl_timing_hints:
  node_create_time_per_infra:
    capo: 30
units:
  calico:
    enabled: false
  calico-crd:
    enabled: false
  calico-ready:
    enabled: false
  cluster:
    depends_on:
      calico: false
      calico-ready: false
  traefik:
    enabled: false
  ingress-nginx:
    enabled: false
  longhorn:
    enabled_conditions:
      - false
  monitoring:
    enabled: false
  monitoring-crd:
    enabled: false
  flux-webui:
    enabled: false
  harbor:
    enabled: false
  # omit multus-ready
```

Leave `ceph`, `nfs-ganesha`, and `os_images_oci_registries` **off / omitted** unless this cloud has those services and they explicitly ask. Do not copy `resources_tag`, `failure_domain`, or monitor IPs from a reference overlay.

RAM cannot hold two 32 GiB workers at once: if an old md0 VM exists, wait until it is **gone** before 1.

## Overlay review (section by section — wait before apply)

Remap **this** cloud. Start from existing `environment-values/workload-clusters/okd-capo/values.yaml` if present. Redact pull secrets — never print them. Refresh flavor/keypair/network UUIDs from this cloud.

Walk **one section at a time** (same replies as RKE2: `next` | `edit <key>=<value>` | `accept remaining` | `stop`). Show only that section’s YAML. Do **not** dump the whole file first. After the last section (or **accept remaining**), show the assembled overlay once and wait for **apply**. Do not write or run `apply-workload-cluster.sh` until they say **apply**.

Worker flavor in the `machine_deployments` section follows the worker SR-IOV answer from Start here. Test-extras keys follow the VT08/VT10 answer; if that was **no**, skip those sections (say so and **next**).

| # | Section | Proposed | Warn if they |
|---|---|---|---|
| 1 | Paths | `sylva-core` dir, `environment-values/workload-clusters/okd-capo`, `KUBECONFIG` = management kubeconfig (not Kind) | |
| 2 | `source_templates` | From existing `values.yaml` if present. If missing, say so and **next** | another lab’s chart branch |
| 3 | `cluster` | `name`, `control_plane_replicas: 3`, **`cluster_pods_cidrs: [10.128.0.0/14]`**, **`cluster_services_cidrs: [172.30.0.0/16]`**; if test extras **yes**: `freeze_first_node_files` | replicas ≠ 3; leaving chart defaults `100.72.0.0/16` / `100.73.0.0/16` (Rancher import dials the management API) |
| 4 | `cluster.capo` | `flavor_name: m1.xxlarge-160`, `ssh_key_name`, `network_id: <private-uuid>` | CP flavor igb / `sriov-net` on CPs |
| 5 | `machine_deployments` | `md0.replicas: 0`; `md0.capo.flavor_name`: `m1.xxlarge-160-igb` if worker SR-IOV else `m1.xxlarge-160` | replicas ≠ 0 on first apply; `sriov-net` / `vnicType: direct` for VT08/VT10 |
| 6 | `openstack` | `external_network_id: <public-uuid>`, Cinder `__DEFAULT__`; Ceph/NFS **off** | leftover Ceph monitors from another lab |
| 7 | `units` | Glance timeout **30m**; always Calico/traefik/ingress-nginx/longhorn/harbor/monitoring off as in the cabpoa block; omit `multus-ready`. If test extras **yes**: `sriov-network-operator` on | 5m default; invented stubs; copying Ceph/NFS units; `traefik` left on |
| 8 | Pull secret | required. Missing → OKD dummy (do not print). Never empty `{"auths":{}}` | omitting the field; printing the secret |
| 9 | Timing + test extras | Always `sylvactl_timing_hints` capo **30**. If extras **yes**: `_internal.static_k8s_version`, `security.restrict_pod_scheduling`, `default_storage_class`. If **no**, say so and **next** | copying `os_images_oci_registries` / `resources_tag` from another lab |

Do not add `sriov-net` / `vnicType: direct` for VT08/VT10. Persist Glance unit timeout **30m**.

```bash
sudo -u sylva -H bash -lc '
cd ~/sylva-core-<checkout>
export KUBECONFIG=<management-cluster-kubeconfig>
./apply-workload-cluster.sh environment-values/workload-clusters/okd-capo
'
```

Watch Assisted. If an agent hits ostree `File exists`, **teardown** that attempt (do not retry the same disks).

While the bootstrap control plane is still in **Waiting for bootkube**, a worker console retrying `ignition-fetch` against `https://<api-vip>:22623/config/worker` is expected (especially if `md0=1` from the first apply). The API VIP Neutron port stays **DOWN** (`…-DO-NOT-DELETE-even-if-DOWN`). Do not delete it. Flux `HealthCheckFailed` after 30s on Cluster/OpenshiftAssistedControlPlane `InProgress` is the health-check window, not a failed install.

`cluster-import-check` is Rancher enrollment, not OKD bootstrap. If its Job logs `timed out waiting for the condition on clusters/<id>` and that object has `apiEndpoint: https://100.73.0.1:443` with `AgentDeployed` absent, the service CIDR collided with management. Recreate the workload cluster with `172.30.0.0/16`. Do not treat a retry of that Job as the fix.

**Gate C-CP:** 3 masters Installed/Done, API up, CPs virtio. Apply is finished when the log says **All done** and `sylva-units-status` is True. Rancher success for this CIDR fix: `clusters.management.cattle.io` for this cluster has `AgentDeployed=True` and `apiEndpoint` `https://172.30.0.1:443` (the OKD service range), not `https://100.73.0.1:443`.

## Apps DNS (required once the ingress routers are Running)

OKD forwards the cluster base domain (usually `sylva`) to the management `k8s-gateway` (`172.24.4.233` on this lab; the management API VIP FIP). `authentication`, `console`, and `ingress` stay Degraded until `*.apps.<cluster>.<baseDomain>` answers. A hypervisor `/etc/hosts` line does not fix in-cluster lookups.

Do this on the **management** overlay (`environment-values/<lab>-rke2-capo/values.yaml`), then one `./apply.sh` as `sylva`. Do not put it in the OKD workload values.

1. Find the nodes that run `router-default` (HostNetwork). Use those node IPs. Do not use the API VIP, the management VIP, or the ingress VIP port (`…-sylva-ingress-vip-port-DO-NOT-DELETE` stays DOWN).
2. Under `units.k8s-gateway.helmrelease_spec.values`, set `extraZonePlugins` to the **full** list from `charts/sylva-units/values.yaml` (`log`, `errors`, `health`, `ready`, `prometheus`, `forward`, `loop`, `reload`, `loadbalance`, `hosts`) and append the template plugin below. A user list **replaces** the chart list. A template-only list deletes `health`, `ready`, and `hosts`, and DNS for `rancher.sylva` and the OKD zone goes down together.
3. The template `configBlock` must be a `(raw)` value so Sylva does not interpret `{{ .Name }}`. Writing `"{{ "{{" }} .Name {{ "}}" }}"` is evaluated again and becomes an empty name (`answer " 60 IN A …"`). That stanza crash-loops CoreDNS; OKD then reports `server misbehaving` on `172.30.0.10:53`.

```yaml
units:
  k8s-gateway:
    helmrelease_spec:
      values:
        extraZonePlugins:
          # copy log, errors, health, ready, prometheus, forward,
          # loop, reload, loadbalance, and hosts from the chart first
          - name: template
            parameters: IN A apps.<cluster>.<baseDomain>
            configBlock:
              (raw): |
                match .*
                answer "{{ .Name }} 60 IN A <router-node-ip>"
                answer "{{ .Name }} 60 IN A <other-router-node-ip>"
                fallthrough
```

`(raw)` must be the only key in that map. After apply, `dig @<management-vip-fip> oauth-openshift.apps.<cluster>.<baseDomain> A` must return the router node IPs, and `kubectl get co authentication console ingress` must be Available and not Degraded. Router node IPs change when those machines are replaced; refresh the A records or the console and oauth checks fail again. A later management apply that rewrites `values.yaml` without this block removes it.

## Worker

If SR-IOV **no**: set `md0.replicas: 1`, apply, done when the worker is Ready.

If SR-IOV **yes**: Glance RHCOS **only around worker boot**, then unset:

```bash
IMG=<rhcos-glance-uuid>
openstack image set --property hw_machine_type=q35 "$IMG"
openstack image set --property hw_vif_model=igb "$IMG"
openstack image set --property hw_viommu_model=intel "$IMG"
```

Set `md0.replicas: 1`. Apply as `sylva`. When the new server is **ACTIVE**:

```bash
openstack image unset --property hw_machine_type "$IMG"
openstack image unset --property hw_vif_model "$IMG"
openstack image unset --property hw_viommu_model "$IMG"
```

Worker flavor must already have `hw:viommu_model=intel`. Gate: guest `sriov_totalvfs` on the igb PF (expect 7). Then policy/operator as needed for VT10 — not host `PHYSICAL_DEVICE_MAPPINGS`.
