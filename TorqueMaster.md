---
title: "WWtorque CentOS7 from Vince"
output: html_document
---

Walkthrough from Vince at McGill about installing Torque on the head node on his cluster.
**Dan's comments in bold**

***
Objective

To describe how to provision Torque a master node using the Warewulf master for the Hydra Cluster. Assumes working setup of WWMaster. To test this run wwinit ALL.
NOTE: Most changes to the chroot environment require rebuild of VNFS and reboot of provisioned node:
```
$ wwvnfs --chroot /var/chroots/hydratm-centos-7
```
Reboot provisioned node.

***
Initial setup

Setup chroot:
```
$ wwmkchroot centos-7 /var/chroots/hydratm-centos-7
```

Update packages:
```
$ rpm --root /var/chroots/hydratm-centos-7 -ivh /root/rpm/epel-release-7-5.noarch.rpm 
$ yum --tolerant --installroot /var/chroots/hydratm-centos-7 update
```

NTP
```
$ yum --tolerant --installroot /var/chroots/hydratm-centos-7 install ntp 
$ chroot /var/chroots/hydratm-centos-7
# systemctl enable ntpd
# exit
```

SSH key
```
$ chmod 700 /var/chroots/hydratm-centos-7/root/.ssh
$ chmod 400 ~/.ssh/authorized_keys
$ cp ~/.ssh/authorized_keys /var/chroots/hydratm-centos-7/root/.ssh/
```

Mount SAN filesystems
```
$ vi /var/chroots/hydratm-centos-7/etc/fstab
192.168.13.10:/mnt/KLEINMAN_BACKUP /mnt/KLEINMAN_BACKUP nfs defaults,async,_netdev 0 0 
192.168.13.10:/mnt/GREENWOOD_BACKUP /mnt/GREENWOOD_BACKUP nfs defaults,async,_netdev 0 0 
192.168.13.10:/mnt/KLEINMAN_SCRATCH /mnt/KLEINMAN_SCRATCH nfs defaults,async,_netdev 0 0 
192.168.13.10:/mnt/GREENWOOD_SCRATCH /mnt/GREENWOOD_SCRATCH nfs defaults,async,_netdev 0 0

$ mkdir /var/chroots/hydratm-centos-7/mnt/KLEINMAN_BACKUP /var/chroots/hydratm- centos-7/mnt/GREENWOOD_BACKUP /var/chroots/hydratm-centos-7/mnt/KLEINMAN_SCRATCH /var/chroots/hydratm-centos-7/mnt/GREENWOOD_SCRATCH

```

Prepare VNFS
```
$ wwvnfs --chroot /var/chroots/hydratm-centos-7
```

Setup WW environment
```
$ vi ~/wwscripts/wwconfig-torquemaster.sh
wwsh -y node new ${NODE} --netdev=eth0 --hwaddr=${GE_HWADDR} --ipaddr=${GE_IPADDR} -- groups=HYDRATM --domain=ldi.lan --netmask 255.255.255.0
wwsh -y provision set --lookup groups HYDRATM --vnfs=hydratm-centos-7 -- bootstrap=3.10.0-229.14.1.el7.x86_64
wwsh -y provision set --fileadd passwd,group,shadow ${NODE}
wwsh -y node set ${NODE} --netdev=eth3 --ipaddr=${XE_IPADDR} --netmask=255.255.255.0 --hwaddr=${XE_HWADDR}
wwsh -y provision set --fileadd=ifcfg-eth3.ww ${NODE} wwsh -y provision set --fileadd=resolv.conf.ww ${NODE} wwsh -y provision set --fileadd=network.ww ${NODE} wwsh -y file sync \* ${NODE}
systemctl restart dhcpd

```















