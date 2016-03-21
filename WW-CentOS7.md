  ---
title: "WWmaster CentOS7 installation walkthrough"
output: html_document
---

#This walkthrough is a combination of RedBarn HPC notes and Vince's notes

Download, burn, and boot from Minimal Install CD for CentOS 7.  Accept all defaults, do not select options. Do enable networking.

RB-HPC suggests to disable or remove Network Manager on the head node.
We might do this later, but not this time
```
#service NetworkManager stop
#chkconfig NetworkManager off
```

##Host names
Head node = marge
**Add backup head node later?**
Worker nodes = lisa[01-09]
NFS file server = wiggum

Set hostname on head node. We will specify worker node hostnames in /etc/hosts
```
hostname marge
```

## CentOS accounts on marge

user name | Pass
----------|-----------
root      | 
devans    | 

MariaDB root pass same as OS root pass

## Configure network interfaces.
### Head node configuration
* 2 NICs on marge
  + eth0 connected to internet
  + eth1 connected to GigE switch to workers

### Hostnames and IP addresses
host name       | IP address
----------------|-----------
marge           | eth0: 10.2.0.128 / eth1 (LAN internal): 172.10.10.2
wiggum/NFS      | eth0: 10.2.0.129 / eth1 (LAN internal): 172.10.10.3
lisa0001        | eth0: 172.10.10.4
lisa0002        | eth0: 172.10.10.5

Configure eth1 on marge.
```
vi /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE=eth1
HWADDR=xx:xx:xx:xx:xx:xx
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPV6INIT=no
USERCTL=no
NM_CONTROLLED=no
PEERDNS=yes
IPADDR=172.10.10.2
NETMASK=255.255.255.0
```

Configure eth0 on marge
```
vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth1
HWADDR=xx:xx:xx:xx:xx:xx
TYPE=Ethernet
ONBOOT=yes
BOOTPROTO=none
IPV6INIT=no
USERCTL=no
NM_CONTROLLED=no
PEERDNS=yes
IPADDR=10.2.0.128
NETMASK=255.255.255.0
```
##This wasn't in the redbarn walkthrough, but was in Vince's fileserver walkthrough. Makes sense to do this, right? 
## We are later configuring gateway and DNS in warewulf. Makes sense to do it in both places?
Configure default gateway
Alaric did this during OS install.
/etc/sysconfig/network
```{.bash}
NETWORKING=yes
HOSTNAME=marge
GATEWAY=xxx.xxx.xxx.xxx
```
Eth0 was configed by Alaric
Should we configure DNS servers, or leave this blank to use /etc/hosts?
/etc/resolv.conf
```{.bash}
nameserver 8.8.8.8
nameserver 8.8.4.4
```

Ensure that network service is started
```
service network start
chkconfig network on

#do we need to do the 2 lines below? I don't think so. I think it's redundant with chkconfig network on. 
#don't need to do below
vi /etc/init.d/network restart
ifconfig eth1
```

##Disable SELinux

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

TODO: add public keys
      disable root access


Reboot

##Install Warewulf dependencies

```
rpm -Uvh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
#rpm -ivh would ignore errors
yum -y update

yum group install 'Development tools'
yum install xinetd   # xinetd is not installed in centos 7 by default
yum install tcpdump tftp tftp-server pigz dhcp httpd 
yum install perl-DBD-MySQL mariadb mariadb-server perl-Term-ReadLine-Gnu mod_perl perl-CGI
yum install libselinux-devel libacl-devel libattr-devel
yum install ntp
yum install cpan
cpan Module::Build
```

We received this error after cpan Module::Build
```
Warning: You do not have write permission for Perl library directories.

To install modules, you need to configure a local Perl library directory or
escalate your privileges.  CPAN can help you by bootstrapping the local::lib
module or by configuring itself to use 'sudo' (if available).  You may also
resolve this problem manually if you need to customize your setup.

What approach do you want?  (Choose 'local::lib', 'sudo' or 'manual')
```
We chose sudo, and it seemed to resolve the problem.


Continue installing warewulf dependencies
```
cpan DateTime
cpan YAML
cpan Crypt::HSXKPasswd #this isn't needed
```

