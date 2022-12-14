# Smol-Metal

Notes for configuring Debian Bookworm nodes for use as VPS hosts.
The steps below setup the system to be further controlled by ansible. Eventually most of this will move into a cloid-init or pre-seed files.

## As Sudo:

1. Fix apt sources

```bash
cat << EOF > /etc/apt/sources.list
deb http://deb.debian.org/debian bookworm main contrib non-free
deb-src http://deb.debian.org/debian bookworm main contrib non-free

deb http://deb.debian.org/debian-security/ bookworm-security main contrib non-free
deb-src http://deb.debian.org/debian-security/ bookworm-security main contrib non-free

deb http://deb.debian.org/debian bookworm-updates main contrib non-free
deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free
EOF
```

2. install deps

```bash
# Package Choice Justifications
# sudo - a user with passwordless sudo for automations
# ssh-import-id - ssh key imports
# wireguard - setup vpn (get keys from bitwarden)
# curl - for onboardme
# nvidia-driver firmware-misc-nonfree linux-headers-amd64 are for GPU

apt-get update && apt-get install -y wireguard \
  ssh-import-id \
  sudo \
  curl \
  nvidia-driver \
  firmware-misc-nonfree \
  linux-headers-amd64 \
  docker.io \
  netplan.io \
  git-extras \
  rsyslog
```

3. add passwordless sudo
```bash
echo "friend ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

4. bridge the network adapter

https://wiki.debian.org/NetworkInterfaceNames

/etc/udev/rules.d/70-persistent-net.rules
```bash

```

```bash
# /etc/netplan/99-bridge.yaml
network:
  bridges:
    br0:
      dhcp4: no
      dhcp6: no
      interfaces: [enp4s0]
      addresses: [192.168.50.101/24]
      routes:
        - to: default
          via: 192.168.50.1
      mtu: 1500
      nameservers:
        addresses: [192.168.50.50]
      parameters:
        stp: true
        forward-delay: 4
  ethernets:
    enp4s0:
      dhcp4: no
      dhcp6: no
  renderer: networkd
  version: 2

sudo netplan --debug generate
sudo netplan --debug apply
```

5. Set grub to enable iommu

```bash
# /etc/default/grub
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet preempt=voluntary iommu=pt amd_iommu=on intel_iommu=on"
GRUB_CMDLINE_LINUX=""

sudo update-grub
sudo reboot now
```

6. Enable GPU passthrough (Optional) 

See: https://github.com/small-hack/smol-gpu-passthrough

```bash
wget https://raw.githubusercontent.com/small-hack/smol-gpu-passthrough/main/setup.sh

bash setup.sh full_run NVIDIA
sudo reboot now
```

## As User:

```bash
ssh-import-id-gh cloudymax
```

```bash
sudo nano /etc/wireguard/wg0.conf

sudo systemctl enable wg-quick@wg0

sudo systemctl restart wg-quick@wg0
```

```bash
sudo wget -O /etc/ssh/sshd_config https://raw.githubusercontent.com/cloudymax/linux_notes/main/sshd_config

sudo systemctl reload sshd
```

## How to run the ansible playbooks

- populate ansible/inventory

- Run the playbook

```bash
# Create a directory for a volume to store settings and a sqlite database
mkdir -p ~/.ara/server

# Start an API server with docker from the image on DockerHub:
docker run --name api-server --detach --tty \
  --volume ~/.ara/server:/opt/ara -p 8000:8000 \
  docker.io/recordsansible/ara-api:latest

# build the runner
docker build -t ansible-runner .

# Run a playbook
docker run --platform linux/amd64 -it \
  -v $(pwd)/ansible:/ansible \
  docker run -it -v $(pwd)/ansible:/ansible \
  ansible-runner ansible-playbook playbooks/install_onboardme.yaml \
  -i sample-inventory.yaml
```

## Playbooks

1. main-playbook.yaml
  - setus up users, ssh keys, basic apt packagaes, apt-update/upgrade prometheus node-exporter

2. install_brew.yaml
  - clones the brew repo, installs it and sets the env vars correctly

3. install_onboardme.yaml
  - installs onboardme
