---
tags:
  - Golang
  - debug
  - pprof
  - gc
  - heap
  - memory
---
# Go Debug Tools

This note summarizes common Go debugging and profiling tools for checking memory, heap, goroutines, GC, CPU usage, blocking, mutex contention, execution trace, and runtime behavior.

## Quick Checklist

| Problem | Tool |
| --- | --- |
| High memory usage | `pprof` heap profile, `runtime.ReadMemStats`, `GODEBUG=gctrace=1` |
| Memory leak | `pprof` heap diff, goroutine profile, object allocation profile |
| High CPU | `pprof` CPU profile, `top`, `perf` |
| Too many goroutines | `pprof` goroutine profile, `runtime.NumGoroutine()` |
| GC too frequent or too slow | `GODEBUG=gctrace=1`, heap profile, `GOMEMLIMIT`, `GOGC` |
| Lock contention | mutex profile, block profile |
| Race condition | `go test -race`, `go run -race` |
| Debug step by step | `dlv` |
| Runtime timeline | `go tool trace` |

## Runtime Memory Stats

Use `runtime.ReadMemStats` when you need a simple in-process memory snapshot.

```go
package main

import (
	"fmt"
	"runtime"
)

func printMemStats() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)

	fmt.Printf("Alloc = %d MB\n", m.Alloc/1024/1024)
	fmt.Printf("TotalAlloc = %d MB\n", m.TotalAlloc/1024/1024)
	fmt.Printf("Sys = %d MB\n", m.Sys/1024/1024)
	fmt.Printf("NumGC = %d\n", m.NumGC)
	fmt.Printf("HeapAlloc = %d MB\n", m.HeapAlloc/1024/1024)
	fmt.Printf("HeapSys = %d MB\n", m.HeapSys/1024/1024)
	fmt.Printf("HeapObjects = %d\n", m.HeapObjects)
}
```

Important fields:

- `Alloc`: currently allocated heap memory.
- `TotalAlloc`: total allocated memory since process start.
- `Sys`: memory requested from OS.
- `HeapAlloc`: current heap allocation.
- `HeapSys`: heap memory obtained from OS.
- `HeapObjects`: number of allocated heap objects.
- `NumGC`: number of completed GC cycles.

## GC Debug

### Print GC Logs

```shell
GODEBUG=gctrace=1 go run main.go
```

Example output:

```text
gc 1 @0.012s 3%: 0.012+0.71+0.003 ms clock, 0.097+0.12/0.60/0+0.027 ms cpu, 4->4->1 MB, 5 MB goal, 8 P
```

Key points:

- `gc 1`: GC cycle number.
- `@0.012s`: time since program started.
- `4->4->1 MB`: heap size before GC, during GC, and live heap after GC.
- `5 MB goal`: next heap target.
- Large live heap after GC usually means real retained objects, not temporary allocation only.

### Control GC

```shell
# default is 100
GOGC=100 go run main.go

# make GC less frequent
GOGC=200 go run main.go

# make GC more frequent
GOGC=50 go run main.go

# disable GC, only for experiments
GOGC=off go run main.go
```

From Go 1.19, `GOMEMLIMIT` can set a soft memory limit:

```shell
GOMEMLIMIT=512MiB go run main.go
```

## pprof

`pprof` is the most important Go profiling tool. It can analyze CPU, heap, goroutine, block, mutex, thread creation, and allocation profiles.

### Enable HTTP pprof

Add this import:

```go
import _ "net/http/pprof"
```

Start an HTTP server:

```go
go func() {
	http.ListenAndServe("localhost:6060", nil)
}()
```

Then check profiles:

```shell
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof http://localhost:6060/debug/pprof/goroutine
go tool pprof http://localhost:6060/debug/pprof/block
go tool pprof http://localhost:6060/debug/pprof/mutex
```

Open web UI:

```shell
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap
```

### Common pprof Commands

Inside `pprof`:

```text
top
top -cum
list functionName
web
peek functionName
traces
```

Meaning:

- `top`: functions using most resources directly.
- `top -cum`: functions with highest cumulative cost.
- `list`: show annotated source code.
- `web`: generate call graph.
- `traces`: show call traces.

### Heap Profile

Check current heap:

```shell
go tool pprof http://localhost:6060/debug/pprof/heap
```

Check allocation history:

```shell
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap
go tool pprof -alloc_objects http://localhost:6060/debug/pprof/heap
```

Useful modes:

- `inuse_space`: memory still retained.
- `inuse_objects`: objects still retained.
- `alloc_space`: total allocated bytes.
- `alloc_objects`: total allocated objects.

For leak analysis, focus on `inuse_space` and compare two heap profiles over time.

### Save and Compare Heap Profiles

```shell
curl -o heap1.pb.gz http://localhost:6060/debug/pprof/heap
sleep 60
curl -o heap2.pb.gz http://localhost:6060/debug/pprof/heap

go tool pprof -base heap1.pb.gz heap2.pb.gz
```

If retained memory keeps growing, inspect:

```text
top
top -cum
list functionName
```

### CPU Profile

