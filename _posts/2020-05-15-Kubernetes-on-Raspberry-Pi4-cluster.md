---
title: "Kubernetes on Raspberry Pi4 cluster (Ubuntu arm64)"
categories:
  - RaspberryPi4
  - Kubernetes
tags:
  - k8s
  - raspberry
  - arm64
---
In this guide I wanted to share my experience on building a complete **Kubernetes Cluster** using Ubuntu 20.04 64 bit on Raspberry Pi 4. I'd like to use it for 64-bit ARM development. Various 64-bit Linux distributions have been shown running on the Raspberry Pi 4, but I wanted to stick to Ubuntu which supports 4 GB ram.

**Hardware**:

* 4 x Raspberry Pi 4 with 4 GB ram
* 4 x 32GB eMMC memory cards
* 1 x Anker PowerPort 6 Port USB Charging Hub
* 4 x Anker PowerLine USB C to USB 3.0 Cable
* 1 x Pi Rack Case for Raspberry Pi 4 Model B
* 1 x NETGEAR GS305 5-Port Gigabit Ethernet Network Switch
* 5 x Cat 6 Ethernet Cable 1 ft

**Planning and installation**

Below I did a brief IP Plan for my home network:

```
Network: 192.168.254.0/26

Gateway: 192.168.254.1
DNS: 192.168.254.1 (running dnsmasq on OPNSense router)

Kubernetes Nodes:
- 192.168.254.10 master
- 192.168.254.11 worker01
- 192.168.254.12 worker02
- 192.168.254.13 worker03
```

At first, I downloaded the [official image](http://cdimage.ubuntu.com/ubuntu/releases/20.04/release/ubuntu-20.04-preinstalled-server-arm64+raspi.img.xz) from Ubuntu's site for Raspberry Pi 4 and wrote it in the SD cards

```
git clone https://github.com/enr0s/k8s-arm64.git
cd flash-micro-sd
wget http://cdimage.ubuntu.com/ubuntu/releases/20.04/release/ubuntu-20.04-preinstalled-server-arm64+raspi.img.xz
xz -d ubuntu-20.04-preinstalled-server-arm64+raspi.img.xz
echo "Writing Raspbian Lite image to SD card"
time dd if=ubuntu-20.04-preinstalled-server-arm64+raspi.img of=/dev/mmcblk0 bs=1M
```

Then I configured the card for headless SSH access

```
mkdir /mnt/rpi/boot
mkdir /mnt/rpi/root
mkdir -p /mnt/rpi/root/home/ubuntu/.ssh/
cat ~/.ssh/id_rsa.pub > /mnt/rpi/root/home/ubuntu/.ssh/authorized_keys
chown -R 1000:1000 /mnt/rpi/root/home/ubuntu
sed -i 's/PasswordAuthentication\ yes/PasswordAuthentication\ no/g' /mnt/rpi/root/etc/ssh/sshd_config
```
Preserve hostname changing cloud init configuration

```
sed -i 's/preserve_hostname.*/preserve_hostname: true/g' /mnt/rpi/root/etc/cloud/cloud.cfg
```

Set the hostname

```
sed -i 's/ubuntu/master.domain.local/g' /mnt/rpi/root/etc/hostname
```

and configured the network using a template:

```
cat <<EOF | tee 99_config.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses:
        - 192.168.254.CHANGEME/26
      gateway4: 192.168.254.1
      nameservers:
          search: [domain.local]
          addresses: [192.168.254.1]
EOF

sed s/CHANGEME/10/g 99_config.yaml> /mnt/rpi/root/etc/netplan/99_config.yaml
echo "network: {config: disabled}" > /mnt/rpi/root/etc/cloud/cloud.cfg.d/99-custom-networking.cfg
```

Remember to umount the SD card partitions before removing.
```
umount /mnt/rpi/boot
umount /mnt/rpi/root
```

Repeat the above, changing the hostname and IP adresses, for each Raspberry Pi.

Based on [alexellis](https://github.com/alexellis)'s [preparation script](https://raw.githubusercontent.com/teamserverless/k8s-on-raspbian/master/script/prep.sh), I created a new version applying my needs.

Then update and run the following commands on each of them:

Update /etc/hosts
```
export DOMAIN_NAME=domain.local
cat <<EOF | sudo tee -a /etc/hosts
192.168.254.10  master.${DOMAIN_NAME} master
192.168.254.11  worker01.${DOMAIN_NAME} worker01
192.168.254.12  worker02.${DOMAIN_NAME} worker02
192.168.254.13  worker03.${DOMAIN_NAME} worker03
EOF
```

Ensure iptables tooling does not use the nftables backend
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

Update the system and install docker/k8s requirements
```
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
```

Install docker
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=arm64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod ubuntu -aG docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

Disable swap
```
sudo swapoff -a
sudo echo "vm.swappiness=0" | sudo tee -a /etc/sysctl.conf
```

Install Kubernetes
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Enabling CGroup Memory
```
sudo cp  /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.orig
orig="$(head -n1 /boot/firmware/cmdline.txt ) ipv6.disable=1 cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1"
echo $orig | sudo tee /boot/firmware/cmdline.txt
```

You can apply all steps before running
```
curl -sL https://raw.githubusercontent.com/enr0s/k8s-arm64/master/scripts/bootstrap.sh | sudo sh
```
