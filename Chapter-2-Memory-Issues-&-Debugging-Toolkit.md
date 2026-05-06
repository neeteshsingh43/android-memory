# Chapter 2: Memory Issues & Debugging Toolkit
#### *(From Symptoms --> Subsystem Truth --> Root Cause Isolation)*

Identifying that a memory issue exists is only 10% of the battle. The true challenge lies in architectural isolation when the blast radius spans across the Java Runtime, Native Heaps, Shared Hardware Buffers, and the Linux Kernel. In a high-performing system, memory bugs are rarely isolated events—they are cross-layer failures where an issue in one subsystem propagates pressure until another layer breaks.

---

## 1. The Performance Engineer's Mental Model

You are never merely debugging "high memory usage." You are auditing a dynamic lifecycle across four distinct dimensions:

[Allocation Profile] --> [Lifetime & Ownership] --> [Reclaimability vs. Persistence] --> [System-Wide Pressure]

* Allocation Profile: Which specific allocator handles the request, what is the allocation frequency, and what is the structural size?
* Lifetime & Ownership: Which context or framework component maintains holding references or open file descriptors long after the business logic has concluded?
* Reclaimability vs. Persistence: Can the kernel page out, unpin, compress, or discard this memory under pressure, or is it locked/pinned permanently into physical RAM?
* System-Wide Pressure: Who is suffering because of this allocation? Is the app killing itself via an internal runtime exception, or is it choking the entire operating system until a hardware watchdog intervenes?

### The Symptom-to-Layer Matrix
Before launching a single tool, classify the system behavior to target the correct layer:

| Observed Symptom | Primary Suspect Subsystem | Root Failure Mode |
| :--- | :--- | :--- |
| OutOfMemoryError Crash | Java Heap / ART | App-level reference leak or asset bloating |
| Gradual Device Sluggishness / Frame Drop | Native Heap / ZRAM | Native leakage (malloc leak) or ZRAM CPU thrashing |
| App Terminated Unexpectedly While Backgrounded | LMKD / System Memory | System-wide memory exhaustion or low oom_score_adj |
| "Camera Failed", Black Preview, UI Reset | DMA-BUF / CMA | Unreleased graphic surfaces or physically fragmented RAM |
| Sudden Screen Blackout + Device Vibration | Linux Kernel Allocators | Kernel Panic (Slab exhaustion or critical page allocation fault) |

---

## 2. Layer 1: The App Runtime & Java/Kotlin Analysis

When an application hits its memory ceiling inside the Android Runtime (ART), simple object logging yields incomplete answers. True isolation requires evaluating a Java Heap Dump (.hprof) via object weighting and dominator graphs.

### Memory Weight Metrics
* Shallow Heap: The exact number of bytes allocated to store the target object instance itself. This includes its primitive fields (integers, booleans, floating points) and the explicit reference pointers to other objects, but excludes the space of the objects it points to.
* Retained Heap: The total quantity of memory that would be instantly reclaimed if this target object was garbage collected. It comprises the object's shallow heap plus the size of its unique down-tree references that are only reachable through this specific object (the dominator tree).

### Diagnostic Archetypes
1. Dominator Tree Verification: Used to systematically surface a single root parent instance (e.g., a massive background Service, a complex Activity layout, or an image caching wrapper) that is anchoring a massive retained heap size.
2. Incoming vs. Outgoing Reference Paths: Evaluating incoming reference chains reveals which framework component or static variable is actively keeping the leaked object alive, tracing an unbrokerable path back to a Garbage Collection (GC) Root.

---

## 3. Layer 2: Native Heap Leak Interception (User Space)

