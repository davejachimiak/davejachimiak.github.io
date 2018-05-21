---
layout: post-no-feature
title: "Efficient Leader Election in Complete Networks"
date: 2018-05-13
categories: languages
---

*Efficient Leader Election in Complete Networks* describes an algorithm with a
time and message complexity linear to the number of nodes in the network. It
focuses on leader election and *only* leader election, leaving aside problems
like multiple leaders arising from network partitions, what triggers leader
election, and how non-leaders discover the identity of the new leader.

The main idea: __During an election, the *candidate* with the highest ID will
become the leader.__ Its complicated mechanics are all about making this happen.

The algorithm requires the nodes to be organized in a *virtual ring*, so that

* each node has a unique identifier that can be sort-ordered with other nodes' IDs,
* each node knows its `neighbor` node, and
* the `neighbor` must be the ID directly below the node's ID in the network.
  Exception: The last node's `neighbor` is the first node.

As with any useful leader election algorithm, it allows *passive* nodes to
become *candidates* at any moment, and many nodes to become *candidates* at the
same time.

Nodes carry state during an election. States are updated as nodes become
*candidates* and receive messages. Nodes receive messages from *neighbors* and
*candidates*.

A node maintains three variables: `status`, `candidate_successor`, and
`candidate_predecessor`.

The possible values for `status` are:

* *passive*
* *candidate*
* *dummy*
* *waiting*
* *leader*

Each node's default `status` is *passive*.

`candidate_successor` holds the node's most recently known successor that has
simultaneously become a *candidate*. The value is updated as messages are
received from other nodes.

`candidate_predecessor` holds the node's most recently known predecessor that
has simultaneously become a *candidate*. The value is updated as messages are
received from other nodes.

A node can spontaenously "wake up" for leader election. It does this by
executing an `initiate` function. This function sets `candidate_successor` and
`candidate_predecessor` to `NULL` and changes the `status` to *candidate*. It
then sends `message1` to `neighbor`.

### Messages

Each node can send and receive three messages. When a message is received, it
updates state based on its current state and the kind of message received.

Messages are sent under the following conditions:

#### `message1(uid initial_sender)`

`message1(current_node_uid)` is sent to `neighbor` when a node `initiate`s itself into
being a *candidate*.

When `message1(A)` is received by a node *B* and its `status` is *passive*, node
*B*'s `status` changes to *dummy* and sends `message1(A)` to node *B*'s
`neighbor`.

#### `message2(uid node_uid)`

`message2(uid)` can be sent to a node under a few conditions:

* A *candidate* node receives `message1` from another *candidate* node, and it
  has no `candidate_successor`.
* A *waiting* node receives `message3` from another node, it doesn't have a
  `candidate_successor`, and the `uid` passed in is before the curren't node's
  `uid` in the *virtual ring*.

#### `message3(uid sender)`

`message3(uid)` can be sent to a node under a few conditions:

* A *candidate* node receives `message1` from a another *candidate* node, and it
  has a `candidate_successor`.
* A *waiting* node receives `message3` from another node, and it has a
  `candidate_successor`.

## In Ruby

## Two-node example

Nodes spontaneously start the leader election algorithm. Each node has a unique
identifyer and knows its *neighbor*, which is successor in the *virtual ring*.
When a node starts the algorithm, it becomes a *candidate*. If more than one
node becomes a *candidate*, rules are used to sort out who should become the
leader. The algorithm uses the unique id of the nodes to determine which
candidate should become the leader; the highest (though it could be the lowest)
id of all spontaneous candidates should become the leader.

Let's start with an example of a network that current has two nodes. Both nodes
start in a *passive* state. If node *A* spontaneously became a *candidate*, it
then sends a message with "*A*" as its payload to its *neighbor*, Node *B*. If
Node *B* receives the message and it's *passive*, then it changes its state to
*dummy*, and forwards the message it its *neighbor* which is Node *A*. Since
node *A* recieves the message with *A* in the payload, it knows that its initial
message to node *B* has been passed back to it, and it therefore knows that its
the leader.

What happens when both nodes simultaneously become candidates? When node *A*
receives message from *B*, *B* is in the payload. Since it's state is
*candidate*, we have to do something a little different. Now node *A* knows that
*B* is its *candidate predecessor* and updates its (`cand_pred`) variable to
*B*. Since node *B* comes before *A* in the logical ring, *A* then sets its
state to *waiting* and sends a message, *message II*, to node *B* with *A* in
the payload.

When node *B* receives *message* it changes its `cand_pred` variable to *A*.
Since *A* does not come before *B* in the logical ring, it doesn't do anything
else.

*message II* hits node *B* with they payload *A*. When a node gets *message II*,
it inspects the payload. If its status is *candidate*, which node *B* is in this
case since its state didn't change in *message*, it now knows that it can't be
the leader, so it sends another kind of message, *message III*, to node *A* with
`cand_pred` — *A* — in the payload.

When *message III* hits node A, it now knows that it's the leader. Three
conditions need to be met for a node to know that it's the leader if multiple
candidates spontaneously become *candidate*s. If it's 1.) hit with *message
III*, 2.) in the *waiting* state, and 3.) the payload of the message is the same
as its unique node id.

<hr>


