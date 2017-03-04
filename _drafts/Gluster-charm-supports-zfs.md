---
layout: post
title: The Gluster charm supports ZFS now
---
Starting with Ubuntu 16.04, the Gluster charm can now setup bricks with ZFS.  How
does this work?  It's all rather easy from the deployment perspective.  Let's
take a look:

Let's deploy a 3 server cluster to Amazon EC2 with ZFS. 
First we need to write some quick yaml to set parameters:
```yaml
gluster:
  volume_name: "test"
  cluster_type: "DistributedAndReplicate"
  replication_level: 3
  filesystem_type: "zfs"
```
What this says is we want a 3x distributed and replicated Gluster cluster
deployed on ZFS formatted bricks.
Since I have the charm pulled down to my local filesystem I'm going to deploy
from there.  You could also deploy from the charm store and it wouldn't make
any difference.

This next command says deploy 3 machines using the gluster.yaml config file
we just wrote and give each machine 10GB of EC2 block storage:
```
juju deploy ./repos/gluster -n 3 --config ~/gluster.yaml --storage brick=10G --series xenial
```
With `juju debug-log` running you'll start to see a bunch of output.  In
particular we're looking for this:
  ```
unit-gluster-1: 16:04:09 INFO unit.gluster/1.juju-log Formatting block device with ZFS: "/dev/xvdf"
unit-gluster-1: 16:04:09 DEBUG unit.gluster/1.juju-log Installing zfs utilsunit-gluster-1: 16:04:09 DEBUG unit.gluster/1.juju-log Installing zfs utils
  ```
Running the mount command over ssh should show:
```
xvdf on /mnt/xvdf type zfs (rw,relatime,xattr,posixacl)
```

Zpool status over ssh:
```
zpool list
NAME   SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
xvdf  9.94G   164K  9.94G         -     0%     0%  1.00x  ONLINE  -
```

After a minute or two the cluster settles and we can query the status over ssh:
```
gluster vol info

Volume Name: chris
Type: Replicate
Volume ID: 5c5e85ad-e6e6-4e1d-8741-04748c5464b5
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: 172.31.8.199:/mnt/xvdf
Brick2: 172.31.27.173:/mnt/xvdf
Brick3: 172.31.38.13:/mnt/xvdf
Options Reconfigured:
cluster.favorite-child-policy: size
diagnostics.stats-dnscache-ttl-sec: 3600
diagnostics.stats-dump-interval: 2
diagnostics.fop-sample-interval: 1
diagnostics.count-fop-hits: On
diagnostics.latency-measurement: On
transport.address-family: inet
performance.readdir-ahead: on
nfs.disable: Off
```

Some of the advantages of deploying ZFS bricks include:
  1. Inline compression
  2. Snapshots
  3. PCI Express SSD L2ARC
  4. Data and metadata checksums for bitrot detection.

The code is available here: [Gluster Charm](https://github.com/cholcombe973/gluster-charm). Stop by, open some PR's
or file bugs.  Any and all help is greatly appreciated!
