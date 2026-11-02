# Automated build of HA k3s Cluster with `kube-vip` and MetalLB

Forked from <https://github.com/timothystewart6/k3s-ansible>.

This playbook is used to build my personal home cluster deployed on Raspberry PIs.

This Ansible collection builds a highly available Kubernetes cluster with k3s. It supports kube-vip for the control
plane virtual IP, multiple CNI options, and either MetalLB or kube-vip for service load balancing.

The current settings deploy

- k3s with embedded etcd on a single master node
- MetalLB

## Setup
For more context on how it works, see:

📄 [Documentation](https://technotim.com/posts/k3s-etcd-ansible/) (including example commands)

Install deps

```console
uv sync
source .venv/bin/activate
```

Install ansible-galaxy deps
## Project guides

- [Getting started](#-getting-started)
- [Configuration variables](#variables)
- [Upgrading an existing cluster](#-upgrading-an-existing-cluster)
- [Local Molecule testing](molecule/README.md)
- [Contributing guidelines](CONTRIBUTING.md)
- [Repository guide for coding agents](AGENTS.md)

## 📖 k3s Ansible Playbook

Build a Kubernetes cluster using Ansible and k3s. The goal is to make a highly available cluster straightforward to
install on machines running:

- [x] Debian (tested on version 13)
- [x] Ubuntu (tested on version 26.04 LTS)
- [x] Rocky (tested on version 10)

Supported processor architectures are:

- [x] x64
- [x] arm64
- [x] armhf

## ✅ System requirements

- The control node, which runs the Ansible commands, must have Ansible 2.11 or newer. For a quick primer, see
  [setting up Ansible](https://technotim.com/posts/ansible-automation/).

- Install the required collections with
  `ansible-galaxy collection install -r ./collections/requirements.yml`.

- [`netaddr` package](https://pypi.org/project/netaddr/) must be available to Ansible. If you have installed Ansible via apt, this is already taken care of. If you have installed Ansible via `pip`, make sure to install `netaddr` into the respective virtual environment.

- Server and agent nodes should support passwordless SSH access. Otherwise, pass `--ask-pass --ask-become-pass` to
  each playbook command.

- Every node in the cluster must have a **unique hostname**. k3s registers each node keyed by its hostname, so
  two nodes with the same hostname cannot join the cluster. `site.yml` asserts this up front and fails fast if
  any duplicate is found.

## 🚀 Getting Started

### 🍴 Preparation

Create a cluster-specific inventory from the sample. The `inventory/` directory ignores custom inventory content so
credentials and environment details aren't committed accidentally.

```bash
cp -R inventory/sample inventory/my-cluster
```

Edit `inventory/my-cluster/hosts.ini` to match the target hosts.
>>>>>>> upstream/master

```console
ansible-galaxy collection install -r ./collections/requirements.yml
```

A new directory based on the `sample` directory within the `inventory` directory has been created, under `inventory/my-cluster`.

Settings are in `ansible.cfg` with adapted inventory path to match.

The `inventory/my-cluster/group_vars/all.yml` has been customised to allow installing embedded ETCD.

The minimum k3s version is `1.19.1`. Select the desired version with the `k3s_version` variable.

## Prepare Cluster

First, copy over ssh key

```console
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@control01
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@cube02
...
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@cubexx
```

### Prepare nodes with packages

<https://github.com/timothystewart6/k3s-ansible/issues/463>

```console
ansible-playbook prepare-nodes.yml -i inventory/my-cluster/hosts.yml
```

### Prepare Storage

Later on we will use longhorn, <https://rpi4cluster.com/k3s-storage-setting/#file-system-and-mount>

Here's how disks are currently setup.

#### RPis

##### USBs

```console
ansible rpi -b -m shell -a "lsblk -f"
```

Verify non-boot disks are sdb, then wipe

```console
ansible rpi -b -m shell -a "wipefs -a /dev/sdb"
ansible rpi -b -m filesystem -a "fstype=ext4 dev=/dev/sdb"
```

Get UUIDs

```console
ansible rpi -b -m shell -a "blkid -s UUID -o value /dev/sdb"
```

Mount

```console
ansible cube03 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=66c05758-9efc-46c3-8fbc-3ed0e84ac3b6 fstype=ext4 state=mounted" -b

ansible cube02 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=73791edf-80b7-4d11-bd1e-ab3cb99e83b7 fstype=ext4 state=mounted" -b

ansible control01 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=33674496-d90e-40c8-962f-728b4ffb11ca fstype=ext4 state=mounted" -b
```

##### NVME SSD

Currently, only control01 (192.168.0.177) has SSD.

This on I have decided to put on the master. I want to partition it so that we have 64GB for k3s_server to use as data drive and for ETCD.
Then the rest I will make available to longhorn if there's anything that requires super fast disk.

```console
ansible rpi -b -m shell -a "lsblk -f"
```

```console
export node=control01
export device=/dev/nvme0n1
```

Wipe & reformat if needed

```console
ansible $node -b -m shell -a "wipefs -a $device"
ansible $node -b -m filesystem -a "fstype=ext4 dev=$device"
ansible $node -b -m shell -a "lsblk -f"
```

Partition the SSD

```console
ansible $node -m shell -a "parted $device --script mklabel gpt" -b
ansible $node -m shell -a "parted $device --script mkpart k3s ext4 0% 64GB" -b
ansible $node -m shell -a "parted $device --script mkpart longhorn ext4 64GB 100%" -b
```

Confirm the partitions with

```console
ansible $node -b -m shell -a "lsblk -f"
```

Then, format the partitions

```console
ansible $node -b -m shell -a "wipefs -a /dev/nvme0n1p1"
ansible $node -b -m shell -a "wipefs -a /dev/nvme0n1p2"
ansible $node -b -m filesystem -a "fstype=ext4 dev=/dev/nvme0n1p1"
ansible $node -b -m filesystem -a "fstype=ext4 dev=/dev/nvme0n1p2"
ansible $node -b -m shell -a "lsblk -f"
```

Get UUIDs

```console
ansible $node -b -m shell -a "blkid -s UUID -o value /dev/nvme0n1p1"
ansible $node -b -m shell -a "blkid -s UUID -o value /dev/nvme0n1p2"
```

Mount them. We're going to mount the large partition under `/mnt/storage02`, following what we did for the USBs, meanwhile the k3s partition under `/mnt/k3sdata`. We also mount that to `/var/lib/rancher/k3s` in order to be compatible the default value for the path <https://docs.k3s.io/cli/server#data>

```console
ansible $node -m ansible.posix.mount -a "path=/mnt/k3sdata src=UUID=1791b0ac-e61e-4ecb-b69d-904ab69fc39a fstype=ext4 state=mounted" -b
ansible $node -m ansible.posix.mount -a "path=/var/lib/rancher/k3s src=UUID=1791b0ac-e61e-4ecb-b69d-904ab69fc39a fstype=ext4 state=mounted" -b

ansible $node -m ansible.posix.mount -a "path=/mnt/storage02 src=UUID=5481387e-a717-4775-99a4-3fb3a59bc531 fstype=ext4 state=mounted" -b
```

We need to also point --data-dir at this new k3sdata partition.

#### x86

I added some old x86 linux machines in. These have HDD/SSD and have a 32GB boot partition. The rest we want to use for longhorn storage. The partition for data is then `/dev/sda2`

```console
ansible x86 -b -m shell -a "lsblk -f"
```

Get UUIDs

```console
ansible x86 -b -m shell -a "blkid -s UUID -o value /dev/sda2"
```

Mount

```console
ansible cube04 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=3e43d753-6398-41f5-9f25-5bd479378606 fstype=ext4 state=mounted" -b

ansible cube05 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=878139eb-6cb4-4c08-bbb6-83fefaa2736d fstype=ext4 state=mounted" -b

ansible cube06 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=7ccf0e05-f26f-43de-97a2-43a36445099c fstype=ext4 state=mounted" -b
```

## ☸️ Create Cluster

Start provisioning of the cluster using the following command:

```bash
ansible-playbook site.yml -i inventory/my-cluster/hosts.yml
```

After deployment, the control plane is accessible through the virtual IP defined by `apiserver_endpoint` in the
inventory variables.

### Apply Node Labels

```console
ansible-playbook label-nodes.yaml -i inventory/my-cluster/hosts.yml
```

### Add New Nodes

If the task is to simply add new nodes we can use the [limit option](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_patterns.html#patterns-and-ad-hoc-commands). E.g.

```console
ansible-playbook site.yml -i inventory/my-cluster/hosts.yml --limit "host1,host2"
```

## 🔥 Remove k3s cluster

```bash
ansible-playbook reset.yml -i inventory/my-cluster/hosts.yml
```

> Reboot the nodes after reset because the virtual IP may remain configured.

### ⏻️ Reboot Cluster Nodes

Reboot all cluster nodes at once or stage the reboot across the cluster.

```bash
ansible-playbook reboot.yml -i inventory/my-cluster/hosts.ini
```

To reboot the nodes in batches, set `concurrent_reboots` to the number of nodes
to reboot at a time (or a percentage). Optionally set `wait_seconds_after_reboot`
to pause after each batch so pods in the freshly rebooted batch can settle
before the next batch reboots.

```bash
ansible-playbook reboot.yml -i inventory/my-cluster/hosts.ini \
  --extra-vars 'concurrent_reboots=2 wait_seconds_after_reboot=30'
```

## 🔁 Upgrading an existing cluster

These version variables select the components used for a **fresh** installation.
They are not a supported direct in-place upgrade path for an existing cluster.
K3s, Calico, and Cilium each require staged upgrades for long-lived clusters.

- **K3s**: do not jump an embedded-etcd cluster straight to Kubernetes 1.36.
  Upgrade one Kubernetes minor version at a time. From the sample default
  (`v1.30.2+k3s2`) the sequence is: the latest supported 1.30 patch, then 1.31,
  1.32, a 1.33 patch that contains etcd 3.5.26 (for example `v1.33.7+k3s3`),
  then 1.34, 1.35, and finally 1.36. Upgrade servers one at a time before
  agents. Take backups and confirm cluster health at each step; this playbook
  does not automate the upgrade, so those remain manual operational steps. See
  [K3s manual upgrades](https://docs.k3s.io/upgrades/manual) and the
  [v1.34 release notes](https://docs.k3s.io/release-notes/v1.34.X).
- **Cilium**: upstream supports only consecutive minor upgrades. Update to the
  latest patch of the current minor, then upgrade 1.17, 1.18, 1.19, and 1.20 in
  order, reading each version's upgrade notes and running preflight checks.
  Do not attempt a direct upgrade from an old Cilium to 1.20.
- **Calico**: starting with 3.28 the v3 resource UID behavior changed. If you
  have operators with OwnerReferences pointing to `projectcalico.org/v3`
  resources, remove and recreate those references around an in-place upgrade.
- **MetalLB**: this project installs application tag `v0.16.0`. A newer
  chart-only tag such as `metallb-chart-0.16.1` is not an application or image
  release and must not be used as the controller or speaker image tag.

## ⚙️ Kube Config

To copy your `kube config` locally so that you can access your **Kubernetes** cluster run:

```bash
scp debian@master_ip:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

If you get file Permission denied, go into the node and temporarly run:
If the copy fails with a permission error, grant the SSH user temporary read access using the least permissive method
available for the target system. Restore the original ownership and permissions immediately after copying. Avoid
world-writable permissions on the kubeconfig because it contains cluster credentials.

For example, copy the file to a temporary user-readable path from the control node:

```bash
ssh debian@master_ip 'sudo install -o "$(id -un)" -m 0600 /etc/rancher/k3s/k3s.yaml /tmp/k3s.yaml'
```

Copy `/tmp/k3s.yaml`, then remove the temporary remote copy:

```bash
scp debian@master_ip:/tmp/k3s.yaml ~/.kube/config
ssh debian@master_ip rm -f /tmp/k3s.yaml
```

You'll then want to modify the config to point to master IP by running:

```bash
sudo nano ~/.kube/config
```

Then change `server: https://127.0.0.1:6443` to match your master IP: `server: https://192.168.0.222:6443`

## 🔨 Testing your cluster

See the commands [here](https://technotim.com/posts/k3s-etcd-ansible/#testing-your-cluster).

### Troubleshooting

Be sure to see [this post](https://github.com/timothystewart6/k3s-ansible/discussions/20) on how to troubleshoot common problems

### Testing the playbook using molecule

This playbook includes a [molecule](https://molecule.rtfd.io/)-based test setup.
It is run automatically in CI, but you can also run the tests locally.
This might be helpful for quick feedback in a few cases.
You can find more information about it [here](molecule/README.md).

### Pre-commit hooks

This repository uses `pre-commit` to check style, syntax, Ansible content, and shell scripts. Install the Python
dependencies, run `pre-commit install` once, and run `pre-commit run --all-files` before submitting a change. See
[CONTRIBUTING.md](CONTRIBUTING.md) for the complete development workflow.

## 🌌 Ansible Galaxy

This collection can now be used in larger ansible projects.

Instructions:

- create or modify a file `collections/requirements.yml` in your project

```yml
collections:
  - name: ansible.utils
  - name: community.general
  - name: ansible.posix
  - name: kubernetes.core
  - name: https://github.com/timothystewart6/k3s-ansible.git
    type: git
    version: master
```

- install via `ansible-galaxy collection install -r ./collections/requirements.yml`
- every role is now available via the prefix `techno_tim.k3s_ansible.` e.g. `techno_tim.k3s_ansible.lxc`

## Thanks 🤝

This repo is really standing on the shoulders of giants. Thank you to all those who have contributed and thanks to these repos for code and ideas:

- [k3s-io/k3s-ansible](https://github.com/k3s-io/k3s-ansible)
- [geerlingguy/turing-pi-cluster](https://github.com/geerlingguy/turing-pi-cluster)
- [212850a/k3s-ansible](https://github.com/212850a/k3s-ansible)
