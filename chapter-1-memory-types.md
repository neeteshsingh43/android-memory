 📘 Chapter 1: Android Memory Types  
 (Allocator, Address Space & Real API Deep Dive)

Memory in Android is not a single pool—it is a **layered system spanning application, runtime, native, and kernel space**.

To debug memory effectively, you must understand:

- Where the memory lives (Virtual Address Space)
- Who allocates it (Allocator)
- Which APIs are used
- Which kernel subsystem backs it
- How it behaves over time

This chapter breaks down each memory type from a **real Android system perspective**.

---

 🧠 1. Native Heap (C/C++ Layer)

 📌 Overview
The native heap is used for dynamic memory allocation in C/C++ layers.  
It provides full control but comes with the responsibility of manual lifecycle management.

 📍 Virtual Address Space
- Appears as `[heap]` in `/proc/<pid>/maps`

 👤 Allocator
- Android bionic libc
- Backed by **jemalloc / scudo** (modern Android)

 🔧 APIs
- `malloc()`, `calloc()`, `realloc()`, `free()`
- `new`, `delete`

 📦 Kernel Backing
- Small allocations → `brk()`
- Large allocations → `mmap()` (anonymous pages)

 📍 Where It Is Used
- Camera HAL buffer processing
- MediaCodec internals
- SurfaceFlinger rendering pipeline
- Vendor native libraries

 🔍 How to Identify
- `adb shell dumpsys meminfo <pid>` → Native Heap section
- RSS increases while Java heap remains stable
- Perfetto → `heapprofd`

 ⚠️ Real-World Scenario
A camera pipeline allocates small metadata buffers per frame using `malloc()`.  
An error path misses `free()` → leak accumulates slowly →  
After long usage (20–30 mins), memory crosses threshold → OOM kill.

---

 📦 2. Java Heap (ART Runtime)

 📌 Overview
The Java heap is managed by the Android Runtime (ART) using garbage collection.  
It simplifies development but introduces **lifecycle-based leaks**.

 📍 Virtual Address Space
- Multiple regions (young, old, large object space)
- Allocated using `mmap()`

 👤 Allocator
- ART GC (generation-based, concurrent collectors)

 🔧 APIs
- Object creation: `new`
- Android APIs:
  - `Bitmap.createBitmap()`
  - `ArrayList`, `HashMap`
  - View system (`TextView`, `RecyclerView`)

 📍 Where It Is Used
- Activities, Fragments
- UI components
- Application-level caches

 🔍 How to Identify
- Android Studio Profiler
- `dumpsys meminfo`
- Frequent GC but memory not reducing

 ⚠️ Real-World Scenario
A `Handler` defined as non-static inner class holds implicit reference to Activity →  
Activity destroyed but not collected →  
Repeated navigation → heap grows → OOM.

---

 🧵 3. Stack Memory (Thread Execution)

 📌 Overview
Stack memory is allocated per thread and is used for function calls and local variables.  
It is fast but limited in size.

 📍 Virtual Address Space
- `[stack]`, `[stack:<tid>]`

 👤 Allocator
- Kernel during thread creation

 🔧 APIs
- Java: `new Thread()`
- NDK: `pthread_create()`, `pthread_attr_setstacksize()`

 📦 Kernel Backing
- `mmap()` with guard pages

 📍 Where It Is Used
- Function call frames
- Local variables
- JNI transitions
- Binder thread pool execution

 🔍 How to Identify
- Tombstone logs (`/data/tombstones`)
- SIGSEGV near stack boundary

 ⚠️ Real-World Scenario
