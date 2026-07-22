---
title: Raft
description: An understandable consensus algorithm for replicated state machines
tags:
  - algorithm
  - distributed-systems
---

Raft is a consensus algorithm for keeping a replicated state machine consistent across
a cluster of servers. Each server applies the same commands in the same order, so the
cluster behaves like one reliable machine even when a minority of its servers fail.[^1]

Raft was designed as a more understandable alternative to Paxos. It divides consensus
into separate problems, most importantly leader election and log replication, while
providing equivalent fault tolerance and performance.[^1]

![The Raft consensus algorithm mascot](raft-mascot.png)

## Server states

Each server is in one of three states:

- **Follower**: accepts log entries and heartbeats from the leader.
- **Candidate**: asks the other servers for votes when it believes the leader has
  failed.
- **Leader**: accepts client commands and replicates them to the followers.

Time is divided into numbered **terms**. A term begins with an election and normally
continues under one leader. A server returns to the follower state when it learns of a
newer term.

## Leader election

The leader sends regular heartbeats. If a follower does not receive one before its
randomized election timeout, it becomes a candidate, increments the term, votes for
itself, and requests votes from the other servers. A candidate that receives votes
from a majority becomes the leader.[^2]

Randomized timeouts make it less likely that several followers start an election at
once. If no candidate wins, the servers start another term and try again.

## Log replication

The leader handles every client command in the same sequence:

1. Append the command to its own log.
2. Send the entry to the followers.
3. Commit the entry once a majority has stored it.
4. Apply the committed command to the state machine and tell the followers to do the
   same.

A five-server cluster can therefore continue with two failed servers. It stops making
progress when it loses its majority, but it does not commit a conflicting result.[^1]

The leader also repairs inconsistent follower logs. It finds the last entry where the
two logs agree, removes the follower's conflicting suffix, and sends the missing
entries again. Raft's voting rules prevent a server with an outdated log from becoming
leader and losing already committed commands.[^2]

Raft assumes servers may crash or become unreachable; it does not protect against
servers that lie or act maliciously. In other words, it is crash fault tolerant, not
Byzantine fault tolerant.[^2]

[^1]: Ongaro, Diego, and John Ousterhout. “The Raft Consensus Algorithm.” <https://raft.github.io/>.

[^2]: “Raft (algorithm).” Wikipedia. <https://en.wikipedia.org/wiki/Raft_(algorithm)>.
