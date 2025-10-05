
# Housing Project

## Summary

It's been few months since I registered my own ASN (*AS209238*) but I never had the occasion to use it, my ISP doesn't support BGP and OVH support [only IPv4 for their bgp service](https://help.ovhcloud.com/csm/fr-network-bgp-service-configuration?id=kb_article_view&sysparm_article=KB0066887). So I decided to host my own server in a community datacenter that accepts my ASN.


## Hardware

The server is a [MS-01](https://minisforumpc.fr/products/minisforum-ms-01-work-station?variant=49292163187005) with the following specs :
- CPU : Intel Core i9-12900H
- RAM : 96 Go DDR4
- Stockage : 3x2 To NVMe SSD

This server is pretty powerful for a home server and has a BMC (IPMI) interface which is a must-have for remote management, I'll probably never use it since it is quite limited compared to the NanoKVM.

Talk about the NanoKVM, I bought both a NanoKVM **and** a JetKVM (because I wasn't sure which one would work best with this server). Even if the JetKVM is more powerful, has more features and a better UI, I ended up using the NanoKVM because it can set a static IP address (it is technically possible with the [JetKVM but requires to edit a specific file that could be reset on firmware update](https://github.com/jetkvm/kvm/issues/37#issuecomment-2622197252)). 


## Install OS

I chose Fedora CoreOS as OS, I like the idea of an immutable OS and :
- I want to be able to rollback to a previous version if something goes wrong
- I like the usage of [Bootc](https://github.com/bootc-dev/bootc) / [OSTree Containers](https://www.opensourcerers.org/2023/06/16/using-ostree-native-containers-as-node-base-images/) for updates and GitOps deployment
- I want automatic updates with [rpm-ostree](https://docs.fedoraproject.org/en-US/iot/applying-updates-UG/)

NixOS was also a good candidate but I have more experience and confidence with Fedora CoreOS as a server OS.

---

In order to bootstrap the OS, I used [Butane](https://coreos.github.io/butane/) to generate an Ignition config file from a simple YAML file (which is avaiable in this repo as `butane/config.bu`). The butane config file is pretty simple, it creates a user `core` with sudo rights and ssh key authentication. The other part is the setup of LUKS + TPM2 for the root partition.

```bash
# On my workstation
docker run --rm -i quay.io/coreos/butane:release < config.bu > config.ign
python3 -m http.server 8080
# On the server in live-USB with CoreOS
sudo coreos-installer install --ignition-url http://192.168.1.124:8080/config.ign /dev/nvme0n1 --insecure-ignition
```

## Data disk setup

Since ZFS is not yet supported on Fedora CoreOS, I decided to use LVM with RAID1 + LUKS encryption for the data disks. The disks are two NVMe drives : `/dev/nvme0n1` and `/dev/nvme1n1`.

```bash
sudo pvcreate /dev/nvme0n1 /dev/nvme1n1
sudo vgcreate vg_data /dev/nvme0n1 /dev/nvme1n1
sudo lvcreate -l 100%FREE -m 1 --type raid1 -n lv_raid1 vg_data
sudo cryptsetup luksFormat --type luks2 /dev/vg_data/lv_raid1
sudo cryptsetup luksOpen /dev/vg_data/lv_raid1 crypt_data
sudo mkfs.xfs -L Data /dev/mapper/crypt_data

sudo mkdir -p /etc/luks/
echo "{{ MY_LUKS_KEY }}" | sudo tee /etc/luks/crypt_data.key > /dev/null
sudo chmod 600 /etc/luks/crypt_data.key
sudo cryptsetup luksClose crypt_data
sudo cryptsetup luksAddKey /dev/vg_data/lv_raid1 /etc/luks/crypt_data.key # In case you want to add a new key in the future

LUKS_UUID=$(sudo blkid -s UUID -o value /dev/vg_data/lv_raid1)
echo "crypt_data UUID=$LUKS_UUID /etc/luks/crypt_data.key luks" | sudo tee -a /etc/crypttab
```

At the start of the next boot, the system will automatically unlock the encrypted volume using the key file stored in `/etc/luks/crypt_data.key`. Then I created a systemd mount unit to mount the encrypted volume at boot. The mount point is `/var/mnt/data` (because of Fedora CoreOS immutability). 

```toml
# /etc/systemd/system/var-mnt-data.mount
[Unit]
Description=Mounts the encrypted RAID data volume
Requires=cryptsetup.target
After=cryptsetup.target

[Mount]
What=/dev/mapper/crypt_data
Where=/var/mnt/data
Type=xfs
Options=defaults,x-systemd.device-timeout=0

[Install]
WantedBy=multi-user.target
```
