---
name: rke2-management-deploy
description: >-
  Step 2 of DevStack then Sylva RKE2 management then OKD workload.
  Bootstraps a Sylva CAPO RKE2 management cluster as one OpenStack VM
  (not RKE2 on the metal).   Prompts for hypervisor IP, SSH user, and SSH
  private-key path (never paste the key).   Always asks for the sylva-core git branch and waits until the user
  types a name (suggested default main; never infer HEAD or treat an
  empty reply as main). After they enter it, fetch and verify that
  branch exists on the sylva-core remote before continuing.   Walks every environment-values section (kustomization.yaml
  and all values.yaml top-level keys, plus secrets auth_url only)
  and waits for edit or next before bootstrap. Overlay may only
  enable units that exist on that branch — never invent stubs
  (e.g. harbor-postgres). One bootstrap.sh at a time. Use
  when the user asks to deploy, bootstrap, or install the RKE2
  management cluster on DevStack.
  Do not use for stack.sh (devstack-deploy), OKD (okd-workload-deploy),
  or unstack (devstack-teardown). Never bootstrap as user stack.
---

# RKE2 management deploy

Intended path: `devstack-deploy` → **this skill** → `okd-workload-deploy`. Kind on the metal is bootstrap only, then pivot.

## Hard rules

- Run Sylva as user **`sylva`** (`docker` group). Never `bootstrap.sh` as `stack`.
- Do **not** run `stack.sh` or restack to “fix” CAPO. If DevStack is missing, stop and use **devstack-deploy**.
- Do **not** run `bootstrap.sh` for OKD (use **okd-workload-deploy**). Management bootstrap is once per successful Gate B.
- Management is **1 CP**, virtio, Neutron **`private`** + FIP on **`public`**. No `sriov-net`, no igb, `md0.replicas: 0`.
- Never paste OpenStack passwords. `secrets.yaml` mode 600, owner `sylva`; `auth_url: http://<HOST_IP>/identity`; password from `/opt/stack/devstack/local.conf` only.
- **Always get a sylva-core git branch and prove it exists.** Do not start the overlay walk, checkout, or `bootstrap.sh` until (1) the user has typed a non-empty branch name and (2) that name exists on the sylva-core remote after `git fetch`. Do not use the checkout’s current HEAD. Do not treat silence, **next**, or **accept remaining** as `main`. Suggested default to *offer* is `main`; they must still enter it (or another name). Store it as `SYLVA_CORE_BRANCH` only after the exists-check passes. This is the sylva-core repo, not `source_templates`.
- **Do not invent overlay units.** `values.yaml` `units:` may only toggle units that exist as top-level units on `$SYLVA_CORE_BRANCH`. Do not add stub blocks (`info` / `unit_templates` / empty `helmrelease_spec`) or commented-out unit YAML. Helm preview fails with `/units/<name> missing info|unit_templates|helmrelease_spec|kustomization_spec` when a stub was invented. `harbor-postgres` is not a top-level unit (it lives under harbor); keep `harbor.enabled: false` and omit `harbor-postgres`. If `cabpoa: true` and the branch has `okd-openshift-mce`, enable that real unit — cabpoa waits on it.
- **One `bootstrap.sh` process.** `pgrep` before start. Do not launch a second copy because Flux shows `HealthCheckFailed` for 30s. Retry only after the previous PID has exited (Helm preview schema failure is the usual case): fix overlay, then one new run. Never `bootstrap.sh` for OKD.

## Start here (ask before SSH)

1. `What IP is the DevStack hypervisor?`
2. `What SSH user?` and `What path is the SSH private key?` (skip path if agent/`~/.ssh/config` works). Never paste the key.
3. **sylva-core branch (required, always):** Ask and **wait** until they type a name:

```text
Which sylva-core git branch should be used?
Suggested default: main
Enter a branch name (required). Do not leave this blank.
```

   Re-ask on empty / “default” / “whatever is checked out”. Accept `main` only if they type `main`.

   Then **verify the branch exists** in the sylva-core checkout (as `sylva`) before section 1. Do not `git checkout` yet.

```bash
cd /home/sylva/sylva-core-<checkout>
git fetch origin
# must print a commit; empty → branch does not exist
git ls-remote --heads origin "refs/heads/<name>"
git rev-parse --verify "origin/<name>"
```

   If `ls-remote` is empty or `rev-parse` fails: tell them the branch was **not found**, show a few `git branch -r | head`, **clear any tentative name**, and ask again. Do not start the overlay walk.

   Only after a commit is shown: record `SYLVA_CORE_BRANCH=<name>` and continue.
4. Do **not** ask the OKD SR-IOV / CP-then-worker questions here.

```bash
ssh -o BatchMode=yes -o IdentitiesOnly=yes -i <key_path> <ssh_user>@<HOST_IP> '...'
```

## Pre-flight (on the hypervisor)

