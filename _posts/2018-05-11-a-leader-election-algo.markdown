---
layout: post-no-feature
title: "Efficient Leader Election in Complete Networks"
date: 2018-05-27
categories: distributed-systems
---

[*Efficient Leader Election in Complete
Networks*](https://ieeexplore.ieee.org/document/1386052/?reload=true) describes
an algorithm with a time and message complexity linear to the number of nodes in
the network. It focuses only on leader election, leaving aside what triggers
leader election and how non-leaders discover the identity of the new leader.

Main idea: __The *candidate* with the highest ID will become the leader during
an election.__

The algorithm requires the nodes to be organized in a *virtual ring*:

* each node has a unique identifier that can be sort-ordered with other nodes' IDs,
* each node knows its `neighbor` node, and
* the `neighbor` must be the ID directly below the node's ID in the network.
  Exception: The last node's `neighbor` is the first node.

For example, node-neighbor organization is like this if a network has nodes
`{A..E}`:

```
NodeA = { neighbor = NodeB }
NodeB = { neighbor = NodeC }
NodeC = { neighbor = NodeD }
NodeD = { neighbor = NodeE }
NodeE = { neighbor = NodeA }
```

The algorithm allows *passive* nodes to become *candidates* at any moment and
many nodes to become *candidates* at the same time.

Nodes carry state during an election. They update state as they become
*candidates* and receive messages. Nodes receive messages from *predecessors*
and *candidates*.

A node maintains three pieces of state: `status`, `candidate_successor`, and
`candidate_predecessor`.

The possible values for `status` are:

* *passive*
* *candidate*
* *dummy*
* *waiting*
* *leader*

Each node's initial `status` is *passive*.

`candidate_predecessor` holds the most recently known *candidate* that comes
before the node. When a *candidate* node holds this value, it knows that
everything between the `candidate_predecessor` and itself in the sense of
direction of the `candidate_predecessor`'s *neighbor* is a *dummy*.

`candidate_successor` holds the most recently known *candidate* that's higher
ranked than the node. Since lower ranked *candidates* cannot become the
*leader*, *candidates* that have a `candidate_successor` value will eventually
become a *dummy*.

It gets to know `candidate_successors` and `predecessors` through 3 kinds of
messages: `message1`, `message2`, and `message3`.

A node can spontaneously "wake up" for leader election. It does this by
executing an `initiate` function. This function sets `candidate_successor` and
`candidate_predecessor` to `NULL` and changes the `status` to *candidate*. It
then sends `message1` to `neighbor`.

`message1` and `message3` come from `candidate_predecessor`s and *neighbors*,
and `message2` comes from `candidate_successor`s. A *candidate* can't be a
*leader* if it has a `candidate_predecessor` higher than its own ID. But if
that's a fact, the node doesn't immediately become a *dummy*; it still has work
to do to let the *candidate* with the highest ID know that it is the *leader*.

There are two ways a node can become a leader. The first is if its the only
*candidate* in an election. A node knows that its the only *candidate* in an
election if it receives `message1(sendingNodeId)` and `sendingNodeId` is its own
ID. This means that the `message1(sendingNodeId = ownId)` message it sent to its
*neighbor* got all the way around the ring and back to itself without hitting
any other *candidates*.

The second way is if the node is the highest ID *candidate* of the election.
Nodes sending and receiving `message2` and `message3` help let the highest ID'd
*candidate* know that it's the *leader*. A node is the leader if it receives
`message3(new_candidate_predecessor)` and `nodeId` is its own ID.

The algorithm achieves the second way with cleverness; it's not an intuitive
process. Let's break down how nodes set state and send messages when the receive
messages. The following code snippets are in Ruby and only show essential parts
of the algorithm. Constant meanings should be self-explanatory and `@id` is the
ID of the node.

### `message1(new_candidate_predecessor)`

```ruby
def message1(new_candidate_predecessor)
  case @status
  when PASSIVE
    @status = DUMMY
    send_message(@neighbor, :message1, new_candidate_predecessor)
  when CANDIDATE
    @candidate_predecessor = new_candidate_predecessor

    if (@id == new_candidate_predecessor)
      @status = LEADER
    elsif (@id > new_candidate_predecessor)
      if @candidate_successor
        @status = DUMMY
        send_message(@candidate_successor, :message3, new_candidate_predecessor)
      else
        @status = WAITING
        send_message(new_candidate_predecessor, :message2, @id)
      end
    end
  end
end
```

If the node hasn't "woken up," it becomes a `DUMMY` and passes the
`new_candidate_predecessor` to its `@neighbor`.

If the node *has* "woken up" — i.e. it's currently a *candidate*—, it saves the
`new_candidate_predecessor` as the `@candidate_predecessor`. What happens next depends on
how the node's `@id` relates to the `new_candidate_predecessor`.

If the `@id` is the `new_candidate_predecessor`, that means the `:message1` originally
sent by `@id` has made its way all around the ring with no other *candidates*.
Now it knows itself as the *leader*.

If the `@id` is higher than the `new_candidate_predecessor`, this means that the
message made itself "around" the ring; it crossed the imaginary border between
the lowest and highest ID'd nodes. This also means that `@id` is a candidate
successor to `new_candidate_predecessor`; `new_candidate_predecessor` is lower
ranked *candidate* than `@id`. Knowing this, we set the node's status to
`WAITING` — a way of letting other received messages know that the node already
received `message1` — and send `message2(@id)` to `new_candidate_predecessor`.
This is the way the algorithm marks the `new_candidate_predecessor`'s
`@candidate_successor` with `@id` and eventually/or become a `DUMMY`.

### `message2(new_candidate_successor)`

```ruby
def message2(new_candidate_successor)
  case @status
  when CANDIDATE
    if @candidate_predecessor
      send_message(new_candidate_successor, :message3, @candidate_predecessor)
      @status = DUMMY
    else
      @candidate_successor = new_candidate_successor
    end
  when WAITING
    @candidate_successor = new_candidate_successor
  end
end
```

Only *waiting* candidate nodes and whose IDs are higher send `message2` to nodes
ID'd *candidates*. In other words, if a message receives `message2`, it knows it
can't be the leader. And since `message2` is the only place where
`@candidate_successor`s are set, then `message1` and `message3` know that if the
node has a `@candidate_successor`, then they can safely mark the node as a
*dummy* before sending off a `message3` to the `@candidate_successor`.

### `message3(new_candidate_predecessor)`

```ruby
def message3(new_candidate_predecessor)
  case @status
  when WAITING
    @candidate_predecessor = new_candidate_predecessor

    if @id == new_candidate_predecessor
      @status = LEADER
    else
      if @candidate_successor
        @status = DUMMY
        send_message(@candidate_successor, :message3, new_candidate_predecessor)
      else
        if @id > new_candidate_predecessor
          send_message(new_candidate_predecessor, :message2, @id)
        end
      end
    end
  end
end
```

If a node message receives `message3(new_candidate_predecessor)` —
`new_candidate_predecessor` being the candidate predecessor to the node that
sent the message —, the node is essentially being told that, now, this
`new_candidate_predecessor` should replace the `@candidate_predecessor`. What
does this mean? To the node it means now all nodes from `@candidate_predecessor`
to itself in the sense of direction of the `@candidate_predecessor`'s neighbor
are dummies. 

If the `@id` is higher than the `new_candidate_predecessor`, this means that the
message made itself "around" the ring; it crossed the imaginary border between
the lowest and highest ID'd nodes. This also means that the
`new_candidate_predecessor` is a candidate successor — a lower ranked
*candidate* —  to `@id`. We send `message2(@id)` to `new_candidate_predecessor`.
This is the way the algorithm marks the `new_candidate_predecessor`'s
`@candidate_successor` with `@id` and eventually/or become a `DUMMY`.

If the `@id` is the `@candidate_predecessor`, then all nodes from itself to
itself are *dummies*; it is the *leader*.