When C/C++ native allocations generated via malloc(), calloc(), or new are not paired with matching free() or delete` invocations, the bytes escape the management of ART and leak directly into the process's Resident Set Size (RSS).

### Approach A: Bionic Malloc Debug & Real-Time Signals
For modern Android systems running on standard platforms, you can instruct Bionic libc to dynamically load an instrumentation shim over the primary allocator (Scudo or jemalloc) using system properties.

#### 1. Configuration Injection
adb shell setprop libc.debug.malloc 1
adb shell setprop libc.debug.malloc.program app_process
adb shell setprop libc.debug.malloc.options "'backtrace=4 guard'"
adb shell setprop libc.debug.malloc.min_size 1024
adb shell stop && adb shell start

#### 2. Point-in-Time Heap Snapshotting via Signal 47
Rather than forcing a process to exit to gather data, you can compel Bionic's malloc debug layer to write out its live internal allocation tables on demand by executing Real-Time signal SIGRTMAX - 17 (which correlates to Signal 47 on modern Android devices):

adb shell kill -47 <TARGET_PID>

Diagnostic Yield: Bionic dumps a raw allocation matrix file directly onto the device filesystem at /data/local/tmp/backtrace_heap.<PID>.txt.

#### 3. Symbols Resolution via llvm-addr2line
The extracted snapshot contains raw, un-symbolicated hex instruction addresses (pc). To transform these back into human-readable source code pointers, run the addresses through the Android NDK binary analysis engine against your unstripped shared object (.so) libraries containing full debug symbols:

$NDK_PATH/toolchains/llvm/prebuilt/linux-x86_64/bin/llvm-addr2line -e ./obj/local/armeabi-v7a/libnative_engine.so -f 0x0004f1a2

### Approach B: Valgrind Massif for Low-Execution (LE) Environments
On highly customized, stripped Android variants or Low-Execution (LE) embedded Linux targets missing the standard Android application framework, heavy Bionic property hooks may not be supported. For these setups, load the target binary using Valgrind Massif to monitor heap structures via instruction-level instrumentation.

valgrind --tool=massif --massif-out-file=/data/local/tmp/massif.out ./my_native_service

Once the system completes execution or displays the target leak state, move the output file to a development host workstation and analyze it using ms_print to generate a detailed ASCII time-graph illustrating allocation peaks alongside the offending call graphs.

---

## 4. Layer 3: Static Binary Architecture (.bss & .data Profiling)

Memory overhead isn’t confined to runtime allocations. Massive runtime penalties can be baked directly into native binaries at compilation time via bloated data structures, oversized global spaces, or improper alignment configurations.

### 1. Auditing the .bss and .data Footprint via llvm-nm
The .bss section holds statically declared variables that remain uninitialized at compile time (zero-initialized by the kernel at boot), whereas .data contains initialized global variables. If a library's baseline memory footprint is unexpectedly massive the instant it is mapped into a process, audit its internal structures:

llvm-nm -S --size-sort --numeric-sort -radix=d ./libnative_core.o

The Signature Focus: Look for symbols tagged with uppercase B (global/static BSS symbols) or D (initialized data symbols). If a global array such as static uint8_t raw_frame_buffer[4194304]; was declared carelessly, nm will instantly flags it as a multi-megabyte memory consumer before your code even performs its first dynamic allocation.

### 2. Struct Layout Padding Analysis via Clang Records
To pinpoint structural optimization opportunities inside complex C++ classes or nested structures, append record layout reporting directly into your build configuration toolchain (Android.mk or CMakeLists.txt):

-Xclang -fdump-record-layouts

Compiler Output Visualization:
*** Type: struct VideoFrameMetadata
   Size: 32 bytes
   Alignment: 8 bytes
   Fields:
     0 | uint32_t width
     4 |   [padding: 4 bytes due to 8-byte alignment restriction]
     8 | uint64_t timestamp

This precise compiler read-back enables you to reorder structural members (placing largest data primitives first) to eliminate internal compiler padding spaces, reducing memory footprints across highly instanced system structures.

---

## 5. Layer 4: System-Wide Pressure & Kernel Interception

When process memory looks flat but the overall system is degrading, you must move beyond application perspectives and evaluate the kernel's memory tracking layer.

### The PSS Fallacy: Why dumpsys meminfo Can Be Incomplete
Standard diagnostic checks typically evaluate PSS (Proportional Set Size) via dumpsys meminfo. PSS averages shared library pages evenly across every process linked to them. While excellent for accounting, PSS cannot separate reclaimable memory from anonymous allocations, meaning it cannot tell you if a process is truly dragging down the system.

To see the raw truth, you must bypass the user-space tools and read the kernel process mappings directly:

adb shell cat /proc/<PID>/smaps

* Private_Dirty: The absolute physical RAM unique to this target process that has been structurally modified. This memory cannot be discarded or paged out; it must remain in physical RAM or be compressed into ZRAM.
* Anonymous (Anon): Memory mappings that are completely unbacked by physical files on disk (such as dynamic heap pools). A steadily climbing Anonymous value is a clear indicator of an active memory leak that is increasing system-wide pressure.

### A. Kernel Space Leaks: Tracking SLAB & SLUB Allocations
The Linux Kernel handles its own structural memory allocations using cache pools managed by the SLAB or SLUB allocator. If kernel extensions or low-level device drivers leak allocations, the user space tools will register no anomalies, but overall free system RAM will drop.

#### Step 1: Macro Verification via meminfo
adb shell cat /proc/meminfo | grep -E "Slab|SReclaimable|SUnreclaim"

* SUnreclaim: Memory locked inside the kernel layer that cannot be paged out or freed under pressure. If SUnreclaim rises continuously while the system runs, you have confirmed an active kernel-space memory leak.

#### Step 2: Micro Object Isolation via slabinfo
adb shell cat /proc/slabinfo

Examine the active object versus total object columns over time:
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
ion_buffer_cache       14250       15000        512           32            4
dentry                 45210       50000        192           21            1

If a specific component like ion_buffer_cache increases continuously without ever dropping, that particular driver subsystem is failing to invoke its matching kmem_cache_free() clean-up sequences.

#### Step 3: Advanced Allocation Backtracing via slub_debug
To catch the precise kernel execution path causing the allocation failure, append the tracking parameters to the kernel command line during system boot:

fastboot boot --cmdline "slub_debug=FZPU" path/to/boot.img

* F (Sanity Checks): Runs real-time validation checks across the allocator’s internal freelists.
* Z (Red-zoning): Appends protective guard spaces on both sides of a slab object to catch memory buffer overflows.
* P (Poisoning): Overwrites freed allocations with specific byte markers (0x6b) to instantly flag Use-After-Free (UAF) actions.
* U (User Tracking): Commands the allocator to log the exact allocation and deallocation call stack histories for every single block.

Once booted with slub_debug=FZPU, look for memory error assertions or allocation traces directly within the kernel log ring buffer using adb shell dmesg.

---

### B. System Stall Isolation via PSI & LMKD
Android relies on two integrated systems to protect overall system integrity when memory drops: PSI (Pressure Stall Information) and the LMKD (Low Memory Killer Daemon).

#### 1. Evaluating PSI Data
Instead of monitoring arbitrary "free memory thresholds," the modern Linux kernel tracks resource starvation directly by measuring time lost to resource shortages:

adb shell cat /proc/pressure/memory

Output Format:
some avg10=12.50 avg60=8.20 avg300=4.10 total=4821045
full avg10=4.10 avg60=2.05 avg300=0.50 total=1204912

* some: Tracks the percentage of time during which at least one thread was stalled waiting on memory operations (e.g., waiting for page reclamation).
* full: Tracks the percentage of time during which all non-idle threads in the system were completely stalled simultaneously. High spikes in the full columns tell you the entire device is locked up waiting on memory IO, meaning an LMKD intervention is imminent.

#### 2. Auditing LMKD Actions
When the system decides to terminate a process, it doesn't choose at random. It references the process's runtime priority value (oom_score_adj) alongside real-time stall statistics. You can trace these exact execution logs via logcat:

adb logcat | grep -i lmkd

Expected Diagnostic Output:
LowMemoryKiller: Kill 'com.android.camera' (1234), adj 906, to free 120MB due to severe memory pressure (PSI)

Critical Analysis Rule: If your application crashes unexpectedly with an exit code of 137 (Killed via SIGKILL), check these logs immediately. If LMKD killed your app while its oom_score_adj was high, your app may not have leaked any memory at all. It might simply have been targeted by the operating system to salvage memory consumed by a silent framework leak elsewhere.

---

## 6. Layer 5: Kernel Memory Architecture & Buddy Allocator Failures

When a hardware allocator or system driver requests a block of RAM from the kernel, the allocation can fail even if meminfo reports that gigabytes of total system memory are completely free. To diagnose these catastrophic hardware timeouts, you must track memory across its internal Zones and Page Orders.

### A. Understanding Kernel Memory Zones
The Linux kernel partitions the device's physical memory architecture into discrete Zones based on addressing limitations and architectural boundaries:

* ZONE_DMA / ZONE_DMA32: Allocated for hardware components restricted to 32-bit addressing limitations that cannot bind directly to high physical memory addresses.
* ZONE_NORMAL: The primary operational pool for standard 64-bit kernels. It backs internal kernel code mappings, page tables, user-space process segments, and anonymous application heaps.
* ZONE_MOVABLE: A specialized, high-efficiency zone populated with pages that are guaranteed to be movable (e.g., file-backed page caches). The kernel utilizes this zone to isolate memory and prevent system-wide fragmentation; pages here can be cleanly shifted out of the way to clear vast, physically contiguous expanses for critical drivers.

### B. Parsing Page Orders via /proc/buddyinfo
The kernel structures every memory zone using the Buddy Allocator. Free physical memory is grouped into blocks of sequential pages defined as Orders. The dimensions of a target block are derived via:

Block Size = 2^Order * 4 KB

When a low-level driver executes a memory command, it specifies an explicit order block. For example, a high-resolution multimedia pipeline might require a steady stream of Order-4 blocks (2^4 * 4KB = 64KB) or Order-9 blocks (2^9 * 4KB = 2MB) of unbroken physical RAM.

To audit these allocator pools in real-time, inspect the buddy information nodes:

adb shell cat /proc/buddyinfo

Output Mapping Matrix:
Node 0, zone      DMA     11      5      3      1      1      1      0      0      0      0      0
Node 0, zone   Normal   4201   1205    304     15      2      0      0      0      0      0      0
Node 0, zone  Movable  15420   8901   4112   2301   1102    840    512    231    104     45     12

Each column directly matches an ascending Page Order, scanning left-to-right from Order-0 up to Order-10:

| Column Sequence | Allocator Order | Physical Block Resolution |
| :--- | :--- | :--- |
| 1 (Far Left) | Order-0 | 4 KB (Single Page Allocation) |
| 2 | Order-1 | 8 KB |
| 3 | Order-2 | 16 KB |
| 4 | Order-3 | 32 KB |
| 5 | Order-4 | 64 KB |
| 6 | Order-5 | 128 KB |
| 7 | Order-6 | 256 KB |
| 8 | Order-7 | 512 KB |
| 9 | Order-8 | 1 MB |
| 10 | Order-9 | 2 MB |
| 11 (Far Right) | Order-10 | 4 MB |

The Fragmentation Signature: Review zone Normal. While the allocator displays over 4,201 free pages at Order-0 (~16.8 MB of total free workspace), the higher-tier indices (Order-5 through Order-10) sit completely at 0. If a system thread requests a single contiguous 128 KB block (Order-5), this allocation will instantly crash, exposing severe physical fragmentation.

### C. Dissecting Kernel Allocation Failure Warning Logs
When high-order page requests cannot be completed, the kernel drops an allocation trace block directly into the system ring buffer. Query this output using dmesg:

adb shell dmesg | grep -A 20 "allocation failure"

Anatomy of a Critical Allocator Log Event:
[ 1402.123456] kswapd0: page allocation failure: order:4, mode:0xcc0(GFP_KERNEL), nodemask=(null)
[ 1402.123460] CPU: 3 PID: 452 Comm: kswapd0 Tainted: G        W         5.15.0-android
[ 1402.123465] Call Trace:
[ 1402.123470]  [<ffffffc00015a2b4>] dump_stack+0xbc/0xf0
[ 1402.123478]  [<ffffffc00028b312>] warn_alloc+0x104/0x18c
[ 1402.123485]  [<ffffffc00028c4ea>] __alloc_pages_nodemask+0xdac/0xe10
[ 1402.123491] Mem-Info:
[ 1402.123495] active_anon:124052 inactive_anon:45102 active_file:84102 inactive_file:95124
[ 1402.123501] Node 0 Zones free:18212kB DMA:412kB Normal:17800kB Movable:0kB
[ 1402.123508] Node 0 DMA: 3*4kB (U) 2*8kB (U) 1*16kB (I) 1*32kB (I) 1*64kB (M) 1*128kB (M) 0*256kB 0*512kB 0*1024kB 0*2048kB 0*4096kB = 412kB
[ 1402.123520] Node 0 Normal: 4450*4kB (U) 0*8kB 0*16kB 0*32kB 0*64kB 0*128kB 0*256kB 0*512kB 0*1024kB 0*2048kB 0*4096kB = 17800kB

#### Step-by-Step Diagnostic Deconstruction:
1. Identify the Order Boundary: Check the line page allocation failure: order:4. This tells you a system component required an Order-4 memory segment (2^4 = 16 contiguous pages = 64 KB). The mode:0xcc0(GFP_KERNEL) flag confirms this was a critical kernel-space execution request.
2. Review Zone Capacities: Line Node 0 Zones free:18212kB DMA:412kB Normal:17800kB Movable:0kB indicates total system free space looks completely fine (~18 MB unallocated). The targeted Normal zone explicitly holds 17,800 KB of free space.
3. Analyze Block Distribution: Examine the structure string: Node 0 Normal: 4450*4kB (U) 0*8kB 0*16kB 0*32kB 0*64kB...
    * 4450*4kB: The allocator possesses 4,450 loose, unlinked Order-0 (4KB) blocks (4450 * 4KB = 17,800KB).
    * 0*8kB through 0*64kB: Orders 1, 2, 3, and 4 are completely empty (0 blocks available).

The Verdict: The requesting component requested exactly one unified 64KB block inside zone Normal. The kernel verified that while it holds thousands of separate 4KB single-page locations, it contains zero contiguous spaces at or above the 64KB line. The allocation fails, spawning a driver fault, system timeout, or a Kernel Panic if the code execution thread is non-reentrant.

---

## 7. Layer 6: Hardware Boundaries & Shared Memory (DMA-BUF & CMA)

When handling intensive multimedia pipelines (such as camera frame processing or 3D rendering), data must pass between the CPU, GPU, and Camera ISP without performance-killing memory copies. This zero-copy pipeline is managed through DMA-BUF files.

### A. Correlating Memory Allocation Context via buf_info
Because DMA buffers bypass typical user-space accounting tools, standard heap profilers will report normal memory metrics even while the device is running out of physical RAM. To verify these allocations, query the kernel's DMA debug interface:

adb shell cat /sys/kernel/debug/dma_buf/buf_info

Review the active table mappings:
Dma-buf Objects:
size            flags           mode            count           exp_name
16777216        00000002        00000003        2               ion
  Attached Devices:
    kgsl-3d0
  Total 1 objects, 16777216 bytes

* exp_name (Exporter Name): Pinpoints the specific low-level kernel driver responsible for creating the memory allocation (e.g., ion, system_heap, or a vendor-specific multimedia block).
* count: The active reference count. If count remains greater than 0 long after an asset or layout has been dismissed, a system component is keeping the buffer pinned in memory.

### B. Locating Open File Descriptors
To find out which user-space component is holding a shared buffer open, check the active file descriptor tables of suspected system daemons (such as surfaceflinger or the camera provider service):

adb shell ls -l /proc/<PID>/fd/ | grep -i "dma_buf"

Once you find an active descriptor match, inspect its internal tracking metadata to see its layout and size metrics:

adb shell cat /proc/<PID>/fdinfo/<FD_NUMBER>

If an application or system service neglects to invoke close() on these file descriptors, the kernel cannot release the backing memory, causing severe CMA (Contiguous Memory Allocator) fragmentation.

---

## 8. Layer 7: Advanced Time-Aligned Profiling (Perfetto Tracing)

When memory anomalies do not occur in isolated bursts, you need an integrated timeline to correlate memory tracking events with system behaviors like scheduling delays, binder transactions, and hardware state changes.

Configure a Perfetto trace file to capture concurrent data streams over a timeline:
* android.native_alloc_samples (heapprofd): Captures native memory allocation call stacks over time.
* linux.ftrace: Monitors low-level kernel event changes, scheduler context switches, and memory page transitions.
* android.binder: Records runtime Binder transaction sizes to capture data bloat across IPC boundaries.

### Expert Analysis Archetypes
* Flame Graph Leaks vs. Peak Demands: Do not just filter for the largest total allocator blocks. Instead, track the allocation timeline to find functions displaying continuous, step-like growth. A minor allocation routine that runs continuously without dropping is a far more dangerous leak than a heavy one-off buffer allocation that cleans up correctly.
* Correlating Memory to System Fluctuations: Map memory expansion curves directly against CPU Throttling events or Binder Spikes. If memory growth aligns perfectly with a spike in incoming IPC transactions, focus your debugging on your inter-process communication boundaries rather than your local heap controllers.

---

## Real-World Debugging Insight:
If your process memory size (RSS) climbs steadily over time, but your inner Java Heap and Native Heap footprints remain completely flat and normal, stop looking at your app's code. Open /proc/<PID>/smaps and check your DMA-BUF allocations or specialized graphics driver memory mappings. You are likely dealing with an unclosed file descriptor leak or an unreleased graphics surface buffer that is quietly locking up system resources.

#### Advanced Defragmentation Tip:
When debugging high-order allocation failures during active lab testing, you can force the kernel to run an immediate memory defragmentation cycle across all allocator zones:

adb shell su -c "echo 1 > /proc/sys/vm/compact_memory"

Capture snapshots of /proc/buddyinfo before and after this instruction. If the structural counts in the high-order columns scale back upward, you have confirmed that your target failure mode is caused by external fragmentation rather than a true physical out-of-memory condition.

---

## Next Step:
Now that you have mastered advanced diagnostics, system pressure monitoring, and low-level kernel tracing, head to Chapter 3: Memory Allocator Internals to learn exactly how jemalloc, Scudo, and ART organize and manage memory blocks behind the scenes.