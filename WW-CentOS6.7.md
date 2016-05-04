---
title: "Warewulf CentOS 6.7 installation walkthrough"
output: html_document
---

# OS install and dependencies
1. Download and burn and boot from the “Minimal Install” CD
2. Set up your partitions however you wish
3. Configure your network interfaces

### CentOS accounts on marge

user name | Pass
----------|-----------
root      | 
devans    | 

MySQL root pass and devans pass same as OS root pass

### Configure network interfaces.
### Head node configuration
* 2 NICs on marge
  + eth0 connected to internet
  + eth1 connected to GigE switch to workers. 
  + Static IPs for eth0 and eth1 assigned by editing ifcfg-eth0/1.

### Hostnames and IP addresses
host name       | IP address
----------------|-----------
marge           | eth0: 10.2.0.128 / eth1 (LAN internal): 172.10.10.2
wiggum/NFS      | eth0: 10.2.0.129 / eth1 (LAN internal): 172.10.10.3
lisa0001        | eth0: 172.10.10.4
lisa0002        | eth0: 172.10.10.5

Set hostname on head node. We will specify worker node hostnames in /etc/hosts
```
hostname marge
```
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

RedBarn strongly recommends disabling – or even removing – NetworkManager on the master. We didn't do this, but we could with the following commands.
```
service NetworkManager stop
chkconfig NetworkManager off
```

4. Disable SELINUX by editing /etc/selinux/config:
(NOTE: The new selinux setting will not take effect until the next reboot)
```
vi /etc/selinux/config
# This file controls the state of SELinux on the system. # SELINUX= can take one of these three values:
# enforcing - SELinux security policy is enforced.
# permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
```
<span style="color:red">SELINUX=disabled </span>
```
# SELINUXTYPE= can take one of these two values:
# targeted - Targeted processes are protected,
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

5. Install dependencies
```
yum -y groupinstall "Development Tools"
yum -y install httpd dhcp tftp-server tftp mod_perl mysql mysql-server wget tcpdump nfs-utils nfs-utils-lib
yum -y update
reboot
```

6. Make sure Apache and MySQL start automatically – and iptables doesn’t.
(Yes – I know this is a bad security practice. Once you get everything thing working, feel free to re-enable this after adding all of your rules)
```
service httpd start
service mysqld start
service iptables stop
chkconfig httpd on
chkconfig mysqld on
chkconfig iptables off
```

7. Enable tftp
```
vi /etc/xinetd.d/tftp --
default: off
description: The tftp server serves files using the trivial file transfer \
protocol. The tftp protocol is often used to boot diskless \
workstations, download configuration files to network-aware printers, \
and to start the installation process for some operating systems. service tftp
{
    socket_type = dgram
    protocol = udp
    wait = yes
    user = root
    server = /usr/sbin/in.tftpd
    server_args = -s /var/lib/tftpboot
    disable = no  ###edit this line
    per_source = 11
    cps = 100 2
    flags = IPv4
}
```
<span style="color:red"> disable = no ###Edit fourth line from bottom to be this.</span>

Restart xinetd
```
service xinetd restart
```

#Install Warewulf
1. For simplicity, we configure yum Warewulf’s own RHEL repository, along with Red Hat’s EPEL repo:
```
wget http://warewulf.lbl.gov/downloads/repo/warewulf-rhel6.repo -O /etc/yum.repos.d/warewulf-rhel6.repo
rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```
warnings that rpm modified outside of yum....
did not check for updates on anything else such as epel repo

2. Using yum, we install Warewulf and its required dependences. (Perl-Term-ReadLine allows for command completion in the Warewulf shell)
```
yum -y install warewulf-common warewulf-provision warewulf-vnfs warewulf-provision- server warewulf-ipmi Warewulf-mic
yum -y install perl-Term-ReadLine-Gnu
```

Versions of components installed.

Name      | warewulf-common
arch      | noarch
version   | 3.6
release   | 1.el6

ipmi warning about dedicate or ontop of eth1....  need to come back to this

3. Configure the cluster interface (the interface marge will use to communicate with the nodes) in /etc/warewulf/provision.conf
```
vi /etc/warewulf/provision.conf
#What is the default network device that marge will use to communicate to the nodes?
network device = eth1 
```

#Configuring a warewulf node image
1. The following command will build a simple vanilla distribution of Centos into the /var/chroots/centos6 folder using yum.

```
wwmkchroot centos-6 /var/chroots/centos6
```

2. Next, we convert the new node OS into a VNFS (Virtual Node File Systsem) usable by Warewulf – and build the bootstrap from our current running environment.

```
wwvnfs --chroot /var/chroots/centos6
wwbootstrap `uname -r`
```
ipmi warning about dedicate or ontop of eth1....  need to come back to this

3. In order to register each node, we will run wwnodescan and then boot up each node on the cluster network – and each DHCP request will be recorded and the MAC addresses stored on the Warewulf master.

NOTE: It is imperative that you understand how the IPMI works on your node motherboards! Many server boards with IPMI will give you the option between using a dedicated IPMI port –or- sharing the LAN1 port. If no link is detected in the IPMI port when the power cord is plugged in, it will assume you wish to share LAN1. Then - if the IPMI is set to use DHCP, this request will come first – and confuse the Warewulf server. Warewulf will register the IPMI MAC address as the node’s LAN1 and not the actual one. So – in order to address this, either make sure there is separate link to the IPMI port before boot (so LAN1 is not shared) or makes sure you have a static address set for the IPMI interface.

```
wwsh dhcp update
wwsh dhcp restart
wwnodescan --netdev=eth0 --ipaddr=172.10.10.4 --netmask=255.255.255.0 --vnfs=centos6 --bootstrap=`uname -r` lisa0001 lisa0002