Tests succeeded but one dependency not OK (File::HomeDir)
Did not install this dependency and just continued without HSXKPasswd.

```
yum install bonnie++
yum install yum-utils
yum install impitool 
yum install emacs
yum install iperf
yum install nfs-utils  
yum install libnfsidmap #we changed this from nfs-utils-lib because that wasn't available on yum
#yum install nfswatch #nfswatch not available in yum

yum -y update

reboot

```

## Enable tftp-server in /etc/xinetd.d/tftp.  Make sure disable = no on the fourth from the bottom line.

```
# default: off
# description: The tftp server serves files using the trivial file transfer \
# protocol. The tftp protocol is often used to boot diskless \
# workstations, download configuration files to network-aware printers, \
# and to start the installation process for some operating systems.
service tftp
{
socket_type = dgram
protocol = udp
wait = yes
user = root
server = /usr/sbin/in.tftpd
server_args = -s /var/lib/tftpboot
disable = no
per_source = 11
cps = 100 2
flags = IPv4
}
```

##Start services, disable firewall
Centos 7 uses firewalld instead of iptables.
What's the difference between systemctl and chkconfig?

```
systemctl disable firewalld
systemctl stop firewalld
systemctl enable httpd
systemctl start httpd
systemctl enable mariadb
systemctl start mariadb
systemctl enable ntpd  
systemctl start ntpd
timedatectl set-ntp yes
systemctl enable xinetd
systemctl restart xinetd

```

Reboot, and ensure services are in proper state after reboot:

```
systemctl status httpd # on 

```

##Issues with apache. What's going on?
Mar 16 18:17:52 marge.psg.net systemd[1]: Starting The Apache HTTP Server...
Mar 16 18:17:53 marge.psg.net httpd[1210]: AH00557: httpd: apr_sockaddr_info_get() failed for marge.psg.net
Mar 16 18:17:53 marge.psg.net httpd[1210]: AH00558: httpd: Could not reliably determine the server's full...sage
Mar 16 18:17:53 marge.psg.net systemd[1]: Started The Apache HTTP Server.
Hint: Some lines were ellipsized, use -l to show in full.

alaric trying to add to dns.....still did not work...come back to this

```
systemctl status mariadb # on
systemctl status firewalld  # off
systemctl status xinetd # on
```

## Install Warewulf

**build_it function creates RPM build environment. First argument is where source files are located (e.g. /root/warewulf/common/), second argument is where RPMs will be built /root/rpmbuild/. Makes everything in first argument dir, then copies all warewulf files to /root/rpmbuild/SOURCES, but I don't understand the very last command rpmbuild -bb ./.spec. The command is still in /root/warewulf/common, is rpmbuild run from there? Might be nice to run the function on the first warewulf component and see where files are created. SPEC file directs rpmbuild kind of like a makefile for make.**

```
# Build Warewulf from SVN
BUILD_DIR=/root/rpmbuild
WW_DIR=/root/warewulf
RPM_DIR=/root/rpmbuild/RPMS/
function build_it { cd $1 && ./autogen.sh && make dist-gzip && make distcheck && cp -fa warewulf-*.tar.gz $2/SOURCES/ && rpmbuild -bb ./*.spec; }
```
**Regarding h2ph command, why convert all C files to perl files in /usr/include? And where are the converted files placed? Don't worry about it.**
**We could probably verify that perl is in /usr/local/lib64/perl5, and if not symlink it there, but let's not worry about that now**

