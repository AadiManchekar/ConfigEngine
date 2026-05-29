---
name: java-tuning
description: >
  Guide for tuning JVM garbage collection and profiling Java applications with
  Java Flight Recorder (JFR). Use this skill whenever the user asks about JVM
  performance, GC pauses, heap sizing, memory pressure, latency spikes, or
  profiling. Trigger on any mention of: GC, G1GC, ZGC, Shenandoah, -Xmx, -Xms,
  heap, metaspace, JFR, jcmd, GC logs, stop-the-world, full GC, OOM, or
  OutOfMemoryError — even if they don't say "tuning" explicitly.
---

# Java GC & JFR Tuning

## How to use this skill

1. **Start with GC logging**: you can't tune what you can't see.
2. **Pick the right collector**: choose based on your latency/throughput tradeoff.
3. **Set JVM flags**: explicit heap sizing, collector flags, GC log config.
4. **Profile with JFR**: identify allocation hotspots, lock contention, thread stalls.
5. **Iterate**: change one thing at a time and re-measure.

---

## Step 1: Enable GC Logging

Never run production without GC logs. They are low-overhead (~1% CPU) and essential for diagnosis.

```bash
# Java 17+ unified logging — use this
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=10M

# More detail (when actively investigating a problem)
-Xlog:gc*,gc+phases=debug:file=/var/log/app/gc-detailed.log:time,uptime,level,tags:filecount=5,filesize=20M
```

**What to look for in GC logs:**
- Pause times consistently above your SLA → tune GC or reduce allocation rate
- Frequent full GCs → heap too small, or memory leak
- Growing old-gen over time → likely leak; take heap dump
- GC triggered very frequently → objects being promoted too fast; check batch size and cache TTLs

---

## Step 2: Heap Sizing

Set `-Xms` equal to `-Xmx` to avoid heap resizing pauses at startup.

```bash
java -Xms2g -Xmx2g \
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -jar application.jar
```

| Flag | Purpose | Guidance |
|---|---|---|
| `-Xms` | Initial heap | Set equal to `-Xmx` |
| `-Xmx` | Max heap | 50–75% of available RAM |
| `-XX:MetaspaceSize` | Initial metaspace | 256m for typical apps |
| `-XX:MaxMetaspaceSize` | Max metaspace | 512m–1g; cap it to avoid unbounded growth |

**Why set Xms = Xmx?** The JVM will grow the heap lazily if they differ, causing resize pauses and inconsistent memory footprint. Pinning them eliminates that variable.

**Why cap metaspace?** Without `-XX:MaxMetaspaceSize`, a class loader leak will grow metaspace unboundedly until the OS kills the process. The cap converts a silent resource leak into an explicit OOM you can detect and alert on.

---

## Step 3: Choose a Garbage Collector

| Collector | Best for | Trade-off |
|---|---|---|
| **G1GC** | Most production apps | Balanced throughput + pause control |
| **ZGC** (Generational, Java 21+) | Low-latency APIs, large heaps | Higher CPU overhead |
| **Shenandoah** | Low-latency, OpenJDK-only | Less widely supported |
| **ParallelGC** | Batch jobs, throughput priority | Long stop-the-world pauses |

**Default recommendation: G1GC.** It is the JVM default since Java 9 and works well for most Spring Boot services. Switch to ZGC only if p99 latency is a hard requirement and you are on Java 21+.

### G1GC

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          # Target pause; G1 tries to meet this, not a hard cap
-XX:G1HeapRegionSize=16m          # Increase for large heaps (>8g); default is auto-calculated
-XX:InitiatingHeapOccupancyPercent=45  # Start concurrent marking earlier if GC is too reactive
```

**When to adjust `InitiatingHeapOccupancyPercent`:** If you see concurrent GC falling behind (evacuation failures, full GCs), lower this to 35–40 so G1 starts marking earlier before old-gen fills up.

### ZGC (Java 21+ — Generational)

```bash
-XX:+UseZGC
-XX:+ZGenerational    # Required for generational ZGC in Java 21+; significant throughput improvement
-XX:ZCollectionInterval=5         # Force a GC cycle every 5 seconds if idle; prevents stale regions
```

ZGC performs almost all work concurrently — pause times are typically <1ms regardless of heap size. The trade-off is 10–20% higher CPU usage versus G1GC.

### Shenandoah

```bash
-XX:+UseShenandoahGC
-XX:ShenandoahGCHeuristics=adaptive   # Default; self-tunes based on allocation rate
```

Similar pause characteristics to ZGC. Prefer ZGC if you are on Oracle JDK; Shenandoah is more common in OpenJDK/Red Hat distributions.

---

## Step 4: Profile with Java Flight Recorder (JFR)

JFR is built into the JDK (Java 11+, no agent needed). It captures CPU samples, allocations, locks, I/O, GC events, and thread activity with very low overhead (~1–2%).

### Start at JVM launch (recommended for production)

```bash
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=60s,filename=/var/log/app/recording.jfr,settings=profile \
     -jar application.jar
