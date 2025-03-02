+++
author = "GreyWind"
title = "Quickly locate program memory leaks"
date = "2024-02-25"
description = "Quickly locate program memory leaks"
tags = [
    "ops",
]
image = "mem-leak.png"
+++
## Introduction

**Memory leaks** have always been a bit of a nuisance in production environments. Unlike memory leaks where memory grows quickly, memory leaks where memory rises slowly are more troublesome. Therefore, program memory needs to be observed.

## Tools

Use different tools for analyzing different languages.

### C/C++

#### jemalloc

**Introduction**

`jemalloc` is an open source memory allocator for memory allocation and management. It was created by Jason Evans, a developer of the FreeBSD system, and is primarily designed to improve the performance of multithreaded programs, especially in highly concurrent environments. jemalloc is available on many Unix-like systems, including FreeBSD, Linux, and macOS.

One of the design goals of `jemalloc` is to avoid memory fragmentation, especially in multithreaded environments. It employs a number of advanced algorithms and techniques to provide efficient memory allocation and release operations. jemalloc has been used in a wide variety of projects, including some large open source software and systems.

**Install**

```bash
wget https://github.com/jemalloc/jemalloc/archive/refs/tags/5.3.0.tar.gz
```

```bash
tar -zxv -f 5.3.0.tar.gz
cd jemalloc-5.3.0
# 要生成jeprof工具，需要修改autogen.sh
sh autogen.sh
make
make install
```

Note: To install the `jeprof` tool, modify the `autogen.sh` file. You also need to install the `autoconf` tool.

```bash
yum install autoconf
```

```bash
#!/bin/sh
#autogen.sh
for i in autoconf; do
    echo "$i"
    $i
    if [ $? -ne 0 ]; then
        echo "Error $? in $i"
        exit 1
    fi
done

echo "./configure --enable-autogen $@"
./configure --enable-autogen --enable-prof $@ # add in this line
if [ $? -ne 0 ]; then
    echo "Error $? in ./configure"
    exit 1
fi
```

**Usage**

In the simplest case, you can check what memory is still allocated but not freed when the program exits.

```c
#include <stdio.h>  
#include <stdlib.h>  
  
void do_something(size_t i)  
{  
  // Leak some memory.  
  malloc(i * 1024);  
}  
  
void do_something_else(size_t i)  
{  
  // Leak some memory.  
  malloc(i * 4096);  
}  
  
int main(int argc, char **argv)  
{  
  size_t i, sz;  
  
  for (i = 0; i < 80; i++)  
  {  
    do_something(i);  
  }  
  
  for (i = 0; i < 40; i++)  
  {  
    do_something_else(i);  
  }  
  
  return 0;  
}
```

Note: Do not include the `jemalloc header file` in the code here, and do not need to link the `jemalloc library` when compiling. You only need to start specifying the path to the `jemalloc library` via `LD_PRELOAD`.

```shell
gcc jemalloc-demo.c -o jemalloc-demo
```

start profiling

```bash
export MALLOC_CONF=prof_leak:true,lg_prof_sample:0,prof_final:true
export LD_PRELOAD=/root/jemalloc-5.3.0/lib/libjemalloc.so.2 
./jemalloc-demo
```

> MALLOC\_CONF parameter meaning
> 
> * prof\_leak: whether to turn on memory leak reporting.
>     
> * lg\_prof\_sample: the interval between memory samples, i.e., how much memory to allocate to start a sample at each interval
>     
> * prof: this parameter is specified during compilation.
>     
> * prof\_prefix: the prefix of the sampling file name
>     
> * prof\_ative: whether to start prof immediately or not.
>     
> * prof\_final: dumps the final memory usage
>     
> * lg\_prof\_interval: how much memory to dump per allocation
>     
> * prof\_gdump: dump each time memory reaches a new high
>     

Viewing Memory Allocations with the `jeprof`

```bash
jeprof ./jemalloc-demo <heap_file>
```

**Code is run to a specific location profiling**

```c
#include <stdio.h>
#include <stdlib.h>
#include <jemalloc/jemalloc.h>

void do_something(size_t i)
{
	malloc(i * 1024);
}

void do_something_else(size_t i)
{
	malloc(i * 4096);
}

  

int main(int argc, char **argv)
{
	size_t i, sz;
	for (i = 0; i < 80; ++i)
	{
		do_something(i);
	}

	// mallctl("prof.dump", NULL, NULL, NULL, 0);
	bool active = true;
	mallctl("prof.active", NULL, NULL, &active, sizeof(bool));
	for (i = 0; i < 40; ++i)
	{
		do_something_else(i);
	}
	mallctl("prof.dump", NULL, NULL, NULL, 0);
	return 0;
}
```

Note: Compilation requires link

```bash
gcc jemalloc-manual.c -o jemalloc-manual -ljemalloc
```

`jeprof` is a jemalloc-related tool used to generate performance analysis reports for `jemalloc`. It can help developers identify memory allocation problems and performance bottlenecks in their programs so that they can optimize their code. `jeprof` is usually used in conjunction with `jemalloc`, and by analyzing a program's memory usage, developers can better understand memory allocation patterns, find potential problems, and make improvements accordingly.

