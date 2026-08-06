# C# / .NET Debug Tools

This page summarizes the .NET diagnostics tools that are closest to common Java
runtime tools such as `jcmd`, `jps`, `jstat`, `jstack`, heap dump collection,
and heap dump analysis.

> Scope: modern .NET / .NET Core applications. Some tools also work with .NET
> Framework on Windows, but the CLI diagnostics workflow is strongest for .NET
> 5+ and later.

## Java To .NET Tool Map

| Java Tool | Common Java Use | Similar .NET Tool | .NET Use |
|---|---|---|---|
| `jps` | List JVM processes | `dotnet-dump ps`, `dotnet-stack ps`, `dotnet-gcdump ps`, `dotnet-counters ps` | List .NET processes that diagnostics tools can attach to |
| `jcmd` | Send diagnostic commands to JVM | `dotnet-dump`, `dotnet-gcdump`, `dotnet-trace`, `dotnet-counters`, `dotnet-monitor` | Collect dumps, GC dumps, traces, counters, logs, and metrics |
| `jstat` | Monitor GC / class / compiler stats | `dotnet-counters` | Live runtime counters: GC heap size, allocation rate, CPU, exceptions, ThreadPool |
| `jstack` | Print thread stacks | `dotnet-stack`, `dotnet-dump analyze` + SOS | Print live managed stacks or inspect stacks from a dump |
| `jmap -dump` | Create heap dump | `dotnet-dump collect`, `dotnet-gcdump collect` | Create full process dump or GC heap dump |
| `jmap -histo` | Heap histogram | `dotnet-gcdump report`, `dotnet-dump analyze` + `dumpheap -stat` | Show heap type statistics |
| MAT / VisualVM | Analyze heap dump | Visual Studio, PerfView, `dotnet-dump analyze`, `dotnet-gcdump report` | Analyze memory, object counts, references, leaks, and GC behavior |
| Java Flight Recorder | Runtime tracing / profiling | `dotnet-trace`, PerfView, Visual Studio Profiler | CPU, GC, exceptions, EventPipe events, runtime traces |

## Install CLI Diagnostic Tools

Install the tools globally:

```powershell
dotnet tool install --global dotnet-counters
dotnet tool install --global dotnet-dump
dotnet tool install --global dotnet-gcdump
dotnet tool install --global dotnet-stack
dotnet tool install --global dotnet-trace
dotnet tool install --global dotnet-monitor
dotnet tool install --global dotnet-symbol
```

Update existing tools:

```powershell
dotnet tool update --global dotnet-counters
dotnet tool update --global dotnet-dump
dotnet tool update --global dotnet-gcdump
dotnet tool update --global dotnet-stack
dotnet tool update --global dotnet-trace
dotnet tool update --global dotnet-monitor
dotnet tool update --global dotnet-symbol
```

Check installed tools:

```powershell
dotnet tool list --global
```

## Process Discovery

Use these commands like Java `jps`.

```powershell
dotnet-dump ps
dotnet-stack ps
dotnet-gcdump ps
dotnet-counters ps
```

Typical output contains:

- Process ID
- Process name
- Runtime executable path
- Command-line arguments, depending on tool version and OS permissions

## Live Runtime Counters

`dotnet-counters` is the closest tool to Java `jstat`. It monitors live runtime
metrics through .NET EventCounters and Meter APIs.

Monitor default runtime counters:

```powershell
dotnet-counters monitor --process-id <PID>
```

Useful providers:

```powershell
dotnet-counters monitor --process-id <PID> System.Runtime
dotnet-counters monitor --process-id <PID> Microsoft.AspNetCore.Hosting
dotnet-counters monitor --process-id <PID> Microsoft-AspNetCore-Server-Kestrel
```

Useful `System.Runtime` counters:

| Counter | Meaning |
|---|---|
| `cpu-usage` | Process CPU usage |
| `working-set` | Process working set memory |
| `gc-heap-size` | Managed heap size |
| `gen-0-gc-count` / `gen-1-gc-count` / `gen-2-gc-count` | GC frequency by generation |
| `alloc-rate` | Managed allocation rate |
| `exception-count` | Exceptions thrown |
| `threadpool-thread-count` | ThreadPool worker thread count |
| `threadpool-queue-length` | Queued ThreadPool work items |
| `monitor-lock-contention-count` | Lock contention count |

Common scenarios:

```powershell
# Quick health check
dotnet-counters monitor --process-id <PID> System.Runtime

# Collect counters to a file for later review
dotnet-counters collect --process-id <PID> --format csv --output counters.csv
```

## Thread Stack Analysis

### Live Thread Stacks

`dotnet-stack` is the closest tool to Java `jstack`.

```powershell
dotnet-stack report --process-id <PID>
```

