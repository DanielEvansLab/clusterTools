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
  host = marge
  port = 8649
}
/* You can specify as many udp_recv_channels as you like as well. */ 
udp_recv_channel {
  port = 8649
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