```
cd /usr/include
h2ph -al * sys/*
# Warning from h2ph: Destination directory /usr/local/lib64/perl5 doesn't exist or isn't a directory
#we didn't see that warning.
mkdir -p $BUILD_DIR/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
mkdir /root/warewulf
```
**downlaod to /root/warewulf. Why is echo p piped to svn? Who cares.**
```
echo p | svn co https://warewulf.lbl.gov/svn/trunk/ /root/warewulf 
#checked out revision 1962
build_it $WW_DIR/common $BUILD_DIR
yum install -y $RPM_DIR/noarch/warewulf-common-*.rpm
build_it $WW_DIR/provision $BUILD_DIR
yum install -y $RPM_DIR/x86_64/warewulf-provision-*.rpm
build_it $WW_DIR/cluster $BUILD_DIR
yum install -y $RPM_DIR/x86_64/warewulf-cluster-*.rpm
## remove debian.tmpl from Makefil* in vnfs/libexec/wwmkchroot
sed -i 's+debian.tmpl+ +g' $WW_DIR/vnfs/libexec/wwmkchroot/Makefil* 
build_it $WW_DIR/vnfs $BUILD_DIR
yum install -y $RPM_DIR/noarch/warewulf-vnfs-*.rpm

```
**Symlinks and dependencies needed to build monitor component?**
**What do the different warewulf components do? I'll RTFM**
```
# monitor
ln -s /usr/lib64/libjson.so.0 /usr/lib64/libjson.so
ln -s /usr/lib64/libjson-c.so.2 /usr/lib64/libjson-c.so 
ln -s /usr/lib64/libsqlite3.so.0 /usr/lib64/libsqlite3.so 
yum -y install json-c-devel json-devel sqlite-devel
build_it $WW_DIR/monitor $BUILD_DIR
yum -y install $RPM_DIR/x86_64/warewulf-monitor-0*.rpm
```
**Line added by Vince below**
We did not install WW ipmi
```
yum -y install $RPM_DIR/x86_64/warewulf-monitor-cl*.rpm # Added by Vince Forgetta
yum -y install $RPM_DIR/x86_64/warewulf-monitor-node*.rpm
# build_it $WW_DIR/ipmi $BUILD_DIR
# yum -y install $RPM_DIR/x86_64/warewulf-ipmi*.rpm
rpm -qa | grep warewulf 
#output of grep command should be below. Checking that all RPMs are built.

our output should match Vince's list below
warewulf-cluster-3.6.99-0.r1954.el7.centos.x86_64
warewulf-cluster-node-3.6.99-0.r1954.el7.centos.x86_64
warewulf-common-3.6.99-0.r1962.el7.centos.noarch
warewulf-common-localdb-3.6.99-0.r1962.el7.centos.noarch
warewulf-monitor-0.0.1-0.r1939.el7.centos.x86_64
warewulf-monitor-cli-0.0.1-0.r1939.el7.centos.x86_64
warewulf-monitor-node-0.0.1-0.r1939.el7.centos.x86_64
warewulf-provision-3.6.99-0.r1960.el7.centos.x86_64
warewulf-provision-gpl_sources-3.6.99-0.r1960.el7.centos.x86_64
warewulf-provision-server-3.6.99-0.r1960.el7.centos.x86_64
warewulf-vnfs-3.6.99-0.r1960.el7.centos.noarch


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

## Configure the cluster interface (the interface marge will use to communicate with the nodes) in /etc/warewulf/provision.conf

```
vi /etc/warewulf/provision.conf
# What is the default network device that will be used to communicate to the nodes?
network device = eth1 
```

##Setup MariaDB
**logged in as root, so ~ = /root/**
```
#set mysql root pass
#http://www.liberiangeek.net/2014/10/reset-root-password-mariadb-centos-7/
sudo systemctl stop mariadb.service
sudo service mariadb stop
sudo mysqld_safe --skip-grant-tables --skip-networking &
mysql -u root
use mysql;
update user set password=PASSWORD("new-password") where User='root';
flush privileges;
exit;

#stop mariadb
systemctl stop mariadb.service

#start mariaDB
systemctl start mariadb.service

vi ~/.my.cnf
[client]
user = root
password =
Add link from Alaric

chmod 0600 ~/.my.cnf

vi /etc/warewulf/database.conf
user = root
password =

NOTE:  user = wwuser was already there...is wwuser special?


vi /etc/warewulf/database-root.conf
user = root
password =
```
## We're not setting up VNFS for now

##initial setup to provision nodes
```
systemctl enable dhcpd
wwinit ALL
#ensure relevant tests are OK
```

##install pdsh
```
yum install libgenders

