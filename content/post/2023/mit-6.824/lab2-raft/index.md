+++
author = "GreyWind"
title = "Lab 2: Raft"
date = "2023-09-05"
description = "Lab 2: Raft"
tags = [
    "MIT-6.824",
]
+++
## Introduction

In lab2 we will implement Raft, a replicated state machine protocol. This lab is divided into four parts, 2A should finish the leader election, 2B must make sure to sync the log between each server, 2C will persist in every server state and 2D take log compaction to reduce logs.

After finishing lab1, we became familiar with Golang, but the concurrency programming is still hard. There are many details to pay attention to Raft, so read the Raft Paper carefully and repeatedly.

## Experiment Description

To get started lab2, read the experiment document:

> [https://pdos.csail.mit.edu/6.824/labs/lab-raft.html](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)

All code should be implemented in `src/raft/raft.go`. The tests are in `src/raft/test_test.go`

## Implementation

### Leader election(2A)

In 2A, we should implement `leader election` and `heartbeats`(empty AppendEntries RPCs). As described in the paper, Fill Some State in Raft struct:

```go
type Raft stuct {
    mu        sync.Mutex          // Lock to protect shared access to this peer's state
	peers     []*labrpc.ClientEnd // RPC end points of all peers
	persister *Persister          // Object to hold this peer's persisted state
	me        int                 // this peer's index into peers[]
	dead      int32               // set by Kill()

	// Your data here (2A, 2B, 2C).
	// Look at the paper's Figure 2 for a description of what
	// state a Raft server must maintain.

    // 2A
	Time time.Time // check leader elect timeout
	state       int // server state
	currentTerm int // term
	votedFor    int // voted to which server
	voteCount int // receive vote number
    Logs []Log  // log entry

    // 2B
	commitIndex int // commit
	lastApplied int
	nextIndex  []int // nextIndex send to follower
	matchIndex []int // follower match Index
	applyCh chan ApplyMsg // send to service command apply

    // 2D
	Lastindex int // snap last index
	LastTerm  int // snap last term
}

type Log stuct {
    Command interface{}
    Term int
}

// server state
const (
	Follower = iota
	Candidate
	Leader
)
```

Then the server will check leader whether is alive periodically. But the timeout must be random because if two servers become candidates at the same and they still have the same timeout, the cluster may never elect a `leader`. If a `candidate` wins the election, it will broadcast to other servers, which will become `followers`. If don't have the leader exceed the timeout period, the new election will start.

> In the lab document, it was noticed don't use Go's `time.Timer` or `time.Ticker`.

```go
// check state
func (rf *Raft) ticker() {
	for !rf.killed() {
		// Your code here (2A)
		// Check if a leader election should be started.
		rf.mu.Lock()
		switch rf.state {
		case Follower:
			if time.Since(rf.Time) > randomTimeout() {
				go rf.StartElection()
			}
		case Candidate:
			if time.Since(rf.Time) > randomTimeout() {
				go rf.StartElection()
			}
		case Leader:
			rf.Broadcast()
		}
        rf.mu.Unlock()
		// pause for a random amount of time between 50 and 350
		// milliseconds.
		ms := 30 + (rand.Int63() % 30)
		time.Sleep(time.Duration(ms) * time.Millisecond)
	}
}
```

When a follower starts to election, it will become a candidate and increment its term. And it will send RequestVote RPCs to another server. When the request is received, it will vote according to the rules(In Raft paper).

```go
func (rf *Raft) StartElection() {
	rf.mu.Lock()
	rf.state = Candidate
	rf.currentTerm += 1
	// vote to itself
	rf.voteCount = 1
	rf.votedFor = rf.me
	rf.persist()

	args := RequestVoteArgs{
		Term:         rf.currentTerm,
		CandidateId:  rf.me,
		LastLogIndex: rf.getLastIndex(),
		LastLogTerm:  rf.getLastLogTerm(),
	}
	rf.mu.Unlock()

	for i := range rf.peers {
		if i != rf.me {
			go rf.sendRequestVote(i, &args, &RequestVoteReply{})
		}
	}
}

func (rf *Raft) sendRequestVote(server int, args *RequestVoteArgs, reply *RequestVoteReply) {
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
	rf.mu.Lock()
	defer rf.mu.Unlock()
    // vote rule
	if !ok || rf.state != Candidate || reply.Term != rf.currentTerm {
		return
	}

	if reply.VoteGranted {
		rf.voteCount++
        // exceed half win the election
		if rf.voteCount > len(rf.peers)/2 {
			rf.SetLeader()
		}
	} else {
        // vote rule
		if reply.Term > rf.currentTerm {
			rf.state = Follower
			rf.currentTerm = reply.Term
			rf.votedFor = -1
			rf.Time = time.Now()
			rf.persist()
		}
	}
}
```

RequestVote RPCs

```go
type RequestVoteArgs struct {
	// Your data here (2A, 2B).
	Term         int
	CandidateId  int
	LastLogIndex int
	LastLogTerm  int
}

type RequestVoteReply struct {
	// Your data here (2A).
	Term        int
	VoteGranted bool
}

func (rf *Raft) RequestVote(args *RequestVoteArgs, reply *RequestVoteReply) {
	// Your code here (2A, 2B).
	rf.mu.Lock()
	defer rf.mu.Unlock()
    // overdue vote
	if args.Term < rf.currentTerm {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
		return
	}
    
    // log unqualified
	if args.LastLogTerm < rf.getLastLogTerm() || (args.LastLogTerm == rf.getLastLogTerm() && args.LastLogIndex < rf.getLastIndex()) {
		reply.VoteGranted = false
		reply.Term = rf.currentTerm
		if args.Term > rf.currentTerm {
			rf.currentTerm = args.Term
			rf.state = Follower
			rf.persist()
		}
		return
	}
	
    // receive a vote should stay follower and vote
	if args.Term > rf.currentTerm || (args.Term == rf.currentTerm && (rf.votedFor == -1 || rf.votedFor == args.CandidateId)) {
		rf.state = Follower
		rf.votedFor = args.CandidateId
		rf.currentTerm = args.Term
		reply.VoteGranted = true
		reply.Term = rf.currentTerm
		rf.Time = time.Now()
		rf.persist()
	}

}
```

### Log Replication(2B)

In 2B, complete the Start() function (when a service sends a new command, it will sync Log)

```go
func (rf *Raft) Start(command interface{}) (int, int, bool) {
	index := -1
	term := -1
	isLeader := true

	// Your code here (2B).
	rf.mu.Lock()
	defer rf.mu.Unlock()
	term = rf.currentTerm
	if rf.state != Leader {
		return index, term, false
	}

	newLog := Log{
		Command: command,
		Term:    rf.currentTerm,
	}
	rf.logs = append(rf.logs, newLog)
	rf.persist()

	rf.matchIndex[rf.me] = len(rf.logs) - 1 + rf.Lastindex
	rf.nextIndex[rf.me] = rf.matchIndex[rf.me] + 1

	for i := range rf.peers {
		if i == rf.me {
			continue
		}

		if rf.matchIndex[i] < rf.Lastindex {
			//do nothing
		} else {
			entry := make([]Log, rf.getLastIndex()-rf.matchIndex[i])
			copy(entry, rf.logs[rf.matchIndex[i]+1-rf.Lastindex:])
			// sync Log
			nargs := AppendEntriesArgs{
				Term:         rf.currentTerm,
				LeaderId:     rf.me,
				PrevLogIndex: rf.matchIndex[i],
				PrevLogTerm:  rf.getLogTerm(rf.matchIndex[i]),
				LeaderCommit: rf.commitIndex,
				Entries:      entry,
			}
			go rf.sendAppenEntries(i, &nargs, &AppendEntriesReply{})
		}
	}
	return len(rf.logs) - 1 + rf.Lastindex, newLog.Term, isLeader
}
```

When `SendAppendEntries` RPCs, the follower log may conflict with the Leader log, in which case it must resend the RPC. Sometimes followers may crash and restart or maintain follower status, the leader should synchronize logs or heartbeats via boardcast.

```go
func (rf *Raft) sendAppenEntries(server int, args *AppendEntriesArgs, reply *AppendEntriesReply) {
	ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
	if !ok {
		return
	}
	//log.Println("send hb")
	rf.mu.Lock()
	defer rf.mu.Unlock()

	if rf.state != Leader {
		return
	}

	if !reply.Success {
		if reply.Term > rf.currentTerm {
			rf.currentTerm = reply.Term
			rf.state = Follower
			rf.votedFor = -1

			rf.persist()
		} else {
			args.PrevLogIndex = reply.Index
			if args.PrevLogIndex < 0 {
				return
			}
			if args.PrevLogIndex-rf.Lastindex < 0 {
				// send snap (2D)
			} else {
				// retry
				args.PrevLogTerm = rf.getLogTerm(args.PrevLogIndex)
				entry := make([]Log, rf.getLastIndex()-args.PrevLogIndex)
				copy(entry, rf.logs[args.PrevLogIndex-rf.Lastindex+1:])
				args.Entries = entry
				// go syncLog
				go rf.sendAppenEntries(server, args, reply)
			}
		}

	} else {
        // sync log success
		if rf.matchIndex[server] < args.PrevLogIndex+len(args.Entries) {
			rf.matchIndex[server] = args.PrevLogIndex + len(args.Entries)
            // commit log
			rf.UpdateCommit()
		}
		if rf.nextIndex[server] < args.PrevLogIndex+len(args.Entries)+1 {
			rf.nextIndex[server] = args.PrevLogIndex + len(args.Entries) + 1
		}
	}
}
```

Broadcast to each follower

```go
func (rf *Raft) Broadcast() {
	if rf.state != Leader {
		return
	}
	prelogindex := rf.getLastIndex()
	prelogterm := rf.getLastLogTerm()
	rf.UpdateCommit()

	for i := range rf.peers {
		if i == rf.me {
			continue
		}
		// log not match
		if (rf.nextIndex[i] <= prelogindex || rf.nextIndex[i]-rf.matchIndex[i] != 1) && rf.getLastIndex() != 0 {
			if rf.matchIndex[i] < rf.Lastindex {
				// need log is remove
                // send snapshot (2D)

			} else {

				// send synclog
				entry := make([]Log, rf.getLastIndex()-rf.matchIndex[i])
				copy(entry, rf.logs[rf.matchIndex[i]+1-rf.Lastindex:])

				nargs := AppendEntriesArgs{
					Term:         rf.currentTerm,
					LeaderId:     rf.me,
					PrevLogIndex: rf.matchIndex[i],
					PrevLogTerm:  rf.getLogTerm(rf.matchIndex[i]),
					LeaderCommit: rf.commitIndex,
					Entries:      entry,
				}
				go rf.sendAppenEntries(i, &nargs, &AppendEntriesReply{})
			}

		} else {
			// log match send heartbeat
			args := AppendEntriesArgs{
				Term:         rf.currentTerm,
				LeaderId:     rf.me,
				PrevLogIndex: prelogindex,
				PrevLogTerm:  prelogterm,
				LeaderCommit: rf.commitIndex,
			}
			go rf.sendAppenEntries(i, &args, &AppendEntriesReply{})
		}
	}
}
```

`AppendEntries` RPCs will keep the server aligned with the leader. There's a lot of detail in the code. It's too long and too big.

```go
type AppendEntriesArgs struct {
	Term         int
	LeaderId     int
	PrevLogIndex int
	PrevLogTerm  int
	LeaderCommit int
	Entries      []Log
}

type AppendEntriesReply struct {
	Term    int
	Success bool
	Index   int
}

func (rf *Raft) AppendEntries(args *AppendEntriesArgs, reply *AppendEntriesReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
    // term not match
	reply.Success = false
	if args.Term < rf.currentTerm {
		reply.Term = rf.currentTerm
		return
	} else if args.Term > rf.currentTerm {
		reply.Term = args.Term
		rf.currentTerm = args.Term
		rf.Time = time.Now()
		rf.state = Follower
		rf.votedFor = -1
		rf.persist()
	} else {
		//term equal
		rf.Time = time.Now()
		rf.state = Follower
		reply.Term = args.Term
	}

	//lack some logs
	if rf.getLastIndex() < args.PrevLogIndex {

		reply.Index = rf.getLastIndex()
		return
	}

	if rf.Lastindex > args.PrevLogIndex {
		if args.PrevLogIndex+len(args.Entries) <= rf.Lastindex {
			reply.Index = rf.Lastindex
			return
		}
		args.PrevLogTerm = args.Entries[rf.Lastindex-args.PrevLogIndex-1].Term
		args.Entries = args.Entries[rf.Lastindex-args.PrevLogIndex:]
		args.PrevLogIndex = rf.Lastindex
	}

	if args.PrevLogTerm != rf.getLogTerm(args.PrevLogIndex) {
		reply.Index = rf.lastApplied
		if reply.Index > rf.Lastindex {
			reply.Index = rf.Lastindex
		}
		if reply.Index > args.PrevLogIndex-1 {
			reply.Index = args.PrevLogIndex - 1
		}
		return
	}

    // sync sucess
	reply.Success = true
	//latest condition
	if rf.getLastIndex() == args.PrevLogIndex && args.PrevLogTerm == rf.getLastLogTerm() {
		if args.LeaderCommit > rf.commitIndex {
			tmp := rf.getLastIndex()
			if tmp > args.LeaderCommit {
				tmp = args.LeaderCommit
			}
			rf.commitIndex = tmp
		}
	}
	//heart beat
	if len(args.Entries) == 0 {
		return
	}
    // overdue entries
	if rf.getLastIndex() >= args.PrevLogIndex+len(args.Entries) && rf.getLogTerm(args.PrevLogIndex+len(args.Entries)) == args.Entries[len(args.Entries)-1].Term {
		return
	}

	i := args.PrevLogIndex + 1
	for i <= rf.getLastIndex() && i-args.PrevLogIndex-1 < len(args.Entries) {
		break
	}
	if i-args.PrevLogIndex-1 >= len(args.Entries) {
		return
	}
    // append to itself
	rf.logs = rf.logs[:i-rf.Lastindex]
	rf.logs = append(rf.logs, args.Entries[i-args.PrevLogIndex-1:]...)
    // commit and persist
	if args.LeaderCommit > rf.commitIndex {
		tmp := rf.getLastIndex()
		if tmp > args.LeaderCommit {
			tmp = args.LeaderCommit
		}
		rf.commitIndex = tmp
	}
	rf.persist()

}
```

Using loops, detect if there are submitted but not applied entries and send the entry information to the channel provided by the test system.

```go
unc (rf *Raft) apply() {
	for !rf.killed() {

		rf.mu.Lock()
		oldApply := rf.lastApplied
		oldCommit := rf.commitIndex

		//after crash
		if oldApply < rf.Lastindex {
			rf.lastApplied = rf.Lastindex
			rf.commitIndex = rf.Lastindex
			rf.mu.Unlock()
			time.Sleep(time.Millisecond * 30)
			continue
		}
		if oldCommit < rf.Lastindex {

			rf.commitIndex = rf.Lastindex
			rf.mu.Unlock()
			time.Sleep(time.Millisecond * 30)
			continue
		}

		if oldApply == oldCommit || (oldCommit-oldApply) >= len(rf.logs) {
			rf.mu.Unlock()
			time.Sleep(time.Millisecond * 5)
			continue
		}
		
		entry := make([]Log, oldCommit-oldApply)
		copy(entry, rf.logs[oldApply+1-rf.Lastindex:oldCommit+1-rf.Lastindex])
		rf.mu.Unlock()
        // apply
		for key, value := range entry {
			rf.applyCh <- ApplyMsg{
				CommandValid: true,
				CommandIndex: key + oldApply + 1,
				Command:      value.Command,
			}

		}

		rf.mu.Lock()
		if rf.lastApplied < oldCommit {
			rf.lastApplied = oldCommit
		}
		if rf.lastApplied > rf.commitIndex {
			rf.commitIndex = rf.lastApplied
		}
		rf.mu.Unlock()

		time.Sleep(time.Millisecond * 30)
	}

}
```

### Persistence (2C)

This is as simple as storing persistent state on a server (described in the Raft paper).

```go
func (rf *Raft) getRaftState() []byte {
	w := new(bytes.Buffer)
	e := labgob.NewEncoder(w)
	e.Encode(rf.votedFor)
	e.Encode(rf.currentTerm)
	e.Encode(rf.logs)
	e.Encode(rf.Lastindex)
	e.Encode(rf.LastTerm)
	return w.Bytes()
}
// persist in local
func (rf *Raft) persist() {
	rf.persister.Save(rf.getRaftState(), rf.persister.snapshot)
}
// read from local
func (rf *Raft) readPersist(data []byte) {
	if data == nil || len(data) < 1 { 
		return
	}
	// Your code here (2C).
	r := bytes.NewBuffer(data)
	d := labgob.NewDecoder(r)
	var votefor int   //Votefor
	var currentTerm int   //term
	var logs []Log //Logs
	var index int //index
	var term int  //term
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if d.Decode(&votefor) != nil ||
		d.Decode(&currentTerm) != nil || d.Decode(&logs) != nil || d.Decode(&index) != nil || d.Decode(&term) != nil {
	} else {
		rf.votedFor = votefor
		rf.currentTerm = currentTerm
		rf.logs = logs
		rf.Lastindex = index
		rf.LastTerm = term
	}
}
```

### Log Compaction (2D)

* Read the paper carefully to understand how a SnapShot can create a log compaction that compacts unrestricted log growth and reduces the storage pressure on peers.
    
* After SnapShot is imported, the original Index system will be greatly changed. The changes related to Index log information need to consider lastSnapShotIndex
    

```go
func (rf *Raft) Snapshot(index int, snapshot []byte) {
	// Your code here (2D).

	rf.mu.Lock()
	defer rf.mu.Unlock()
	if index <= rf.Lastindex || index > rf.commitIndex {
		return
	}
    // Clipping log
	count := 1
	oldIndex := rf.Lastindex
	for offset, value := range rf.logs {
		if offset == 0 {
			continue
		}
		count++
		rf.Lastindex = offset + oldIndex
		rf.LastTerm = value.Term
		if offset+oldIndex == index {
			break
		}
	}

	newLog := make([]Log, 1)
	newLog = append(newLog, rf.logs[count:]...)
	rf.logs = newLog

	rf.persister.Save(rf.getRaftState(), snapshot)
}
```

InstallSnapRPC

```go
type InstallSnapshotRPC struct {
	Term             int
	LeaderId         int
	LastIncludeIndex int
	LastIncludeTerm  int
	//offset           int
	Data []byte
}
type InstallSnapshotReply struct {
	Term int
}

func (rf *Raft) InstallSnapShot(args *InstallSnapshotRPC, reply *InstallSnapshotReply) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	reply.Term = rf.currentTerm
	if args.Term < rf.currentTerm {
		return
	}

	if args.Term > rf.currentTerm {
		rf.currentTerm = args.Term
		rf.votedFor = -1
		rf.state = Follower
		rf.persist()
	}

	if args.LastIncludeIndex <= rf.Lastindex {
		return
	}
	rf.Time = time.Now()
    // remove old log
	tmpLog := make([]Log, 1)
    if rf.getLastIndex() > args.LastIncludeIndex+1 {
	    tmpLog = append(tmpLog, rf.logs[args.LastIncludeIndex+1-rf.Lastindex:]...)
	}
    rf.Lastindex = args.LastIncludeIndex
	rf.LastTerm = args.LastIncludeTerm
	rf.logs = tmpLog

	if args.LastIncludeIndex > rf.commitIndex {
		rf.commitIndex = args.LastIncludeIndex
	}
	if args.LastIncludeIndex > rf.lastApplied {
		rf.lastApplied = args.LastIncludeIndex
	}

	rf.persister.Save(rf.getRaftState(), args.Data)
	msg := ApplyMsg{
		Snapshot:      args.Data,
		SnapshotValid: true,
		SnapshotTerm:  rf.LastTerm,
		SnapshotIndex: rf.Lastindex,
	}
	go func() { rf.applyCh <- msg }()
}
```