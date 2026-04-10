# Home Lab

## core - i5 Intel Nuc - Ubuntu 24.04.4 LTS
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0  3.6T  0 disk
└─sda1        8:1    0  3.6T  0 part /srv
nvme0n1     259:0    0  1.8T  0 disk
├─nvme0n1p1 259:1    0    1G  0 part /boot/efi
└─nvme0n1p2 259:2    0  1.8T  0 part /
```


### Platform Bootstrap

#### Folder Structure
```sudo mkdir -p /srv/homelab/{compose,control,data,logs,backups}
sudo chown -R $USER:$USER /srv/homelab

mkdir -p /srv/homelab/control/{caddy,openwebui,searxng,redis}
mkdir -p /srv/homelab/data/{postgres,qdrant,artifacts}
```

#### Remove snapd garbage
```
snap list
sudo systemctl stop snapd
sudo systemctl disable snapd
sudo systemctl mask snapd
sudo apt purge snapd -y
sudo rm -rf /snap
sudo rm -rf /var/snap
sudo rm -rf /var/lib/snapd
sudo rm -rf ~/snap
sudo apt-mark hold snapd
which snap
systemctl list-units | grep snap
sudo systemctl disable lxd-installer.socket
sudo systemctl stop lxd-installer.socket
sudo apt purge lxd-installer -y
sudo rm -f /etc/systemd/system/snapd.mounts-pre.target
sudo systemctl daemon-reload
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl reset-failed
sudo apt autoremove -y
sudo shutdown -r now
```

#### System Update
```
sudo apt update
sudo apt upgrade -y
sudo shutdown -r now
```

#### Docker Install
```
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
sudo shutdown -r now
docker run hello-world
```

### Services

#### Open WebUI
```
openssl rand -hex 32
```