`jeporf` compares two dumps

```bash
jeprof <path_to_binary> --base=heap1 heap2
```

`jeprof` allows you to visualize the `profile.out`

```bash
jeprof --show_bytes --pdf <path_to_binary> ./profile.out > ./profile.pdf
```

Note: The use of `jeprof` drawing `pdf`, you need `Graphiz` tools in the `dot` to generate `graphics` and `GhostScript` tools `ps2pdf` will be converted `PostScript` to `PDF`.

```bash
yum install graphviz ghostscript
```

### Rust

#### tikv\_jemalloc

Using `jemalloc` as a rust allocator

Add in `Cargo.toml`

```toml
tikv-jemallocator = { version = "0.5.4", features = ["profiling" "unprefixed_malloc_on_supported_platforms"] }
```

Add in Code

```rust
#[cfg(not(target_env = "msvc"))]
#[global_allocator]
static ALLOC: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;

#[allow(non_upper_case_globals)]
#[export_name = "malloc_conf"]
pub static malloc_conf: &[u8] = b"prof:true,prof_active:true,lg_prof_sample:19\0";
```

Then you can use jemalloc, which gives you the heap file periodically by setting a parameter, and you can view the file with `jeprof`.

But here the jemalloc\_pprof tool is used to convert the heap file to the format of the pprof tool. Add in `Cargo.toml`

```ini
jemalloc_pprof="0.1.0"
```

you can change code like this:

```rust
#[tokio::main]
async fn main() {
    let mut v = vec![];
    for i in 0..1000000 {
        v.push(i);
    }
    let app = axum::Router::new()
        .route("/debug/pprof/heap", axum::routing::get(handle_get_heap));

    // run our app with hyper, listening globally on port 3000
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}

use axum::http::StatusCode;
use axum::response::IntoResponse;

pub async fn handle_get_heap() -> Result<impl IntoResponse, (StatusCode, String)> {
    let mut prof_ctl = jemalloc_pprof::PROF_CTL.as_ref().unwrap().lock().await;
    require_profiling_activated(&prof_ctl)?;
    let pprof = prof_ctl
        .dump_pprof()
        .map_err(|err| (StatusCode::INTERNAL_SERVER_ERROR, err.to_string()))?;
    Ok(pprof)
}

/// Checks whether jemalloc profiling is activated an returns an error response if not.

fn require_profiling_activated(prof_ctl: &jemalloc_pprof::JemallocProfCtl) -> Result<(), (StatusCode, String)> {
    if prof_ctl.activated() {
        Ok(())
    } else {
        Err((axum::http::StatusCode::FORBIDDEN, "heap profiling not activated".into()))
    }
}
```

curl the endpoint and explore it with any pprof compatible tooling.

```bash
curl localhost:3000/debug/pprof/heap > heap.pb.gz

pprof -http=:8080 heap.pb.gz
```

Note: If the program is executed in `docker`, `kubernetes`, etc., the binaries need to be extracted, and `pprof` needs to be configured with startup parameters.

```bash
PPROF_BINARY_PATH=. pprof -http=:8080 heap.pb.gz
```

### Java

#### async\_profiler

async-profiler is an open source Java performance analysis tool , the principle is based on `HotSpot`s API, with minimal performance overhead to collect program run-time stack information , memory allocation and other information for analysis .

async-profiler can be used to analyze

* CPU cycles
    
* Hardware and Software performance counters like `cache misses`, `branch misses`, `page fault`, `context switches` etc.
    
* Allocations in Java Heap
    
* Contented lock attempts, including both Java object monitors and ReentrantLocks
    

**Install**

```bash
$ tree async-profiler-3.0-macos
async-profiler-3.0-macos
├── CHANGELOG.md
├── LICENSE
├── README.md
├── bin
│   └── asprof
└── lib
    ├── async-profiler.jar
    ├── converter.jar
    └── libasyncProfiler.dylib

3 directories, 7 files
```

**Usage**

