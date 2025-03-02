+++
author = "GreyWind"
title = "Lab 1: MapReduce"
date = "2023-08-16"
description = "Lab 1: MapReduce"
tags = [
    "MIT-6.824",
]
+++
## Introduction

In lab 1 we will build a MapReduce system. Two main parts are implemented, one is a worker process that calls `Map` and `Reduce` functions and handles reading and writing files, and the other is a coordinator process that hands out tasks to workers and copes with failed workers.

Before you start the lab, must be familiar with `Golang concurrency programming` and `RPC`. Understanding the MapReduce paper is also essential.

## Experiment Description

To get started the lab 1, you should read the lab document.

> [https://pdos.csail.mit.edu/6.824/labs/lab-mr.html](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)

Because the RPC using based on `Unix Socket`, you must have a Linux System to Finish the experiment. (I choose the `Ubuntu 22.04.2`)

A simple sequential MapReduce version in `src/main/mrsequential.go`. The code is helpful to understand to the whole system works.

Access to the system is `main/mrcoordinator.go`. When starting the program, it will call `MakeCoordinator()` to create a master. Secondly, the `server()` function will start `Unix Socket Listen`. Then, the main process will call `Done()` to check the total MapReduce whether finished. All of the above functions must be implemented in `mr/coordinator.go`.

As for the worker, we should put our implementation in `mr/worker.go`, and also refer the `mrsequential.go`.

Using the script `test-mr.sh` to check the program pass. Sometimes `ctrl + c` might not exis program, you should use `ps` to find the `pid` of the mrcoordinator process and `kill` it.

## Implementation

### Execution Overview

The master first assigns all map tasks to workers. In the `map` state, the worker will generate intermediate files `"mr-X-Y"`, where X is the Map task number, and Y is the Reduce task number. Then in the `reduce` state, the worker will input intermediate files, after being processed by reduce function, it will put the results into `"mr-out-Y"`.

### Master Implementation

No lock is used, just ensured the data operation is thread-safe.

First, the master will allocate tasks to workers, so we should definite task data structure. Filename handled by map function, Id is the worker num and it could be a map worker or a reduce worker. So we can merge the two tasks.

```go
type Task struct {
    Filename string
    Id int
}
```

Then, the master should record the state of the system. There is `map, reduce, finish` three states, just used `int32 (0,1,2)`to refer to them. Because we can't use lock, make sure the work gets race task is used `channel` for synchronization. The `MapTaskNum` and `ReduceTaskNum` also needs to record the number of tasks assigned. Our system must tolerate machine failures gracefully. We should record the working time(`Unix TimeStamp`), a working time of more than `10 seconds` is considered to be the failure of the machine. Each worker's time has to be recorded and modified, and there are data races. To get good performance, I used `sync.Map`.

```go
type Coordinator struct {
	State int32 // 0 map 1 reduce 2 finish

	MapTaskNum    int
	ReduceTaskNum int

	MapTask    chan Task
	ReduceTask chan Task

	TaskTime sync.Map

	files []string
}

const TimeOut = 10

type TimeStamp struct {
	Time int64
	Fin  bool
}
```

As the system begins, the master assigns map tasks to `MapTask chan` and record the working start time in `TaskTime`.

```go
for i, file := range files {
		now := time.Now().Unix()
		c.MapTask <- Task{FileName: file, Id: i} // send to chan
		c.TaskTime.Store(i, TimeStamp{now, false}) // record time
}
```

When a work sends a `request` to the master, it should get `worker task, worker num, system state and a succeed mark` to confirm that the task was acquired.

```go
type TaskRespond struct {
	WorkerTask    Task
	MapTaskNum    int
	ReduceTaskNum int
	State         int32

	Succeed bool
}
```

The master will send a `reply` to workers in different states.

```go
func (c *Coordinator) GetTask(args *TaskRequest, reply *TaskRespond) error {
	state := atomic.LoadInt32(&c.State)
	if state == 0 {
		maptask, ok := <-c.MapTask
		if ok {
			reply.WorkerTask = maptask
		} else {
			return errors.New("fail call task")
		}
	} else if state == 1 {
		reducetask, rok := <-c.ReduceTask
		if rok {
			reply.WorkerTask = reducetask
		} else {
			return errors.New("fail call task")
		}
	}
	reply.MapTaskNum = c.MapTaskNum
	reply.ReduceTaskNum = c.ReduceTaskNum
	reply.State = c.State

	return nil
}
```

If a worker finish work, the master also will handle it in different states.

```go
func (c *Coordinator) GetTaskFin(args *TaskRespond, _ *ExampleReply) error {
	state := args.State

	now := time.Now().Unix()
	id := args.WorkerTask.Id
	start, _ := c.TaskTime.Load(id)
	total := now - start.(TimeStamp).Time

	if total > TimeOut {
		return errors.New("fail fin task")
	}
	c.TaskTime.Store(id, TimeStamp{now, true})

	if state == 0 {
		if lenTaskFin(&c.TaskTime) == c.MapTaskNum {
			close(c.MapTask) // close chan
			c.TaskTime = sync.Map{}
			for i := 0; i < c.ReduceTaskNum; i++ {
				c.ReduceTask <- Task{Id: i}
				c.TaskTime.Store(i, TimeStamp{now, false})
			}
			atomic.StoreInt32(&c.State, 1)
		}
	} else if state == 1 {
		if lenTaskFin(&c.TaskTime) == c.ReduceTaskNum {
			atomic.StoreInt32(&c.State, 2)
			close(c.ReduceTask) // close chan
		}
	}
	return nil
}
```

At last, to tolerate machine failures we should assign its worker to another worker machine. In this system, I create a `goroutine` to check task finish periodically`(2 seconds)`.

```go
func (c *Coordinator) checkWorker() {
	for {
		time.Sleep(2 * time.Second)
		state := atomic.LoadInt32(&c.State)
		if empty(&c.TaskTime) {
			continue
		}
		if state == 0 {
			for i := 0; i < c.MapTaskNum; i++ {
				now := time.Now().Unix()
				tmp, _ := c.TaskTime.Load(i)
				if !tmp.(TimeStamp).Fin && now-tmp.(TimeStamp).Time > TimeOut {
					fmt.Println("map task timeout")
					c.MapTask <- Task{FileName: c.files[i], Id: i}
					c.TaskTime.Store(i, TimeStamp{now, false})
				}
			}
		} else if state == 1 {
			for i := 0; i < c.ReduceTaskNum; i++ {
				now := time.Now().Unix()
				tmp, _ := c.TaskTime.Load(i)
				if !tmp.(TimeStamp).Fin && now-tmp.(TimeStamp).Time > TimeOut {
					fmt.Println("reduce task timeout")
					c.ReduceTask <- Task{Id: i}
					c.TaskTime.Store(i, TimeStamp{now, false})
				}
			}
		}
	}
}
```

### Worker Implementation

The worker loop fetches tasks from the master for execution

```go
func Worker(mapf func(string, string) []KeyValue, reducef func(string, []string) string) {
	for {
		args := TaskRequest{}
		reply := TaskRespond{}
		CallGetTask(&args, &reply)
		if !reply.Succeed {
			goto sleep
		}
		if reply.State == 0 {
			doMapTask(&reply, mapf)
		} else if reply.State == 1 {
			doReduceTask(&reply, reducef)
		} else if reply.State == 2 {
			break
		}
		CallTaskFin(&reply)
	sleep:
		time.Sleep(1 * time.Second)
	}
}
```

The map and reduce work can refer to some code from `mrsequential.go` for reading Map input files, for sorting `intermedate key/value pairs` between the Map and Reduce, and for storing Reduce output in files.

To ensure that nobody observes partially written files in the presence of crashes, the MapReduce paper mentions the trick of using a temporary file and atomically renaming it once it is completely written. You can use `ioutil.TempFile` to create a temporary file and `os.Rename` to atomically rename it.