*Note: A range of IPs can be indicated as lisa00[01-02]
*Note: netdev is the cluster interface of the nodes
*Note: ipaddr is the first IP to be assigned to the nodes. It will increment for each new node
```
Output of wwnodescan:
```
Added to data store:  lisa0001:  172.10.10.4/255.255.255.0/00:15:5d:00:7a:0c
Added to data store:  lisa0002:  172.10.10.5/255.255.255.0/00:15:5d:00:7a:0a
```

LISA0002 PXE booted using Legacy mode for NIC.
UEFI NIC isn't working. 
Legacy mode on VM does not support 1Gb eth, so we will need to install everything on baremetal in order to get at least 1Gb.

may be able to build a version that switches adaptors down the road after it is installed....

wwsh to get to warewulf shell....
help gives a list of available commands.

If you screw up the config on a node, delete it like this:
```
wwsh node delete lisa0001
```
Then, run wwnodescan to add it again. Or you can use node new followed by provision set. See the provision documentation at the warewulf site.

4. Once the nodes have been registered, we need to update the master’s DHCP conf to include all of the node’s MACs and IPs.
```
wwsh dhcp update
wwsh dhcp restart
```
Then reboot.
```
reboot
```
Your nodes should be booting successfully now. Power them on in the order you want them numbered.

#Customizing Warewulf
1. Configuring the Parallel Distributed Shell (pdsh).
We need to install pdsh and its dependency to the master. Then we share root’s ssh key to the node image. Then, we rebuild the node image and manually reboot each node.

```
yum -y install http://apt.sw.be/redhat/el6/en/x86_64/rpmforge/RPMS/libgenders-1.14- 2.el6.rf.x86_64.rpm
yum -y install http://pkgs.repoforge.org/pdsh/pdsh-2.27-1.el6.rf.x86_64.rpm
cd /root
chmod 700 /var/chroots/centos6/root/.ssh
ssh-keygen (use empty passphrases)
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 400 ~/.ssh/authorized_keys
cp ~/.ssh/authorized_keys /var/chroots/centos6/root/.ssh/ 
wwvnfs --chroot /var/chroots/centos6
       -- Manually reboot each node –
```

Once each node has rebooted, we must establish an ssh connection with each one so that the keys get properly associated with each node’s MAC address. (I know this is a pain – if anyone has a more automated way of doing this, please let me know)

```
ssh lisa0001
       -- answer ‘yes’ and exit
ssh lisa0002
       -- answer ‘yes’ and exit
```

After booting nodes, took awhile to ssh or ping them, but it started working after a bit.

Once finished, the following command should work:
```
pdsh -w lisa00[01-02] echo $hostname
```

2. Hosts file
Add marge’s cluster interface IP to the hosts template that will get passed to the nodes.
File did not exist so we created it.
```
vi /etc/warewulf/hosts-template
127.0.0.1 localhost localhost.localdomain
172.10.10.2 marge marge.mydomain.local
```

Propagate Warewulf-generated /etc/hosts file to the nodes
```
wwsh provision set --fileadd dynamic_hosts lisa0001 lisa0002 
*Note: A range of nodes can be indicated as lisa00[02-19]
 
