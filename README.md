# Cursor skills

Agent skills for DevStack, a Sylva RKE2 management cluster, and an OKD workload cluster.

| Folder | Use when |
|---|---|
| `devstack-deploy` | Install DevStack with `stack.sh` |
| `devstack-teardown` | Unstack or clean DevStack |
| `rke2-management-deploy` | Bootstrap the Sylva RKE2 management cluster |
| `rke2-management-teardown` | Delete that management cluster |
| `okd-workload-deploy` | Apply a Sylva CAPO OKD workload cluster |
| `okd-workload-teardown` | Delete an OKD workload cluster |
| `okd-worker-sriov` | Simulated igb SR-IOV PF on an existing OKD worker |

## Use them in every project

Clone this repo, then link each skill into your personal skills directory:

```bash
git clone https://github.com/vjayaramrh/cursorskills.git
mkdir -p ~/.cursor/skills
for d in devstack-deploy devstack-teardown rke2-management-deploy rke2-management-teardown okd-workload-deploy okd-workload-teardown okd-worker-sriov; do
  ln -sfn "$PWD/cursorskills/$d" "$HOME/.cursor/skills/$d"
done
```

Run that from the directory that contains the clone. Cursor loads skills from `~/.cursor/skills/` in any open project.

Do not put passwords, pull secrets, or SSH private keys in this repo. The skills ask for a key path and read passwords from the host.