```shell
$ cd bin
$ ./asprof
Usage: asprof [action] [options] <pid>
Actions:
  start             start profiling and return immediately
  resume            resume profiling without resetting collected data
  stop              stop profiling
  dump              dump collected data without stopping profiling session
  check             check if the specified profiling event is available
  status            print profiling status
  meminfo           print profiler memory stats
  list              list profiling events supported by the target JVM
  load              load agent library (jattach action)
  jcmd              run JVM diagnostic command (jattach action)
  collect           collect profile for the specified period of time
                    and then stop (default action)
Options:
  -e event          profiling event: cpu|alloc|lock|cache-misses etc.
  -d duration       run profiling for <duration> seconds
  -f filename       dump output to <filename>
  -i interval       sampling interval in nanoseconds
  -j jstackdepth    maximum Java stack depth
  -t, --threads     profile different threads separately
  -s, --simple      simple class names instead of FQN
  -n, --norm        normalize names of hidden classes / lambdas
  -g, --sig         print method signatures
  -a, --ann         annotate Java methods
  -l, --lib         prepend library names
  -o fmt            output format: flat|traces|collapsed|flamegraph|tree|jfr
  -I include        output only stack traces containing the specified pattern
  -X exclude        exclude stack traces with the specified pattern
  -L level          log level: debug|info|warn|error|none
  -F features       advanced stack trace features: vtable, comptask
  -v, --version     display version string

  --title string    FlameGraph title
  --minwidth pct    skip frames smaller than pct%
  --reverse         generate stack-reversed FlameGraph / Call tree

  --loop time       run profiler in a loop
  --alloc bytes     allocation profiling interval in bytes
  --live            build allocation profile from live objects only
  --lock duration   lock profiling threshold in nanoseconds
  --wall interval   wall clock profiling interval
  --total           accumulate the total value (time, bytes, etc.)
  --all-user        only include user-mode events
  --sched           group threads by scheduling policy
  --cstack mode     how to traverse C stack: fp|dwarf|lbr|vm|no
  --signal num      use alternative signal for cpu or wall clock profiling
  --clock source    clock source for JFR timestamps: tsc|monotonic
  --begin function  begin profiling when function is executed
  --end function    end profiling when function is executed
  --ttsp            time-to-safepoint profiling
  --jfrsync config  synchronize profiler with JFR recording
  --fdtransfer      use fdtransfer to serve perf requests
                    from the non-privileged target

<pid> is a numeric process ID of the target JVM
      or 'jps' keyword to find running JVM automatically
      or the application name as it would appear in the jps tool

Example: asprof -d 30 -f profile.html 3456
         asprof start -i 1ms jps
         asprof stop -o flat jps
         asprof -d 5 -e alloc MyAppName
```

```java
import java.util.ArrayList;
import java.util.Random;
import java.util.UUID;

/**
 * <p>
 * 模拟热点代码
 *
 * @Author niujinpeng
 */
public class HotCode {

    private static volatile int value;

    private static Object array;

    public static void main(String[] args) {
        while (true) {
            hotmethod1();
            hotmethod2();
            hotmethod3();
            allocate();
        }
    }

    /**
     * 生成 6万长度的数组
     */
    private static void allocate() {
        array = new int[6 * 1000];
        array = new Integer[6 * 1000];
    }

    /**
     * 生成一个UUID
     */
    private static void hotmethod3() {
        ArrayList<String> list = new ArrayList<>();
        UUID uuid = UUID.randomUUID();
        String str = uuid.toString().replace("-", "");
        list.add(str);
    }

    /**
     * 数字累加
     */
    private static void hotmethod2() {
        value++;
    }

    /**
     * 生成一个随机数
     */
    private static void hotmethod1() {
        Random random = new Random();
        int anInt = random.nextInt();
    }
}
```

get java pid

```bash
jps
ps -ef | grep java
```

profiling cpu state

```shell
./asprof -d 20 -f cpu.html <java-pid>
```

profiling memory state

```shell
./asprof -e alloc -d 20 -f alloc.html <java-pid>
```

### Go

#### pprof

`Golang pprof` is the official `golang` profiling tool, very easy to use.

Add in Code

```go
import _ "net/http/pprof"

go func() {
	http.ListenAndServe("localhost:6060", nil)
}()
```

Analyzing memory with the `pprof` tool

```bash
go tool pprof -seconds=10 -http=:9999 http://localhost:6060/debug/pprof/heap
```

Sometimes there may be network isolation problems, can not directly from the development machine to access the test machine, online machine, or test machine, online machine does not have go installed, then you can do this

```bash
curl http://localhost:6060/debug/pprof/heap?seconds=30 > heap.out

go tool pprof -http=:9999 heap.out
```

### BCC

`bcc` is based on `ebpf`, and uses the `memleak` utility to detect memory leaks.

**Install**

```bash
yum install bcc-tools
```

```bash
yum install kernel-devel-$(uname -r)
```

```bash
memleak -p <pid>
```

### GDB

If it still doesn't work, you can use `gdb` to dump the memory state directly.

Observing memory at different times with `pmap`.

```bash
pmap -x <pid> > 1.txt
pmap -x <pid> > 2.txt
```

Compare to see suspicious areas of memory growth, use `gdb` to save memory state.

```bash
gdb -p <pid>

gdb> dump /path/to/save/dump_file <start_address> <end_address>
```

Use `strings` or `hexdump` to view the contents of the binary file.

```bash
strings ./dump_file
hexdump -C ./dump_file
```

## Reference

[https://github.com/jemalloc/jemalloc](https://github.com/jemalloc/jemalloc)

[https://www.polarsignals.com/blog/posts/2023/12/20/rust-memory-profilin](https://www.polarsignals.com/blog/posts/2023/12/20/rust-memory-profiling)g

[https://github.com/async-profiler/async-profiler](https://github.com/async-profiler/async-profiler)

[https://github.com/google/pprof](https://github.com/google/pprof)

[https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)