```
what is dynamic_hosts?

Why in the world do we need to run wwsh provision set --fileadd dynamic_hosts to the nodes? It seems like wwnodescan already registers the nodes in the master /etc/hosts, so why do this again? 

What does the output of wwsh file show dynamic_hosts look like? We're generating /etc/hosts with it.

Propagate Warewulf-generated /etc/hosts file to marge
```
wwsh file show dynamic_hosts > /etc/hosts
*Note: This should be run every time new nodes are added to the cluster. A cron job is recommended...
```

3. Sharing wwmaster’s /home with the nodes.

Each node will mount this as their own /home and gives us a good launching point for jobs across the cluster.

On the master, edit /etc/exports (changes in red):
```
vi /etc/exports
/home/ 172.10.10.2/255.255.255.0(rw,no_root_squash,async)
```

Next is to edit the nodes’ /etc/fstab. In order to do this, we directly modify the chrooted OS we made earlier (/var/chroots/centos6):
```
vi /var/chroots/centos6/etc/fstab
#GENERATED_ENTRIES#
tmpfs /dev/shm tmpfs defaults 0 0
devpts /dev/pts devpts gid=5,mode=620 0 0
sysfs /sys sysfs defaults 0 0
proc /proc proc defaults 0 0

172.10.10.2:/home/ /home/ nfs rw,soft,bg 0 0
```

Tell marge to run the NFS server at startup
```
chkconfig nfs on
service rpcbind start
service nfs start
```
Why isn't rpcbind have a chkconfig on?

Then, rebuild the node image from the modified chroot
```
wwvnfs --chroot /var/chroots/centos6
```
Then reboot the nodes to load the modified VNFS.
```
pdsh -n lisa00[01-02] reboot
```

4. Synchronizing user/group accounts to the nodes
Create new user
```
adduser devans
passwd devans
```
Use same passwd as root account. 

Provision the master's /etc/passwd to the nodes
```
wwsh file import /etc/passwd
wwsh provision set lisa[0000-0002] --fileadd passwd
```

Provision the master's /etc/group to nodes.
```
wwsh file import /etc/group
wwsh provision set lisa[0000-0002] --fileadd group
```

In order to provide password-less access for users to the nodes, each time you create a user, make sure you generate a key pair and add it to authorized_keys. Since the home folder is shared between all nodes, you only need to do this once for each user at inception. Our user will be ‘devans’.

```
su - devans
cd ~
mkdir .ssh
chmod 700 .ssh
ssh-keygen -t dsa -P ""
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys 
chmod 400 ~/.ssh/authorized_keys
exit
```

Once complete, reboot to allow passwordless logins.

5. Install EPEL repo on the nodes
So as to avoid any versioning differences between the master and the nodes, let’s make sure the nodes’ filesystem is using the same EPEL repo as the master.

```
rpm --root /var/chroots/centos6 -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```
rebuild vnfs
```
wwvnfs --chroot /var/chroots/centos6
```
reboot nodes to deliver the latest vnfs.

6. Configure node's HD as swap or scrath.
come back to this once we have the drives.....

#Installing Ganglia
This procedure assumes that you have installed the EPEL repo on both the master and nodes, as described in Section 3: Installing Warewulf

Also see:
https://github.com/ganglia/monitor-core/wiki/Ganglia-Quick-Start

1. Using yum, install the ganglia binaries on the master node:
```
yum install -y ganglia*
```

2. Ganglia has two conf files that need editing. The first is for the server daemon, gmetad. This resides only on the server. We must edit it to contain the name of the cluster and the name of the master node (which in this case, is simply localhost).

```
vi /etc/ganglia/gmetad.conf  
(Line 39)
data_source “Warewulf” localhost
```
In gmetad.conf, data_source MUST start with lowercase d.

3. Next, edit /etc/ganglia/gmond.conf. This information will reside on both the master and the nodes – indicating where the gmetad daemon is and how to communicate with it. (changes marked with hash )

```
#-- /etc/ganglia/gmond.conf – (lines 19-47)
cluster {
  name = "Warewulf" ##edit this
  owner = "unspecified"
  latlong = "unspecified"
  url = "unspecified"
}
/* The host section describes attributes of the host, like the location */ 
host {
  location = "unspecified"
}
/* Feel free to specify as many udp_send_channels as you like. Gmond used to only support having a single channel */
udp_send_channel {
  host = marge ##edit this
  port = 8649
}
/* You can specify as many udp_recv_channels as you like as well. */ 
udp_recv_channel {
  port = 8649 ##edit this
}
/* You can specify as many tcp_accept_channels as you like to share an xml description of the state of the cluster */
tcp_accept_channel {
  port = 8649
}
```             

4. Start gmetad and gmond on the master and configure them to start automatically every boot.

```
/etc/init.d/gmetad start
/etc/init.d/gmond start
chkconfig gmetad on
chkconfig gmond on
```

5. Now, install only gmond on the nodes, copy the config file from the server, then make sure it starts at boot time on the nodes.

```
yum --installroot=/var/chroots/centos6 install ganglia-gmond
cp /etc/ganglia/gmond.conf /var/chroots/centos6/etc/ganglia/gmond.conf 
chroot /var/chroots/centos6
chkconfig gmond on
exit
```

6. One of the ganglia RPMs we installed (ganglia-web) actually builds the root of the ganglia web app, but we need to make sure apache is running and configured to start at boot time first. If you installed Warewulf – this should already be done.

```
/etc/init.d/httpd start
chkconfig httpd on
```

7. By default, ganglia-web only allows connections from localhost. If you would like the ganglia utility viewable by all on your network, adjust the /etc/httpd/conf.d/ganglia.conf file to be:
```
vi /etc/httpd/conf.d/ganglia.conf
  #
  # Ganglia monitoring system php web frontend
  #
  Alias /ganglia /usr/share/ganglia
  <Location /ganglia>
    Order allow,deny
    Allow from all
    # Allow from .example.com
