# Android Memory Deep Dive
### System-level understanding with real debugging insights

This project provides a comprehensive, system-level guide to understanding and managing memory within the Android ecosystem, bridging the gap between high-level application development and low-level kernel behavior.

---

## 📚 Table of Contents

### 📘 Chapter 1: Android Memory Types
A deep dive into the layered memory system, covering Virtual Address Space, Allocators, and Kernel backing for:
* Native & Java Heaps
* Stack & Static Memory
* DMA-BUF, Page Cache, and Anonymous Memory

### 🛠️ Chapter 2: Memory Issues & Debugging Toolkit
Focuses on identifying common memory failures and the modern toolchain used to solve them:
* **Common Issues:** Native leaks, Java collection leaks, Stack overflows, and CMA exhaustion.
* **Debugging Tools:** `dumpsys meminfo`, `procrank`, `Perfetto` (`heapprofd`), and Android Studio Profiler.
* **System Logs:** Analyzing `/proc/pid/smaps`, `dmesg` for reserved memory, and tombstone logs.

### ⚙️ Chapter 3: How Allocators Work
Deep dive into the internal mechanics of memory allocation:
* **Native Allocators:** Understanding **jemalloc** and **scudo** behavior in Android Bionic.
* **ART Runtime:** How the Android Runtime manages the Java heap through generation-based concurrent garbage collection.
* **Kernel Allocators:** The role of `kmalloc`, `vmalloc`, and the Slab/Buddy systems.

### 🧪 Chapter 4: Android Tuning Parameters & Hooks
Exploring the "knobs" available for system-level performance engineering:
* **ZRAM Configuration:** Tuning compression ratios and swap sizes.
* **Heap Limits:** Modifying `dalvik.vm.heapsize` and growth limits.
* **Vendor Hooks:** Understanding reserved memory carveouts and GKI (Generic Kernel Image) extensions.

### 📉 Chapter 5: Memory Reclaim Policies
Understanding how Android survives under extreme memory pressure:
* **LMK (Low Memory Killer):** How `lmkd` selects processes to kill based on `oom_adj` scores.
* **PSI (Pressure Stall Information):** Modern kernel metrics for detecting system thrashing.
* **Page Cache Management:** How the kernel automatically reclaims file-backed memory.

### 📜 Chapter 6: Guidelines & Best Practices
Actionable strategies for performance engineering:
* Lifecycle-aware memory management to prevent Handler and Activity leaks.
* Efficient buffer sharing using `DMA-BUF` and `Ashmem`.
* Proactive monitoring using PSI and memory pressure signals.

---

## 🚀 Focus Areas
* **Real Debugging:** Moving beyond theory to practical `adb` and `Perfetto` workflows.
* **Android Internals:** Understanding the relationship between APIs, Allocators, and the Linux Kernel.
* **Performance Engineering:** Learning to balance memory usage with system responsiveness and stability.
