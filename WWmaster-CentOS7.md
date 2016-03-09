---
title: "WWmaster CentOS7 from Vince"
output: html_document
---


**Head node Marge, worker nodes Lisa[01-9]** 

Assumes:
1. Internet connectivity.
2. SELinux and firewall disabled.
3. EPEL repository enabled.

```
hostname marge
```

### CentOS accounts
user name | Pass
----------|-----------
root      | 
devans    | 


## Configure network interfaces.
### Head node configuration
* hostname is marge
* network interfaces
  + eth0 connected to internet, IP assigned via DHCP
  + eth1 connected to GigE switch to workers
  

### Hostnames and IP addresses
host name   | IP address
------------|-----------
marge       | 10.2.0.128 / LAN internal 172.10.10.2
ringo/NFS| 10.2.0.129 / LAN internal 172.10.10.3
n0001       | 
n0002       | 

After manually assigning IP to eth1, needed to start it with:
/etc/sysconfig/network-scripts/ifcfg-eth1
ONBOOT="yes"

---
Hydra-FileServer-CentOS7 write-up from Vince
---

##OS installation
Installed CentOS 7 from USB key.
To do later:
Restrict SSH login to root and devans:
```
$ vi /etc/ssh/sshd_config
# add following
AllowUsers root vforget

$ systemctl restart sshd
```


```
Filesystem                                     Size  Used Avail Use% Mounted on
/dev/mapper/centos-root                        222G  1.1G  221G   1% /
devtmpfs                                       7.8G     0  7.8G   0% /dev
tmpfs                                          7.8G     0  7.8G   0% /dev/shm
tmpfs                                          7.8G  8.9M  7.8G   1% /run
tmpfs                                          7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/sda1                                      4.7G  170M  4.5G   4% /boot
/dev/mapper/centos-home                         47G   33M   47G   1% /home
```

Disable SELinux

```
$ vi /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these two values:
#     targeted - Targeted processes are protected,
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

```

Reboot



Disable firewall

```
$ systemctl disable firewalld
$ systemctl stop firewalld
```

Update packages

```
yum update
```

##NTP

```
$ yum install ntp
$ systemctl enable ntpd  #
$ systemctl start ntpd
$ timedatectl set-ntp yes
```

##Networking

Vince had lots of hydra-specific details

Add hostname to /etc/hosts
```
$ cat /etc/hosts
192.168.13.10 D1P-HYDRAFS01 D1P-HYDRAFS01.ldi.lan
127.0.0.1   D1P-HYDRAFS01 D1P-HYDRAFS01.ldi.lan localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

##NFS

Install:
```
$ wget http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
$ rpm -Uvh epel-release-7*.rpm
$ yum update
$ yum install bonnie++
$ yum install yum-utils
$ yum install impitool
$ yum install emacs
$ yum install iperf

$ yum install nfs-utils nfs-utils-lib nfswatch
```
Run on startup:
```
$ systemctl enable nfs-server
$ systemctl start nfs-server
```

According to following advice, no additional services are required to run NFSv4:
http://serverfault.com/questions/530908/nfsv4-and-rpcbind
However, downstream Warewulf config needs rpcbind:

```
$ systemctl enable rpcbind
$ systemctl start rpcbind
```

Add following to /etc/exports:
```
/mnt/KLEINMAN_BACKUP 192.168.13.10/24(rw,async,no_subtree_check,no_root_squash)
/mnt/KLEINMAN_SCRATCH 192.168.13.10/24(rw,async,no_subtree_check,no_root_squash)
/mnt/GREENWOOD_BACKUP 192.168.13.10/24(rw,async,no_subtree_check,no_root_squash)
/mnt/GREENWOOD_SCRATCH 192.168.13.10/24(rw,async,no_subtree_check,no_root_squash)
```

Load/Reload exported file systems:
```
$ exportfs -rv
```

On client add to /etc/fstab:
```
192.168.13.10:/mnt/KLEINMAN_BACKUP /mnt/KLEINMAN_BACKUP nfs defaults,async 0 0
192.168.13.10:/mnt/GREENWOOD_BACKUP /mnt/GREENWOOD_BACKUP nfs defaults,async 0 0
192.168.13.10:/mnt/KLEINMAN_SCRATCH /mnt/KLEINMAN_SCRATCH nfs defaults,async 0 0
192.168.13.10:/mnt/GREENWOOD_SCRATCH /mnt/GREENWOOD_SCRATCH nfs defaults,async 0 0
```

Mount file systems:
```
$ mount -a
```

##Optimization

```
$ /etc/sysconfig/nfs
RPCNFSDCOUNT=16
```

##Install a few dependencies
```

