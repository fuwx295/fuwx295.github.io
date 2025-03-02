+++
author = "GreyWind"
title = "Lab 4: Sharded Key/Value Service"
date = "2023-10-07"
description = "Lab 4: Sharded Key/Value Service"
tags = [
    "MIT-6.824",
]
+++

## Introduction

In lab 4, we will build a key/value storage system that "shards", or partitions, the keys over a set of replica groups.

> A shard is a subset of the key/value pairs.
> 
> For example, all the keys starting with "a" might be one shard, all the keys starting with "b" another, etc. The reason for sharding is performance. Each replica group handles puts and gets for just a few of the shards, and the groups operate in parallel. thus total system throughput (puts and gets per unit time) increases in proportion to the number of groups.

The sharded key/value store has two components. First, the component is the `"shard controller"`. The shard controller decides which replica group should serve each shard. Second, a set of replica groups. Each replica group is responsible for a subset of the shards. A replica consists of a handful of servers that use ***Raft*** to replicate the group's shards.

Clients consult the `shard controller` to find the replica group for a key, and replica groups consult the `controller` to find out what shards to serve. There is a single `shard controller` for the whole system, implemented as a fault-tolerant service using ***Raft.***

## Experience Description

To get started lab4, read the experiment document:

> [https://pdos.csail.mit.edu/6.824/labs/lab-shard.html](https://pdos.csail.mit.edu/6.824/labs/lab-shard.html)

In 3A, we should implement the `shard controller`, in `shardctrler/server.go` and `client.go`. implementation must support the RPC interface described in `shardctrler/common.go`, which consists of `Join`, `Leave`, `Move`, and `Query` RPCs. These RPCs are intended to allow an administrator (and the tests) to control the `shardctrler`: to add new replica groups, to eliminate replica groups, and to move shards between replica groups.

In 3B, modify `shardkv/client.go`, `shardkv/common.go`, and `shardkv/server.go`.

Each shardkv server operates as part of a replica group. Each replica group serves `Get`, `Put`, and `Append` operations for some of the key-space shards. Use `key2shard()` in `client.go` to find which shard a key belongs to. Multiple replica groups cooperate to serve the complete set of shards. A single instance of the `shardctrler` service assigns shards to replica groups; when this assignment changes, replica groups have to hand off shards to each other, while ensuring that clients do not see inconsistent responses.

Storage system must provide a linearizable interface to applications that use its client interface. That is completed application calls to the `Clerk.Get()`, `Clerk.Put()`, and `Clerk.Append()` methods in `shardkv/client.go` must appear to have affected all replicas in the same order. `A Clerk.Get()` should see the value written by the most recent `Put/Append` to the same key. This must be true even when `Gets` and `Puts` arrive at about the same time as configuration changes.

## Implementation

### Shard controller (4A)

To support `Join`, `Leave`, `Move`, and `Query` RPCs. Combine them into one `command` RPCs. Modify the `common.go`.

```go
type CommandArgs struct {
	Servers   map[int][]string //Join
	GIDs      []int            //leave
	Shard     int              //Move
	GID       int              //Move
	Num       int              //Query
	Op        string           // four optype
	ClientId  int64
	CommandId int
}

type CommandReply struct {
	Err    Err
	Config Config
}
```

The `Clerk` and `Server` frame struct like [Lab3](https://greywind.cn/lab-3-fault-tolerant-keyvalue-service), the controller using `Raft` the sync `configuration` log.

The main task is the implementation of four RPCs.

```go
func (sc *ShardCtrler) JoinProcess(Servers map[int][]string) {
	lastConfig := sc.GetLastConfig()
	lastGroup := make(map[int][]string)
	// copy
	for k, v := range lastConfig.Groups {
		lastGroup[k] = v
	}
	newConfig := Config{
		Num:    lastConfig.Num + 1,
		Shards: lastConfig.Shards,
		Groups: lastGroup,
	}
	for gid, servers := range Servers {
		if _, ok := newConfig.Groups[gid]; !ok {
			newServers := make([]string, len(servers))
			copy(newServers, servers)
			newConfig.Groups[gid] = newServers
		}
	}

	g2s := Group2Shard(newConfig)
	// find big and small untill blance
	for {
		from, to := getGid_MaxShards(g2s), getGid_MinShards(g2s)

		if from != 0 && len(g2s[from])-len(g2s[to]) <= 1 {
			break
		}
		g2s[to] = append(g2s[to], g2s[from][0])
		g2s[from] = g2s[from][1:]
	}
	var newShards [NShards]int
	for gid, shards := range g2s {
		for _, shard := range shards {
			newShards[shard] = gid
		}
	}

	newConfig.Shards = newShards
	sc.configs = append(sc.configs, newConfig)
}

func (sc *ShardCtrler) LeaveProcess(GIDs []int) {
	lastConfig := sc.GetLastConfig()
	lastGroup := make(map[int][]string)
	orphanShards := make([]int, 0)
	for k, v := range lastConfig.Groups {
		lastGroup[k] = v
	}
	newConfig := Config{Num: lastConfig.Num + 1, Shards: lastConfig.Shards, Groups: lastGroup}
	g2s := Group2Shard(newConfig)
	for _, gid := range GIDs {
		if _, ok := newConfig.Groups[gid]; ok {
			delete(newConfig.Groups, gid)
		}
		if shards, ok := g2s[gid]; ok {
			orphanShards = append(orphanShards, shards...)
			delete(g2s, gid)
		}

	}
	var newShards [NShards]int

	// find target small to add leave group
	if len(newConfig.Groups) != 0 {
		for _, shard := range orphanShards {
			to := getGid_MinShards(g2s)
			g2s[to] = append(g2s[to], shard)
		}
		for gid, shards := range g2s {
			for _, shard := range shards {
				newShards[shard] = gid
			}
		}
	}

	newConfig.Shards = newShards
	sc.configs = append(sc.configs, newConfig)
}

func (sc *ShardCtrler) MoveProcess(gid int, shard int) {
	lastConfig := sc.GetLastConfig()
	lastGroup := make(map[int][]string)
	for k, v := range lastConfig.Groups {
		lastGroup[k] = v
	}
	newConfig := Config{Num: lastConfig.Num + 1, Shards: lastConfig.Shards, Groups: lastGroup}
	newConfig.Shards[shard] = gid
	sc.configs = append(sc.configs, newConfig)
}

func (sc *ShardCtrler) QueryProcess(num int) Config {
	if num == -1 || num >= len(sc.configs) {
		return sc.configs[len(sc.configs)-1]
	}
	return sc.configs[num]
}
```

The `apply()` function and `command()` function to handle all logs and requests.

```go
func (sc *ShardCtrler) apply() {
	for !sc.killed() {
		select {
		case ch := <-sc.applyCh:
			if ch.CommandValid {
				sc.mu.Lock()
				opchan := sc.GetChan(ch.CommandIndex)
				op := ch.Command.(Op)
				res := result{
					ClientId:  op.ClientId,
					CommandId: op.CommandId,
				}

				if sc.Client2Seq[op.ClientId] < op.CommandId {

					switch op.Command {
					case Join:
						sc.JoinProcess(op.Servers)
					case Leave:
						sc.LeaveProcess(op.GIDs)
					case Move:
						sc.MoveProcess(op.GID, op.Shard)
					case Query:
						res.Res = sc.QueryProcess(op.Num)
					}

					sc.Client2Seq[op.ClientId] = op.CommandId
					sc.mu.Unlock()
					opchan <- res
				} else {
					sc.mu.Unlock()
				}
			}
		}
	}
}

func (sc *ShardCtrler) Command(args *CommandArgs, reply *CommandReply) {
	if sc.killed() {
		reply.Err = WrongLeader
		return
	}
	_, isLeader := sc.rf.GetState()
	if !isLeader {
		reply.Err = WrongLeader
		return
	}
	sc.mu.Lock()
	if sc.Client2Seq[args.ClientId] >= args.CommandId {
		reply.Err = OK
		if args.Op == Query {
			reply.Config = sc.QueryProcess(args.Num)

		}
		sc.mu.Unlock()
		return
	}

	op := Op{
		Command:   args.Op,
		CommandId: args.CommandId,
		ClientId:  args.ClientId,

		Servers: args.Servers, //Join
		GIDs:    args.GIDs,    //leave
		Shard:   args.Shard,   //Move
		GID:     args.GID,     //Move
		Num:     args.Num,     //Query
	}

	index, _, isLeader := sc.rf.Start(op)
	if !isLeader {
		reply.Err = WrongLeader
		sc.mu.Unlock()
		return
	}
	ch := sc.GetChan(index)
	sc.mu.Unlock()

	select {
	case app := <-ch:
		if app.ClientId == op.ClientId && app.CommandId == op.CommandId {
			reply.Err = OK
			reply.Config = app.Res
		} else {

			reply.Err = WrongLeader

		}

	case <-time.After(time.Millisecond * 33):
		reply.Err = TimeOut
	}

	go func() {
		sc.mu.Lock()
		delete(sc.Index2Cmd, index)
		sc.mu.Unlock()
	}()
}
```

### Sharded Key/Value Server (4B)

The lab4 is the storage system, we should use **raft(lab2)** and **shard controller(4A)** to build a sharded key/value system. The total design is like **lab3**. We should modify it to fit the sharded environment, and make it approach a production environment.

There are two challenges:

**Garbage collection of state**

When a replica group loses ownership of a shard, that replica group should eliminate the keys that it lost from its database. It is wasteful for it to keep values that it no longer owns, and no longer serves requests for.

**Client requests during configuration changes**

The simplest way to handle configuration changes is to disallow all client operations until the transition has been completed. While conceptually simple, this approach is not feasible in production-level systems; it results in long pauses for all clients whenever machines are brought in or taken out. It would be better to continue serving shards that are not affected by the ongoing configuration change.

First, implement `clerk` and merge `Get/Put/Append` RPCS (like **lab3**) in `common.go`.

```go
type CommandArgs struct {
	Key       string 
	Value     string
	Op        Operation
	ClientId  int64
	CommandId int
}

type CommandReply struct {
	Value string
	Err   Err
}
```

```go
func (ck *Clerk) Get(key string) string {
	return ck.Command(&CommandArgs{Key: key, Op: GetType})
}

func (ck *Clerk) PutAppend(key string, value string, op string) {
	ck.Command(&CommandArgs{Key: key, Value: value, Op: Operation(op)})
}

func (ck *Clerk) Put(key string, value string) {
	ck.PutAppend(key, value, "Put")
}
func (ck *Clerk) Append(key string, value string) {
	ck.PutAppend(key, value, "Append")
}

func (ck *Clerk) Command(args *CommandArgs) string {
	args.ClientId = ck.ClientId
	ck.CommandId++
	args.CommandId = ck.CommandId
	for {
		shardId := key2shard(args.Key)
		gid := ck.config.Shards[shardId]
		if servers, ok := ck.config.Groups[gid]; ok {
			for si := 0; si < len(servers); si++ {
				srv := ck.make_end(servers[si])
				var reply CommandReply
				ok := srv.Call("ShardKV.Command", args, &reply)
				if ok && (reply.Err == OK || reply.Err == ErrNoKey) {
					return reply.Value
				}
				if ok && reply.Err == ErrWrongGroup {
					break
				}
			}
		}

		time.Sleep(100 * time.Millisecond)
		ck.config = ck.sm.Query(-1)
	}
}
```

`ShardKV` still refers the **lab3**, but we should add something and modify it.

```go
type ShardKV struct {
	mu           sync.Mutex
	me           int
	rf           *raft.Raft
	applyCh      chan raft.ApplyMsg
	makeEnd      func(string) *labrpc.ClientEnd
	gid          int
	masters      []*labrpc.ClientEnd
	maxRaftState int // snapshot if log grows this big

	// Your definitions here.

	dead int32 // set by Kill()

	Config     shardctrler.Config // 需要更新的最新的配置
	LastConfig shardctrler.Config // 更新之前的配置，用于比对是否全部更新完了

	DB KVStateMachine // Storage system

	ComNotify      map[int]chan OpReply // notify
	ClientId2ComId map[int64]int

	sck *shardctrler.Clerk // sck is a client used to contact shard master
}

func StartServer(servers []*labrpc.ClientEnd, me int, persister *raft.Persister, maxraftstate int, gid int, masters []*labrpc.ClientEnd, make_end func(string) *labrpc.ClientEnd) *ShardKV {
	// call labgob.Register on structures you want
	// Go's RPC library to marshall/unmarshall.
	labgob.Register(Op{})

	kv := new(ShardKV)
	kv.me = me
	kv.maxRaftState = maxraftstate
	kv.makeEnd = make_end
	kv.gid = gid
	kv.masters = masters

	// Your initialization code here.

	kv.DB = &KVMemory{make([]KVShard, shardctrler.NShards)}
	kv.ClientId2ComId = make(map[int64]int)

	// Use something like this to talk to the shardctrler:
	// kv.mck = shardctrler.MakeClerk(kv.masters)
	kv.sck = shardctrler.MakeClerk(kv.masters)
	kv.ComNotify = make(map[int]chan OpReply)

	snapshot := persister.ReadSnapshot()
	if len(snapshot) > 0 {
		kv.DecodeSnapShot(snapshot)
	}

	kv.applyCh = make(chan raft.ApplyMsg)
	kv.rf = raft.Make(servers, me, persister, kv.applyCh)

	go kv.apply()
	go kv.ConfigDetectedLoop()

	return kv
}
```

`KVStateMachine` to Describe DataBase, notice it is **sharded**.

```go
type KVStateMachine interface {
	Get(key string) (string, Err)
	Put(key, value string) Err
	Append(key, value string) Err
	GetShard(shardId int) *KVShard
}

// KVMemory implement KVStateMachine
type KVMemory struct {
	KV []KVShard
}

type KVShard struct {
	KVS       map[string]string
	ConfigNum int // what version this Shard is in
}

func (kv *KVMemory) Get(key string) (string, Err) {
	shardId := key2shard(key)
	value, ok := kv.KV[shardId].KVS[key]
	if ok {
		return value, OK
	}
	return "", ErrNoKey
}

func (kv *KVMemory) Put(key, value string) Err {
	shardId := key2shard(key)
	kv.KV[shardId].KVS[key] = value
	return OK
}

func (kv *KVMemory) Append(key, value string) Err {
	shardId := key2shard(key)
	kv.KV[shardId].KVS[key] += value
	return OK
}

func (kv *KVMemory) GetShard(shardId int) *KVShard {
	return &kv.KV[shardId]
}

func (kv *ShardKV) applyStateMachine(op *Op) {
	switch op.OpType {
	case PutType:
		kv.DB.Put(op.Key, op.Value)
	case AppendType:
		kv.DB.Append(op.Key, op.Value)
	}
}
```

Then the `ShardKV` receives `command` RPCs, we need to handle it.

```go
// Put Append Get
func (kv *ShardKV) Command(args *CommandArgs, reply *CommandReply) {
	shardId := key2shard(args.Key)
	kv.mu.Lock()
	if kv.Config.Shards[shardId] != kv.gid {
		reply.Err = ErrWrongGroup
		kv.mu.Unlock()
		return
	} else if kv.DB.GetShard(shardId).KVS == nil {
		reply.Err = ShardNotArrived
		kv.mu.Unlock()
		return
	}
	kv.mu.Unlock()

	command := Op{
		OpType:    args.Op,
		Key:       args.Key,
		Value:     args.Value,
		ClientId:  args.ClientId,
		CommandId: args.CommandId,
		ShardId:   shardId,
	}
	reply.Err = kv.startCommand(command, TimeOut)

	// Get function should check twice
	if reply.Err == OK && command.OpType == GetType {
		kv.mu.Lock()
		if kv.Config.Shards[shardId] != kv.gid {
			reply.Err = ErrWrongGroup
		} else if kv.DB.GetShard(shardId).KVS == nil {
			reply.Err = ShardNotArrived
		} else {

			reply.Value, reply.Err = kv.DB.Get(args.Key)
		}
		kv.mu.Unlock()
	}
}
// using raft to sync log
func (kv *ShardKV) startCommand(command Op, timeoutPeriod time.Duration) Err {
	kv.mu.Lock()
	index, _, isLeader := kv.rf.Start(command)
	if !isLeader {
		kv.mu.Unlock()
		return ErrWrongLeader
	}

	ch := kv.getWaitCh(index)
	kv.mu.Unlock()

	timer := time.NewTicker(timeoutPeriod)
	defer timer.Stop()

    // wait to finish
	select {
	case re := <-ch:
		kv.mu.Lock()
		delete(kv.ComNotify, index)
		if re.CommandId != command.CommandId || re.ClientId != command.ClientId {
			// One way to do this is for the server to detect that it has lost leadership,
			// by noticing that a different request has appeared at the index returned by Start()
			kv.mu.Unlock()
			return ErrInconsistentData
		}
		kv.mu.Unlock()
		return re.Err
	case <-timer.C:
		return ErrOverTime
	}
}
```

The make should **linearized** and **consistent**, we should pull the latest `configuration` by `shard controller`.

```go
func (kv *ShardKV) ConfigDetectedLoop() {
	kv.mu.Lock()
	curConfig := kv.Config
	rf := kv.rf
	kv.mu.Unlock()

	for !kv.killed() {
		// only leader needs to deal with configuration tasks
		if _, isLeader := rf.GetState(); !isLeader {
			time.Sleep(UpConfigLoopInterval)
			continue
		}
		kv.mu.Lock()
		// send finished ?
		if !kv.allSent() {
			SeqMap := make(map[int64]int)
			for k, v := range kv.ClientId2ComId {
				SeqMap[k] = v
			}

			for shardId, gid := range kv.LastConfig.Shards {
				//check and send shard to other
				if gid == kv.gid && kv.Config.Shards[shardId] != kv.gid && kv.DB.GetShard(shardId).ConfigNum < kv.Config.Num {
					sendDate := kv.cloneShard(kv.Config.Num, kv.DB.GetShard(shardId).KVS)
					args := SendShardArg{
						LastAppliedCommandId: SeqMap,
						ShardId:              shardId,
						Shard:                sendDate,
						ClientId:             int64(gid),
						CommandId:            kv.Config.Num,
					}

					// shardId -> gid -> server names
					serversList := kv.Config.Groups[kv.Config.Shards[shardId]]
					servers := make([]*labrpc.ClientEnd, len(serversList))
					for i, name := range serversList {
						servers[i] = kv.makeEnd(name)
					}
					go kv.sendAddShrad(servers, &args)
				}
			}
			kv.mu.Unlock()
			time.Sleep(UpConfigLoopInterval)
			continue
		}

		// received finished ?
		if !kv.allReceived() {
			kv.mu.Unlock()
			time.Sleep(UpConfigLoopInterval)
			continue
		}

		// current configuration is configured, poll for the next configuration
		curConfig = kv.Config
		sck := kv.sck
		kv.mu.Unlock()
		// pull new configuration
		newConfig := sck.Query(curConfig.Num + 1)
		if newConfig.Num != curConfig.Num+1 {
			time.Sleep(UpConfigLoopInterval)
			continue
		}
        
        //sync pull op
		command := Op{
			OpType:    UpConfigType,
			ClientId:  int64(kv.gid),
			CommandId: newConfig.Num,
			UpConfig:  newConfig,
		}
		kv.startCommand(command, UpConfigTimeout)
	}

}
```

Send the `shard` from one shard to another shard, must using an RPCs, which is named `AddShard`.

```go
type SendShardArg struct {
	LastAppliedCommandId map[int64]int // for receiver to update its state
	ShardId              int
	Shard                KVShard // Shard to be sent
	ClientId             int64
	CommandId            int
}

type AddShardReply struct {
	Err Err
}

func (kv *ShardKV) sendAddShrad(servers []*labrpc.ClientEnd, args *SendShardArg) {
	index := 0
	start := time.Now()
	for {
		var reply AddShardReply

		ok := servers[index].Call("ShardKV.AddShard", args, &reply)
		// send shard success or timeout
		if ok && reply.Err == OK || time.Now().Sub(start) >= 2*time.Second {
			kv.mu.Lock()
			command := Op{
				OpType:    RemoveShardType,
				ClientId:  int64(kv.gid),
				ShardId:   args.ShardId,
				CommandId: kv.Config.Num,
			}
			kv.mu.Unlock()
			// remove shard
			kv.startCommand(command, TimeOut)
			break
		}
		index = (index + 1) % len(servers)
		if index == 0 {
			time.Sleep(UpConfigLoopInterval)
		}
	}
}

func (kv *ShardKV) AddShard(args *SendShardArg, reply *AddShardReply) {
	command := Op{
		OpType:         AddShardType,
		ClientId:       args.ClientId,
		CommandId:      args.CommandId,
		ShardId:        args.ShardId,
		Shard:          args.Shard,
		ClientId2ComId: args.LastAppliedCommandId,
	}
	reply.Err = kv.startCommand(command, AddShardsTimeout)
}
```

At last, `apply` function to apply all op (`Get/Append/Put`, `update config`). The apply function is like **lab3**, but you should think about updating the config.

```go
func (kv *ShardKV) apply() {
	for {
		if kv.killed() {
			return
		}
		select {
		case ch := <-kv.applyCh:
			if ch.CommandValid == true {
				kv.mu.Lock()
				op := ch.Command.(Op)
				reply := OpReply{
					ClientId:  op.ClientId,
					CommandId: op.CommandId,
					Err:       OK,
				}
				if op.OpType == PutType || op.OpType == GetType || op.OpType == AppendType {

					shardId := key2shard(op.Key)
					if kv.Config.Shards[shardId] != kv.gid {
						reply.Err = ErrWrongGroup
					} else if kv.DB.GetShard(shardId).KVS == nil {
						// 如果应该存在的切片没有数据那么这个切片就还没到达
						reply.Err = ShardNotArrived
					} else {
						if !kv.ifDuplicate(op.ClientId, op.CommandId) {
							kv.ClientId2ComId[op.ClientId] = op.CommandId
							kv.applyStateMachine(&op)
						}
					}
				} else {
					// request from server of other group
					switch op.OpType {
					case UpConfigType:
						kv.upConfigHandler(op)
					case AddShardType:
						// 如果配置号比op的SeqId还低说明不是最新的配置
						if kv.Config.Num < op.CommandId {
							reply.Err = ConfigNotArrived
							break
						}
						kv.addShardHandler(op)
					case RemoveShardType:
						// remove operation is from previous UpConfig
						kv.removeShardHandler(op)
					default:
						log.Fatalf("invalid command type: %v.", op.OpType)
					}
				}

				// 如果需要snapshot，且超出其stateSize
				if kv.maxRaftState != -1 && kv.rf.GetRaftStateSize() > kv.maxRaftState {
					snapshot := kv.PersistSnapShot()
					kv.rf.Snapshot(ch.CommandIndex, snapshot)
				}

				ch := kv.getWaitCh(ch.CommandIndex)
				ch <- reply

				kv.mu.Unlock()

			}
			if ch.SnapshotValid == true {
				if len(ch.Snapshot) > 0 {
					// 读取快照的数据
					kv.mu.Lock()
					kv.DecodeSnapShot(ch.Snapshot)
					kv.mu.Unlock()
				}
				continue
			}

		}
	}
}
```

The snapshot is very easy to implement, just refer **lab3**.

## Reference

There are many difficulties in finishing **lab4**, there are blogs that may help you.

> [github\_lab4](https://github.com/OneSizeFitsQuorum/MIT6.824-2021/blob/master/docs/lab4.md)
> 
> [cdsn\_lab4b](https://blog.csdn.net/weixin_45938441/article/details/125566763?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522169640361916800225587562%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=169640361916800225587562&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-125566763-null-null.142^v94^insert_down28v1&utm_term=mit%206.824%20lab4&spm=1018.2226.3001.4187)