Confirm Gate A already passed: token, hypervisor, `cpu_mode=host-model` in `/etc/nova/nova-cpu.conf`, flavor `m1.xxlarge-160`. Placement inventories non-empty. Keypair + `private` / `public` UUIDs exist. Docker can resolve `ghcr.io` before Kind.

The `private` subnet must have public `dns_nameservers` (for example `8.8.8.8` and `1.1.1.1`). DevStack dnsmasq does not forward. Without that, the node’s `curl -sfL https://get.rke2.io` fails silently (`set +e` in the userdata), `rke2-server.service` is never installed, and bootstrap.sh exits with `Unit timeout exceeded` on `Kustomization/cluster` after 18 minutes. `INSTALL_RKE2_ARTIFACT_PATH` is exported in a different cloud-init command, so local artifacts are not used by that curl. Set the subnet DNS before the first VM boots. A VM that already failed cloud-init must be replaced; it will not retry runcmd.

Enable `cabpoa` now (needed later for OKD Assisted). Leave harbor/monitoring off unless they asked.

## Overlay review (section by section — wait before bootstrap)

Remap **this** cloud. Do not copy another lab’s UUIDs or passwords. Fill proposals from this cloud (`openstack flavor/keypair/network list`) and from an existing `values.yaml` if present. Do **not** cat `secrets.yaml` into chat.

Walk **one YAML section at a time**. Show only that section’s proposed keys, then wait:

```text
Section N/<total>: <name>
<yaml for this section only>
Reply: next | edit <key>=<value> | accept remaining | stop
```

- **next** — keep this section, show the following one.
- **edit** — merge, re-show **this** section, wait again.
- **accept remaining** — keep this section and all later ones as proposed (no more stops until the final apply).
- **stop** — do not write or bootstrap.

Do **not** dump the whole file first. After the last section (or **accept remaining**), show the assembled `values.yaml` once and wait for **apply**. Do not write files or run `bootstrap.sh` until they say **apply**.

Walk **every** top-level key in `values.yaml` in file order. If a key exists that is not in the table, still stop on it. If a listed key is missing, say so and **next**. Nested `cluster.capo` and `machine_deployments` stay separate stops (high-stakes remap). For `units`, show each `*.enabled` flag; do not paste long kube-rbac-proxy patches unless they ask to edit a named operator. Never print `secrets.yaml` passwords or OCI registry credentials.

| # | Section | Proposed | Warn if they |
|---|---|---|---|
| 1 | Paths | `sylva-core` dir, `environment-values/<lab>-rke2-capo`, **`SYLVA_CORE_BRANCH` (must already be set)**; show disk HEAD vs that name | HEAD ≠ entered branch; missing `SYLVA_CORE_BRANCH` |
| 2 | `kustomization.yaml` | `../base` + `metal3-crds` shim (needed for cabpoa/Assisted) | drop metal3-crds while cabpoa is on |
| 3 | `source_templates` | Prefer **drop** so `$SYLVA_CORE_BRANCH` chart defaults apply. If present, show and offer drop; do not copy another lab | pinning another lab’s chart branch/tag |
| 4 | `os_images_oci_registries` | Prefer **drop** unless they asked to pin. Do not invent OCI urls/tags or a cosign pubkey | invented tag (e.g. 1.1.4 on an image that only has 1.1.1/1.1.2) |
| 5 | `sylva_diskimagebuilder_images` | Prefer **drop**. Management CP image is ubuntu hardened RKE2 from the chart, not RHCOS/SCOS | forcing RHCOS on the RKE2 CP |
| 6 | `os_image_server_additional_selectors` | Existing selectors (may stay, e.g. `os: scos`, for later OKD Glance sync) | |
| 7 | `cluster` | `name: <lab>-rke2`, `control_plane_replicas: 1`, capo+cabpr | replicas ≠ 1 |
| 8 | `cluster.capo` | ubuntu hardened, `flavor_name: m1.xxlarge-160`, `ssh_key_name: <keypair>`, `network_id: <private-uuid>` | another lab’s UUID |
| 9 | `machine_deployments` | `md0.replicas: 0` | replicas ≠ 0 or `sriov-net` |
| 10 | `openstack` | `storageClass` cinder-default/`__DEFAULT__`, `external_network_id: <public-uuid>` | |
| 11 | `proxies` | From file (often empty strings) | |
| 12 | `ntp` | From file | |
| 13 | `units` | `cabpoa: true`; if the branch has `okd-openshift-mce`, enable it (cabpoa depends on it). harbor/monitoring/postgres/pushgateway **false**. Omit `harbor-postgres`. Keep existing kube-rbac-proxy patches; do not paste them unless they edit a named operator. No commented unit stubs. If `k8s-gateway.extraZonePlugins` is already present for the OKD apps zone, **keep it** and only refresh router node IPs (see **okd-workload-deploy**, Apps DNS). Never replace that list with only a `template` plugin | `cabpoa: false`; invented `harbor-postgres` stub; a partial `extraZonePlugins` list (it replaces the chart list and crash-loops management DNS) |
| 14 | `secrets.yaml` | `auth_url: http://<HOST_IP>/identity`, username `admin` | never print the password; source from `local.conf` |