wget ftp://rpmfind.net/linux/sourceforge/s/sy/sys-integrity-mgmt-platform/yum/el/7/ext/x86_64/pdsh-2.29-1el7.x86_64.rpm

wget ftp://fr2.rpmfind.net/linux/sourceforge/s/sy/sys-integrity-mgmt-platform/yum/el/7/ext/x86_64/pdsh-rcmd-ssh-2.29-1el7.x86_64.rpm

yum -y install pdsh-rcmd-ssh-2.29-1el7.x86_64.rpm pdsh-2.29-1el7.x86_64.rpm
```

##Configure a Warewulf node image
Build a simple vanilla distribution of Centos into the /var/chroots/centos7 folder using yum
wwmkchroot was expecting a template file to be named exactly centos-7.tmpl, so we issued the copy command below
```
cp /usr/libexec/warewulf/wwmkchroot/centos-7.tmpl usr/libexec/warewulf/wwmkchroot/centos7.tmpl
wwmkchroot centos7 /var/chroots/centos7
```

convert the new node OS into a VNFS (Virtual Node File System) usable by Warewulf – and build the bootstrap from our current running environment.
```
wwvnfs --chroot /var/chroots/centos7
wwbootstrap `uname -r`

Number of drivers included in bootstrap: 447
depmod: WARNING: could not open /var/tmp/wwinitrd.sne6ea2IAnjM/initramfs/lib/modules/3.10.0-327.10.1.el7.x86_64/modules.order: No such file or directory
depmod: WARNING: could not open /var/tmp/wwinitrd.sne6ea2IAnjM/initramfs/lib/modules/3.10.0-327.10.1.el7.x86_64/modules.builtin: No such file or directory
Number of firmware images included in bootstrap: 93
Building and compressing bootstrap

```

## Register each node by running wwnodescan and then booting up each node on the cluster network – and each DHCP request will be recorded and the MAC addresses stored on the Warewulf master marge.
```
wwsh dhcp update
```

The wwsh dhcp restart command reported the error below. 
```
wwsh dhcp restart

Restarting the DHCP service
ERROR:  
ERROR:  
Done.
```
Vince also reported this error on the warewulf google group, and there was no fix. A workaround is to simply restart dhcpd.
```
systemctl restart dhcpd
```

```
wwnodescan --netdev=eth0 --ipaddr=172.10.10.4 --netmask=255.255.255.0 --vnfs=centos7 -- bootstrap=`uname -r` lisa00[01-02]
#Note: A range of IPs can be indicated as n00[00-02]
#Note: netdev is the node's (Lisa's) cluster interface
#Note: ipaddr is the first IP to be assigned to the nodes. It will increment for each new node
```
Output with errors is:
WARNING:  Auto-detected "bootstrap=3", cluster "10", domain "0-327.10.1.el7.x86_64"
ERROR:  Invalid value for Warewulf::Node->nodename:  "bootstrap=3"
Use of uninitialized value $name in concatenation (.) or string at /usr/share/perl5/vendor_perl/Warewulf/Provision.pm line 139.
Use of uninitialized value $name in concatenation (.) or string at /usr/share/perl5/vendor_perl/Warewulf/Provision.pm line 139.
Use of uninitialized value $name in concatenation (.) or string at /usr/share/perl5/vendor_perl/Warewulf/Provision.pm line 139.
ERROR:  There was an error restarting the DHCPD server
Added to data store:  bootstrap=3:  172.10.10.4/255.255.255.0/84:2b:2b:52:66:e0
ERROR:  Node name "lisa0001" contains unparseable dotted notation.  Removing all suffixes.
WARNING:  Auto-detected "lisa0001", cluster "", domain ""
ERROR:  There was an error restarting the DHCPD server
Added to data store:  lisa0001:  172.10.10.5/255.255.255.0/84:2b:2b:52:5e:a1
(command is still running)


now no more errors....

still need to ctrl-c to get out



Once nodes are registered, need to update head's DHCP conf to include all of the node's MACs and IPs
```
wwsh dhcp update
wwsh dhcp restart