```

`settings=profile` captures more detail than the default. Use `settings=default` if overhead is a concern.

### Attach to a running process (no restart needed)

```bash
# Find the PID
jps -l

# Start a recording
jcmd <pid> JFR.start duration=60s filename=/tmp/recording.jfr settings=profile

# Check recording status
jcmd <pid> JFR.check

# Stop and dump early if needed
jcmd <pid> JFR.stop name=1 filename=/tmp/recording.jfr
```

### What to look for in JFR

Open the `.jfr` file in **JDK Mission Control (JMC)**: free tool from https://jdk.java.net/jmc/

| Tab | What to look for |
|---|---|
| **Allocation** | Top allocating methods → reduce object churn in hot paths |
| **GC** | Pause time distribution, promotion rates, full GC frequency |
| **Method Profiling** | CPU hotspots — where is the app actually spending time? |
| **Lock Instances** | Contended monitors → candidates for `ConcurrentHashMap`, finer locking |
| **Thread Stalls** | I/O waits, socket timeouts blocking request threads |
| **Exceptions** | High exception rates (exceptions are expensive; common in misconfigured retry loops) |

### Continuous JFR in production (Java 14+)

For always-on low-overhead recording with automatic dump on OOM or crash:

```bash
-XX:StartFlightRecording=disk=true,dumponexit=true,filename=/var/log/app/recording.jfr,maxage=1h,maxsize=500m
```

This keeps a rolling 1-hour window on disk and dumps it on process exit — invaluable for post-mortem analysis.

---

## Putting It Together: Full Production JVM Flags

```bash
# G1GC: general-purpose Spring Boot service
java \
  -Xms2g -Xmx2g \
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:G1HeapRegionSize=16m \
  -XX:InitiatingHeapOccupancyPercent=45 \
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=10M \
  -XX:StartFlightRecording=disk=true,dumponexit=true,filename=/var/log/app/recording.jfr,maxage=1h,maxsize=500m \
  -jar application.jar
```

```bash
# ZGC: low-latency API, Java 21+
java \
  -Xms4g -Xmx4g \
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseZGC \
  -XX:+ZGenerational \
  -XX:ZCollectionInterval=5 \
  -Xlog:gc*:file=/var/log/app/gc.log:time,uptime:filecount=5,filesize=10M \
  -XX:StartFlightRecording=disk=true,dumponexit=true,filename=/var/log/app/recording.jfr,maxage=1h,maxsize=500m \
  -jar application.jar
```

---

## Diagnostic Cheatsheet

| Symptom | Likely cause | First action |
|---|---|---|
| High p99 latency spikes | Stop-the-world GC pauses | Check GC logs for pause times; consider ZGC |
| Gradual memory growth | Heap or metaspace leak | Take heap dump: `jcmd <pid> VM.heap_dump /tmp/heap.hprof` |
| `OutOfMemoryError: Java heap space` | Heap too small or leak | Increase `-Xmx`; take heap dump; analyze with JFR allocation tab |
| `OutOfMemoryError: Metaspace` | Class loader leak | Cap metaspace and monitor; JFR class loading tab |
| Frequent full GCs | Old-gen filling up | Lower `InitiatingHeapOccupancyPercent`; check for large object allocations |
| High CPU, low throughput | GC overhead | JFR method profiling to find allocation hotspots |
| Thread pool exhaustion | Blocking I/O in request threads | JFR thread stalls tab; move to `@Async` or virtual threads |
