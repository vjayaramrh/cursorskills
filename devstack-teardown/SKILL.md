---
name: devstack-teardown
description: >-
  Tears down an OpenStack DevStack cloud with unstack.sh and optional clean.sh.
  Use when the user asks to unstack, destroy, wipe, remove, or tear down
  DevStack, or to fully reset a hypervisor before a fresh stack.sh.
  Do not use for deploying DevStack (see skill devstack-deploy).
  Never unstack a live lab without explicit user confirmation.
---

# DevStack teardown

## Hard rules

- **Ask the user to confirm** before running `unstack.sh` or `clean.sh`. This stops OpenStack, drops DBs, and can leave orphan libvirt domains.
- Do **not** use teardown as a “fix” for PCI, NFD, or CAPO on a live cloud unless the user explicitly wants the cloud gone.
- Run as user **`stack`** from `/opt/stack/devstack`.
- `unstack.sh` then `clean.sh` if they want a **full** wipe before a new deploy. `unstack.sh` alone is not always enough for a clean restack.
- After teardown, leftover **libvirt** guests named `instance-*` can block the next `n-cpu`. Destroy them only after confirming they belong to this DevStack.

## Order

1. Confirm target host (`HOST_IP` / hostname) and that the user wants **this** cloud destroyed.
2. Stop applying Sylva/CAPO against it (out of scope for this skill; warn if VMs still exist that they care about).
3. Unstack:

```bash
sudo -u stack -H bash -lc 'cd /opt/stack/devstack && ./unstack.sh'
```

4. If they asked for a full wipe / restack-ready host:

```bash
sudo -u stack -H bash -lc 'cd /opt/stack/devstack && ./clean.sh'
```

5. Leftover QEMU/libvirt (only if `virsh list --all` still shows DevStack instances):

```bash
sudo virsh list --all
# if stale instance-* domains remain:
# sudo virsh destroy <name>
# sudo virsh undefine <name>
```

6. Do **not** delete `/opt/stack/devstack/local.conf` unless they asked. Passwords live there (`chmod 600`).

## Gate (torn down)

```bash
systemctl list-units 'devstack@*' --no-pager | head
# expect no active devstack@ units
sudo virsh list --all
```

A new cloud is **devstack-deploy**, not another `stack.sh` on a half-dead tree.
