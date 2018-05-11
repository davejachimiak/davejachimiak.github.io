---
layout: post-no-feature
title: "A Leader Election Algorithm"
date: 2018-05-13
categories: languages
---

*Efficient Leader Election in Complete Networks* is a paper from a group of
computer scientists from the Universidad Publica de Navarra in Spain. It dropped
in 2005 in conference proceedings for the 13th Euromicro Conference on Parallel,
Distributed and Network-Based Processing.

<hr>

First of all, what's a complete network? It's is a network where any node it is
able to communicate directly with all other nodes.

Even though the algorithm is for such a network, the algorithm relies on the
nodes also being organized in a ring. A ring network is organized like a
linked-list that has a last item that points to the first item: each node only
knows about and communicates with a neighbor.

While it's not possible for a ring network to be a complete network, it is
possible for a complete network to organize itself into ring. Insofar as a ring
topology inside a complete network isn't *truly* a ring network, we and the
paper this post is based on call it a *virtual ring*.

The paper assumes that each node knows its neighbor in the virtual ring and
doesn't describe how a node can come to such information. But we can imagine
some scenarios where a node finding out which node is its neighbor would be
trivial. In complete networks that use the Internet Protocol (IP), where nodes
know about all other nodes, a node can order the adddresses of the nodes with
itself included and know that its neighbor is the next one down on the list. If
the node is the last in the list, its neighbor is at the top.

Or perhaps the nodes don't know every node in the network, but have some way of
find that out through some other machine — in the network or not — that's
responsible for knowing.

So, it helps to know that there are many paths to determining a *virtual ring*
with a *sense of direction* in a complete network. If you're using IP
addresses as unique identifiers for the nodes, getting a node to know its
neighbor is trivial.

<hr>

Second of all, what's leader election and why is it important?