</Location>
```

8. Rebuild the image and restart the nodes. Browse to http://10.2.0.128/ganglia or http://marge.psg.net/ganglia from your LAN to verify installation.

```
wwvnfs --chroot /var/chroots/centos6
pdsh -w lisa00[00-99] reboot
```

ganglia server is responding but getting - error collecting ganglia data  port 8652...

selinux was reenabled..... not sure why. Once disabled again, it worked.

#Installing OpenMPI
This procedure assumes that you have installed the EPEL repo on both the master and nodes, as described in Section 2: Installing Warewulf. We also assume that you are using a Gigabit network, and not Infiniband. This can be done, but is not detailed here.

1. Get the needed files from openmpi.org
```
cd ~
wget http://www.open-mpi.org/software/ompi/v1.4/downloads/openmpi-1.4.5.tar.gz
wget http://svn.open-mpi.org/svn/ompi/branches/v1.4/contrib/dist/linux/buildrpm.sh 
wget http://svn.open-mpi.org/svn/ompi/branches/v1.4/contrib/dist/linux/openmpi.spec
```

2. Make sure the buildrpm.sh script is executable.
```
chmod +x buildrpm.sh
```

3. For some strange reason Centos 6.x does not give you a default RPM-building environment. You must create one yourself. Logged in as root, issue these commands:
```
cd ~
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS} 
echo '%_topdir /root/rpmbuild' > ~/.rpmmacros
```

4. Edit the buildrpm.sh file so that it will build a single rpm file (changes in hash marks).
```
vi ~/buildrpm.sh – (Lines 33-41)
Note that this script can build one or all of the following RPMs: 
SRPM, all-in-one, multiple.
#If you want to build the SRPM, put "yes" here 
build_srpm=${build_srpm:-"yes"}
#If you want to build the "all in one RPM", put "yes" here 
build_single=${build_single:-"yes"}  #make this yes
#If you want to build the "multiple" RPMs, put "yes" here 
build_multiple=${build_multiple:-"no"}
```

Note: Optimally, I would prefer to build with the multiple RPMs option and just install the openmpi-runtime package on the nodes – to reduce VNFS size, but that has given me some compilation errors I have not been able to fix. Please feel free to send me any advice on this topic.

5. Build the RPM (this will take awhile)
```
./buildrpm.sh openmpi-1.4.5.tar.gz
```

6. Install the new RPM on the master.
```
rpm -ivh /root/rpmbuild/RPMS/x86_64/openmpi-1.4.5-2.x86_64.rpm
```

7. Install the new RPM on the nodes
```
yum --installroot /var/chroots/centos6 install /root/rpmbuild/RPMS/x86_64/openmpi- 1.4.5-2.x86_64.rpm
```

8. Rebuild VNFS and reboot nodes.
```
wwvnfs --chroot /var/chroots/centos6
pdsh -w n00[00-99] reboot
```

#SLURM install, and Munge!!!!
This procedure assumes that you have installed the EPEL repos on both the master and nodes, as described in Section 2: Installing Warewulf.

1. We will be using Munge for authentication, so it must be configured and installed before SLURM.
Add the munge and slurm users
```
useradd munge
useradd slurm
```

Find the new users’ entries in /etc/passwd, and change the homedir and shell: (changes followed by hash marks). UID and GID are likely different on our system. 
```
vi /etc/passwd
munge:x:502:502::/home/munge:/sbin/nologin 
slurm:x:503:503::/home/slurm:/sbin/nologin
```

Sync the users to the nodes.
```
wwsh file import /etc/passwd
wwsh file import /etc/group
```

Install munge.
```
yum install munge munge-devel
```

Create the authentication key for use with Munge
```
dd if=/dev/urandom bs=1 count=1024 >/etc/munge/munge.key
#red barn lists the line below too, but seems redundant with first line. We only ran first line.
#echo -n "foo" | sha1sum | cut -d' ' -f1 >/etc/munge/munge.key //(where foo is your password)
chown munge.munge /etc/munge/munge.key
chmod 400 /etc/munge/munge.key
/etc/init.d/munge start
chkconfig munge on
```

Copy the authentication key to the nodes, then reboot nodes.
```
yum --installroot=/var/chroots/centos6 install munge 
chroot /var/chroots/centos6
chkconfig munge on
exit
cp -p /etc/munge/munge.key /var/chroots/centos6/etc/munge/munge.key 
chown -R munge.munge /var/chroots/centos6/etc/munge
chown -R munge.munge /var/chroots/centos6/var/log/munge
chown -R munge.munge /var/chroots/centos6/var/lib/munge
mv /var/chroots/centos6/etc/rc3.d/S40munge /var/chroots/centos6/etc/rc3.d/S96munge
```

var/log/munge was created in the chroot, but then it wasn't on the nodes! Looks like it's being excluded.
vi etc/warewulf/vnfs.conf
Comment out the line about excluding /var/log. We need this directory in the nodes vnfs. After removing this excludes line, everything worked well. 

Rebuild the VNFS and reboot the nodes.
```
wwvnfs --chroot /var/chroots/centos6
pdsh -w n00[00-99] reboot
```

Test munge with the following commands:
```
munge -n | unmunge                // test locally
munge -n | ssh n0000 unmunge      // test remotely
NOTE – make sure your clocks are synced (or at least very close) or munge will report an error
```

##We stopped here##

#NTP
http://www.cyberciti.biz/faq/rhel-fedora-centos-configure-ntp-client-server/

Also on Jeff Layton's HPC-mag article.
The next package to install is ntp. NTP is a way for clocks on disparate nodes to synchronize to each other. This aspect is important for clusters because, if the clocks are too skewed relative to each other, the application might have problems. You need to install ntp on both the master node and the VNFS, but start by installing it on the master node.

```
yum install ntp
```

Configuring NTP on the master node isn’t too difficult and might not require anything more than making sure it starts when the node is rebooted. To do this, first make sure NTP is running on the master node after being installed and that it starts when the master node is rebooted (this process should be familiar to you by now):

```
chkconfig --list | grep ntpd
chkconfig ntpd on
chkconfig --list | grep ntpd
service ntpd restart
```

I used the install defaults on Scientific Linux 6.2 on my test system. The file /etc/ntp.conf defines the NTP configuration, with three important lines that point to external servers for synchronizing the clock of the master node:
```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html). 
server 0.rhel.pool.ntp.org                 
server 1.rhel.pool.ntp.org             
server 2.rhel.pool.ntp.org
```

This means the master node is using three external sources for synchronizing clocks. To check this, you can simply use the ntpq command along with the lpeers option:
```
ntpq
ntpq> lpeers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 10504.x.rootbsd 198.30.92.2      2 u   27   64    3   40.860  141.975   4.746
 wwwco1test12.mi 64.236.96.53     2 u   24   64    3  106.821  119.694   4.674
 mail.ggong.info 216.218.254.202  2 u   25   64    3   98.118  145.902   4.742
 ntpq> quit
