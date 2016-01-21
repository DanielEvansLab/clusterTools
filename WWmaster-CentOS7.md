---
title: "WWmaster CentOS7 from Vince"
output: html_document
---

Walkthrough from Vince at McGill about setting up WWmaster on his cluster.
**Dan's comments in bold**

***
Objective

To describe deployment of Warewulf master node for the Hydra Cluster.
The Warewulf Master is deployed on same server and the HYDRA File Server on host, ~~D1P-HYDRAFS01~~. **Change to our server name, Head node Marge, worker nodes Lisa[01-9]** This deployment assumes that the server is configured as per document Hydra-FileServer-CentOS7. **We don't have that doc from Vince**

Assumes:
1. Internet connectivity.
2. SELinux and firewall disabled.
3. EPEL repository enabled.

***
Install

Warewulf dependencies

**pigz? Will perl-DBD-MySQL talk with mariadb? Why use mariaDB instead of MySQL?**
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

**build_it function creates RPM build environment. First argument is where source files are located (e.g. /root/warewulf/common/), second argument is where RPMs will be built /root/rpmbuild/. Makes everything in first argument dir, then copies all warewulf files to /root/rpmbuild/SOURCES, but I don't understand the very last command rpmbuild -bb ./.spec. The command is still in /root/warewulf/common, is rpmbuild run from there? Might be nice to run the function on the first warewulf component and see where files are created.**

```
# Build Warewulf from SVN
$ BUILD_DIR=/root/rpmbuild
$ WW_DIR=/root/warewulf
$ RPM_DIR=/root/rpmbuild/RPMS/
$ function build_it { cd $1 && ./autogen.sh && make dist-gzip && make distcheck && cp -fa warewulf-*.tar.gz $2/SOURCES/ && rpmbuild -bb ./*.spec; }
```
**Regarding h2ph command, why convert all C files to perl files in /usr/include? And where are the converted files placed? **
**We should probably verify that perl is in /usr/local/lib64/perl5, and if not symlink it there**

```
$ cd /usr/include
$ h2ph -al * sys/*
# Warning from h2ph: Destination directory /usr/local/lib64/perl5 doesn't exist or isn't a directory
$ mkdir -p $BUILD_DIR/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
$ mkdir /root/warewulf
```
**downlaod to /root/warewulf. Why is echo p piped to svn? **
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

**Not sure I understand pdsh command**
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






