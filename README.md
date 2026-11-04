# Automated build of HA k3s Cluster with `kube-vip` and MetalLB

Forked from <https://github.com/timothystewart6/k3s-ansible>.

This playbook is used to build my personal home cluster deployed on Raspberry PIs.

The current settings deploy

- k3s with embedded etcd on a single master node
- MetalLB

## Setup

Install deps

```shell
uv sync
source .venv/bin/activate
```

Install ansible-galaxy deps

```shell
ansible-galaxy collection install -r ./collections/requirements.yml
```

A new directory based on the `sample` directory within the `inventory` directory has been created, under `inventory/my-cluster`.

Settings are in `ansible.cfg` with adapted inventory path to match.

The `inventory/my-cluster/group_vars/all.yml` has been customised to allow installing embedded ETCD.

## Prepare Cluster

First, copy over ssh key

```shell
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@control01
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@cube02
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@cube03
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@cube04
...
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@cubexx
```

### Prepare nodes with packages

<https://github.com/timothystewart6/k3s-ansible/issues/463>

```shell
ansible-playbook prepare-nodes.yml -i inventory/my-cluster/hosts.yml
```

### Prepare Storage

Later on we will use longhorn, <https://rpi4cluster.com/k3s-storage-setting/#file-system-and-mount>

Here's how disks are currently setup.

#### RPis

##### USBs

```shell
ansible rpi -b -m shell -a "lsblk -f"
```

Verify non-boot disks are sdb, then wipe

```shell
ansible rpi -b -m shell -a "wipefs -a /dev/sdb"
ansible rpi -b -m filesystem -a "fstype=ext4 dev=/dev/sdb"
```

Get UUIDs

```shell
ansible rpi -b -m shell -a "blkid -s UUID -o value /dev/sdb"
```

Mount

```shell
ansible cube03 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=841b96dc-e516-48c7-a152-4111018ef934 fstype=ext4 state=mounted" -b

ansible cube02 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=ee3c972a-c87d-4bf7-8c92-ff6c3093746a fstype=ext4 state=mounted" -b

ansible control01 -m ansible.posix.mount -a "path=/mnt/storage01 src=UUID=b05c1c36-e6c3-4d37-add8-e78a6dd180b3 fstype=ext4 state=mounted" -b
```

##### NVME SSD

Currently, only control01 (192.168.0.177) has SSD.

This I have decided to put on the master. I want to partition it so that we have 64GB for k3s_server to use as data drive and for ETCD.
Then the rest I will make available to longhorn if there's anything that requires super fast disk.

```shell
ansible rpi -b -m shell -a "lsblk -f"
```

```shell
export node=control01
export device=/dev/nvme0n1
```

Wipe & reformat if needed

```shell
ansible $node -b -m shell -a "wipefs -a $device"
ansible $node -b -m filesystem -a "fstype=ext4 dev=$device"
ansible $node -b -m shell -a "lsblk -f"
```

Partition the SSD

```shell
ansible $node -m shell -a "parted $device --script mklabel gpt" -b
ansible $node -m shell -a "parted $device --script mkpart k3s ext4 0% 64GB" -b
ansible $node -m shell -a "parted $device --script mkpart longhorn ext4 64GB 100%" -b
```

Confirm the partitions with

```shell
ansible $node -b -m shell -a "lsblk -f"
```

Then, format the partitions

```shell
ansible $node -b -m shell -a "wipefs -a /dev/nvme0n1p1"
ansible $node -b -m shell -a "wipefs -a /dev/nvme0n1p2"
ansible $node -b -m filesystem -a "fstype=ext4 dev=/dev/nvme0n1p1"
ansible $node -b -m filesystem -a "fstype=ext4 dev=/dev/nvme0n1p2"
ansible $node -b -m shell -a "lsblk -f"
```

Get UUIDs

```shell
ansible $node -b -m shell -a "blkid -s UUID -o value /dev/nvme0n1p1"
ansible $node -b -m shell -a "blkid -s UUID -o value /dev/nvme0n1p2"
```

Mount them. We're going to mount the large partition under `/mnt/storage02`, following what we did for the USBs, meanwhile the k3s partition under `/mnt/k3sdata`. We also mount that to `/var/lib/rancher/k3s` in order to be compatible the default value for the path <https://docs.k3s.io/cli/server#data>

```shell
ansible $node -m ansible.posix.mount -a "path=/mnt/k3sdata src=UUID=17a7eed6-7435-40f2-bbd9-246e3c077ecb fstype=ext4 state=mounted" -b

ansible $node -m ansible.posix.mount -a "path=/var/lib/rancher/k3s src=UUID=17a7eed6-7435-40f2-bbd9-246e3c077ecb fstype=ext4 state=mounted" -b

ansible $node -m ansible.posix.mount -a "path=/mnt/storage02 src=UUID=13a73c3f-3552-45b7-beed-33d469984ed7 fstype=ext4 state=mounted" -b
```

We need to also point --data-dir at this new k3sdata partition.

#### x86

I added some old x86 linux machines in. These have HDD/SSD and have a 32GB boot partition. The rest we want to use for longhorn storage. The partition for data is then `/dev/sda2`

```shell
ansible x86 -b -m shell -a "lsblk -f"
```

Get UUIDs

```shell
ansible x86 -b -m shell -a "blkid -s UUID -o value /dev/sda2"
```

Mount

```shell
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

```shell
ansible-playbook label-nodes.yml -i inventory/my-cluster/hosts.yml
```

### Add New Nodes

If the task is to simply add new nodes we can use the [limit option](https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_patterns.html#patterns-and-ad-hoc-commands). E.g.

```shell
ansible-playbook site.yml -i inventory/my-cluster/hosts.yml --limit "host1,host2"
```

## 🔥 Remove k3s cluster

```bash
ansible-playbook reset.yml -i inventory/my-cluster/hosts.yml
```

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


## ⚙️ Kube Config

```shell
mkdir -p ~/.kube
scp root@control01:/etc/rancher/k3s/k3s.yaml ~/.kube/config 
```

Then, edit the file to change `server: https://127.0.0.1:6443` to match your master IP: `server: https://192.168.0.177:6443`