```

The output indicates that the master node is syncing time with three different servers on the Internet. Knowing how to point NTP to a particular server will be important for configuring the VNFS.

The next step is to install NTP into the chroot environment that is the basis for the VNFS (again, with Yum). Yum is flexible enough that you can tell it the root location where you want it installed. This works perfectly with the chroot environment:
```
yum --tolerant --installroot /var/chroots/centos6 -y install ntp
```

Now that NTP is installed into the chroot, you just need to configure it by editing the etc/ntp.conf file that is in the chroot. For the example, in this article, the full path to the file is /var/chroots/centos6/etc/ntp.conf. The file should look like this,
```
# For more information about this file, see the man pages
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).

#driftfile /var/lib/ntp/drift

restrict default ignore
restrict 127.0.0.1
server 172.10.10.2
restrict 172.10.10.2 nomodify
```
where 172.10.10.2 is the IP address of the master node. (It uses the private cluster network.)

To get NTP ready to run on the compute nodes, you have one more small task. When ntp is installed into the chroot, it is not configured to start automatically, so you need to force it to run. A simple way to do this is to put the commands into the etc/rc.d/rc.local file in the chroot. For this example, the full path to the file is /var/chroots/centos6/etc/rc.d/rc.local and should look as follows:
```
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.

touch /var/lock/subsys/local