Use it for:

- ThreadPool starvation
- Deadlocks
- Requests stuck in synchronous blocking calls
- High CPU investigation before collecting a deeper trace

### Dump-Based Thread Stacks

Collect a dump:

```powershell
dotnet-dump collect --process-id <PID> --type Full --output app.dmp
```

Analyze it:

```powershell
dotnet-dump analyze app.dmp
```

Useful SOS commands inside `dotnet-dump analyze`:

```text
threads
clrthreads
clrstack
clrstack -all
syncblk
dumpasync
dumpthreadpool
exit
```

Use `syncblk` to inspect managed lock contention. Use `dumpasync` when async
tasks are suspected.

## Heap Dump Collection

.NET has two common dump types for memory investigation:

| Dump Type | Tool | Best For | Tradeoff |
|---|---|---|---|
| Full process dump | `dotnet-dump collect --type Full` | Deep memory, threads, locks, exceptions, object references | Larger file, more sensitive data, more disk usage |
| GC dump | `dotnet-gcdump collect` | Managed heap type statistics and leak triage | Smaller file, less complete than full dump |

### Full Process Dump

```powershell
dotnet-dump collect --process-id <PID> --type Full --output app-full.dmp
```

Other dump types:

```powershell
dotnet-dump collect --process-id <PID> --type Mini --output app-mini.dmp
dotnet-dump collect --process-id <PID> --type Heap --output app-heap.dmp
dotnet-dump collect --process-id <PID> --type Triage --output app-triage.dmp
```

Use `Full` when you need the most complete memory analysis. Use smaller dump
types when production disk, transfer, or privacy constraints matter.

### GC Heap Dump

```powershell
dotnet-gcdump collect --process-id <PID> --output app.gcdump
```

Generate a quick heap report:

```powershell
dotnet-gcdump report app.gcdump
```

Use `gcdump` when you need a lightweight managed heap histogram similar to:

```bash
jmap -histo <pid>
```

## Heap Dump Analysis

### Analyze With `dotnet-dump`

Open a full dump:

```powershell
dotnet-dump analyze app-full.dmp
```

Common SOS commands:

```text
dumpheap -stat
dumpheap -type <TypeName>
dumpobj <ObjectAddress>
gcroot <ObjectAddress>
eeheap -gc
gcheapstat
finalizequeue
analyzeoom
clrstack
clrthreads
syncblk
```

Memory leak workflow:

1. Run `dumpheap -stat` to find large object types.
2. Run `dumpheap -type <TypeName>` to list instances.
3. Pick a large or suspicious object address.
4. Run `gcroot <ObjectAddress>` to find why the object is still referenced.
5. Check static fields, event handlers, caches, timers, queues, and long-lived
   singletons.

Example:

```text
dumpheap -stat
dumpheap -type MyApp.CustomerSession
dumpobj 000001f4aabbccdd
gcroot 000001f4aabbccdd
```

### Analyze With Visual Studio

Visual Studio can open `.dmp` files and inspect:

- Managed call stacks
- Threads
- Exceptions
- Modules
- Memory usage
- Heap objects, depending on dump type and available symbols

This is often the easiest option on Windows.

### Analyze With PerfView

PerfView is useful for:

- GC heap investigation
- Allocation analysis
- CPU profiling
- EventPipe / ETW trace analysis
- Comparing memory snapshots

PerfView is especially useful when you need deeper allocation and GC behavior
than a simple heap histogram provides.

## CPU And Performance Tracing

`dotnet-trace` is similar in spirit to Java Flight Recorder because it collects
runtime event data for offline analysis.

Collect a trace:

```powershell
dotnet-trace collect --process-id <PID> --output app.nettrace
```

Collect CPU samples:

```powershell
dotnet-trace collect --process-id <PID> --profile cpu-sampling --output cpu.nettrace
```

Convert trace format when needed:

```powershell
dotnet-trace convert app.nettrace --format speedscope
```

Open `.nettrace` files with:

- Visual Studio
- PerfView
- Speedscope, after converting to speedscope format

## Production Diagnostics

`dotnet-monitor` is designed for production diagnostics. It exposes HTTP
endpoints and can collect artifacts on demand or by rule.

It can collect:

- Dumps
- GC dumps
- Traces
- Logs
- Metrics

Common use cases:

- Kubernetes diagnostics sidecar
- Production dump collection without shelling into containers
- Triggering dumps when CPU, memory, or request failures cross thresholds
- Centralizing diagnostics collection for multiple services

## Symbols And Cross-Machine Dump Analysis

When analyzing a dump on a different machine, you may need runtime files,
symbols, DAC, DBI, and native modules.

Use `dotnet-symbol`:

