+++
author = "GreyWind"
title = "Raft Paper"
date = "2023-08-31"
description = "Raft Paper"
tags = [
    "MIT-6.824",
]
image = "4edbcd64-afa3-4eff-8d34-2624bf875149.webp"
+++
## Introduction

Raft is a consensus algorithm for managing a replicated log. Consensus algorithms allow a collection of machines to work as a coherent group that can survive the failures of some of its members. Although Paxos can solve consensus, it is indigestible for most people. Raft enhance understandability and also provides a better foundation for building practical system.

This paper called [In Search of an Understandable Consensus Algorithm(Extended Version)](https://raft.github.io/raft.pdf) was published in 2014 by Stanford University. This paper is very in-depth, I recommend reading it several times.

## Replicated state machines

Consensus algorithms typically arise in the context of ***replicated state machines***. Replicated state machines are typically implemented using a replicated log. The consensus algorithms manage a replicated log containing state machine commands from clients. The state machines process identical sequences of commands from the logs so that they produce the same outputs.

## The Raft consensus algorithm

Raft decomposes the consensus problem into three relatively independent subproblems:

* **Leader election: A new leader must be chosen when an existing leader fails**
    
* **Log replication: The leader must accept log entries from clients and replicate them across the cluster, forcing the other logs to agree with their own.**
    
* **Safety: If any server has applied a particular log entry to its state machine, then no other server may apply a different command for the same log index.**
    

### Raft basics

At any time Raft cluster, each server is in one of three states: ***leader***, ***follower***, or ***candidate***. In normal operation there is exactly one leader and all of the other servers are followers.

***Followers*** are passive: They issue no requests on their own but simply respond to requests from ***leaders*** and ***candidates***. The ***leader*** handles all client requests. The ***candidate*** elects a new leader.

Raft divides time into `terms` of arbitrary length. Each term begins with an election, in which one or more candidates attempt to become ***leader***. If a ***candidate*** wins the election, then it serves as a ***leader*** and other servers become ***followers***.

Each server stores a `current term` number, which increases monotonically over time. Current terms are exchanged whenever servers communicate; if one server’s current term is smaller than the other’s, then it updates its current term to the larger value. If a candidate or leader discovers that its term is out of date, it immediately reverts to ***follower*** state. If a server receives a request with a stale term number, it rejects the request.

Raft servers communicate using remote procedure calls, and the basic consensus algorithm requires only two types of RPCs. `RequestVote RPCs` are initiated by ***candidates*** during elections, and `AppendEntries RPCs` are initiated by ***leaders*** to replicate log entries and to provide a form of heartbeat.

### Leader election

A server remains in ***follower*** state as long as it receives valid RPCs from a ***leader*** or ***candidate***. ***Leaders*** send periodic heartbeats to all ***followers*** to maintain their authority. If a ***follower*** receives no communication over a period called the *election timeout*, then it assumes there is no viable ***leader*** and begins an election to choose a new ***leader***.

To begin an election, a ***follower*** increments its `current term` and transitions to ***candidate*** state. It then votes for itself and issues `RequestVote RPCs` in parallel to each of the other servers in the cluster. A ***candidate*** continues in this state until one of three things happens:

* (a) it wins the election
    
* (b) another server establishes itself as a leader
    
* (c) a period goes by with no winner.
    

> Raft uses randomized election timeouts to ensure that split votes are rare and that they are resolved quickly. To prevent split votes in the first place, election timeouts are chosen randomly from a fixed interval (e.g., 150–300ms).

### Log replication

Each client request contains a command to be executed by the replicated state machines. The ***leader*** appends the command to its log as a new entry, then issues `AppendEntries RPCs` in parallel to each of the other servers to replicate the entry. When the entry has been safely replicated (as described below), the ***leader*** applies the entry to its state machine and returns the result of that execution to the client.

Each log entry stores a state machine command along with the term number when the entry was received by the ***leader***. Each log entry also has an integer index identifying its position in the log.

Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines. A log entry is committed once the ***leader*** that created the entry has replicated it on a majority of the servers. This also commits all preceding entries in the ***leader***’s log, including entries created by previous ***leaders***. The ***leader*** keeps track of the highest index it knows to be committed, and it includes that index in future `AppendEntries RPCs` (including heartbeats) so that the other servers eventually find out. Once a ***follower*** learns that a log entry is committed, it applies the entry to its local state machine (in log order).

Raft maintains the following properties, which together constitute the Log Matching Property:

* If two entries in different logs have the same `index and term`, then they store the same command.
    
* If two entries in different logs have the same `index and term`, then the logs are identical in all preceding entries.
    

When sending an `AppendEntries RPC`, the ***leader*** includes the `index and term` of the entry in its log that immediately precedes the new entries. If the follower does not find an entry in its log with the same index and term, then it refuses the new entries.

To bring a ***follower***’s log into consistency with its own, the leader must find the latest log entry where the two logs agree, delete any entries in the follower’s log after that point, and send the follower all of the leader’s entries after that point. All of these actions happen in response to the consistency check performed by `AppendEntries RPCs.` The leader maintains a `nextIndex` for each ***follower,*** which is the index of the next log entry the leader will send to that follower. When a ***leader*** first comes to power, it initializes all `nextIndex` values to the index just after the last one in its log. If a follower’s log is inconsistent with the leader’s, the AppendEntries consistency check will fail in the next `AppendEntries RPC`. After a rejection, the leader decrements `nextIndex` and retries the `AppendEntries RPC`.

### Safety

The previous sections described how Raft elects leaders and replicates log entries. However, the mechanisms described so far are not quite sufficient to ensure that each state machine executes the same commands in the same order. For example, a ***follower*** might be unavailable while the ***leader*** commits several log entries, then it could be elected ***leader*** and overwrite these entries with new ones; as a result, different state machines might execute different command sequences.

#### Election restriction

Raft uses the voting process to prevent a ***candidate*** from winning an election unless its log contains all committed entries. A candidate must contact a majority of the cluster to be elected, which means that every committed entry must be present in at least one of those servers. If the candidate’s log is at least as up-to-date as any other log in that majority (where “up-to-date” is defined precisely below), then it will hold all the committed entries.

#### Commiting entries from previous terms

Once a log entry for the current term is accepted by the majority of machines(half), the ***Leader*** commits it. If the ***Leader*** dies while submitting the entry, the next Leader will continue to try to complete the copy of the entry. However, the ***Leader*** cannot immediately confirm that an entry from a previous term has been committed, even if it is already stored on most machines.

Therefore, Raft does not attempt to submit log entries from the previous term by checking the number of copies. Only the number of copies is checked to submit a log entry for the `current term`. When an entry for the current term is submitted, all entries currently in that entry are implicitly submitted.

## Cluster membership changes

For the configuration change mechanism to be safe, there must be no point during the transition where two leaders can be elected for the same term.

To ensure the security of node changes, Raft adopts a two-stage approach. First, the cluster switches to a joint consensus state, and after the joint consensus is committed, the system switches to the new configuration.

* Log entries are replicated to all servers in both configurations.
    
* Any server from either configuration may serve as a leader.
    
* The agreement requires separate majorities from *both* the old and new configurations.
    

The cluster configuration uses special entry storage and transport in the log. The process is as follows:

1. The Leader receives a request and switches the cluster configuration from `C_old` to `C_new`
    
2. The Leader stores `C_old` and `C_new` as the joint consensus `C_old`, new configuration in a log entry
    
3. Append this log entry to all machines with the old and new configurations.
    
4. When a machine receives such a log entry and adds it to its logs (no submission is required), all subsequent operations are performed based on this configuration.
    
5. When `C_old`, new is accepted by most machines, the Leader commits it. In this case, any machine configured with `C_old` or `C_new` cannot be selected as the Leader.
    
6. The Leader creates a log entry for `C_new`, appends it to all machines, and submits it
    

The following problems still exist when the cluster member changes:

1. The new machine does not store any logs when it joins the cluster, and it takes a while to catch up with the old machine, which may reduce the availability of the cluster in a short period. When adding nodes to a cluster, Raft introduces a new phase in which new machines can normally receive additional requests, but are not considered voting nodes, and consensus can be reached without considering these new nodes. Perform the preceding operations when the logs of the new node catch up with the progress of the old node.
    
2. The Leader of the cluster may not be part of the new configuration. In this case, when the Leader submits `C_new`, it should itself be pulled down. This results in a period when the Leader manages a cluster that does not contain itself and replicates logs but does not consider itself a principal.
    
3. Those servers that are taken down may break the cluster, and since these machines cannot receive heartbeats, elections may be held. A canvassing request with a new term is then sent, causing the cluster Leader to revert to the Follower. Although the election will not succeed in this case, the new Leader will still be generated in the new cluster, but the removed machine will still time out the election, resulting in poor overall availability.
    

> To prevent problem 3 from happening, Raft adds a restriction: If the server receives a vote request within the timeout period of receiving a heartbeat from the current Leader, it will not renew the term and vote. This allows a Leader not to be ousted by a larger term vote as long as he can maintain the heartbeat of the current cluster.

## Log compaction

As logs grow, the machine cannot store all logs in memory, so snapshots need to be introduced to periodically keep the state of the system in persistent storage so that logs from the start to the snapshot point are safely removed from memory.

Each machine in the cluster independently manages its snapshot, which contains only the committed log entries in its log. In addition to saving the current state of the state machine, you also need to save some metadata:

* Log index of the last log entry included in the snapshot
    
* Log term of the last log entry included in the snapshot
    

This metadata is mainly used for continuity checks on append requests (where the previous log entries need to be compared). If you want to support the cluster node changes mentioned above, the snapshot must also contain the latest configuration information at the snapshot point. After the snapshot is written, all logs and historical snapshots before the snapshot point are deleted.

Occasionally, when the ***Leader*** needs to synchronize its logs to some newly added or backward nodes, it needs to send snapshots. The ***leader*** will send `installSnapshotRPC`:

```go
type SnapshotRequest struct {
    term // leader term
    leaderId // so follower can redirect clients
    lastIncludedIndex // the snapshot replaces all entries up through and including this inedex
    lastIncludedTerm // term of lastIncludeIndex
    
    offset // byte offset where chunk is positioned in the snapshot file
    data[] // raw bytes of the snapshot chunk, staring at offset
    done // true if this is the last chunk
}

type SnapshotReply struct {
    term // currentTerm, for leader to update itself
}
```

1. Reply immediately if `term < currentTerm`
    
2. Create new snapshot file if first chunk (offset is 0)
    
3. Write data into snapshot file at given offset
    
4. Reply and wait for more data chunks if done is false
    
5. Save snapshot file, discard any existing or partial snapshot with a smaller index
    
6. If existing log entry has same index and term as snapshot’s last included entry, retain log entries following it and reply
    
7. Discard the entire log
    
8. Reset state machine using snapshot contents (and load snapshot’s cluster configuration)
    

## Raft Implementation

server

```go
type Raftnode struct {
    // persisten state on all servers
    currentTerm // latest term server has seen (initialized to 0 on first boot, increases monotonically)
    votedFor // candidateId that received vote in current term (or null if none)
    log[] // log entries; each entry contains command for state machine, and term when entry was received by leader (first index is 1)
    
    // volatile state on all servers
    commitIndex // index of highest log entry known to be committed (initialized to 0, increases monotonically)
    lastApplied // index of highest log entry applied to state machine (initialized to 0, increases monotonically)
    
    // volatile state on leaders
    nextIndex[] // for each server, index of the next log entry to send to that server (initialized to leader last log index + 1)
    matchIndex[] // for each server, index of highest log entry known to be replicated on server (initialized to 0, increases monotonically)
}
```

AppenEntries

```go
type AppendEntriesRequest struct {
    term // leader’s term
    leaderId // so follower can redirect clients
    prevLogIndex //index of log entry immediately preceding new ones
    prevLogTerm // term of prevLogIndex entry
    entries[] // log entries to store (empty for heartbeat; may send more than one for efficiency)
    leaderCommit // leader’s commitIndex
}

type AppendEntriesReply struct {
    term // currentTerm, for leader to update itself
    success // true if follower contained entry matching prevLogIndex and prevLogTerm
}
```

1. Reply false if `term < currentTerm`
    
2. Reply false if log doesn’t contain an entry at `prevLogIndex` whose term matches `prevLogTerm`
    
3. If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it
    
4. Append any new entries not already in the log
    
5. If `leaderCommit > commitIndex`, set `commitIndex = min(leaderCommit, index of last new entry)`
    

RequsetVote

```go
type RequestVoteRequest struct {
    term // candidate’s term
    candidateId // candidate requesting vote
    lastLogIndex // index of candidate’s last log entry (§5.4)
    lastLogTerm // term of candidate’s last log entry (§5.4)
}

type RequsetVoteReply struct {
    term // currentTerm, for candidate to update itself
    voteGranted // true means candidate received vote
}
```

1. Reply false if `term < currentTerm`
    
2. If votedFor is null or `candidateId,` and ***candidate***’s log is at least as up-to-date as receiver’s log, grant vote
    

**Rules for Servers**

All Servers:

* If `commitIndex` &gt; `lastApplied`: increment `lastApplied`, apply `log[lastApplied]` to state machine
    
* If RPC request or response contains `term T` &gt; `currentTerm`: set `currentTerm = T`, convert to `follower`
    

Followers:

* Respond to RPCs from candidates and leaders
    
* If `election timeout` elapses without receiving `AppendEntries RPC` from current leader or granting the vote to the candidate: convert to `candidate`
    

Candidates:

* On conversion to candidate, start election:
    
    Increment `currentTerm`
    
    Vote for `self`
    
    Reset election `timer`
    
    Send `RequestVote RPCs` to all other servers
    
* If votes received from majority of servers: become `leader`
    
* If AppendEntries RPC received from new leader: convert to `follower`
    
* If election `timeout` elapses: start new election
    

Leaders:

* Upon election: send initial empty AppendEntries RPCs (heartbeat) to each server; repeat during idle periods to prevent election timeouts
    
* If command received from client: append entry to local log, respond after entry applied to state machine
    
* If `last log index` ≥ `nextIndex` for a `follower`: send AppendEntries RPC with log entries starting at `nextIndex`
    
* If successful: update `nextIndex` and `matchIndex` for `follower`
    
* If AppendEntries fails because of log inconsistency: decrement `nextIndex` and retry
    
* If there exists an N such that `N > commitIndex`, a majority of `matchIndex[i] ≥ N`, and `log[N].term == currentTerm`: set `commitIndex = N`.