chkconfig ntpd on
service ntpd start
```

Although you probably have better, more elegant ways to do this, I chose to do it this way for the sake of expediency.

Now ntp is installed and configured in the chroot environment, but because it has changed, you now have to rebuild the VNFS for the compute nodes. 
```
wwnvfs --chroot /var/chroots/centos6
pdsh -w lisa00[01-04] reboot
```

Check that ntp is working on each node.
```
ssh lisa0001
ntpq
ntpq> lpeers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 172.10.10.2      .STEP.          16 u    2   64    0    0.000    0.000   0.000
ntpq> quit
```


2. Install SLURM on master
To build RPMs directly, copy the distributed tar-ball into a directory and execute rpmbuild -ta slurm-14.03.9.tar.bz2 with the appropriate Slurm version number. The rpm file will be installed under the $(HOME)/rpmbuild directory of the user building them. 

```
#wget http://www.schedmd.com/download/archive/slurm-2.3.4.tar.bz2 
#newest version is 15.08.11, but doesn't seem to be available via wget. Download using GUI browser then copy to marge?
yum install readline-devel openssl-devel pam-devel
rpmbuild -ta slurm*.tar.bz2
cd ~/rpmbuild/RPMS/x86_64
yum install slurm-2.3.4-1.el6.x86_64.rpm slurm-devel-2.3.4-1.el6.x86_64.rpm slurm- plugins-2.3.4-1.el6.x86_64.rpm slurm-munge-2.3.4-1.el6.x86_64.rpm
```

3. Generate your config file. You can do this by going to
https://computing.llnl.gov/linux/slurm/configurator.html
and saving the result to /etc/slurm/slurm.conf However, here is my conf file for your convenience:

Please note you may need to adjust the Sockets and CoresPerSocket parameters based on your hardware configuration. Also, please note that the last two lines in the file begin with “NodeName” and “PartitionName” respectively. I don’t want the word wrap to make you think there should be line breaks where there shouldn’t be).

```
vi /etc/slurm/slurm.conf
#
# Example slurm.conf file. Please run configurator.html
# (in doc/html) to build a configuration file customized # for your environment.
#
#
# slurm.conf file generated by configurator.html. 
#
# See the slurm.conf man page for more information. 
#
ClusterName=Warewulf ##this is the cluster name we specified in ganglia conf files.
ControlMachine=marge
ControlAddr=172.10.10.2
#BackupController=
#BackupAddr=
#
SlurmUser=slurm
#SlurmdUser=root
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge
#JobCredentialPrivateKey= 
#JobCredentialPublicCertificate= 
StateSaveLocation=/tmp
SlurmdSpoolDir=/tmp/slurmd
SwitchType=switch/none
MpiDefault=none 
SlurmctldPidFile=/var/run/slurmctld.pid 
SlurmdPidFile=/var/run/slurmd.pid 
ProctrackType=proctrack/pgid
#PluginDir=
CacheGroups=0
#FirstJobId=
ReturnToService=0
#MaxJobCount=
#PlugStackConfig=
#PropagatePrioProcess=
#PropagateResourceLimits= 
#PropagateResourceLimitsExcept=
#Prolog=
#Epilog=
#SrunProlog=
#SrunEpilog=
#TaskProlog=
#TaskEpilog=
#TaskPlugin=
#TrackWCKey=no
#TreeWidth=50
#TmpFs=
#UsePAM=
#
# TIMERS
SlurmctldTimeout=300
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0
#
# SCHEDULING
SchedulerType=sched/backfill
#SchedulerAuth=
#SchedulerPort=
#SchedulerRootFilter=
SelectType=select/linear
FastSchedule=1
#PriorityType=priority/multifactor 
#PriorityDecayHalfLife=14-0
#PriorityUsageResetPeriod=14-0
#PriorityWeightFairshare=100000
#PriorityWeightAge=1000
#PriorityWeightPartition=10000
#PriorityWeightJobSize=1000
#PriorityMaxAge=1-0
#
# LOGGING
SlurmctldDebug=3
#SlurmctldLogFile=
SlurmdDebug=3
#SlurmdLogFile=
JobCompType=jobcomp/none
#JobCompLoc=
#
# ACCOUNTING
#JobAcctGatherType=jobacct_gather/linux
#JobAcctGatherFrequency=30
#
#AccountingStorageType=accounting_storage/slurmdbd
#AccountingStorageHost=
#AccountingStorageLoc=
#AccountingStoragePass=
#AccountingStorageUser=
#
# COMPUTE NODES
NodeName=lisa00[01-16] Sockets=2 CoresPerSocket=6 ThreadsPerCore=2 State=UNKNOWN PartitionName=allnodes Nodes=lisa00[00-16] Default=YES MaxTime=INFINITE State=UP
#ALL OF THE NODES ABOVE MUST BE RESOLVABLE OR ELSE SLURM WILL NOT START!
```

4. Start Slurm and configure to start automatically each boot
```
/etc/init.d/slurm start
chkconfig slurm on
```

5. Install Slurm and configure it to start automatically on the nodes.
```
cd ~/rpmbuild/RPMS/x86_64
#use the right slurm version
yum --installroot=/var/chroots/centos6 install slurm-2.3.4-1.el6.x86_64.rpm slurm- plugins-2.3.4-1.el6.x86_64.rpm slurm-munge-2.3.4-1.el6.x86_64.rpm
cp -p /etc/slurm/slurm.conf /var/chroots/centos6/etc/slurm/slurm.conf 
chroot /var/chroots/centos6
chkconfig slurm on
mv /etc/rc3.d/S90slurm /etc/rc3.d/S97slurm
exit
```

6. Rebuild VNFS, reboot nodes
```
wwvnfs --chroot /var/chroots/centos6
pdsh –w n00[00-99] reboot
```

7. Check that slurm sees the nodes
```
scontrol show nodes
-- If nodes’ state shows as DOWN, run the following: --
scontrol update NodeName=ClusterNode0 State=Resume 
scontrol show nodes
```


#Install HPL Benchmark
This procedure assumes that you have installed the EPEL repo on both the master and nodes, as described in Section 2: Installing Warewulf.

1. Install the following Linear Algebra libraries and HPL prereqs on the master.
```
yum install atlas atlas-devel blas blas-devel lapack lapack-devel compat-gcc* 
yum --installroot=/var/chroots/centos6 install compat-gcc*
```

2. Get the source for the benchmark and move it into the proper folder (to make our lives easier later)
```
cd ~
wget http://www.netlib.org/benchmark/hpl/hpl-2.2.tar.gz 
tar xfvz hpl-2.2.tar.gz
mv hpl-2.2 hpl
cd hpl
cp setup/Make.Linux_PII_CBLAS .
```

3. Edit the Makefile (~/hpl/Make.Linux_PII_CBLAS) , giving it the location of OpenMPI and the ATLAS/BLAS. (changes are the uncommented lines)

```
# ----------------------------------------------------------------------
# - Message Passing library (MPI) --------------------------------------
# ----------------------------------------------------------------------
# MPinc tells the C compiler where to find the Message Passing library
# header files, MPlib is defined to be the name of the library to be
# used. The variable MPdir is only used for defining MPinc and MPlib.
#
MPdir = /usr/lib64/openmpi
MPinc = -I$(MPdir)
MPlib = /usr/lib64/libmpi.so
#
# ----------------------------------------------------------------------
# - Linear Algebra library (BLAS or VSIPL) -----------------------------
# ----------------------------------------------------------------------
# LAinc tells the C compiler where to find the Linear Algebra library
# header files, LAlib is defined to be the name of the library to be
# used. The variable LAdir is only used for defining LAinc and LAlib.
#
LAdir = /usr/lib64/atlas
LAinc =
LAlib = $(LAdir)/libcblas.a $(LAdir)/libatlas.a
#