```powershell
dotnet-symbol app-full.dmp
```

Set symbol paths for Windows debugging tools or Visual Studio when native stack
frames are important.

## Debugging In Containers

Typical container workflow:

```powershell
# Find container
docker ps

# Enter container
docker exec -it <container-id> /bin/bash

# List dotnet processes
dotnet-dump ps

# Collect dump inside container
dotnet-dump collect --process-id <PID> --type Full --output /tmp/app-full.dmp

# Copy dump to host
docker cp <container-id>:/tmp/app-full.dmp ./app-full.dmp

# Analyze on host
dotnet-dump analyze ./app-full.dmp
```

Notes:

- The diagnostics tool architecture must match the target process architecture.
- Linux containers may need permissions for diagnostics IPC.
- Full dumps can contain secrets, tokens, request data, connection strings, and
  personally identifiable information.

## Common Troubleshooting Playbooks

### High Memory

```powershell
dotnet-counters monitor --process-id <PID> System.Runtime
dotnet-gcdump collect --process-id <PID> --output high-memory.gcdump
dotnet-gcdump report high-memory.gcdump
dotnet-dump collect --process-id <PID> --type Full --output high-memory.dmp
dotnet-dump analyze high-memory.dmp
```

Inside `dotnet-dump analyze`:

```text
dumpheap -stat
eeheap -gc
finalizequeue
```

### Memory Leak

1. Confirm growth with `dotnet-counters`.
2. Capture two or more `gcdump` files over time.
3. Compare top object types and sizes.
4. Capture a full dump if object roots are needed.
5. Use `gcroot` to find retention paths.

Useful commands:

```powershell
dotnet-counters monitor --process-id <PID> System.Runtime
dotnet-gcdump collect --process-id <PID> --output leak-1.gcdump
dotnet-gcdump collect --process-id <PID> --output leak-2.gcdump
dotnet-dump collect --process-id <PID> --type Full --output leak-full.dmp
```

### High CPU

```powershell
dotnet-counters monitor --process-id <PID> System.Runtime
dotnet-stack report --process-id <PID>
dotnet-trace collect --process-id <PID> --profile cpu-sampling --output high-cpu.nettrace
```

Then open the trace with Visual Studio, PerfView, or Speedscope.

### Deadlock Or ThreadPool Starvation

```powershell
dotnet-counters monitor --process-id <PID> System.Runtime
dotnet-stack report --process-id <PID>
dotnet-dump collect --process-id <PID> --type Full --output hang.dmp
dotnet-dump analyze hang.dmp
```

Inside `dotnet-dump analyze`:

```text
clrthreads
clrstack -all
syncblk
dumpthreadpool
dumpasync
```

### Crash Or Unhandled Exception

```powershell
dotnet-dump collect --process-id <PID> --type Full --output crash.dmp
dotnet-dump analyze crash.dmp
```

Inside `dotnet-dump analyze`:

```text
pe
clrstack
clrthreads
dumpheap -stat
```

## Tool Selection Cheat Sheet

| Problem | Start With | Then Use |
|---|---|---|
| Need process ID | `dotnet-dump ps` | `dotnet-stack ps`, `dotnet-gcdump ps` |
| Is memory growing? | `dotnet-counters` | `dotnet-gcdump`, `dotnet-dump` |
| Need heap histogram | `dotnet-gcdump report` | `dotnet-dump analyze` + `dumpheap -stat` |
| Need object retention root | `dotnet-dump analyze` | `gcroot` |
| Need live thread stacks | `dotnet-stack report` | `dotnet-dump analyze` |
| Need CPU profile | `dotnet-trace` | Visual Studio, PerfView, Speedscope |
| Need production artifact collection | `dotnet-monitor` | Dumps, traces, logs, metrics |
| Need cross-machine dump support | `dotnet-symbol` | Visual Studio, WinDbg, `dotnet-dump` |

## Safety Notes

- Full dumps can contain secrets and user data. Treat them as sensitive.
- Dump collection can pause or slow the target process.
- Large processes can produce very large dump files.
- Prefer `dotnet-gcdump` first when only managed heap statistics are needed.
- Prefer `dotnet-dump collect --type Full` when you need roots, threads, locks,
  exceptions, and deeper state.
- In production, consider `dotnet-monitor` to standardize secure collection.

## References

- [.NET diagnostic tools overview](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/tools-overview)
- [Diagnostics in .NET](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/)
- [`dotnet-counters`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-counters)
- [`dotnet-dump`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-dump)
- [`dotnet-gcdump`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-gcdump)
- [`dotnet-stack`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-stack)
- [`dotnet-trace`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-trace)
- [`dotnet-monitor`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-monitor)
- [`dotnet-symbol`](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/dotnet-symbol)
