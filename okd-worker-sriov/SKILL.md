---
name: okd-worker-sriov
description: >-
  Gives one OKD worker on an existing Sylva CAPO cluster a simulated
  SR-IOV PF (QEMU igb, q35, vIOMMU) so NFD can set
  network-sriov.capable. Prompts for hypervisor IP, SSH user, and
  key path. Use when the user asks for SR-IOV, VT10, igb, or
  sriov_totalvfs on the OKD worker. Do not use for host Neutron
  SR-IOV, stack.sh, RKE2 bootstrap, or creating the OKD cluster
  (that is okd-workload-deploy). Never bootstrap.sh or kubectl scale.
---

# OKD worker SR-IOV

Same safety rules as the skills in [vjayaramrh/cursorskills](https://github.com/vjayaramrh/cursorskills). Creating the cluster is **okd-workload-deploy**. Deleting it is **okd-workload-teardown**. This skill only replaces the worker NIC with Nova’s emulated igb PF.

This is **not** host `q-sriov-agt`, `PHYSICAL_DEVICE_MAPPINGS`, or `vnic-type: direct`. A real PF passed through with vfio does not expose `sriov_totalvfs`.

## Hard rules

- Require an existing OKD cluster: 3 control planes Installed, API up. If it does not exist, stop and use **okd-workload-deploy**.
- Control planes stay **pc + virtio** on `m1.xxlarge-160`. Never set q35, igb, or `hw:viommu_model` on that flavor or on the shared Glance image except for the window below.
- Worker only: flavor `m1.xxlarge-160-igb` with `hw:viommu_model=intel`. Do not add `sriov-net` or `vnicType: direct`.
- Drive `md0` only through `values.yaml` and `./apply-workload-cluster.sh` as **`sylva`**. Do not `kubectl scale`. Do not `bootstrap.sh` or `apply.sh`.
- Glance `hw_machine_type`, `hw_vif_model`, and `hw_viommu_model` go on the RHCOS/SCOS image **only while the worker boots**, then come off as soon as that server is **ACTIVE**.
- RAM cannot hold two 32 GiB workers. If an md0 VM exists, set replicas to 0 and wait until it is **gone** before 1.
- Do not restack DevStack. Do not delete ports named `DO-NOT-DELETE`. Do not print passwords, pull secrets, or SSH private keys.
- QEMU must be ≥ 8.0 and libvirt ≥ 9.3, with host IOMMU on (`intel_iommu=on` or `amd_iommu=on`, and `iommu=pt`). If those fail, stop. Do not change `local.conf` on a live cloud.

## Start here

Ask, then wait:

1. `What IP is the DevStack hypervisor?` SSH user and private-key **path**. Never paste the key.
2. Workload kubeconfig path on that host (lab default `/tmp/lab53-okd.kubeconfig`). Management kubeconfig stays the one **okd-workload-deploy** used.
3. `Worker PF only, or also the VT10 operator (sriov-network-operator and a SriovNetworkNodePolicy)?` (`pf` / `vt10`).

Confirm before any write: 3 control planes are virtio on `m1.xxlarge-160`, and `md0` will go 0 then 1.

## 1. Flavor

As `stack`, on this cloud:

```bash
openstack flavor show m1.xxlarge-160-igb
```

If it is missing, create it as a copy of the CP flavor (same ram, disk, vcpus) and set only:

```bash
openstack flavor set m1.xxlarge-160-igb --property hw:viommu_model=intel
```

Do not put that property on `m1.xxlarge-160`.

In `environment-values/workload-clusters/okd-capo/values.yaml`, `machine_deployments.md0.capo.flavor_name` is `m1.xxlarge-160-igb`. `cluster.capo.flavor_name` stays `m1.xxlarge-160`. Show that diff and wait for **apply**.

## 2. Boot the worker

Find the Glance image this cluster boots. As `stack`:

```bash
IMG=<that-image-uuid>
openstack image set --property hw_machine_type=q35 "$IMG"
openstack image set --property hw_vif_model=igb "$IMG"
openstack image set --property hw_viommu_model=intel "$IMG"
```

Set `md0.replicas: 0`, apply as `sylva`, and wait until the old worker server is gone. Then set `md0.replicas: 1` and apply once:

```bash
sudo -u sylva -H bash -lc '
cd ~/sylva-core-<checkout>
export KUBECONFIG=<management-cluster-kubeconfig>
./apply-workload-cluster.sh environment-values/workload-clusters/okd-capo
'
```

When the new server is **ACTIVE**, unset the three image properties. The running worker keeps them in Nova metadata. Later control-plane rebuilds stay virtio.

```bash
openstack image unset --property hw_machine_type "$IMG"
openstack image unset --property hw_vif_model "$IMG"
openstack image unset --property hw_viommu_model "$IMG"
```

## 3. Gate

On the worker (`oc debug node/<worker> -- chroot /host` with the workload kubeconfig):

```bash
lspci -nn | grep -i ether
cat /sys/bus/pci/devices/0000:01:00.0/sriov_totalvfs
```

Pass: `8086:10c9` and `sriov_totalvfs` **7**. Then:

```bash
oc get node <worker> -o jsonpath='{.metadata.labels.feature\.node\.kubernetes\.io/network-sriov\.capable}{"\n"}'
```

Pass for the PF path: that label is `true`. Stop here when the answer was **pf**.

If the label stays empty, the NIC is not a PF. Do not add a NodeFeatureRule to fake it, and do not switch the port to `vnic-type: direct`.

## 4. VT10 operator

Only when the answer was **vt10**.

Enable the existing units (do not invent stubs):

```yaml
units:
  sriov-network-operator:
    enabled: true
  sriov-crd:
    depends_on:
      multus: false
```

Apply the workload cluster once as `sylva`.

`sriov-network-config-daemon` init `mkdir /opt/cni: file exists` is the RHCOS `/opt/cni` symlink. On the worker:

```text
/etc/tmpfiles.d/zz-opt-cni.conf
L+ /opt/cni - - - - /var/lib/cni
```

If the device-plugin or config-daemon image pull from `registry.opensuse.org` times out, sideload those images with `oc debug` and `podman load`. Do not disable the InfiniBand plugin; the operator enum cannot skip it.

After the node is `network-sriov.capable=true`, add one policy in the operator namespace (on this chart, `cattle-sriov-system`). Confirm the namespace before create. `numVfs` greater than 1 is denied on this OpenStack NIC.

```yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: igb-test
  namespace: cattle-sriov-system
spec:
  deviceType: netdevice
  numVfs: 1
  nicSelector:
    vendor: "8086"
    deviceID: "10c9"
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
  resourceName: igbvf
  priority: 99
  isRdma: false
```

Gate:

```bash
oc get ds -n cattle-sriov-system sriov-network-config-daemon sriov-device-plugin
```

Both `DESIRED` ≥ 1 and `READY` equals `DESIRED`. `desired` 0 fails VT10.

VT08/VT10 smoke, when they ask to run it, uses the **workload** kubeconfig:

```bash
cd ~/validation_center/flavour-validation-subgroup/flavour-validation-scripts/Script\ VT08\ \&\ VT10
KUBECONFIG=<workload-kubeconfig> ./run.sh
```

Pass is both `multus_cni_smoke` and `sriov_cni_smoke` PASS. That does not prove a 25G dataplane.
