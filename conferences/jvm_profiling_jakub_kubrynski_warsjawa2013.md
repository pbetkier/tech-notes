# JVM profiling - Jakub Kubryński, Michał Warecki

## Introduction

Not too many materials online on how to do it, basicaly you need to gain the experience by yourself.

Typical Java system stack:
 * Application
 * Frameworks
 * JVM
 * Libraries
 * System calls
 * Kernel
 * Hardware

Performance problems may occur at each level. You may profile either going top-down (application first) or bottom-up (hardware first). Top-down approach is more natural for developers and is what you will do most of the time, but some problems are visible only at the OS level.

Profiling means spotting the problem. Once it's done, you can start tuning. You can tune either for low latency or for high throughput.

## Java memory model (HotSpot)

Based on an observation, that most objects die young. Old objects probably will live even more. This is not always true, e.g. cache.

### Memory model

Heap:
 * Young generation:
   * Eden: where new objects are created.
   * Survivor from: when Eden gets full, not collected objects go here.
   * Survivor to: when survivor from gets full, not collected objects go here.
   * (After survivor to gets full, not collected objects go back to survivor from).

 * Old generation:
   * Tenured: objects whose age exceeded max_tenuring_threshold go here. Age gets bumped after each switch between spaces (GC launch) in young generation.

Non-heap:
 * Method area:
   * Perm gen: small space for classes, methods, internalized strings. NOT old objects.
   * (If you intern strings a lot, make sure to increase perm gen space!).
 * Native area:
   * Code cache.
   * Call stack.

### Object layout

Each Object introduces some overhead because of the header (8 bytes, 12 for arrays):
 * Hash code plus some flags like lock state, age (4 bytes).
 * Reference to Class (4 bytes).
 * (length for arrays, 4 bytes).

The overhead is actually bigger, because of object's memory alignment. Each object needs to be aligned to the nearest chunk of 1-word size. That's why each new Boolean\* takes 16 bytes on 32bit platform. On 64bit it will be more unless you use compression flag. Small booleans on the other hand take only 1 byte if used inside an array.

\* for Boolean you should actually use singletons Boolean.TRUE and Boolean.FALSE instead of creating new objects. The reference takes only 1-word size.

### GC roots

An object, which by definition cannot be garbage collected, e.g. thread, used monitor, class, JNIs. In practise it will be e.g. main BeanFactory of Spring.

Garbage collector starts from GC roots to check which objects are still reachable.

### Object size

Three parameters define object size:
 * Shallow: header, references, primitives - the size of the object itself.
 * Deep: sum of shallow sizes for objects in a tree starting from this object.
 * Retained: how much memory could be freed if this object got collected.

## GC tuning

### GC algorithms

* Mark-sweep: selected objects get removed. Introduces fragmentation: after that there are free slots scattered across the memory, so problems with allocating big objects.
* Mark-compact: defragments the memory after objects removal into one, single, contiguous area.
* Copying GC.
* G1: based on heuristics.

Mark-sweep and mark-compact are used in old generation, copying GC in young generation.

### Multithreaded GC

* Serial: non multithreaded.
* Parallel: pauses exist, but GC is faster.
* Concurrent: solution being 100% concurrent doesn't exist.
* Incremental: multiple small pauses.

Even Azul GC, which is advertised as no-pause GC, may introduce small pauses. It works by detecting Safe Points: moments when most threads don't access the heap, e.g. are inside native code. GC is run during those Safe Points.

### Analyzing GC

Few tips:
* Conditions as similar to production environment as possible, e.g. conducted on a test server. Not on a laptop: phantom bottlenecks may occur (bottlenecks not existing in production), different heuristics used.
* Data volume and nature as in production: beware of phantom bottlenecks, differences in cache utilisation.
* No micro-benchmarks, there will be differences in production.

### Tuning

For throughput:
* Increase heap: GC will occur much less often.
* If you see that heap gets full on some node, you may detach it from the load balancer, call System.gc() and attach it back.

For low latency (minimal pauses):
* Try parallel GC.
* If it doesn't help, try concurrent GC.
* G1 GC is still a young solution, but it may perform better.

## Thread statuses

A thread may be in one of the following states:
 * NEW
 * RUNNABLE: JVM doesn't differentiate whether the thread is actually running on the processor (if it got scheduled by the OS).
 * BLOCKED: hanging on a monitor.
 * WAITING: Object.wait(), doesn't wait actively on a monitor, waits until some other object wakes him up.
 * TIMED_WAITING: the same as above, plus will wake up after given time.
 * TERMINATED

## Unix tools

* htop: configurable, search, filters, process tree, setting priorities, kill, process tagging, user selection.
* iostat: load on I/O, CPU, I/O waits (if it's a lot, it means that the disks are your bottlenecks), reads and writes per second, in MB per second.
* free: how much memory available, how much is used.
* dstat.
* perf: system profiler, e.g. instructions per cycle, context switches, CPU migrations, processor cache misses.

## JDK tools

* jps: ps for java processes.
* jinfo: flags, properties, sperators.
* jmap: 
   * heap: analysis, check configuration, utilistaion.
   * histo: class instances histogram.
* jstat: each n seconds dumps info about:
   * class: how many classes are loaded.
   * gc: memory utilisation.
   * printcompilation: methods compiled to native by JIT.
* jstack: thread states dump, analysis.
* java -XX:+PrintFlagsFinal: shows all possible flags and their current settings.
* java-object-layout (from github): info about object layout for a given class. You may set this as an external tool in IntelliJ.
* jvisualvm: memory and CPU monitoring, graphs, perform gc, threads inspection, gc tracking (visual gc). Don't use sampler feature, and profiler especially, on production, you will take it down. There are better, separate tools for profiling.

## JIT

You can steer JIT using -XX:+CompilationThreshold flag. By default method compilation takes place after 10000 calls. Micro-benchmarks must take into account, that JIT may not have been used during benchmarking, but take place in production. You can check which methods got inlined, which not and why using -XX:+PrintInlining flag.

## JProfiler

Instrumentation or sampling. Instrumentation affects code performance, but enables more features than sampling.

You should use it only in dev environment, it would be too heavy to use it in production environment (or at least use it at night). In production you should use some kind of APM like NewRelic. You can dump your production database, import it on dev and analyse your system on real data.

Some of the features:
* Databases: database connections analysis, see open connections, JDBC, JPA/Hibernate, mongoDB, Cassandra.
* CPU views: method call times, you can analyze % of call time in the call stack.
* Live memory: you can mark current state, launch some action and see the difference after launch. 

Simplest way to locate memory leak: view biggest / most occuring objects, show selection in heap walker, use selected instances, you see cumulated incoming references in heap walker - who references those interesting objects. You can view the object graph, show paths from GC root.