reboot
```
Your nodes should be booting successfully now. Power them on in the order you want them numbered.

## SSH key
```
cd /root
chmod 700 /var/chroots/centos7/root/.ssh
ssh-keygen (use empty passphrases)
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 400 ~/.ssh/authorized_keys
cp ~/.ssh/authorized_keys /var/chroots/centos7/root/.ssh/
wwvnfs --chroot /var/chroots/centos7

-- Manually reboot each node --
```

##Then should be able to ssh into nodes
```
ssh lisa0001
 --answer 'yes' and exit
 
 Do this for all worker nodes

[root@marge ~]# ssh lisa0001
ssh: connect to host lisa0001 port 22: Connection refused


```

After this, should be able to use pdsh like so:
```
pdsh -w lisa00[01-02]

--Output--
lisa0001: lisa0001
lisa0002: lisa0002
```

## Hosts file
### Add Marge's cluster NIC IP to the hosts template that will get passed to the nodes
```
vi /etc/warewulf/hosts-template
127.0.0.1 localhost localhost.localdomain marge
172.10.10.2 marge marge.mydomain.local
```

### Propogate warewulf generated /etc/hosts file to the nodes
```
wwsh provision set --fileadd dynamic_hosts lisa0001 lisa0002
*Note: A range of nodes can be indicated as n00[02-19]

```

### Propogate warewulf generated /etc/hosts file to marge
```
wwsh file show dynamic_hosts > /etc/hosts
*Note: This should be run every time new nodes are added to the cluster. A cron job is recommended...
```


##NFS

Run on startup:
```
$ systemctl enable nfs-server
$ systemctl start nfs-server
```

According to following advice, no additional services are required to run NFSv4:
http://serverfault.com/questions/530908/nfsv4-and-rpcbind
However, downstream Warewulf config needs rpcbind:

rpcbind enables computer inter-communication
```
$ systemctl enable rpcbind
$ systemctl start rpcbind
```

Share marge's home with all nodes
```
vi /etc/exports
/home/ 10.10.10.0/255.255.255.0(rw,no_root_squash,async)
We need to change this
```

Next is to edit the nodes’ /etc/fstab. In order to do this, we directly modify the chrooted OS we made earlier (/var/chroots/centos7):
```
-- /var/chroots/centos6/etc/fstab --
#GENERATED_ENTRIES#
tmpfs /dev/shm tmpfs defaults 0 0
devpts /dev/pts devpts gid=5,mode=620 0 0
sysfs /sys sysfs defaults 0 0
proc /proc proc defaults 0 0
10.10.10.1:/home/ /home/ nfs rw,soft,bg 0 0
 
```

Mount file systems:
```
$ mount -a
```

##Optimization
Richard did that.

```
$ /etc/sysconfig/nfs
RPCNFSDCOUNT=16
```








Setup VNFS
**We should talk about this. Should we remove some dirs from the chroot to be nfs mounted to save RAM in the chroot? Alaric would say no, since we bought lots of RAM, but more RAM is always good.**
#We decided to leave it commented out, so we're not using VNFS.
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

$ wget ftp://rpmfind.net/linux/sourceforge/s/sy/sys-integrity-mgmt-platform/yum/el/7/ext/x86_64/pdsh-2.29-1el7.x86_64.rpm

$ wget ftp://fr2.rpmfind.net/linux/sourceforge/s/sy/sys-integrity-mgmt-platform/yum/el/7/ext/x86_64/pdsh-rcmd-ssh-2.29-1el7.x86_64.rpm

yum -y install pdsh-rcmd-ssh-2.29-1el7.x86_64.rpm pdsh-2.29-1el7.x86_64.rpm

```

SSH key
**What is the last line doing?**
```
$ ssh-keygen (use empty passphrases)
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys 
$ chmod 400 ~/.ssh/authorized_keys
$ ssh-keygen -f ~/.ssh/identity #used empty pass too
```
To do later:
Restrict SSH login to root and devans:
```
$ vi /etc/ssh/sshd_config
# add following
AllowUsers root devans

$ systemctl restart sshd
```
End of To do later.

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
#we didn't run this.
$ pdsh -w marge "SLEEPTIME=0 /warewulf/bin/wwgetfiles"
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