## Deploy

Do not write overlay files until they say **apply**. Before replacing anything in `environment-values/<lab>-rke2-capo/`:

1. **Backup** the current files as `sylva` (timestamp, same dir). Never overwrite the backup.

```bash
ENV=/home/sylva/sylva-core-<checkout>/environment-values/<lab>-rke2-capo
STAMP=$(date -u +%Y%m%dT%H%M%SZ)
cp -a "$ENV/values.yaml" "$ENV/values.yaml.bak-$STAMP"
# if secrets.yaml or kustomization.yaml will be rewritten:
# cp -a "$ENV/secrets.yaml" "$ENV/secrets.yaml.bak-$STAMP"
# cp -a "$ENV/kustomization.yaml" "$ENV/kustomization.yaml.bak-$STAMP"
ls -l "$ENV"/values.yaml*
```

2. **Write** the reviewed `values.yaml` (omit dropped keys). Keep `secrets.yaml` mode 600; do not print it.
3. Then fetch, re-verify, and **checkout** `$SYLVA_CORE_BRANCH` before `bootstrap.sh`. Refuse if unset or missing on origin. Show `git log -1 --oneline`. Do not bootstrap if checkout fails. If the tree is dirty (local OKD overlay edits), `git stash push -u -m "pre-$SYLVA_CORE_BRANCH"` first — do not discard.

Pass the branch **into** the `sylva` shell (single-quoted `bash -lc` will not see the agent’s env):

```bash
# if this prints a PID, stop — monitor that run; do not start another
pgrep -a -f 'bash ./bootstrap.sh environment-values/<lab>-rke2-capo' || true

sudo -u sylva -H bash -lc "
cd ~/sylva-core-<checkout>
export SYLVA_CORE_BRANCH='$SYLVA_CORE_BRANCH'
git fetch origin
git checkout \"\$SYLVA_CORE_BRANCH\"
git log -1 --oneline
export KUBECONFIG=~/.kube/config
nohup ./bootstrap.sh environment-values/<lab>-rke2-capo \
  >> ~/sylva-core-<checkout>/bootstrap.log 2>&1 &
"
```

Do not start a second copy. If Helm preview fails on a missing unit schema, wait until the PID is gone, remove the invented unit from `values.yaml` (backup first), then one new `bootstrap.sh`.

If `bootstrap.sh` already exited after the cluster timeout, but the VM later becomes Ready and the `pivot` Job completes, do not start `bootstrap.sh` again. `check_pivot_has_ran` refuses that. Run one `./apply.sh environment-values/<lab>-rke2-capo` as `sylva`. It waits for `sylva-units-status` and deletes the Kind cluster.

## Monitor (expected; not failure)

Post status on each phase. Kind `KUBECONFIG` (`~/.kube/config`) until pivot; then `management-cluster-kubeconfig`.

| Phase | What you see | Do not |
|---|---|---|
| Preview / Kind | Helm preview, Kind node `sylva-control-plane` Ready | invent units; restack |
| Kind units | `HealthCheckFailed` ~30s on kyverno / `sync-openstack-images` while Helm is `InProgress` | start a second bootstrap |
| CAPO VM | Cluster KS → HelmRelease `cluster` → Nova `BUILD` then `ACTIVE` on `private` | treat Cluster `InProgress` 30s health timeout as a hang |
| CP init | Machine Provisioning → Running; `management-cluster-kubeconfig` appears | curl the instance IP for 6443 |
| API | FIP on the **VIP** (`*-sylva-virtual-ip-port-DO-NOT-DELETE-even-if-DOWN`). Port stays **DOWN**. `curl -k https://<VIP-FIP>:6443` → **401** is success. Instance `:6443` often fails | delete the VIP port; use ICMP |
| Units on RKE2 | kyverno → cert-manager → capi/cabpr/capo → `okd-openshift-mce` → cabpoa | disable MCE while cabpoa is on |
| Pivot | Kind still listed; `remaining_ks` may spike (~100) while `sylva-units` upgrades and `root-dependency-3` appears | restart bootstrap |
| Done | Log **All done**, Kind deleted, remaining KS/HR = 0 | |

## Gate B

- `management-cluster-kubeconfig` exists and Kind is gone (`kind get clusters` empty).
- `curl -k https://<VIP-FIP>:6443` returns HTTP 401 (or other TLS HTTP from the apiserver). ICMP is not the test.
- RKE2 CP is virtio on `private` only (`vnic_type=normal`, `vif_type=ovs`).
- `kubectl get cluster,machine` on that kubeconfig: Available, 1 CP, md0=0.

Then: `export KUBECONFIG=.../management-cluster-kubeconfig`. Next skill: **okd-workload-deploy**.