```

4. Compile the benchmark and copy the new binary to the user's home directory.
```
make arch=Linux_PII_CBLAS
cp bin/Linux_PII_CBLAS/xhpl /home/devans 
chown devans.devans /home/devans/xhpl
```

5. Configure your HPL.dat file. Just plug your node specifications into the following online tool, and it will generate the file for you:
http://www.advancedclustering.com/faq/how-do-i-tune-my-hpldat-file.html

6. Once xhpl and HPL.dat are both in your user’s home directory, you may run the benchmark in one of two ways:

###Using OpenMPI directly. 
Create a file named machines in the user's home directory, and give it the following contents, adjusting for your own nodes.

```
vi /home/devans/machines
lisa0000
lisa0001
lisa0002
lisa0003
lisa0004
lisa0005
lisa0006
lisa0007
```
Next, invoke the benchmark using mpirun, and make sure the –np attribute equals the number of your nodes, multiplied by the number of cores in each node. (eg, 8 nodes x 2 cpus x 4 cores = 64)

```
su - devans
mpirun --machinefile machines -np 64 ./xhpl
```

### Using Slurm
Allocate the resources for the benchmark and make sure the –n attribute equals the number of your nodes, multiplied by the number of cores in each node. (eg, 8 nodes x 2 cpus x 4 cores = 64)

```
salloc –n64 sh
```

Inside the provided shell, call the benchmark using mpirun. Note: No parameters are needed – as they are already inferred from the allocation we did previously.

```
mpirun ./xhpl
```

Sample HPL.out file generated by the benchmark. The score in this example for our sample cluster was 328 Gflops.
```
================================================================================ 
HPLinpack 2.0 -- High-Performance Linpack benchmark -- September 10, 2008 Written by A. Petitet and R. Clint Whaley, Innovative Computing Laboratory, UTK Modified by Piotr Luszczek, Innovative Computing Laboratory, UTK
Modified by Julien Langou, University of Colorado Denver ================================================================================
An explanation of the input/output parameters follows: 
T/V : Wall time / encoded variant.
N : The order of the coefficient matrix A.
NB : The partitioning blocking factor.
P      : The number of process rows.
Q      : The number of process columns.
Time : Time in seconds to solve the linear system. 
Gflops : Rate of execution for solving the linear system.

