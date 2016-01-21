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