A JNI function allocates a large array on stack:
```c
uint8_t buffer[1920 * 1080];


4. Static Memory (.text / .data / .bss)
📌 Overview

Static memory is allocated at load time and remains for the entire process lifecycle.

📍 Virtual Address Space
Mapped during ELF loading:
.text → code
.data → initialized globals
.bss → zero-initialized globals
👤 Allocator
Linker (/system/bin/linker64)
🔧 APIs
No runtime APIs (defined at compile time)
📦 Kernel Backing
File-backed (.text, .rodata)
Anonymous (.bss)
📍 Where It Is Used
Global configuration
Lookup tables
Codec constants
🔍 How to Identify
nm, size, readelf
High baseline memory at startup
⚠️ Real-World Scenario

A 25MB lookup table defined globally in a camera library →
Every process loading the library consumes that memory →
High baseline RAM even when feature unused.

⚙️ 5. Kernel Memory (Driver & OS Layer)
📌 Overview

Kernel memory is used for system-level operations and hardware interaction.

📍 Virtual Address Space
Kernel space (not visible to user processes)
👤 Allocator
kmalloc, kzalloc, vmalloc
Slab allocator (object caching)
🔧 APIs (Driver Side)
kmalloc(size, GFP_KERNEL)
kzalloc
vmalloc
📦 Backing
Slab caches
Buddy allocator
📍 Where It Is Used
Camera drivers (V4L2)
Binder IPC
Networking stack
🔍 How to Identify
/proc/meminfo
slabtop
kmemleak
⚠️ Real-World Scenario

Camera driver allocates buffers but does not free on stream stop →
Slab grows continuously →
System memory pressure → apps killed by LMK.

🎮 6. DMA / ION / DMABUF (Shared Buffers)
📌 Overview

Used for high-performance data sharing across hardware components without copying.

📍 Virtual Address Space
Allocated in kernel, mapped into userspace via mmap()
👤 Allocator
DMA-BUF framework
CMA (Contiguous Memory Allocator)
🔧 Android APIs
AHardwareBuffer_allocate()
GraphicBuffer (AOSP)
ANativeWindow
📦 Kernel Backing
Physically contiguous memory (CMA)
📍 Where It Is Used
Camera → ISP → GPU → Display
Video playback
XR pipelines
🔍 How to Identify
/sys/kernel/debug/dma_buf/
dumpsys SurfaceFlinger
⚠️ Real-World Scenario

Camera preview buffers not released →
CMA exhausted →
New allocations fail → black screen / preview stuck.

📂 7. Page Cache (File-Backed Memory)
📌 Overview

Page cache stores file data in memory to improve performance.

📍 Virtual Address Space
File-backed mappings
👤 Allocator
Kernel page cache subsystem
🔧 APIs
read(), open()
Java: FileInputStream
mmap()
📦 Kernel Backing
Page cache
📍 Where It Is Used
APK loading
Shared libraries (.so)
ART oat files
🔍 How to Identify
/proc/meminfo → Cached
Drops under memory pressure
⚠️ Real-World Scenario

Large log files read repeatedly →
Cache grows → appears like leak →
Actually reclaimed automatically.

📊 8. Anonymous Memory (RSS)
📌 Overview

Represents actual RAM used by a process.

📍 Virtual Address Space
Heap + stack + anonymous mmap
👤 Allocator
Combined: ART + libc + kernel
🔧 APIs
malloc
mmap(MAP_ANONYMOUS)
📦 Kernel Backing
Anonymous RAM pages
📍 Where It Is Used
All processes
🔍 How to Identify
procrank
/proc/<pid>/smaps
⚠️ Real-World Scenario

Small leaks across multiple layers →
Individually minor → collectively large →
Process killed due to high RSS.

🧬 9. mmap (Explicit Memory Mapping)
📌 Overview

Used to map files or shared memory directly into address space.

📍 Virtual Address Space
Visible in /proc/<pid>/maps
👤 Allocator
Kernel (mmap syscall)
🔧 Android APIs
mmap(), munmap()
ASharedMemory_create() (ashmem replacement)
📦 Kernel Backing
File-backed or anonymous
📍 Where It Is Used
Shared memory
ML models
Graphics buffers
🔍 How to Identify
/proc/<pid>/maps, smaps
⚠️ Real-World Scenario

ML model mapped per request but never unmapped →
Virtual memory grows → eventual crash.

🔍 Final Mental Model

Every memory allocation in Android follows this chain:

👉 API → Allocator → Virtual Address → Kernel Subsystem → Physical Memory
