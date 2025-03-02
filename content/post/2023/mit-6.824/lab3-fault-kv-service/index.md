+++
author = "GreyWind"
title = "Lab 3: Fault-tolerant Key/Value Service"
date = "2023-09-30"
description = "Lab 3: Fault-tolerant Key/Value Service"
tags = [
    "MIT-6.824",
]
+++
## Introduction

In lab3 we will build a fault-tolerant key/value storage service using your **Raft library** from Lab2. This service will be a replicated state machine, consisting of several key/value servers that use Raft for replication. This lab has two parts. In 3A, we will implement a key/value service using Raft, but without using snapshots. In 3B, we will use a snapshot from Lab2D, which allows Raft to discard old log entries.

Before starting the lab, you should review the extended ***Raft paper***, in particular Sections 7 and 8.

## Experience Description

To get started lab3, read the experiment document:

> [https://pdos.csail.mit.edu/6.824/labs/lab-kvraft.html](https://pdos.csail.mit.edu/6.824/labs/lab-kvraft.html)

Clients can send three different RPCs to the key/value service: `Put(key, value)`, `Append(key, arg)`, and `Get(key)`. Each client talks to the service through a `Clerk` with `Put/Append/Get` methods. A `Clerk` manages RPC interactions with the servers.

An important thing is Linearizability. Linearizability is convenient for applications because it's the behavior you'd see from a single server that processes requests one at a time. It is harder if the service is replicated since all servers must choose the same execution order for concurrent requests, must avoid replying to clients using state that isn't up to date, and must recover their state after a failure in a way that preserves all acknowledged client updates.

All code and tests are in `src/kvraft`. We will need to modify `kvraft/client.go`, `kvraft/server.go`, and perhaps `kvraft/common.go`.

## Implementation

### Linearizability

To make sure of the Linearizability, the paper gives a solution.

> For example, if the leader crashes after committing the log entry but before responding to the client, the client will retry the command with a new leader, causing it to be executed a second time.
> 
> The solution is for clients to assign unique serial numbers to every command. Then the state machine tracks the latest serial number processed for each client, along with the associated response.

If it receives a command whose serial number has already been executed, it responds immediately without re-executing the request.

### Client (3A)

To make it easier to send RPCs, combine three different RPCs (Get/Put/Append) into one. So, we should modify the `common.go`.

```go
type CommandArgs struct {
    Op string // Get/Put/Append
    Key string // Get/Put/Append
    Value string // Put/Append
    
    CommandId int // linearizability
    ClientId int64 // linearizability
}

type CommandReply struct {
    Err Err // Put/Append
    Value string // Get
}
```

Then for `Clerk` client, we should make it send Command RPCS to each server which includes ***raft note***. To assure Linearizability, we use ClientId and CommandId to mark a Clerk client.

```go
type Clerk struct {
    Leader Id int
    CommandId int
    ClientId int64
}

func (ck *Clerk) Command(args *CommandArgs) string {
	args.ClientId = ck.ClientId
	args.CommandId = ck.CommandId
	LeaderId := ck.LeaderId

	for {
		reply := CommandReply{}
		ok := ck.servers[LeaderId].Call("KVServer.Command", args, &reply)
		if ok {
			switch reply.Err {
			case OK:
				ck.LeaderId = LeaderId
				ck.CommandId++
				return reply.Value
			case ErrNoKey:
				ck.LeaderId = LeaderId
				ck.CommandId++
				return ""
			}
		}
		LeaderId = (LeaderId + 1) % len(ck.servers)
	}
}

func (ck *Clerk) Get(key string) string {
	// You will have to modify this function.
	return ck.Command(&CommandArgs{Key: key, Value: "", Op: "Get"})
}

func (ck *Clerk) PutAppend(key string, value string, op string) {
	// You will have to modify this function.
	ck.Command(&CommandArgs{Key: key, Value: value, Op: op})
}

func (ck *Clerk) Put(key string, value string) {
	ck.PutAppend(key, value, "Put")
}
func (ck *Clerk) Append(key string, value string) {
	ck.PutAppend(key, value, "Append")
}
```

### Server (3A)

As the paper, the state machine should track the latest serial number processed for each client, along with the associated response. So using `Client2ComId map[int64]int` and `ComNotify map[int]chan Op` record them.

```go
type KVServer struct {
    LastApplied int
    StateMachine KVStateMachine
    
    Client2ComId map[int64]int
    ComNotify map[int] chan Op
}
```

Using `KVStateMachine` to Describe the `kv` storage structure (like a database).

```go
type KVStateMachine interface {
	Get(key string) (string, Err)
	Put(key, value string) Err
	Append(key, value string) Err
}

// kv datebase implement kvstatemachine
type MemoryKV struct {
	KV map[string]string
}

func (kv *MemoryKV) Get(key string) (string, Err) {
	value, ok := kv.KV[key]
	if ok {
		return value, OK
	}
	return "", ErrNoKey
}
func (kv *MemoryKV) Put(key, value string) Err {
	kv.KV[key] = value
	return OK
}
func (kv *MemoryKV) Append(key, value string) Err {
	kv.KV[key] += value
	return OK
}
```

When the server receives an RPC from the `Client`, it should sync it by ***Raft library***, then wait until the raft library sends a message by channel. The `command method` used to sync by ***raft***, the `apply function` listens to channel message and apply to `KVStateMachine`.

```go
func (kv *KVServer) Command(args *CommandArgs, reply *CommandReply) {
	if kv.killed() {
		reply.Err = ErrWrongLeader
		return
	}
	kv.mu.Lock()

	// put and append re-executing
	if args.Op != "Get" && kv.ClientId2ComId[args.ClientId] >= args.CommandId {
		reply.Err = OK
		kv.mu.Unlock()
		return
	}
	kv.mu.Unlock()

	op := Op{
		Key:       args.Key,
		Value:     args.Value,
		Command:   args.Op,
		CommandId: args.CommandId,
		ClientId:  args.ClientId,
	}
    // using raft library to sync log
	index, _, isLeader := kv.rf.Start(op)
	if !isLeader {
		reply.Err = ErrWrongLeader
		return
	}


	kv.mu.Lock()
	ch := kv.GetChan(index)
	kv.mu.Unlock()
    // waiting message from raft
	select {
	case apply := <-ch:
		if apply.ClientId == op.ClientId && apply.CommandId == op.CommandId {
            // Get
			if args.Op == "Get" {
				kv.mu.Lock()
				reply.Value, reply.Err = kv.StateMachine.Get(apply.Key)
				kv.mu.Unlock()
			}
            // Put or Append
			reply.Err = OK
		} else {
            // timeout, it should re-executing
			reply.Err = TimeOut
		}
	case <-time.After(time.Millisecond * 33):
		reply.Err = TimeOut
	}
    // delete channel
	go func() {
		kv.mu.Lock()
		delete(kv.ComNotify, index)
		kv.mu.Unlock()
	}()
}

func (kv *KVServer) apply() {
	for !kv.killed() {
		select {
		// server get applych from raftnote
		case ch := <-kv.applyCh:
			// command sycn success
			if ch.CommandValid {
				kv.mu.Lock()
				if ch.CommandIndex <= kv.LastApplied {
					kv.mu.Unlock()
					continue
				}

				kv.LastApplied = ch.CommandIndex
				opchan := kv.GetChan(ch.CommandIndex)
				op := ch.Command.(Op)

				// apply to stateMachine(kvdatebase)
				if kv.ClientId2ComId[op.ClientId] < op.CommandId {
					kv.applyStateMachine(&op)
					kv.ClientId2ComId[op.ClientId] = op.CommandId
				}
				kv.mu.Unlock()

				opchan <- op
			}
		}
	}
}
```

### SnapShot (3B)

The snapshot must include the `KVStateMachine (database)`. The `ClientLd2ComId` also should be stored, which can avoid command execution a second time.

```go
func (kv *KVServer) apply() {
	for !kv.killed() {
		select {
		// server get applych from raftnote
		case ch := <-kv.applyCh:
			// an apply
			if ch.CommandValid {

                .....
				if kv.maxraftstate != -1 && kv.rf.GetRaftStateSize() > kv.maxraftstate {
					kv.rf.Snapshot(ch.CommandIndex, kv.PersisterSnapshot())
				}
                .....

			}

			// a snap
			if ch.SnapshotValid {
				kv.mu.Lock()
				if ch.SnapshotIndex > kv.LastApplied {
					kv.DecodeSnapshot(ch.Snapshot)
					kv.LastApplied = ch.SnapshotIndex
				}
				kv.mu.Unlock()
			}

		}
	}
}

// start
func StartKVServer(servers []*labrpc.ClientEnd, me int, persister *raft.Persister, maxraftstate int) *KVServer {
	// call labgob.Register on structures you want
	// Go's RPC library to marshall/unmarshall.
	labgob.Register(Op{})

	kv := new(KVServer)
	kv.me = me
	kv.maxraftstate = maxraftstate

	// You may need initialization code here.

	kv.applyCh = make(chan raft.ApplyMsg)
	kv.rf = raft.Make(servers, me, persister, kv.applyCh)
	kv.ClientId2ComId = make(map[int64]int)
	kv.ComNotify = make(map[int]chan Op)
	kv.StateMachine = &MemoryKV{make(map[string]string)}

	// read snapshot
	snapshot := persister.ReadSnapshot()
	if len(snapshot) > 0 {
		kv.DecodeSnapshot(snapshot)
	}

	go kv.apply()
	return kv
}
```