The following parameter values will be used:

N : 150000
NB : 128
PMAP : Row-major process mapping 
P:8
Q:8
PFACT : Right
NBMIN : 4
NDIV: 2
RFACT : Crout
BCAST : 1ringM
DEPTH : 1
SWAP : Mix (threshold = 64)
L1 : transposed form
U : transposed form
EQUIL : yes
ALIGN : 8 double precision words
--------------------------------------------------------------------------------
- The matrix A is randomly generated for each test.
- The following scaled residual check will be computed:
||Ax-b||_oo / ( eps * ( || x ||_oo * || A ||_oo + || b ||_oo ) * N )
- The relative machine precision (eps) is taken to be 1.110223e-16 
- Computational tests pass if scaled residuals are less than 16.0
================================================================================ 
T/V       N     NB  P Q Time    Gflops 
-------------------------------------------------------------------------------- 
WR11C2R4 150000 128 8 8 6859.11 3.280e+02 -------------------------------------------------------------------------------- ||Ax-b||_oo/(eps*(||A||_oo*||x||_oo+||b||_oo)*N)= 0.0019772 ...... PASSED ================================================================================
Finished 1 tests with the following results:
1 tests completed and passed residual checks, 
0 tests completed and failed residual checks,
0 tests skipped because of illegal input values. --------------------------------------------------------------------------------
End of Tests. ================================================================================ 
(END)
                              
```



