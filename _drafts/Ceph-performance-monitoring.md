---
layout: post
title: Ceph Performance Monitoring
---

Monitoring Ceph has proven more difficult than I originally anticipated. When I started work at
Canonical I had an idea of what data would be required to make an accurate recommendation of
what needed to be changed on a given Ceph cluster to improve performance.  Performance means
different things in different situations.  It could mean:
1. How many iops my cluster sustain to handle my small file operations.
2. How much throughput my cluster can sustain for my large file operations.
3. How many simultaneous clients can be supported
4. Read throughput for write once read many applications

There are other situations of course but I think this gets the point across.

So what makes Ceph more difficult than your average SAN?  Well partially the
problem lies in the amount of knobs Ceph exposes.  At last count on Ceph's
Jewel release there were approx [1103](https://gist.github.com/cholcombe973/e51db4674fe2fcb667fb4e44bd003e3b) different configuration options exposed through
the admin socket!  Some of them really move the needle but it's hard to tell
which one(s) matter for a given situation. Mark Nelson spent quit a lot of time
back in [2013](http://ceph.com/community/ceph-performance-part-1-disk-controller-write-throughput/) trying to [quantify](http://ceph.com/community/ceph-performance-part-2-write-throughput-without-ssd-journals/) that.

## Deployment Patterns
In my experience there's been 2 major patterns around Ceph and distributed
storage in general.  The first has been the shared cluster.  The goal here is
to cram as many clients as possible onto the same cluster to reduce cost. If you
have thousands of clients that only need the occasional GB or iops then this
makes perfect sense.

The second pattern I've seen is high performance or size requirements that justify creating a dedicated cluster for one client.  If you have a client that mentions they need 30GB/s write throughput it generally doesn't make sense to have them included in the shared pool.

What about a pattern which is in between these two extremes?  Well if you have
a client that isn't sure you can start them out in the shared pool and graduate
them to the dedicated cluster should their needs escalate.  I've done this before
and while it is slightly painful to migrate clients to a different pool it
generally saves on cost.

## Optimizing the Patterns

Lets say you are in situation one.  You have a medium size cluster with many
different clients all exhibiting different usage patterns. The sum of it all
is a base low level IO with occasional spikes that are not predictable. How do
you optimize a cluster with many different clients?  Generally you need to
try and get the aggregate IO size to be as large as possible.  Clustered storage
generally behaves badly with small IO sizes due to physics.  In the links above
they illustrate that as IO size increases you get a logarithmic graph.  
