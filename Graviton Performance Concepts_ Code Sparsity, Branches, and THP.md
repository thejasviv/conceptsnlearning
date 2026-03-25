# Graviton Performance Concepts: Code Sparsity, Branches, and THP

## 1. Code Sparsity in Graviton

In the context of AWS Graviton (and ARM/AArch64 processors in general), **code sparsity** refers to how scattered or widely distributed frequently executed instruction code is across memory.

When an application runs—particularly long-running applications on a JVM (Java Virtual Machine) that dynamically compile code—the executable instructions can become fragmented over time. Instead of "hot" (frequently used) code being packed tightly together, it gets spread out across the CodeCache. This lack of compactness is known as code sparsity.

### Why is Code Sparsity an Issue?

Graviton processors rely heavily on a hardware component called the **Branch Target Buffer (BTB)**, which is part of the CPU's branch prediction unit. The BTB tries to predict the destination of indirect branches so the CPU can fetch instructions ahead of time and keep execution running smoothly.

Code sparsity causes a mechanical bottleneck for a few reasons:

- **AArch64 Branch Limits**: The AArch64 architecture has a 128 MB immediate branch range limit.
- **Forced Indirect Branches**: If an application's CodeCache grows large and becomes sparse, the target of a function call might be too far away in memory for a direct branch. To bridge that gap, the JIT compiler is forced to emit indirect branches.
- **BTB Pressure and CPU Stalls**: Indirect branches place massive pressure on the BTB. Because the hot code is scattered across a vast address range, the BTB simply cannot keep track of all the different destination regions. This leads to a spike in branch prediction misses and front-end stalls, meaning the CPU is idling and wasting cycles while it waits for the correct instructions to be fetched from memory.

### How to Mitigate Code Sparsity

To address this degradation (commonly in Java environments), you can:

1. **Tweak JVM Compiler Flags**: Disabling tiered compilation (e.g., `-XX:-TieredCompilation`) limits the JIT to only use the C2 compiler, reducing the total volume of compiled code and increasing code locality.
2. **Enable Transparent Huge Pages**: Utilizing THP (e.g., `-XX:+UseTransparentHugePages`) speeds up address translations, indirectly helping branch prediction logic.
3. **Group Hot Code**: Utilize features like HotCodeHeap in newer OpenJDK versions to periodically identify and relocate hot methods into a compact, dedicated segment of memory.

---

## 2. Direct vs. Indirect Branches

Whenever code contains an `if` statement, loop, or function call, the CPU needs to "branch" (jump) to a different part of the executing code.

### Direct Branches

The destination address is hardcoded right into the instruction itself, usually as an offset.

**Advantage**: Highly predictable. The CPU reads the instruction, instantly knows where it's going next, and can fetch instructions ahead of time efficiently.

### Indirect Branches

The destination address isn't known immediately. The instruction tells the CPU to look inside a specific register or memory location to find the target address.

**Disadvantage**: The CPU has to wait to read that register/memory before knowing where to go. To avoid stalling, the BTB tries to guess the destination based on past behavior. If the BTB is overwhelmed (like in cases of high code sparsity), it guesses incorrectly, and the CPU wastes time.

---

## 3. Transparent Huge Pages (THP)

To understand Transparent Huge Pages, we must look at how operating systems manage memory.

### The TLB Bottleneck

Modern operating systems divide memory into small chunks called "pages," typically **4 Kilobytes (4KB)** in size. When mapping virtual to physical memory, the CPU caches recent translations in the **Translation Lookaside Buffer (TLB)**.

If you have a massive application, tracking millions of tiny 4KB pages overflows the TLB. When the CPU can't find the address in the TLB (a **TLB miss**), it does a slow, manual lookup in main memory, hurting performance.

### The Huge Pages Solution

Instead of 4KB pages, the OS can use much larger blocks of contiguous memory—typically **2 Megabytes (2MB)** or even **1 Gigabyte (1GB)**. Because each page holds vastly more data, the CPU needs far fewer entries in the TLB. This drastically reduces TLB misses.

### The "Transparent" Part

**Transparent Huge Pages (THP)** is a Linux kernel feature that automates the use of Huge Pages. In the background, the OS tries to find contiguous 4KB pages and "promotes" them into 2MB Huge Pages completely transparently to the application.

> **Note**: While THP is great for reducing TLB misses, it can sometimes cause latency spikes (compaction stalls) if physical memory is heavily fragmented and the OS struggles to create contiguous 2MB blocks on the fly. Because of this, it is often recommended to set THP to `madvise` so it is only used when an application explicitly asks for it.