```


##Install

Warewulf dependencies

**pigz? It is a parallel gzip. Not needed for warewulf, but nice to have.**
**Will perl-DBD-MySQL talk with mariadb? Why use mariaDB instead of MySQL? Probably not a big deal.**
**libselinux-devel aren't we diabling SELinux? **
**Crypt::HSXKPasswd? Looks like it's a specific function in the Crypt Perl module. What does it do?**

```
$ yum group install 'Development tools'
$ yum install tcpdump tftp tftp-server pigz dhcp nfs-utils nfs-utils-lib ntp httpd 
$ yum install perl-DBD-MySQL mariadb mariadb-server perl-Term-ReadLine-Gnu mod_perl perl-CGI
$ yum install libselinux-devel libacl-devel libattr-devel
$ yum install cpan
$ cpan Module::Build
$ cpan YAML
$ cpan DateTime
$ cpan Crypt::HSXKPasswd

```

Reboot

Start services

```
$ systemctl enable httpd
$ systemctl start httpd
$ systemctl enable mariadb
$ systemctl start mariadb
```

Reboot, and ensure services are in proper state after reboot:

```
$ systemctl status httpd # on 
$ systemctl status mariadb # on
$ systemctl status firewalld  # off
```

Install Warewulf

**build_it function creates RPM build environment. First argument is where source files are located (e.g. /root/warewulf/common/), second argument is where RPMs will be built /root/rpmbuild/. Makes everything in first argument dir, then copies all warewulf files to /root/rpmbuild/SOURCES, but I don't understand the very last command rpmbuild -bb ./.spec. The command is still in /root/warewulf/common, is rpmbuild run from there? Might be nice to run the function on the first warewulf component and see where files are created. SPEC file directs rpmbuild kind of like a makefile for make.**

```
# Build Warewulf from SVN
$ BUILD_DIR=/root/rpmbuild
$ WW_DIR=/root/warewulf
$ RPM_DIR=/root/rpmbuild/RPMS/
$ function build_it { cd $1 && ./autogen.sh && make dist-gzip && make distcheck && cp -fa warewulf-*.tar.gz $2/SOURCES/ && rpmbuild -bb ./*.spec; }
```
**Regarding h2ph command, why convert all C files to perl files in /usr/include? And where are the converted files placed? Don't worry about it.**
**We could probably verify that perl is in /usr/local/lib64/perl5, and if not symlink it there, but let's not worry about that now**

```
$ cd /usr/include
$ h2ph -al * sys/*
# Warning from h2ph: Destination directory /usr/local/lib64/perl5 doesn't exist or isn't a directory
$ mkdir -p $BUILD_DIR/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
$ mkdir /root/warewulf
```
**downlaod to /root/warewulf. Why is echo p piped to svn? Who cares.**
```
$ echo p | svn co https://warewulf.lbl.gov/svn/trunk/ /root/warewulf 
$ build_it $WW_DIR/common $BUILD_DIR
$ yum install -y $RPM_DIR/noarch/warewulf-common-*.rpm
$ build_it $WW_DIR/provision $BUILD_DIR
$ yum install -y $RPM_DIR/x86_64/warewulf-provision-*.rpm
$ build_it $WW_DIR/cluster $BUILD_DIR
$ yum install -y $RPM_DIR/x86_64/warewulf-cluster-*.rpm
## remove debian.tmpl from Makefil* in vnfs/libexec/wwmkchroot
$ sed -i 's+debian.tmpl+ +g' $WW_DIR/vnfs/libexec/wwmkchroot/Makefil* 
$ build_it $WW_DIR/vnfs $BUILD_DIR
$ yum install -y $RPM_DIR/noarch/warewulf-vnfs-*.rpm
```
**Symlinks and dependencies needed to build monitor component?**
**What do the different warewulf components do? I'll RTFM**
```
# monitor
$ ln -s /usr/lib64/libjson.so.0 /usr/lib64/libjson.so
$ ln -s /usr/lib64/libjson-c.so.2 /usr/lib64/libjson-c.so 
$ ln -s /usr/lib64/libsqlite3.so.0 /usr/lib64/libsqlite3.so 
$ yum -y install json-c-devel json-devel sqlite-devel
$ build_it $WW_DIR/monitor $BUILD_DIR
$ yum -y install $RPM_DIR/x86_64/warewulf-monitor-0*.rpm
```
**Line added by Vince below**
```
$ yum -y install $RPM_DIR/x86_64/warewulf-monitor-cl*.rpm # Added by Vince Forgetta
$ yum -y install $RPM_DIR/x86_64/warewulf-monitor-node*.rpm
# build_it $WW_DIR/ipmi $BUILD_DIR
# yum -y install $RPM_DIR/x86_64/warewulf-ipmi*.rpm
$ rpm -qa | grep warewulf 
#output of grep command should be below. Checking that all RPMs are built.

warewulf-cluster-3.6.99-0.r1933.el7.centos.x86_64 
warewulf-cluster-node-3.6.99-0.r1933.el7.centos.x86_64 
warewulf-common-3.6.99-0.r1936.el7.centos.noarch
warewulf-monitor-0.0.1-0.r1939.el7.centos.x86_64 
warewulf-monitor-cli-0.0.1-0.r1939.el7.centos.x86_64
warewulf-provision-3.6.99-0.r1945.el7.centos.x86_64 
warewulf-provision-gpl_sources-3.6.99-0.r1945.el7.centos.x86_64 
warewulf-provision-server-3.6.99-0.r1945.el7.centos.x86_64 
warewulf-vnfs-3.6.99-0.r1947M.el7.centos.noarch 

```

Configure

Set NIC interface used to provision OS image

**eno1? Will we use eth1? Does Vince's use of eno1 indicate he is using a different type of NIC?**

```
$ vi /etc/warewulf/provision.conf
# What is the default network device that the master will use to communicate with the nodes?
network device = eno1

```

Enable tftp

```
$ vi /etc/xinetd.d/tftp
# default: off
# description: The tftp server serves files using the trivial file transfer \
# protocol. The tftp protocol is often used to boot diskless \
# workstations, download configuration files to network-aware printers, \
# and to start the installation process for some operating systems. 
service tftp
{
  socket_type     = dgram
  protocol        = udp
  wait            = yes
  user            = root
  server          = /usr/sbin/in.tftpd
  server_args     = -s /var/lib/tftpboot
  disable         = no
  per_source      = 11
  cps             = 100 2
  flags           = IPv4
}

```

Restart xinetd

```
$ systemctl restart xinetd
```

Setup MariaDB
**And why not MySQL? I don't see where MariaDB is specified instead of MySQL. How does warewulf know whether to use MariaDB or MySQL?**
**logged in as root, so ~ = /root/**
```
$ vi ~/.my.cnf
[client]
user = root
password =
$ chmod 0600 ~/.my.cnf

$ vi /etc/warewulf/database.conf
user = root
password =

$ vi /etc/warewulf/database-root.conf
user = root
password =
```

Setup VNFS
**We should talk about this. Should we remove some dirs from the chroot to be nfs mounted to save RAM in the chroot? Alaric would say no, since we bought lots of RAM, but more RAM is always good.**
```
$ vi /etc/warewulf/vnfs.conf
# uncomment hybridpath =
```

Initial setup to provision nodes
```
$ systemctl enable dhcpd
$ wwinit ALL
#ensure relevant tests pass
$ wwbootstrap `uname -r`
```

Install pdsh

```
$ yum install libgenders

$ wget ftp://rpmfind.net/linux/sourceforge/s/sy/sys-integrity-mgmt-
platform/yum/el/7/ext/x86_64/pdsh-2.29-1el7.x86_64.rpm

$ wget ftp://fr2.rpmfind.net/linux/sourceforge/s/sy/sys-integrity-mgmt- platform/yum/el/7/ext/x86_64/pdsh-rcmd-ssh-2.29-1el7.x86_64.rpm

yum -y install pdsh-rcmd-ssh-2.29-1el7.x86_64.rpm pdsh-2.29-1el7.x86_64.rpm

```

SSH key
**What is the last line doing?**
```
$ ssh-keygen (use empty passphrases)
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 
$ chmod 400 ~/.ssh/authorized_keys
$ ssh-keygen -f ~/.ssh/identity
```

***
Provision Configuration

**Not sure I understand pdsh command. Run help on wwgetfiles to see what it does.**
Sync users, groups, and passwords
```
$ wwsh -y file import /etc/passwd
$ wwsh -y file import /etc/group
$ wwsh -y file import /etc/shadow
$ wwsh -y provision set --fileadd passwd,group,shadow \* 
$ wwsh file sync \*
# To sync files right away
$ pdsh -w D1P-HYDRARS01 "SLEEPTIME=0 /warewulf/bin/wwgetfiles"
```

Gigabit network card

```
$ vi ~/wwtemplates/ifcfg-eth3.ww
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
USERCTL=no
NM_CONTROLLED=no
PREFIX=24
HWADDR=%{NETDEVS::ETH3::HWADDR}
IPADDR=%{NETDEVS::ETH3::IPADDR}
NAME=eth3
DEVICE=eth3
DEFROUTE=yes
PEERDNS=no
DNS1=172.21.8.112
DNS2=172.21.8.111
DOMAIN=ldi.lan
```

Import and sync file:
**Will need to change second line**
```
$ wwsh file import ~/wwtemplates/ifcfg-eth3.ww --path=/etc/sysconfig/network-scripts/ifcfg-eth3 --name=ifcfg-eth3.ww
$ wwsh -y node set D1P-HYDRARS02 --netdev=eth3 --ipaddr=192.168.13.21 --netmask=255.255.255.0 --hwaddr=0c:c4:7a:1f:8b:9b
$ wwsh provision set --fileadd=ifcfg-eth3.ww D1P-HYDRARS[01-04] 
$ wwsh file sync
$ systemctl restart dhcpd
```

Gateway
```
# vi ~/wwtemplates/network.ww
GATEWAY=172.21.13.1
GATEWAYDEV=eth0
```

Setup domain name server
**Change second to last line to LISA[01-06], use the right IPs**
```
$ vi ~/wwtemplates/resolv.conf.ww
search ldi.lan
nameserver 172.21.8.112
nameserver 172.21.8.111
$ wwsh -y file import ~/wwtemplates/resolv.conf.ww --path=/etc/resolv.conf -- name=resolv.conf.ww
$ wwsh -y provision set --fileadd=resolv.conf.ww D1P-HYDRARS[01-04]
$ wwsh file sync
```

Import and sync file:

```
$ wwsh file import ~/wwtemplates/network.ww --path=/etc/sysconfig/network -- name=network.ww
$ wwsh –y provision set --fileadd=network.ww D1P-HYDRARS[01-04]
$ wwsh file sync \*
$ systemctl restart dhcpd
```

Additional packages
**lsof lists all open files. Could be a good way to monitor activity if you build a shell script to parse through it.**
```
$ yum install emacs
$ yum install lsof

```






