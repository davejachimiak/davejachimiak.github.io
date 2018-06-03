---
layout: post-no-feature
title: "Distributed Systems Notes"
date: 2018-06-03
categories: distributed-systems
---

I came across some unfamiliar terms and ideas when I read and [wrote
about](/2018/05/a-leader-election-algo.html) a paper called *[Efficient Leader
Election in Complete
Networks](https://ieeexplore.ieee.org/document/1386052/?reload=true)*. The
following is a chunk I took out of that post that was going to be background or
introductory material. I'll probably update this post if I decide to write more
about distributed systems on this blog.

### Complete Network

A complete network is network where any node is *able* to communicate directly
with all other nodes.

### Ring Network

A ring network is organized like a linked list whose last node points to the
first; each node only knows about and communicates with its neighbor.

While it’s not possible for a ring network to be a complete network, it is
possible for a complete network to organize itself into a ring. This is called a
*virtual ring* insofar as the complete network that contains the ring is
probably not operating like a ring network for all of its distributed
operations. In complete networks that use the Internet Protocol (IP) a node can
order the addresses of the nodes in the network and know that its *neighbor* is
next in the list from itself. If the node is the last in the list, its neighbor
is at the top.

### Edge

An edge is a connection between two nodes.

### Cycle

A cycle is a series of edges that form a path from a node back to itself.

### Chord

A chord is an edge of between two nodes that's not part of a cycle.

### Sense of direction

A network or part of a network has a sense of direction if there's a distinct
way that messages flow from node to node. For example, in a [ring
network](#ring-network) the sense of direction is from *neighbor* to *neighbor*.

### Why’s leader election a thing?

One goal in a distributed database with no data partitioning is for all nodes in
the network to eventually agree on the values of the data. If two database
clients try to simultaneously update the same piece of data and nothing
coordinates those updates, then the network will likely become confused about
the piece of data's value after that point. The way in which it will be confused
depends on the replication strategy of the database.

Having a coordinating process — a leader — helps eliminate confusion about data
in the network.

When initiating a network, it’s easy enough to manually appoint some node as the
leader. But what happens if the leader fails or otherwise becomes unavailable?

This is where leader election comes in. A leader election algorithm kicks off in
replicas when their heuristics for knowing when the leader becomes unavailable
trigger one. One necessary feature of fully distributed, leader election
algorithms is that replicas should be able to kick off their part of the
algorithm spontaneously and independent of the state of other replicas.

One such algorithm is *[Efficient Leader Election in Complete
Networks](/2018/05/a-leader-election-algo.html)*. There are problems that arise
from network partions that aren’t addressed by the paper. For example, if
there’s a network {A..H}, Node E is the leader, and there's a network partition
such that replicas A..E can reach the leader, but replicas F..H cannot, then
replicas F..H will choose a new leader among themselves, leading to 2 leaders in
the network. These kinds of problems are addressed in more fully-described
consensus protocols, like [Raft](https://raft.github.io/raft.pdf).
