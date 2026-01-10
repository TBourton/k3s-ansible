# Automated build of HA k3s Cluster with `kube-vip` and MetalLB

Forked from <https://github.com/timothystewart6/k3s-ansible>.

This playbook is used to build my personal home cluster deployed on Raspberry PIs.

This playbook will build an HA Kubernetes cluster with `k3s`, `kube-vip` and MetalLB via `ansible`.

The current settings deploy

- k3s with embedded etcd on a single master node
- MetalLB

## Setup

Install deps

```console
uv venv --python 3.11
source .venv/bin/activate

uv pip install -r requirements.txt
```

A new directory based on the `sample` directory within the `inventory` directory has been created, under `inventory/my-cluster`.

Settings are in `ansible.cfg` with adapted inventory path to match.

The `inventory/my-cluster/group_vars/all.yml` has been customised to allow installing embedded ETCD.

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
ansible 192.168.1.79 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=66c05758-9efc-46c3-8fbc-3ed0e84ac3b6 fstype=ext4 state=mounted" -b

ansible 192.168.1.109 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=73791edf-80b7-4d11-bd1e-ab3cb99e83b7 fstype=ext4 state=mounted" -b

ansible 192.168.1.177 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=33674496-d90e-40c8-962f-728b4ffb11ca fstype=ext4 state=mounted" -b
```

##### NVME SSD

Currently, only control01 (192.168.1.177) has SSD.

This on I have decided to put on the master. I want to partition it so that we have 64GB for k3s_server to use as data drive and for ETCD.
Then the rest I will make available to longhorn if there's anything that requires super fast disk.

```console
ansible rpi -b -m shell -a "lsblk -f"
```

```console
export node=192.168.1.177
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

Mount them. We're going to mount the large partition under /mnt/storage02, following what we did for the USBs, meanwhile the k3s partition under `/mnt/k3sdata`

```console
ansible $node -m ansible.posix.mount -a "path=/mnt/k3sdata src=UUID=1791b0ac-e61e-4ecb-b69d-904ab69fc39a fstype=ext4 state=mounted" -b
ansible $node -m ansible.posix.mount -a "path=/mnt/storage02 src=UUID=5481387e-a717-4775-99a4-3fb3a59bc531 fstype=ext4 state=mounted" -b
```

We need to also point --data-dir at this new k3sdata partition.

##### USB SSD

I Brought a 1TB SSD to act as additional storage, it's currently installed in cube02.

```console
ansible rpi -b -m shell -a "lsblk -f"
```

The disk should be sde

```console
export node=192.168.1.109
export device=/dev/sde
ansible $node -b -m shell -a "wipefs -a $device"
ansible $node -b -m filesystem -a "fstype=ext4 dev=$device"
```

Get UUID

```console
ansible $node -b -m shell -a "blkid -s UUID -o value $device"
```

Mount to /mnt/storage03

```console
ansible $node -m ansible.posix.mount -a "path=/mnt/storage03 src=UUID=3aa268b0-c8f7-4d68-8a63-a04fc58b4116 fstype=ext4 state=mounted" -b
```

#### x86

I added some old x86 linux machines in. These with the OS i partiton the HDDs into a 32GB boot partition. The rest we want to use for longhorn storage. The partition for data is then `/dev/sda2`

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

After deployment control plane will be accessible via virtual ip-address which is defined in inventory/group_vars/all.yml as `apiserver_endpoint`

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

>You should also reboot these nodes due to the VIP not being destroyed

## ⚙️ Kube Config

To copy your `kube config` locally so that you can access your **Kubernetes** cluster run:

```bash
scp debian@master_ip:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

If you get file Permission denied, go into the node and temporarly run:

```bash
sudo chmod 777 /etc/rancher/k3s/k3s.yaml
```

Then copy with the scp command and reset the permissions back to:

```bash
sudo chmod 600 /etc/rancher/k3s/k3s.yaml
```

You'll then want to modify the config to point to master IP by running:

```bash
sudo nano ~/.kube/config
```

Then change `server: https://127.0.0.1:6443` to match your master IP: `server: https://192.168.1.222:6443`

## 🔨 Testing your cluster

See the commands [here](https://technotim.live/posts/k3s-etcd-ansible/#testing-your-cluster).

### Troubleshooting

Be sure to see [this post](https://github.com/timothystewart6/k3s-ansible/discussions/20) on how to troubleshoot common problems

### Testing the playbook using molecule

This playbook includes a [molecule](https://molecule.rtfd.io/)-based test setup.
It is run automatically in CI, but you can also run the tests locally.
This might be helpful for quick feedback in a few cases.
You can find more information about it [here](molecule/README.md).

### Pre-commit Hooks

This repo uses `pre-commit` and `pre-commit-hooks` to lint and fix common style and syntax errors.  Be sure to install python packages and then run `pre-commit install`.  For more information, see [pre-commit](https://pre-commit.com/)

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