```shell
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

Use this when CPU is high or latency is caused by expensive computation.

### Goroutine Profile

```shell
curl http://localhost:6060/debug/pprof/goroutine?debug=2
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

Use this to find:

- goroutine leaks;
- goroutines blocked on channel receive/send;
- goroutines stuck on lock or IO;
- unexpected background workers.

### Block Profile

Block profile is disabled by default. Enable it in code:

```go
runtime.SetBlockProfileRate(1)
```

Then collect:

```shell
go tool pprof http://localhost:6060/debug/pprof/block
```

It helps find blocking on:

- channel operations;
- `select`;
- mutexes;
- condition variables.

### Mutex Profile

Enable mutex profile:

```go
runtime.SetMutexProfileFraction(1)
```

Then collect:

```shell
go tool pprof http://localhost:6060/debug/pprof/mutex
```

It helps find lock contention.

## go test Profiles

Profiles can also be generated from tests.

```shell
go test -bench=. -benchmem
go test -bench=. -cpuprofile cpu.out
go test -bench=. -memprofile mem.out
go test -bench=. -blockprofile block.out
go test -bench=. -mutexprofile mutex.out
```

Analyze:

```shell
go tool pprof cpu.out
go tool pprof mem.out
```

## Trace

`go tool trace` shows a timeline of goroutines, scheduler events, syscalls, network blocking, and GC.

Generate trace from tests:

```shell
go test -trace trace.out ./...
go tool trace trace.out
```

Generate trace in code:

```go
f, _ := os.Create("trace.out")
defer f.Close()

trace.Start(f)
defer trace.Stop()
```

Analyze:

```shell
go tool trace trace.out
```

Use trace when you need to understand scheduling, blocking, latency spikes, or GC timing across the whole program.

## Race Detector

Use race detector to find data races.

```shell
go test -race ./...
go run -race main.go
go build -race
```

Common race patterns:

- shared map without lock;
- shared variable updated by multiple goroutines;
- closure capturing loop variable incorrectly;
- object reused while another goroutine still accesses it.

## Delve

`dlv` is the standard debugger for Go.

Install:

```shell
go install github.com/go-delve/delve/cmd/dlv@latest
```

Debug executable:

```shell
dlv exec ./app -- arg1 arg2
```

Debug package:

```shell
dlv debug .
```

Attach to process:

```shell
dlv attach <pid>
```

Common commands:

```text
b main.main
c
n
s
locals
args
p variableName
bt
goroutines
goroutine <id>
```

## Runtime Debug Endpoints

Useful pprof HTTP endpoints:

```text
/debug/pprof/
/debug/pprof/heap
/debug/pprof/profile?seconds=30
/debug/pprof/goroutine?debug=2
/debug/pprof/block
/debug/pprof/mutex
/debug/pprof/threadcreate
/debug/pprof/trace?seconds=5
```

Do not expose these endpoints publicly in production. Bind them to `localhost`, internal network, or protect them behind authentication.

## OS Tools

Linux process memory:

```shell
ps -o pid,rss,vsz,cmd -p <pid>
top -p <pid>
cat /proc/<pid>/status
cat /proc/<pid>/smaps
```

Check file descriptors:

```shell
ls /proc/<pid>/fd | wc -l
lsof -p <pid>
```

Check threads:

```shell
ps -T -p <pid>
```

Check syscalls:

```shell
strace -p <pid>
```

Linux performance profiling:

```shell
perf top -p <pid>
perf record -F 99 -p <pid> -g -- sleep 30
perf report
```

## Common Investigation Flow

### High Memory

1. Check process RSS by `top` or `ps`.
2. Check Go heap by `pprof` heap profile.
3. Compare two heap profiles with `-base`.
4. Check goroutine count and goroutine profile.
5. Check GC logs with `GODEBUG=gctrace=1`.
6. If RSS is high but Go heap is not high, check stacks, mmap, cgo, file cache, fragmentation, or OS-level memory.

### High CPU

1. Collect CPU profile for 30 seconds.
2. Use `top` and `top -cum`.
3. Check whether CPU is in user code, runtime, GC, syscall, or cgo.
4. Use `go tool trace` if scheduling or blocking is unclear.

### Goroutine Leak

1. Check `runtime.NumGoroutine()`.
2. Dump `/debug/pprof/goroutine?debug=2`.
3. Group by stack trace.
4. Look for goroutines waiting on channels, locks, timers, or network reads.

### GC Problem

1. Enable `GODEBUG=gctrace=1`.
2. Check heap growth and live heap after GC.
3. Use heap profile to find retained objects.
4. Tune allocation behavior first.
5. Tune `GOGC` or `GOMEMLIMIT` only after understanding allocation and retention.

## Summary

- Use `pprof` first for CPU, heap, goroutine, block, and mutex problems.
- Use `GODEBUG=gctrace=1` to understand GC behavior.
- Use `go tool trace` for scheduler, blocking, and latency timeline.
- Use `go test -race` for concurrency data races.
- Use `dlv` for step debugging.
- Use OS tools when Go heap does not explain process RSS or system behavior.
