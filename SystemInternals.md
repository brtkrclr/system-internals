---
title: System Internals
description: From bits to containers, a connected, from-scratch tour of how a Linux system really works.
permalink: /
---

# System Internals

Notes on how a Linux machine actually works, from the transistor up to the container. Each part builds on the previous ones. Look for the **🔗 Connections** boxes: they show where a concept from one part shows up again in another. That's the point of these notes. Nothing in a computer works in isolation.

**Contents**

0. [The Big Picture: How Everything Connects](#0-the-big-picture-how-everything-connects)
1. [Data Representation](#1-data-representation)
2. [CPU Fundamentals](#2-cpu-fundamentals)
3. [Operating System Fundamentals](#3-operating-system-fundamentals)
4. [Processes](#4-processes)
5. [Threads and Concurrency](#5-threads-and-concurrency)
6. [Memory Management](#6-memory-management)
7. [Files and I/O](#7-files-and-io)
8. [Signals](#8-signals)
9. [Inter-Process Communication (IPC)](#9-inter-process-communication-ipc)
10. [Namespaces and cgroups (Containers)](#10-namespaces-and-cgroups-containers)
11. [Putting It All Together: One Command, Every Layer](#11-putting-it-all-together-one-command-every-layer)
12. [Questions and Answers](#12-questions-and-answers)

🧪 **New:** [interactive visuals for every part](visuals/index.html): click bits, walk page tables, step through system calls.

---

# 0. The Big Picture: How Everything Connects

> 🧪 **Interactive visuals:** [Part 0, visualized](visuals/part0-big-picture.html): an interactive page with a demo for each section.


A Linux system has three main levels: **hardware**, the **kernel**, and **user processes**.

```
+---------------------------- USER SPACE (Ring 3) -----------------------------+
|   bash     python     nginx     postgres     ...                             |
|   each one is a PROCESS (Part 4) made of one or more THREADS (Part 5)        |
|                                                                              |
|   libc:  printf() -> write()   malloc() -> brk()/mmap()   fork() -> clone()  |
+----------------+---------------------------------------^------------+--------+
                 | system calls (3.3)          signals   |            | page faults,
                 v                             (Part 8)  |            v exceptions (3.4)
+------------------------------ KERNEL (Ring 0) -------------------------------+
|                                                                              |
|   task_struct  (one per thread: the hub everything hangs off)                |
|     |-- saved registers, state, prio ...... scheduler       (3.5, 3.6)       |
|     |-- mm     -> mm_struct -> VMAs, page tables ........... Memory (Part 6)  |
|     |-- files  -> fd table -> struct file -> inode ......... Files  (Part 7)  |
|     |-- signal / sighand -> pending signals, handlers ...... Signals (Part 8) |
|     |-- cred   -> uid, gid, capabilities ................... (7.11, 10.4)     |
|     `-- nsproxy, cgroups -> what this task can see/use ..... Containers (10)  |
|                                                                              |
|   subsystems: scheduler | memory manager | VFS + filesystems | drivers | net |
+------------------------------------+-----------------------------------------+
                                     | privileged instructions, MMU, interrupts, DMA
                                     v
+--------------------------------- HARDWARE -----------------------------------+
|   CPU cores: registers, caches, TLB, MMU   (Part 2)                          |
|   RAM: page frames   |   disks   |   network cards   |   timers              |
|   ... and all of it is just bits and bytes (Part 1)                          |
+------------------------------------------------------------------------------+
```

## 0.1 The Three Illusions

The kernel's main job is to give every process three illusions, on top of hardware that is shared by everyone:

| Illusion | What the process believes | Reality | How the kernel fakes it | Where |
| :---- | :---- | :---- | :---- | :---- |
| **1. My own CPU** | "I run continuously." | Hundreds of tasks share a few cores. | Timer interrupts + scheduler + context switches that save and restore registers in `task_struct`. | 2.3, 3.4–3.6 |
| **2. My own memory** | "I own a huge, contiguous address space starting near 0." | Physical RAM is shared, scattered and smaller. | Virtual memory: page tables + MMU + page faults. | Part 6 |
| **3. My own machine** | "I have my files, my terminal, my user id, my PID 1." | Everything is shared kernel objects. | fd tables, credentials, signals, and (for containers) namespaces and cgroups. | Parts 7–10 |

When the illusions break, you get the classic bugs. Two threads sharing illusion 2 cause race conditions (Part 5). A deleted file still held open makes `df` and `du` disagree (Part 7). A forked child shares a file offset with its parent (Part 7.2).

## 0.2 The One Data Structure to Remember: `task_struct`

If you remember only one thing, remember this: **in Linux, every thread of every process is a `struct task_struct`**, and almost every topic in these notes is "a field of `task_struct` and what it points to".

```
task_struct (one per process, and one per THREAD)
 |- pid, tgid             identity                                    -> 4.5
 |- __state               R / S / D / T / Z                           -> 4.8
 |- thread (saved regs)   restored on context switch    <- illusion 1 -> 3.5
 |- se / prio / policy    scheduler bookkeeping                       -> 3.6
 |- parent, children      the process tree                            -> 4.5
 |- mm      --> mm_struct     (page tables, memory maps)  <- illusion 2 -> 6.4
 |- files   --> files_struct  (fd table)                  \            -> 7.1
 |- fs      --> fs_struct     (cwd, root)                  |           -> 7.3
 |- cred    --> credentials   (uid, gid, capabilities)     | illusion 3-> 7.11
 |- signal/sighand -->        (signal handlers, pending)   |           -> 8.3
 `- nsproxy --> namespaces    (what makes a container)    /            -> 10.1
```

## 0.3 Reading Order

- **Parts 1–2** are the hardware vocabulary: bytes, alignment, caches, registers, privilege rings.
- **Part 3** is the contract between hardware and software: kernel, system calls, interrupts, scheduling, boot.
- **Parts 4–6** are about running code: processes, threads, memory.
- **Parts 7–9** are about talking to the world: files, signals, other processes.
- **Part 10** recombines everything to build containers.
- **Part 11** traces one command through every layer.
- **Part 12** is a set of questions that only make sense once you see the connections.

---

# 1. Data Representation

> 🧪 **Interactive visuals:** [Part 1, visualized](visuals/part1-data-representation.html): an interactive page with a demo for each section.


## 1.1 Bits, Bytes and Binary Numbers

A CPU is made of billions of **transistors**. A transistor is essentially an electronic switch. It has only two reliable states: **ON/OFF**, **Low Voltage/High Voltage**, or **0/1**.

Modern CPUs contain tens of billions of these tiny switches.

- A **bit** is the smallest unit of data in computing. It can be `0` or `1`.
- A **byte** is a group of `8 bits`.
- Most modern systems measure memory, files, and storage in bytes and multiples of bytes.

If we have *n* bits we can represent *2^n* different values.

Example: with 2 bits, each can be 0 or 1, giving `00, 01, 10, 11`, so four outcomes (2² = 4). A **state** is just one specific bit arrangement: `0110`, `0001` and `1011` are three different states of 4 bits.

Units:

- `1 byte = 8 bits`
- `1 KiB = 1024 bytes` (often written "KB")
- `1 MiB = 1024 KiB`

Strictly speaking, **KB/MB/GB** are decimal (SI) units (1 KB = 1000 bytes), while **KiB/MiB/GiB** are binary (1 KiB = 1024 bytes). Disk manufacturers use the decimal units, while RAM and many OS tools use the binary ones. That's why a "1 TB" disk shows up as ~931 GiB.

Why it matters:

- Bits are the raw language of hardware.
- Bytes are the practical unit used by software, memory addressing, and file sizes.

### Binary Numbers

Binary is just a positional number system, like decimal. Decimal uses powers of 10:

```
352 = 3 × 10² + 5 × 10¹ + 2 × 10⁰
```

Binary uses powers of 2:

```
1011₂ = (1 × 2³) + (0 × 2²) + (1 × 2¹) + (1 × 2⁰) = 8 + 0 + 2 + 1 = 11
```

> 🔗 **Connections:** You will meet "n bits → 2^n values" over and over: 9 bits of a virtual address index **512** page-table entries (6.3), 12 offset bits address a **4096-byte** page (6.3), 2 bits of the `CS` register encode **4** privilege rings (2.6), and 64 signals fit in one 64-bit pending mask (8.4).

## 1.2 Hexadecimal

Binary becomes difficult for humans to read. Instead of **1101111010101100**, we group bits into sets of four: **1101 1110 1010 1100**. Each group (called a **nibble**) maps to exactly one hexadecimal digit (0–9, A–F):

```
1101 1110 1010 1100
  D    E    A    C    ->  0xDEAC
```

The `0x` prefix marks a number as hexadecimal. Two hex digits = one byte, which is why memory dumps and addresses are almost always shown in hex.

Example: you'll see hex everywhere in Linux:

```bash
$ cat /proc/self/maps | head -2
5603c3a00000-5603c3a02000 r--p 00000000 103:02 1835123   /usr/bin/cat
5603c3a02000-5603c3a07000 r-xp 00002000 103:02 1835123   /usr/bin/cat
```

Each range is a virtual memory area (6.4). `0x5603c3a02000 - 0x5603c3a00000 = 0x2000 = 8192 bytes = 2 pages`.

## 1.3 Bytes

*The smallest **addressable** unit of memory on almost all modern CPUs.*

Here's something many people don't know: **originally, a byte was NOT always 8 bits.**

In the 1950s and 1960s, different computers used different byte sizes (6-bit, 7-bit, 8-bit, 9-bit). There was **no universal standard**.

### Why 6 Bits?

Because early computers mostly processed uppercase English letters. How many characters? **A–Z (26 letters) + numbers + punctuation**. Six bits give **2⁶ = 64** characters.

### Why 7 Bits?

Then ASCII appeared. ASCII needs 128 characters. How many bits? **2⁷ = 128**.

### Why Did 8 Bits Win?

This was one of the best engineering compromises ever made. Eight bits provide **2⁸ = 256 values**, enough to represent:

- characters
- small integers
- machine instruction opcodes
- memory values

while still being efficient for hardware. IBM's System/360 (1964) standardized the **8-bit byte**, and nearly all modern architectures followed.

**Question:** Why not make a byte 16 bits? Because the letter `A` (`01000001`) would become `0000000001000001`: twice the memory, bandwidth, and cache usage. Wasteful.

## 1.4 Word Size (32-bit vs 64-bit Systems)

A **word** is the natural data size the CPU handles most efficiently. More precisely:

> A word is the size of the CPU's general-purpose registers.

| Architecture | Word size | Handles per operation |
| :---- | :---- | :---- |
| 32-bit CPU | 32 bits (4 bytes) | 4 bytes at a time |
| 64-bit CPU | 64 bits (8 bytes) | 8 bytes at a time |

Word size = how much data the CPU can naturally process in one operation.

*Note (x86 terminology):* In general architecture theory, word size equals register width (32 or 64 bits). In x86 assembly specifically, a **WORD** is strictly 16 bits (2 bytes), a **DWORD** (Double Word) is 32 bits (4 bytes), and a **QWORD** (Quad Word) is 64 bits (8 bytes), due to legacy 16-bit 8086 compatibility.

### Pointers and Address Space

- A pointer is just a variable that stores a **memory address**.
- **32-bit architecture:** a pointer typically occupies **4 bytes**. This allows the CPU to address roughly **4 GB** (2³² bytes) of virtual memory.
- **64-bit architecture:** a pointer occupies **8 bytes**. The address space is exponentially larger.

The main idea is still the same: **one address → one byte**. The only thing that changed is **how large the address itself is**.

By the late 90s and early 2000s, databases, web servers, and high-end workstations were slamming face-first into that hard 4 GB ceiling. They could not fit their active working data into RAM because a 32-bit pointer couldn't produce a number large enough to address anything past the 4 GB mark. Moving to 64-bit was born out of pure necessity to expand that address space.

### Canonical Addresses (48-bit / 57-bit Virtual Addresses)

A 64-bit pointer could in theory address 2⁶⁴ bytes (16 EiB). No CPU actually translates all 64 bits: that would need huge page tables for no benefit. x86-64 CPUs use **48-bit virtual addresses** (256 TiB), or **57-bit** with 5-level paging (LA57, 128 PiB).

The rule is that the unused upper bits **must be copies of the highest used bit** (bit 47). Such an address is called **canonical**. This splits the 64-bit space into two halves with a giant hole in the middle:

```
0xFFFF_FFFF_FFFF_FFFF  +----------------------+
                       |  kernel space        |  bit 47 = 1, bits 63..48 all 1
0xFFFF_8000_0000_0000  +----------------------+
                       |                      |
                       |  NON-CANONICAL HOLE  |  any access -> #GP fault -> SIGSEGV
                       |                      |
0x0000_7FFF_FFFF_FFFF  +----------------------+
                       |  user space          |  bit 47 = 0, bits 63..48 all 0
0x0000_0000_0000_0000  +----------------------+
```

That's why user-space addresses you see in a debugger look like `0x7ffc...` (stack) or `0x55...`/`0x7f...` (binary and libraries), and kernel addresses look like `0xffffffff8...`. A 48-bit address is 12 hex digits.

> 🔗 **Connections:** The 48 bits are exactly `9+9+9+9+12`: four page-table levels plus a page offset (6.3). The user/kernel split here is the same split protected by the `U/S` bit in page-table entries (6.3) and by CPU rings (2.6).

### PAE and the 3 GB / 1 GB User/Kernel Split on 32-bit Linux

This concerns historical 32-bit x86 Linux systems.

A 32-bit virtual address contains 32 bits, so it can represent 2³² = 4 GiB of virtual address space.

A traditional Linux configuration divides this into:

- **3 GiB user space:** the address space available to a user process.
- **1 GiB kernel space:** the region reserved for the kernel in that process's virtual address-space layout.

This is a virtual-address-space split, not a division of physical RAM.

**PAE (Physical Address Extension)** allows supported 32-bit x86 systems to address more than 4 GiB of physical memory (up to 64 GiB) by using 64-bit page-table entries with wider physical frame numbers and an additional page-table level.

Important distinction: PAE does not give a single ordinary 32-bit process a 64-bit virtual address space. Each process still sees 4 GiB; the machine as a whole can just hold more processes in RAM. (This is also why 32-bit kernels needed `HIGHMEM`: RAM that didn't fit in the kernel's 1 GiB window, see 6.2.)

## 1.5 Endianness

Endianness answers a very specific question:

**When a value occupies multiple bytes, in what order are those bytes stored in memory?**

There are two common types:

- **Little-endian**: least significant byte stored first
- **Big-endian**: most significant byte stored first

Example value `0x12345678` stored in memory starting at address `0x1000`:

| Address | Little-Endian | Big-Endian |
| :---: | :---: | :---: |
| 0x1000 | 0x78 | 0x12 |
| 0x1001 | 0x56 | 0x34 |
| 0x1002 | 0x34 | 0x56 |
| 0x1003 | 0x12 | 0x78 |

Notice something important: the **addresses do not change**, the **number of bytes does not change**. Only the **order of the bytes** changes.

You can see it yourself:

```bash
$ printf '\x78\x56\x34\x12' > n.bin
$ xxd n.bin
00000000: 7856 3412                                xV4.
$ od -An -tx4 n.bin        # interpret as one 4-byte int on this (little-endian) machine
 12345678
```

### Which Architectures Use Which

| Architecture | Endianness |
| :---- | :---- |
| x86, x86-64 (Intel, AMD) | Little-endian |
| ARM / AArch64 (phones, Apple Silicon, Graviton) | Bi-endian in hardware, but run little-endian in practice |
| RISC-V | Little-endian |
| PowerPC (older), SPARC, IBM s390x (mainframes) | Big-endian |
| Network protocols (IP, TCP, UDP headers) | Big-endian ("network byte order") |

Little-endian has a nice property: the address of a value is also the address of its low byte, so reading a 64-bit value as 32-bit or 8-bit at the same address gives the low part.

### Network Byte Order (`htonl` / `ntohl`)

Internet protocols were standardized as **big-endian**. A little-endian machine must convert multi-byte fields before putting them on the wire:

```c
#include <arpa/inet.h>

uint16_t port = 8080;            // 0x1F90
uint16_t wire = htons(port);     // host-to-network short: bytes become 1F 90 in memory

struct sockaddr_in addr;
addr.sin_port = htons(8080);     // forgetting this binds to port 36895 (0x901F)!
addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
```

- `htons` / `htonl`: host to network (16 / 32-bit)
- `ntohs` / `ntohl`: network to host

On a big-endian host these are no-ops; on x86 they swap bytes (a single `bswap`/`rol` instruction).

> 🔗 **Connections:** Endianness matters any time bytes leave one CPU's interpretation: sockets (9.5), binary file formats like ELF (whose header declares its endianness, 4.9), and reading raw memory dumps from `/proc/<pid>/mem`.

## 1.6 Alignment

Alignment means:

**Data is stored at memory addresses that are multiples of its size (or the architecture's alignment requirement).**

For example, a 4-byte `int` is ideally stored at an address divisible by 4. The reason is that the CPU does not fetch data from RAM one byte at a time; it fetches fixed-size chunks.

Think of RAM not as a single long line of individual bytes, but as a series of **boxes**, each holding exactly 4 bytes:

```
Box 0      Box 1      Box 2        Box 3
[0 1 2 3]  [4 5 6 7]  [8 9 10 11]  [12 13 14 15]
```

The CPU can grab **one whole box** in a single trip. It cannot grab "half of box 0 and half of box 1" in one trip. If what it needs spans two boxes, it has to make **two trips** and glue the pieces together.

Alignment just means: **does the data you want fit entirely inside one box, or does it straddle two boxes?**

Say you want to store a 4-byte integer (like a normal `int` in C).

**Case 1: Aligned.** You store it starting at byte address 4 (the start of Box 1).

```
Box 0      Box 1      Box 2
[. . . .]  [I I I I]  [. . . .]
            ^ your whole int fits perfectly inside one box
```

One trip to memory. Done.

**Case 2: Misaligned.** You store it starting at byte address 2 instead.

```
Box 0      Box 1
[. . I I]  [I I . .]
     ^ 2 bytes of your int are in Box 0, the other 2 are in Box 1
```

Now the CPU has to:

1. Fetch Box 0, grab the 2 bytes of your int that live there
2. Fetch Box 1, grab the other 2 bytes
3. Stitch those pieces together into one number

Twice the work for the exact same 4 bytes of data. That's the whole problem alignment is about.

### Alignment in Practice (Cache-Line Splits, Architectures That Fault, Atomicity)

On modern x86 the "box" is really a **64-byte cache line** (1.8), so most misaligned accesses are actually free as long as they stay inside one line. The real costs show up at boundaries:

| Situation | What happens |
| :---- | :---- |
| Misaligned, but within one cache line | Usually no penalty on modern x86. |
| Crosses a **cache-line** boundary | Two cache lines must be fetched: slower ("split load"). |
| Crosses a **page** boundary (4 KiB) | Two TLB lookups, and possibly **two page faults** (2.4, 6.5). |
| Strict-alignment CPUs (old ARM, SPARC, some MIPS) | Hardware **faults** → Linux delivers **`SIGBUS`** to the process (8.2). |
| x86 SSE `movaps` on unaligned data | `#GP` fault → `SIGSEGV`. |

**Atomicity:** On x86, a naturally aligned load or store of 1, 2, 4 or 8 bytes is guaranteed to be **atomic**: no other core can see half of it. A misaligned value that straddles two cache lines is **not** atomic, and a `lock`-prefixed instruction on it becomes a "split lock" that stalls the whole memory bus. Modern Linux can detect and warn about or kill split-lock offenders.

> 🔗 **Connections:** Alignment is why atomics and mutexes (5.3) rely on the compiler aligning your variables, why `malloc` returns 16-byte-aligned memory, and why page-aligned buffers are required for `O_DIRECT` I/O (7.6).

## 1.7 Padding

Padding exists because of alignment requirements. Consider:

```c
struct A {
    char c;
    int  i;
};
```

Sizes are **char = 1 byte** and **int = 4 bytes**.

Without alignment padding:

```
Address 1000        : c (1 byte)
Address 1001 - 1004 : i (4 bytes, starting at 1001)
```

The problem: `i` starts at address `1001`, which is **not divisible by 4**. So the compiler inserts 3 bytes of padding after `c`:

```
Address 1000        : c (1 byte)
Address 1001 - 1003 : padding (3 bytes)
Address 1004 - 1007 : i (4 bytes)
```

Now `1004 % 4 == 0`: perfect alignment, and the total struct size is 8 bytes.

Alignment makes memory accesses efficient (or valid), and padding is how the compiler achieves alignment. The total struct size must also be padded to a multiple of its largest member's alignment requirement (so that every element of an array of structs is aligned too).

Example of struct layout optimization:

```c
// Bad layout (12 bytes total due to internal & trailing padding)
struct Bad {
    char c; // 1 byte + 3 bytes padding (to align 'a' to a 4-byte boundary)
    int  a; // 4 bytes
    char d; // 1 byte + 3 bytes trailing padding (round total up to a multiple of 4)
};

// Optimized layout (8 bytes total)
struct Good {
    int  a; // 4 bytes (offset 0)
    char c; // 1 byte  (offset 4)
    char d; // 1 byte  (offset 5) + 2 bytes trailing padding (offsets 6-7)
};
```

Rule of thumb: order struct members from largest to smallest alignment.

You can check it:

```c
#include <stdio.h>
#include <stddef.h>
struct Bad  { char c; int a; char d; };
struct Good { int a; char c; char d; };
int main(void) {
    printf("Bad=%zu Good=%zu offsetof(Bad,a)=%zu\n",
           sizeof(struct Bad), sizeof(struct Good), offsetof(struct Bad, a));
}
// Bad=12 Good=8 offsetof(Bad,a)=4
```

> 🔗 **Connections:** The kernel cares a lot about this. `task_struct`, `struct page` (one per 4 KiB of RAM!) and `struct inode` are carefully ordered so hot fields share a cache line, and so that millions of copies don't waste RAM on padding. One extra byte in `struct page` costs about 4 MB on a 16 GB machine.

## 1.8 Cache Line

A cache line is the **unit of data transferred between RAM and the CPU cache**.

Typically: **1 cache line = 64 bytes** (Apple Silicon uses 128-byte lines).

RAM is effectively divided into 64-byte lines:

```
Line 0: bytes   0-63
Line 1: bytes  64-127
Line 2: bytes 128-191
```

### What Happens When You Access One Byte

Say your program reads a single byte at address `100`. That address falls inside Line 1 (bytes 64–127).

The CPU doesn't fetch just byte 100. It fetches **the entire 64-byte line** (bytes 64–127) and stores that whole line in cache:

```
You asked for:        byte 100
CPU actually fetches: [64 65 66 ... 100 ... 126 127]   <- whole line, one trip
```

This pays off because programs usually have **spatial locality**:

> If I access one memory location, I'm likely to access nearby locations soon.

### Temporal Locality

The other half of locality:

> If I accessed a memory location recently, I'm likely to access it again soon.

Loop counters, the top of the stack, a hot function's code and a frequently used lock are all reused constantly, so keeping them in cache pays off. Caches evict the **least recently used** lines first, betting on temporal locality.

Classic example combining both kinds, a 2D array in C (stored row by row):

```c
int a[4096][4096];

// FAST: walks memory sequentially -> uses all 16 ints of each fetched 64-byte line
for (int i = 0; i < 4096; i++)
    for (int j = 0; j < 4096; j++)
        sum += a[i][j];

// SLOW (often 5-10x): jumps 16 KiB between accesses -> one cache line per int,
// and a new page every access -> TLB misses too
for (int j = 0; j < 4096; j++)
    for (int i = 0; i < 4096; i++)
        sum += a[i][j];
```

Same number of instructions, wildly different speed. That's the "performance is about locality" idea from 2.4.

### False Sharing

**Performance hazard:** If two threads running on different CPU cores modify separate variables that reside within the same 64-byte cache line, the cores must repeatedly invalidate and reload each other's copy of that line, causing high cache-coherency latency.

```
            one 64-byte cache line
   +-----------------+-----------------+--------------------+
   |  counter_A (8B) |  counter_B (8B) |  ... 48 more bytes |
   +-----------------+-----------------+--------------------+
        ^ written by core 0      ^ written by core 1

core 0 writes A -> its copy becomes Modified, core 1's copy is Invalidated
core 1 writes B -> must fetch the line back, core 0's copy is Invalidated
... the line ping-pongs between cores, though they never share data logically
```

#### Fixing False Sharing (Padding / `alignas(64)`)

Give each hot, independently written variable **its own cache line**:

```c
#include <stdalign.h>

struct counters {
    alignas(64) long a;   // starts a new 64-byte line
    alignas(64) long b;   // starts another one
};                        // sizeof == 128

// or manual padding:
struct padded_counter {
    long value;
    char pad[64 - sizeof(long)];
};
```

The Linux kernel uses `____cacheline_aligned_in_smp` for this, and goes one step further with **per-CPU variables**: each core increments its own copy, and the values are summed only when someone reads them.

> 🔗 **Connections:** False sharing is a direct consequence of the cache coherence protocol (MESI, 2.4), shows up as a concurrency *performance* bug (Part 5), and is why the kernel keeps per-CPU run queues (3.6) and per-CPU page lists (6.2).

## 1.9 Integers: Unsigned, Signed and Two's Complement

Bits have no meaning by themselves. The byte `11111011` is **251** if you read it as an unsigned number, and **−5** if you read it as a signed one. The type tells the CPU (and the compiler) how to interpret the bits.

### Unsigned Integers

Plain binary (1.1): *n* bits hold **0 to 2ⁿ − 1**.

| C type (Linux x86-64) | Bytes | Range |
| :---- | :---- | :---- |
| `uint8_t` / `unsigned char` | 1 | 0 to 255 |
| `uint16_t` | 2 | 0 to 65,535 |
| `uint32_t` / `unsigned int` | 4 | 0 to ~4.29 billion |
| `uint64_t` / `unsigned long` | 8 | 0 to ~1.8 × 10¹⁹ |

### Signed Integers: Two's Complement

Every modern CPU represents negative numbers in **two's complement**. The highest bit has a **negative weight**:

```
8-bit two's complement:  bit weights  -128  64  32  16   8   4   2   1

 5  =  0000 0101   =                    0 + 0 + 0 + 0 + 0 + 4 + 0 + 1
-5  =  1111 1011   =  -128 + 64 + 32 + 16 + 8 + 0 + 2 + 1  =  -5
```

To negate a number: **invert all bits, then add 1** (`-x == ~x + 1`).

```
 5        = 0000 0101
 ~5       = 1111 1010
 ~5 + 1   = 1111 1011   = -5
```

- Range with *n* bits: **−2ⁿ⁻¹ to 2ⁿ⁻¹ − 1**. For 8 bits: −128 to 127; for 32 bits: about ±2.1 billion.
- The top bit is the **sign bit**: 1 means negative.
- `-1` is **all ones** (`0xFF`, `0xFFFFFFFF`, ...).
- Why it won: addition and subtraction use the **same circuit** for signed and unsigned numbers. The ALU (2.3) just adds bits; only the interpretation of the result (and the flags) differs.

**Sign extension:** widening a signed value copies the sign bit into the new high bits: `-5` as 8 bits `FB` becomes `FFFFFFFB` as 32 bits. Widening an unsigned value fills with zeros.

### Overflow

```c
uint8_t  u = 255;  u = u + 1;          // 0: unsigned arithmetic wraps around (defined, modulo 2^8)
int8_t   s = 127;  s = s + 1;          // -128 on real hardware...
int      i = INT_MAX; i = i + 1;       // ...but in C, SIGNED overflow is UNDEFINED BEHAVIOR:
                                       // the compiler may assume it never happens and optimize accordingly
```

A classic signed/unsigned bug:

```c
char buf[100];
ssize_t n = read(fd, buf, sizeof buf);   // returns -1 on error
if (n < sizeof buf) { ... }              // sizeof is size_t (unsigned): n is converted to unsigned,
                                         // -1 becomes 18446744073709551615, and the check is FALSE
```

### Integer Sizes Are Not Universal

| Type | Linux/macOS 64-bit (LP64) | Windows 64-bit (LLP64) |
| :---- | :---- | :---- |
| `int` | 4 bytes | 4 bytes |
| `long` | **8 bytes** | **4 bytes** |
| pointer, `size_t` | 8 bytes | 8 bytes |

That's why portable code uses `<stdint.h>` types (`int32_t`, `uint64_t`) and `size_t`/`ssize_t` for sizes.

> 🔗 **Connections:** Two's complement is everywhere in these notes. A failed syscall returns a **negative** errno in `rax` (−4095 to −1, 3.3), which is just a huge unsigned number. The exit status is **8 bits**, so `exit(-1)` shows up as **255** and `exit(256)` as **0** (4.9). And a 32-bit signed `time_t` overflows in **2038** (3.8).

## 1.10 Floating-Point Numbers (IEEE 754)

Integers can't represent 0.5 or 6.02 × 10²³. **Floating point** stores numbers in scientific notation, in binary:

```
value = (-1)^sign  ×  1.fraction  ×  2^(exponent - bias)

double (64 bits):   | sign 1 | exponent 11 | fraction 52 |     bias 1023, ~15-17 decimal digits
float  (32 bits):   | sign 1 | exponent  8 | fraction 23 |     bias  127, ~6-9 decimal digits
```

Example: `-6.5` = `-110.1₂` = `-1.101₂ × 2²` → sign 1, exponent 2 + 1023, fraction `101000...`.

### Why `0.1 + 0.2 != 0.3`

Just like 1/3 = 0.3333... has no exact decimal representation, **0.1 has no exact binary representation**: it's `0.000110011001100110011...₂`, cut off after 52 bits. So 0.1 and 0.2 are each stored slightly wrong, and the errors don't cancel:

```python
>>> 0.1 + 0.2
0.30000000000000004
>>> 0.1 + 0.2 == 0.3
False
>>> abs((0.1 + 0.2) - 0.3) < 1e-9      # compare with a tolerance instead
True
```

Practical rules:

- **Never use floating point for money.** Use integer cents or a decimal type.
- Compare floats with a tolerance, not `==`.
- A `double` represents every integer exactly only up to **2⁵³** (~9 × 10¹⁵). Beyond that, gaps appear. JavaScript numbers are doubles, which is why huge JSON IDs get corrupted in JS.

### Special Values

| Value | Bits | Produced by |
| :---- | :---- | :---- |
| `+0` and `-0` | exponent 0, fraction 0 | They compare equal, but `1/-0.0 = -inf` |
| `+inf`, `-inf` | exponent all 1s, fraction 0 | `1.0 / 0.0`, overflow |
| `NaN` (not a number) | exponent all 1s, fraction ≠ 0 | `0.0 / 0.0`, `sqrt(-1)`. **`NaN != NaN`** is true! |
| Subnormals | exponent 0, fraction ≠ 0 | Tiny numbers near zero (can be much slower on some CPUs) |

> 🔗 **Connections:** Floating point runs on the FPU / SIMD units (2.3), whose registers must be saved on every context switch (3.5). **Floating-point** division by zero does **not** raise `SIGFPE` by default; it quietly returns `inf`. Only **integer** division by zero traps (`#DE` → `SIGFPE`, 3.4).

## 1.11 Text Encoding: ASCII, Unicode and UTF-8

Text is also just bytes; an **encoding** maps characters to numbers.

- **ASCII** (1963): 128 characters in 7 bits (1.3): `A` = 65 = `0x41`, `a` = `0x61`, `0` = `0x30`, newline = `0x0A`.
- **Unicode**: assigns a **code point** to every character in every script, written `U+XXXX`: `é` = U+00E9, `€` = U+20AC, `😀` = U+1F600. There are over a million possible code points (U+0000 to U+10FFFF), so they don't fit in one byte.
- **UTF-8** (the encoding used by Linux, the web, and most files): encodes each code point as **1 to 4 bytes**:

| Code point range | Bytes | Bit pattern |
| :---- | :---- | :---- |
| U+0000 – U+007F | 1 | `0xxxxxxx` (identical to ASCII) |
| U+0080 – U+07FF | 2 | `110xxxxx 10xxxxxx` |
| U+0800 – U+FFFF | 3 | `1110xxxx 10xxxxxx 10xxxxxx` |
| U+10000 – U+10FFFF | 4 | `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx` |

```bash
$ printf 'Aé€😀' | xxd
00000000: 41c3 a9e2 82ac f09f 9880                 A.........
#         A  é     €        😀
#         1  2     3        4 bytes
$ printf 'Aé€😀' | wc -c     # bytes
10
$ printf 'Aé€😀' | wc -m     # characters (needs a UTF-8 locale)
4
```

Why UTF-8 won: every ASCII file is already valid UTF-8; it has **no endianness** problem (it's a byte sequence, unlike UTF-16, 1.5); and a byte like `0x2F` (`/`) or `0x00` never appears inside a multi-byte character, so C strings and Unix paths keep working.

Consequences:

- `strlen()` counts **bytes**, not characters. Truncating a UTF-8 string at a byte limit can cut a character in half.
- Windows, Java and JavaScript use **UTF-16** internally (2 or 4 bytes per character), which is a frequent source of conversion bugs.
- To the Linux kernel, a **filename is just bytes**: any byte except `/` and NUL (`0x00`) is allowed. The kernel doesn't know or care about encodings; only user space interprets them (via the locale, `LANG=en_US.UTF-8`).

---

# 2. CPU Fundamentals

> 🧪 **Interactive visuals:** [Part 2, visualized](visuals/part2-cpu.html): an interactive page with a demo for each section.


## 2.1 Clock Cycle

A CPU has a **clock signal**, an electrical pulse that oscillates between high and low voltage at a fixed rate, derived from a crystal oscillator (usually quartz). Every "tick" of this clock is one **clock cycle**.

- **Clock speed** (e.g., 3.5 GHz) means the clock ticks 3.5 billion times per second, so one cycle is ~0.29 ns.
- Each tick synchronizes the CPU's internal components so they all move in lockstep, like a metronome for a marching band.

### Dynamic Frequency Scaling (Turbo Boost / DVFS)

The clock speed is not fixed. Power consumption grows roughly with **frequency × voltage²**, so modern CPUs constantly change both. This is **DVFS** (Dynamic Voltage and Frequency Scaling):

- **P-states** (performance states): frequency/voltage pairs used while running. Lower P-state number = faster.
- **Turbo Boost** (Intel) / **Precision Boost** (AMD): briefly run some cores above the base clock when there is thermal and power headroom.
- **C-states** (idle states): when a core has nothing to run, it stops its clock (C1), then powers parts of itself down (C6...). Deeper = more savings but slower wake-up.

Linux controls this through the **cpufreq** subsystem and its *governors*:

```bash
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
schedutil          # frequency follows the scheduler's view of utilization
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
2394000            # kHz
```

> 🔗 **Connections:** When no task is runnable, the scheduler runs the **idle task (PID 0)** (4.5), which executes `hlt`/`mwait` to enter a C-state. The `schedutil` governor gets its numbers directly from the scheduler (3.6). Benchmarks are noisy partly because frequency changes under you.

## 2.2 Instruction Cycle and Pipelining

A CPU is a machine that repeatedly fetches instructions, interprets them, and performs operations. The kernel schedules CPU time, but the CPU itself runs a continuous instruction cycle.

The **Fetch-Decode-Execute cycle** is the process a processor uses to run each machine-level instruction. Modern CPUs don't run these steps for one instruction at a time: they **pipeline** them, so different instructions are in different stages at the same moment (the classic 5-stage pipeline below):

| Instr \ Clock Cycle | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 1 | IF | ID | EX | MEM | WB |  |  |  |
| 2 |  | IF | ID | EX | MEM | WB |  |  |
| 3 |  |  | IF | ID | EX | MEM | WB |  |
| 4 |  |  |  | IF | ID | EX | MEM | WB |

IF: Instruction Fetch, ID: Instruction Decode, EX: Execute, MEM: Memory Access, WB: Write Back (to register)

Like a laundry: while load 1 is in the dryer, load 2 is in the washer. Each instruction still takes 5 cycles, but one finishes **every** cycle.

Real modern CPUs go further: they are **superscalar** (several instructions per cycle) and **out-of-order** (they execute instructions as soon as their inputs are ready, not strictly in program order).

### Superscalar and Out-of-Order Execution

**Superscalar:** the core has several execution units (multiple ALUs, load/store units, FPUs) and can **issue several instructions per cycle**. Modern cores sustain 4–8 instructions per cycle on good code. The metric is **IPC** (instructions per cycle); `perf stat` shows it.

**Out-of-order (OoO):** instead of stalling when one instruction waits (say, for a load from RAM that takes 300 cycles), the core looks ahead and runs later, independent instructions.

```
1: load  r1, [x]      ; cache miss, waits ~100 ns
2: add   r2, r1, 1    ; depends on 1 -> must wait
3: mul   r3, r4, r5   ; independent -> executes NOW, out of order
4: add   r6, r3, 7    ; depends on 3 only -> also executes now
```

The machinery that makes this safe:

- **Register renaming:** the 16 architectural registers (`rax`...) are mapped onto ~200+ physical registers, removing false dependencies.
- **Reorder buffer (ROB):** instructions can *execute* in any order but **retire** (become visible) strictly in program order. If instruction 1 faults, results of 3 and 4 are thrown away. This gives **precise exceptions**: the kernel always sees a clean state where "everything before the faulting instruction happened, nothing after did". Page faults (6.5) depend on this.

### Branch Prediction

A pipeline needs to know which instruction comes next *before* the current branch (`if`, loop end) is resolved. So the CPU **guesses**, using the history of each branch. Modern predictors are >95% accurate.

A wrong guess (a **misprediction**) means flushing the pipeline: ~15–20 wasted cycles.

The classic demonstration:

```c
// data[] has random values 0..255
for (int i = 0; i < N; i++)
    if (data[i] >= 128)      // random -> ~50% mispredicted
        sum += data[i];

// sort(data) first -> the branch becomes FFFF...FTTTT...T
// -> nearly perfectly predicted -> often several times faster
```

### Speculative Execution (Spectre / Meltdown)

Branch prediction + out-of-order execution = **speculative execution**: the CPU *runs* instructions down the guessed path before knowing if the guess was right. If wrong, the results are discarded architecturally (registers, memory)... but **not micro-architecturally**: data that was pulled into the cache stays there.

That leftover is a **side channel**: timing a memory access tells you whether it was cached (fast, ~1 ns) or not (slow, ~100 ns).

- **Meltdown (2018):** on many Intel CPUs, a user-mode load from a *kernel* address was speculatively executed before the permission check (the `U/S` bit, 6.3) killed it. The secret byte was used to touch one of 256 array entries; timing the array revealed the byte. **Fix: KPTI** (3.3): don't map kernel memory in user page tables at all.
- **Spectre (2018):** train the branch predictor so a *victim* (kernel, browser JS engine, another process) speculatively runs a bounds-checked access out of bounds. Fixes are many and partial: retpolines, `lfence` barriers, IBRS/IBPB microcode, array index masking.

```bash
$ grep . /sys/devices/system/cpu/vulnerabilities/*
/sys/devices/system/cpu/vulnerabilities/meltdown:Not affected
/sys/devices/system/cpu/vulnerabilities/spectre_v2:Mitigation: Enhanced / Automatic IBRS ...
```

> 🔗 **Connections:** Spectre and Meltdown tie together pipelining (2.2), caches (2.4), privilege levels (2.6), page tables (6.3) and system call cost (3.3, KPTI). They're the reason syscalls and context switches got measurably slower after 2018.

## 2.3 Core Functional Units

### Control Unit (CU)

- The "manager" of the CPU. Decodes instructions and generates the control signals that tell every other component what to do and when (which registers to read, whether to write to memory, what operation the ALU should perform, etc.).
- Doesn't do calculations itself; it orchestrates.

### Arithmetic Logic Unit (ALU)

- Does the actual math and logic: addition, subtraction, multiplication, AND/OR/NOT/XOR, comparisons, shifts.
- Modern CPUs often have multiple ALUs so they can execute several operations in the same cycle (superscalar, 2.2).

### Floating Point Unit (FPU)

- A specialized unit for floating-point math (scientific calculations, graphics, etc.), separate from the integer ALU because float math needs different circuitry.

### SIMD / Vector Units

**SIMD** = Single Instruction, Multiple Data. One instruction operates on a whole vector register:

```
scalar:   a0+b0                      (1 result per instruction)
SIMD:    [a0 a1 a2 a3 a4 a5 a6 a7]
       + [b0 b1 b2 b3 b4 b5 b6 b7]   (8 x 32-bit results per AVX2 instruction)
```

| ISA extension | Register width | Registers |
| :---- | :---- | :---- |
| SSE (x86) | 128 bits | `xmm0`–`xmm15` |
| AVX / AVX2 (x86) | 256 bits | `ymm0`–`ymm15` |
| AVX-512 (x86) | 512 bits | `zmm0`–`zmm31` |
| NEON / SVE (ARM) | 128 / up to 2048 bits | `v0`–`v31` / `z0`–`z31` |

Used by `memcpy`, `strlen`, compression, crypto, video codecs, and ML libraries. Compilers auto-vectorize simple loops at `-O2`/`-O3`.

> 🔗 **Connections:** Vector registers are part of a task's CPU state, so a context switch (3.5) must save and restore them too (using `XSAVE`, which can be kilobytes for AVX-512). This is why the kernel itself avoids floating point and SIMD except inside explicit `kernel_fpu_begin()`/`kernel_fpu_end()` sections.

### Registers

A register is an extremely fast storage location inside the CPU. Registers are small, high-speed storage located directly on the CPU chip, and they hold the "machine state" of a running process. Accessing a register takes about a single clock cycle.

- **General-Purpose Registers:** used for calculations and temporarily storing data, such as `rax` or `rbx` on x86-64 (`eax`/`ebx` are their 32-bit halves). x86-64 has 16: `rax rbx rcx rdx rsi rdi rbp rsp r8–r15`.
- **Program Counter (PC):** also known as the **Instruction Pointer (IP, `rip`)**, points to the next instruction to execute. (Important for context switching.)
- **Stack Pointer (SP, `rsp`):** tracks the current "top" of the run-time stack, which stores local variables and function call linkage information.
- **Frame Pointer (`rbp`):** used alongside the stack pointer to manage individual stack frames for function parameters and local variables.
- **Processor Status / Flags Register (`rflags`):** holds the CPU's status flags (zero, carry, overflow, interrupt-enable…). In textbook architectures it also holds the user/kernel mode bit. On x86 the mode is **not** stored here: it's the CPL in the `CS` register (see 2.6).
- **Control registers** (kernel only): e.g. **`CR3`** holds the physical address of the current process's top-level page table (6.3). Changing `CR3` = switching address spaces.

> 🔗 **Connections:** "Saving a process" for a context switch (3.5) literally means saving these registers into the task's kernel stack / `task_struct->thread`. The system call ABI (3.3) is defined in terms of registers: syscall number in `rax`, arguments in `rdi, rsi, rdx, r10, r8, r9`.

## 2.4 Memory Hierarchy and Caches

Since RAM is slow relative to the CPU, there's a tiered memory system built into or near the chip. Caches are implemented using **SRAM (Static RAM)** cells; main memory uses **DRAM**.

- **L1 cache**: smallest, fastest, closest to the core, split into L1i (instructions) and L1d (data). Usually per-core. ~32–64 KiB each.
- **L2 cache**: larger, slightly slower, often per-core or shared between a couple of cores. ~1–2 MiB.
- **L3 cache**: largest, shared across all cores, slower than L1/L2 but still much faster than RAM. Tens of MiB.

This hierarchy exists because there's a tradeoff between speed and size: you can't make something both huge and instant.

From fastest to slowest:

1. Registers
2. L1 cache
3. L2 cache
4. L3 cache
5. RAM
6. SSD
7. HDD

Core idea:

- Small, fast storage is close to the CPU.
- Large, cheap storage is farther away.
- This is why performance problems are often about **locality**, not raw instruction count (see the 2D array example in 1.8).
- **Latency** is how long one operation takes.
- **Throughput** is how much total work can be done over time.

```bash
$ lscpu | grep -i cache
L1d cache:   384 KiB (8 instances)
L1i cache:   256 KiB (8 instances)
L2 cache:    10 MiB (8 instances)
L3 cache:    24 MiB (1 instance)
```

### Latency Numbers Every Programmer Should Know

Approximate, order-of-magnitude values on a modern machine:

| Operation | Time | Scaled: if 1 cycle (0.3 ns) were 1 second |
| :---- | :---- | :---- |
| Register access / 1 CPU cycle | ~0.3 ns | 1 s |
| L1 cache hit | ~1 ns | 3 s |
| Branch mispredict | ~5 ns | 15 s |
| L2 cache hit | ~4 ns | 13 s |
| L3 cache hit | ~10–20 ns | 1 min |
| Main memory (DRAM) | ~80–100 ns | 5 min |
| System call (simple, e.g. `getppid`) | ~100–300 ns | 5–15 min |
| Context switch (direct cost) | ~1–5 µs | 1–5 hours |
| Minor page fault | ~1 µs | ~1 hour |
| NVMe SSD random read | ~20–100 µs | 1–4 days |
| Network round trip, same datacenter | ~500 µs | ~3 weeks |
| HDD seek / major page fault from disk | ~5–10 ms | ~1 year |
| Network round trip, continent to continent | ~100–150 ms | ~15 years |

The lesson: **a major page fault or a disk read is millions of instructions of lost time**. That's why the page cache (6.7) and readahead exist, and why a process waiting on disk is put to sleep (state `D`/`S`, 4.8) so others can use the CPU.

### TLB (Translation Lookaside Buffer)

Every memory access uses a *virtual* address that must be translated to a physical one (6.3). Doing a 4-level page-table walk (4 extra memory reads!) on **every** load would be absurdly slow. So the MMU caches recent translations in the **TLB**:

```
virtual page number --> [ TLB ] --hit (~1 cycle)--> physical frame number
                           |
                         miss
                           v
                 hardware page walk: CR3 -> PGD -> PUD -> PMD -> PTE  (~10s-100s of cycles)
                           |
                   PTE not present? --> PAGE FAULT -> kernel (6.5)
```

- Typical sizes: ~64 entries L1 dTLB, ~1,500–2,000 entries in the shared L2 TLB.
- 2,000 entries × 4 KiB pages = only **8 MiB of "reach"**. A program randomly touching 10 GiB will miss the TLB constantly. **Huge pages** (2 MiB) give 512× more reach per entry (6.9).
- On a **context switch** to a different process, the old translations are wrong. Without help, the TLB must be flushed. **PCID** tags entries with an address-space ID to avoid that (3.5).
- When the kernel changes a mapping (e.g. `munmap`), other cores might still cache it, so it sends them an inter-processor interrupt to invalidate: a **TLB shootdown**. This is one reason `munmap`/`mmap` are not free (6.6).

### Cache Coherence (MESI)

Each core has its own L1/L2. If core 0 and core 1 both cache address X and core 0 writes it, core 1 must not keep reading the stale value. Hardware keeps caches **coherent** with a protocol; the classic one is **MESI**, where each cache line is in one of four states:

| State | Meaning |
| :---- | :---- |
| **M**odified | Only this cache has it, and it's dirty (differs from RAM). |
| **E**xclusive | Only this cache has it, clean. Can be written without telling anyone (→ M). |
| **S**hared | Several caches may have it, clean. Writing requires invalidating the others first. |
| **I**nvalid | Not usable; must be fetched. |

```
core 0: read X   -> E
core 1: read X   -> both S
core 0: write X  -> core 0 broadcasts "invalidate X": core 0 = M, core 1 = I
core 1: read X   -> core 0 supplies the data, both S again
```

> 🔗 **Connections:** MESI is what makes atomic instructions and locks work across cores (5.3: the `lock` prefix takes a line in M state), and it's exactly the mechanism behind false sharing (1.8). Writes to *shared* data are expensive by nature: every write invalidates other cores.

### Cache Write Policies (Write-Back vs Write-Through)

What happens when the CPU writes to a cached line?

- **Write-through:** write to the cache **and** to the next level immediately. Simple, always consistent, but every write costs memory bandwidth.
- **Write-back:** write only to the cache and mark the line **dirty**. It's written to the next level only when evicted. Much faster; CPU caches today are write-back.

> 🔗 **Connections:** This exact idea repeats one level up in the OS. The **page cache** (6.7) is a write-back cache of the disk: `write()` only dirties a page in RAM; it's written to disk later by flusher threads, unless you call `fsync()`. The `D` (dirty) bit in page-table entries (6.3) plays the same role as the dirty bit on a cache line. Same idea, different layer.

## 2.5 Buses: How Things Talk to Each Other

Internal pathways that carry information:

- **Data bus**: carries the actual data being moved.
- **Address bus**: carries the memory address being read/written.
- **Control bus**: carries control signals (read/write, clock sync signals, interrupts).

The three-bus model is a conceptual/teaching model. Modern systems use point-to-point interconnects (PCIe, on-chip ring/mesh fabrics, Infinity Fabric) and an on-die memory controller.

### Modern Interconnects (PCIe, On-Chip Fabrics)

```
        +-------------------- CPU package --------------------+
        |  core  core  core  core      <- ring / mesh fabric  |
        |   L2    L2    L2    L2          connects cores,     |
        |  [========== shared L3 ==========]  L3 slices,       |
        |  memory controller --+  PCIe root complex --+       |
        +----------------------|----------------------|-------+
                               v                      v
                        DDR5 DIMMs (RAM)     PCIe lanes: NVMe SSD, GPU, NIC
```

- **PCIe** is a packet-based, point-to-point serial link. Each device gets dedicated lanes (x1, x4, x16). Devices expose their registers through **memory-mapped I/O (MMIO)**: the driver writes to special physical addresses that land in the device, not RAM.
- Multi-socket servers connect CPUs with links like Intel UPI or AMD Infinity Fabric. Memory attached to the other socket is slower to reach: that's **NUMA** (6.2).

```bash
$ lspci | head -3
00:00.0 Host bridge: Intel Corporation ...
01:00.0 Non-Volatile memory controller: Samsung NVMe SSD Controller ...
```

### DMA (Direct Memory Access)

Without DMA, the CPU would copy every byte from a disk or NIC itself. With DMA, the **device writes directly into RAM**:

```
1. driver: allocate a buffer in RAM, give its physical address to the device (MMIO write)
2. device: transfers the data into RAM by itself  (CPU is free to run other tasks)
3. device: raises an INTERRUPT: "done"            (3.4)
4. kernel: interrupt handler marks the I/O complete, wakes the waiting task  (3.6)
```

An **IOMMU** (Intel VT-d, AMD-Vi) translates and restricts device addresses, the same way the MMU does for processes, so a buggy or malicious device can't write anywhere in RAM.

> 🔗 **Connections:** DMA + interrupts is how a `read()` from disk actually completes (Part 11): the process sleeps, the disk DMAs into a page-cache page (6.7), an interrupt fires, and the scheduler wakes the process. DMA buffers need specific physical memory, which is why `ZONE_DMA`/`ZONE_DMA32` exist (6.2).

## 2.6 CPU Privilege Levels (Rings)

Why do we need privilege levels? One buggy program could crash the entire OS. One malicious program could steal everyone's data. That's why we have **CPU rings**.

```
   +-----------------------------------------+
   | Ring 3: user mode (bash, python, nginx) |
   |   +---------------------------------+   |
   |   | Ring 2 (unused on Linux)        |   |
   |   |   +-------------------------+   |   |
   |   |   | Ring 1 (unused on Linux)|   |   |
   |   |   |   +-----------------+   |   |   |
   |   |   |   | Ring 0: kernel  |   |   |   |
   |   |   |   +-----------------+   |   |   |
   |   |   +-------------------------+   |   |
   |   +---------------------------------+   |
   +-----------------------------------------+
```

- **Ring 0:** kernel mode
- **Rings 1 and 2:** rarely used. Historically designed for device drivers, giving them more privilege than user software but less than the core kernel. Most modern operating systems (Windows, Linux, macOS) ignore Rings 1 and 2 entirely and run everything in Ring 0 or Ring 3 for simplicity and performance.
- **Ring 3:** user mode, everyday software

What Ring 3 code **cannot** do (the CPU raises a `#GP` fault, which Linux turns into `SIGSEGV`):

- execute privileged instructions: `hlt`, `cli`/`sti` (disable/enable interrupts), `mov cr3`, `wrmsr`, `in`/`out` to I/O ports
- access pages whose page-table entry has the `U/S` bit set to "supervisor" (6.3)

### Tracking the Mode (CPL)

**How does the CPU know whether it's in Ring 0 or Ring 3?**

On x86, the CPU derives the **Current Privilege Level (CPL)** from the two least significant bits of the current Code Segment (`CS`) selector: `00` = Ring 0, `11` = Ring 3. During a controlled transition such as a `syscall`, interrupt, or exception, the processor loads a kernel code segment selector, changing the CPL from 3 to 0. When returning with `sysret` or `iret`, it restores the user code segment, returning the CPL to 3. User-mode code cannot arbitrarily modify `CS` or the CPL, which is why privilege enforcement is hardware-based.

The only doors from Ring 3 into Ring 0 are ones the kernel set up in advance:

- the `syscall` instruction → entry point stored in the `MSR_LSTAR` register (3.3)
- interrupts and exceptions → entry points in the Interrupt Descriptor Table (3.4)

### Hardware Virtualization (VMX Root / "Ring −1")

A virtual machine's guest kernel *also* wants to run in Ring 0. Intel VT-x / AMD-V solve this with a second, orthogonal dimension:

- **VMX root mode:** the hypervisor (KVM, Hyper-V, ESXi). Informally called "Ring −1".
- **VMX non-root mode:** the guest. It has its own Rings 0–3, so the guest kernel really runs in Ring 0, but certain events (privileged instructions, some I/O, EPT faults) cause a **VM exit** to the hypervisor.

Memory is virtualized with **EPT/NPT** (Extended/Nested Page Tables): guest-virtual → guest-physical (guest's page tables) → host-physical (hypervisor's EPT). A TLB miss in a VM can therefore need up to 24 memory reads, which is why huge pages matter even more in VMs.

On Linux, KVM is just a kernel module (`kvm.ko`, `kvm_intel.ko`) and each VM is a normal process (QEMU) whose threads are the virtual CPUs. **A VM is a process.**

> 🔗 **Connections:** Containers (Part 10) are *not* VMs: they don't use VMX at all. A container is a normal Ring 3 process with restricted views (namespaces). It shares the host kernel. A VM brings its own kernel.

## 2.7 Function Calls and the Stack

Registers (2.3) and the process stack (4.4) meet in one place: **function calls**. Understanding this makes stack traces, stack overflows and buffer-overflow exploits obvious.

### `call` and `ret`

- **`call f`**: pushes the **return address** (the address of the next instruction) onto the stack, then jumps to `f`.
- **`ret`**: pops that address off the stack into `rip` and jumps back.

The stack grows **down** (toward lower addresses), and `rsp` always points at its top.

### Stack Frames

Each active function call owns a **stack frame**: its return address, saved registers, and local variables.

```c
long add3(long a, long b, long c) {
    long tmp = a + b;
    return tmp + c;
}
int main(void) { return add3(1, 2, 3); }
```

```asm
main:
    mov  edx, 3          ; 3rd argument -> rdx
    mov  esi, 2          ; 2nd argument -> rsi
    mov  edi, 1          ; 1st argument -> rdi
    call add3            ; push return address, jump
    ret                  ; result is already in rax

add3:                    ; (unoptimized, with frame pointer)
    push rbp             ; save caller's frame pointer
    mov  rbp, rsp        ; my frame starts here
    sub  rsp, 16         ; room for locals
    ...
    mov  rax, ...        ; return value in rax
    leave                ; rsp = rbp; pop rbp
    ret                  ; pop return address -> rip
```

```
higher addresses
+---------------------------+
| main's frame              |
|   ...                     |
+---------------------------+
| return address into main  |  <- pushed by `call`
+---------------------------+
| saved rbp (main's)        |  <- rbp points here during add3
+---------------------------+
| tmp (local variable)      |
| ...                       |  <- rsp (top of stack)
+---------------------------+
lower addresses: the stack grows this way
```

Frames chained through saved `rbp` values are what a debugger walks to print a **backtrace**. (Optimized code often omits the frame pointer; debuggers then use DWARF unwind tables instead.)

### The x86-64 Calling Convention (System V ABI)

The rules every compiler on Linux follows, so that code from different compilers and libraries can call each other:

| Item | Convention |
| :---- | :---- |
| Integer/pointer arguments 1–6 | `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`; more go on the stack |
| Floating-point arguments | `xmm0`–`xmm7` |
| Return value | `rax` (and `rdx` for 128-bit), `xmm0` for floats |
| **Callee-saved** (a function must restore them) | `rbx`, `rbp`, `r12`–`r15`, `rsp` |
| **Caller-saved** (may be clobbered by any call) | `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11` |
| Stack alignment | `rsp` must be 16-byte aligned at each `call` (1.6, SIMD needs it) |
| Red zone | 128 bytes below `rsp` that a leaf function may use without moving `rsp` |

Compare with the **syscall** convention (3.3): same order, except the 4th argument goes in **`r10`** instead of `rcx`, because the `syscall` instruction overwrites `rcx` with the return address.

The **red zone** is also why the Linux kernel is compiled with `-mno-red-zone`: an interrupt (3.4) arriving in kernel mode pushes its frame directly below the current `rsp` and would overwrite it. (In user mode this is safe, because interrupts switch to the kernel stack.)

### Every Thread Has Two Stacks

- a **user stack** (8 MiB default for the main thread, 4.4; `mmap`ed for other threads, 5.1)
- a **kernel stack** (16 KiB on x86-64, pointed to by `task_struct->stack`, 4.2): when a thread makes a syscall or is interrupted, the CPU switches to it (3.3), so the kernel never trusts or uses user-space stack memory.

### Stack Buffer Overflows

Local arrays live in the frame **right below the saved registers and the return address**. Writing past the end of one walks **up** the stack and overwrites them:

```c
void greet(const char *name) {
    char buf[16];
    strcpy(buf, name);     // no length check: a 40-byte name overwrites saved rbp
}                          // and the RETURN ADDRESS -> `ret` jumps wherever the attacker wants
```

This was the classic exploit for decades. Today's layered defenses:

| Defense | How it helps | See |
| :---- | :---- | :---- |
| **Stack canaries** (`-fstack-protector`) | A random value placed before the return address; checked before `ret`; mismatch → `abort()` ("stack smashing detected") | |
| **NX bit** | The stack isn't executable, so injected code can't run there | 6.3 |
| **ASLR** | The attacker doesn't know where useful code is | 4.4 |
| **Shadow stacks** (Intel CET) | The CPU keeps a second, protected copy of return addresses and faults on mismatch | |
| Safe APIs / languages | `strncpy`/`snprintf`, bounds-checked languages (Rust, Go) | |

Infinite recursion is the other way to break the stack: each call adds a frame until `rsp` crosses into the guard gap → `SIGSEGV` (6.5).

---

# 3. Operating System Fundamentals

> 🧪 **Interactive visuals:** [Part 3, visualized](visuals/part3-os.html): an interactive page with a demo for each section.


## 3.1 Kernel

At its core, a kernel is the central, foundational component of an operating system. It acts as a bridge between the software applications you run and the physical hardware (CPU, memory, disk, devices).

### Core Responsibilities

#### Process Management

A **process** is an instance of an executing program. The kernel handles starting, pausing, resuming, scheduling and terminating processes.

Each process uses the CPU for a small fraction of a second (a **time slice**), then pauses, and another process gets the CPU, and so on. The time slice is so small that the system appears to run many processes at the same time (**multitasking**). The act of one process giving up the CPU to another is a **context switch** (see 3.5).

#### Memory Management

The kernel is responsible for allocating, protecting, mapping, and reclaiming memory for processes. It also owns a private area of memory that user processes can't access.

Modern CPUs include a **Memory Management Unit (MMU)**: a hardware component that translates the virtual addresses generated by a process into physical addresses in RAM. When a process reads from or writes to memory, it always uses virtual addresses. The MMU performs the translation by consulting the process's memory address map, known as the **page table** (see Part 6). The kernel must switch that map on every context switch between processes.

#### Device Drivers and Management

Devices are accessible only in kernel mode to prevent unauthorized or improper access by user-space applications, ensuring system stability and security. A driver is kernel code that knows how to talk to one kind of hardware; to user space, devices are presented as files in `/dev` (7.12).

#### System Calls

System calls provide the controlled interface through which user-space programs request services from the kernel (see 3.3).

### Monolithic vs Microkernel

Where do drivers, filesystems and the network stack live?

```
      MONOLITHIC (Linux)                         MICROKERNEL (seL4, QNX, MINIX 3)
+-------------------------------+         +-------+ +-------+ +--------+ +-------+
|  user apps                    |  ring 3 |  app  | |  FS   | | driver | |  net  |  <- servers are
+-------------------------------+         +---+---+ +---+---+ +---+----+ +---+---+     user processes
|  scheduler, MM, VFS, ext4,    |  ring 0     |   messages (IPC)  |          |
|  TCP/IP, all drivers ...      |         +---v-------------------v----------v---+
|  ONE address space            |         | tiny kernel: scheduling, IPC, memory |  ring 0
+-------------------------------+         +--------------------------------------+
```

| | Monolithic | Microkernel |
| :---- | :---- | :---- |
| Speed | Fast: a function call between subsystems | Slower: every service request is an IPC message + context switch |
| Robustness | A driver bug can crash the whole kernel | A crashed driver is just a process; restart it |
| Examples | Linux, FreeBSD | seL4, QNX, MINIX 3 |

macOS/iOS (**XNU**) and Windows NT are usually called **hybrid**: microkernel-inspired design, but most services run in kernel space for speed.

Linux is monolithic **but modular**: code can be loaded and unloaded at runtime as modules.

### Kernel Modules

A kernel module (`.ko` file) is a piece of kernel code (usually a driver or filesystem) that can be loaded into the **running** kernel. Once loaded, it runs in Ring 0 with full privileges, in the same address space as the rest of the kernel.

```bash
$ lsmod | head -3               # what's loaded (reads /proc/modules)
Module                  Size  Used by
nvme                   57344  3
ext4                 1003520  2

$ modinfo ext4 | head -2        # metadata
filename:  /lib/modules/6.8.0-45-generic/kernel/fs/ext4/ext4.ko.zst
license:   GPL

$ sudo modprobe vfat            # load + its dependencies (from modules.dep)
$ sudo modprobe -r vfat         # unload
```

> 🔗 **Connections:** When you plug in a USB device, the kernel sends a **uevent**; `udevd` reads the device's `MODALIAS` and runs `modprobe` to load the right driver automatically (7.12). The **initramfs** (3.7) exists largely to hold the modules needed to mount the root filesystem. Loading a module requires the `CAP_SYS_MODULE` capability (10.4), which containers normally drop.

## 3.2 User Space vs Kernel Space

- **User mode:** restricted (Ring 3)
- **Kernel mode:** privileged (Ring 0)

**User space** is the portion of a process's virtual address space where user applications execute. It contains the program's code, data, heap, stack, and shared libraries. User-space programs run in user mode and cannot directly access kernel memory or execute privileged instructions.

The term is also used for the memory the kernel gives to user processes as a whole: "user space" = the collection of all running processes' memory, as opposed to the kernel's own memory.

Why it matters:

- System calls, interrupts, faults and traps transition from user mode into kernel mode.
- Security depends on this boundary being enforced.

## 3.3 System Calls

This is the bridge between user mode and kernel mode. A **system call** is:

> A controlled request for the kernel to perform a privileged operation on behalf of a user process.

It is the **only legal way for user space to request a service from the kernel**. User space **cannot directly access hardware or kernel memory**, so system calls act as **controlled entry points** into the kernel.

Categories and examples:

- **Process control:** `fork()`, `execve()`, `exit()`, `wait()`
- **File management:** `open()`, `read()`, `write()`, `close()`, `rename()`
- **Memory management:** `brk()`, `mmap()`, `munmap()`
- **Signals:** `kill()`
- **Networking:** `socket()`, `connect()`
- **IPC:** `pipe()`
- **Other:** timing, scheduling

`printf()` is **not** a system call. `malloc()` is usually **not** a system call. Many library functions make a system call only when necessary:

- `printf()` formats text into a stdio buffer in user space and calls `write()` when the buffer is flushed (7.6).
- `malloc()` is a memory manager built on top of `brk()` / `mmap()`; it only asks the kernel for more memory when its own pool runs out (6.6).

Typical path:

```
application -> libc -> syscall -> kernel -> hardware
```

Why it matters:

- `libc` is usually a wrapper, not the actual OS implementation.
- Every `read`, `write`, `fork`, or `open` eventually crosses a privilege boundary.

You can watch it happen with `strace`:

```bash
$ strace -e trace=openat,read,write,close cat hello.txt
openat(AT_FDCWD, "hello.txt", O_RDONLY) = 3
read(3, "Hello, World!\n", 131072)      = 14
write(1, "Hello, World!\n", 14)         = 14
read(3, "", 131072)                     = 0      <- 0 bytes = end of file
close(3)                                = 0
```

*Linux manual pages are divided into sections:*

| Section | Contents |
| :---- | :---- |
| 1 | User commands (`ls`, `cat`, `grep`) |
| 2 | **System calls** ⭐ |
| 3 | Library functions (`printf`, `malloc`) |
| 5 | File formats (`/etc/passwd`, `proc`) |
| 7 | Overviews and conventions (`signal`, `epoll`) |
| 8 | System administration commands (`mount`, `ip`) |

So `man 2 write` is the syscall and `man 3 printf` is the library function.

### How a System Call Enters the Kernel (`syscall` Instruction vs Legacy `int 0x80`)

On x86-64, `write(1, "hi\n", 3)` in libc boils down to:

```asm
mov  rax, 1        ; syscall number: 1 = write  (see arch/x86/entry/syscalls/syscall_64.tbl)
mov  rdi, 1        ; arg1: fd
lea  rsi, [msg]    ; arg2: buffer
mov  rdx, 3        ; arg3: count
syscall            ; -> Ring 0
; on return: rax = bytes written, or -errno (e.g. -9 = -EBADF)
```

What the `syscall` instruction does in hardware:

1. Saves the user `rip` into `rcx` and `rflags` into `r11`.
2. Loads the kernel entry address from the `MSR_LSTAR` register and switches CPL to 0 (2.6).
3. Jumps to `entry_SYSCALL_64`, which switches to the thread's **kernel stack**, saves all user registers (a `struct pt_regs`), and calls `do_syscall_64()`.
4. `do_syscall_64()` dispatches on the number in `rax` → `__x64_sys_write()` → `ksys_write()` → VFS (7.4). [Before Linux 6.9 this was a lookup in the `sys_call_table[rax]` array of function pointers; it's now a `switch` statement in `x64_sys_call()`, a hardening against Spectre-style branch target attacks (2.2).]
5. On the way back, the kernel checks for pending work (pending signals → deliver them (8.4), "need reschedule" flag → call the scheduler (3.6)), then returns with `sysret`.

ABI summary (x86-64): number in `rax`; args in `rdi, rsi, rdx, r10, r8, r9` (note `r10`, not `rcx`, because `syscall` clobbers `rcx`); result in `rax`. Negative values between −4095 and −1 are errors; libc turns them into `-1` and sets `errno`.

The legacy 32-bit way was the software interrupt **`int 0x80`** (vector 128 in the IDT, 3.4) with the number in `eax` and args in `ebx, ecx, edx, esi, edi, ebp`. It still works on x86-64 for 32-bit compatibility, but it's slower. 32-bit x86 also had `sysenter`.

### vDSO (System Calls Without a Mode Switch)

Some "system calls" are called so often and need so little privilege that crossing into the kernel is wasteful. The classic one is reading the time (`clock_gettime`, `gettimeofday`, `time`, `getcpu`).

The kernel maps a tiny shared library, the **vDSO** (virtual Dynamic Shared Object), into every process. It contains user-space implementations of these calls that read time data from a page the kernel keeps updated. No `syscall` instruction, no mode switch: ~20 ns instead of ~200 ns.

```bash
$ cat /proc/self/maps | grep -E 'vdso|vvar'
7ffd8a5f1000-7ffd8a5f5000 r--p 00000000 00:00 0   [vvar]   <- data page written by the kernel
7ffd8a5f5000-7ffd8a5f7000 r-xp 00000000 00:00 0   [vdso]   <- code
```

That's why `strace` does *not* show `clock_gettime` for most programs: it never enters the kernel.

### KPTI and the Cost of Crossing the Boundary

Before 2018, the kernel was mapped (with the supervisor-only `U/S` bit) in the upper half of every process's page tables, so a system call didn't need to switch page tables. **Meltdown** (2.2) showed that user code could speculatively read those kernel pages anyway.

**KPTI** (Kernel Page-Table Isolation) gives each process **two** sets of page tables:

```
user mode:   page tables map user memory + a tiny trampoline of kernel entry code
kernel mode: page tables map user memory + the whole kernel

every syscall / interrupt:   switch CR3 to kernel tables   ...work...   switch CR3 back
```

Switching `CR3` would normally flush the TLB on each syscall, a large cost for syscall-heavy workloads (databases, network servers). **PCID** (2.4, 3.5) tags TLB entries so the two tables can coexist in the TLB. CPUs that aren't vulnerable to Meltdown don't need KPTI at all (`/sys/devices/system/cpu/vulnerabilities/meltdown`).

Takeaway: **system calls are cheap, but not free.** Batching (one big `read` instead of thousands of small ones), stdio buffering (7.6), the vDSO, `epoll` (7.8) and `io_uring` (7.9) all exist to cross the boundary less often.

## 3.4 Interrupts and Exceptions

- **Interrupt**: asynchronous signal from hardware or a timer (e.g., timer interrupt, network packet arrived, key pressed). It has nothing to do with the instruction currently running.
- **Exception**: synchronous event caused by the current instruction (e.g., page fault, divide-by-zero).

Why it matters:

- Interrupts let the kernel preempt work and handle devices. The **timer interrupt** (every 1–4 ms, `CONFIG_HZ`) is what lets the scheduler take the CPU from a task that never makes a syscall (illusion 1).
- Exceptions are how the OS handles invalid memory access, faults, and traps.

### Faults vs Traps vs Aborts

x86 classifies exceptions by **where execution resumes**:

| Kind | When reported | Saved `rip` points to | Example | Typical outcome |
| :---- | :---- | :---- | :---- | :---- |
| **Fault** | Before the instruction completes | The **faulting** instruction (it will be retried) | Page fault `#PF`, `#GP`, divide error `#DE` | Kernel fixes it (loads the page) and re-runs the instruction, or sends a signal |
| **Trap** | After the instruction completes | The **next** instruction | Breakpoint `int3`, `int 0x80`, single-step | Debugger stops; execution continues after |
| **Abort** | Severe, imprecise | Unreliable | Double fault, machine check (hardware error) | Kernel panic or killing the process |

The "fault → fix → retry" pattern is the foundation of virtual memory: demand paging and copy-on-write (6.5) are just page **faults** the kernel resolves and then transparently restarts.

How exceptions reach your program as signals:

| CPU exception | Linux signal |
| :---- | :---- |
| `#PF` on an invalid address | `SIGSEGV` |
| `#DE` divide by zero (integer) | `SIGFPE` |
| `#UD` invalid opcode | `SIGILL` |
| `#BP` breakpoint (`int3`) | `SIGTRAP` |
| `#AC` alignment check (only when alignment checking is enabled) | `SIGBUS` |
| `#PF` on an `mmap`ed file page beyond the end of the file (6.6) | `SIGBUS` |

### Interrupt Descriptor Table (IDT)

How does the CPU know which kernel function to run for vector 14 (page fault)? The kernel fills an **IDT**, a table of 256 entries, and loads its address with `lidt` at boot.

```
vector  0   #DE divide error        -> asm_exc_divide_error
vector  3   #BP breakpoint          -> asm_exc_int3
vector  6   #UD invalid opcode
vector 13   #GP general protection  -> asm_exc_general_protection
vector 14   #PF page fault          -> asm_exc_page_fault  -> handle_mm_fault() (6.5)
vectors 0-31    reserved for CPU exceptions
vectors 32-255  device interrupts (via the APIC), IPIs, and 0x80 = legacy syscall
```

On an interrupt or exception the CPU: switches to Ring 0 if needed, switches to the kernel stack, pushes `ss, rsp, rflags, cs, rip` (and an error code for some exceptions), and jumps to the IDT entry. `iret` undoes it.

```bash
$ cat /proc/interrupts | head -4
           CPU0       CPU1
  0:         38          0   IO-APIC    2-edge      timer
 16:     152839          0   PCI-MSI    nvme0q1
LOC:   48213091   47922310   Local timer interrupts
```

### Hard IRQs vs Softirqs

An interrupt handler runs while **interrupting** whatever was running, often with further interrupts disabled. It must be very short. So Linux splits interrupt work in two halves:

```
NIC raises IRQ
   |
   v
TOP HALF (hard IRQ handler)        - runs immediately, interrupts disabled-ish, microseconds
   - acknowledge the device
   - grab the packet descriptor
   - schedule the bottom half
   |
   v
BOTTOM HALF (deferred)             - runs soon after, interrupts enabled
   - softirq   NET_RX: run the TCP/IP stack on the packet
   - tasklet / workqueue / threaded IRQ for drivers
   - under heavy load: handed to the ksoftirqd/N kernel thread so user tasks still get CPU
```

```bash
$ cat /proc/softirqs | head -4
                CPU0       CPU1
      HI:          0          0
   TIMER:    1928374    1877766
  NET_RX:     412283     398112
```

> 🔗 **Connections:** If you see `ksoftirqd` eating CPU in `top`, the machine is overwhelmed with network or block I/O interrupts. Interrupts are also the second way (besides syscalls) the CPU enters the kernel, so they cause **mode switches** (3.5) and are the moment the scheduler gets a chance to **preempt** (3.6).

## 3.5 Mode Switch vs Context Switch

### Mode Switch

A **mode switch** is the transition of CPU execution between **user mode** and **kernel mode**, allowing the operating system to safely perform privileged operations on behalf of an application. It's the **same process**; only the privilege level changes.

Common triggers:

- A **system call** (e.g., `read()`, `write()`, `open()`)
- A **hardware interrupt** (e.g., timer or network packet)
- A **CPU exception** (e.g., page fault or divide-by-zero)

After the kernel completes the work, execution returns to user mode.

### Context Switch

A **context switch** is when the kernel saves the execution context of one task (process or thread) and restores another's, so the CPU continues executing the new task.

Usually involves saving/restoring:

- registers
- program counter
- stack pointer
- FPU/SIMD state (2.3)
- page table state (`CR3`), only when switching to a thread of a *different* process
- kernel bookkeeping

In Linux this is `schedule()` → `context_switch()` → `switch_mm()` (address space) + `switch_to()` (registers and kernel stack).

Why it matters:

- Context switches are not free.
- Too many switches hurt performance.

**Note:** A **mode switch** changes the CPU's privilege level, whereas a **context switch** changes which task is executing. A mode switch does **not** necessarily involve a context switch (a context switch, however, always happens in kernel mode).

```
time --->
task A (user) --syscall--> [kernel, on behalf of A] --return--> task A (user)      mode switches only
task A (user) --read() blocks--> [kernel: schedule()] --> task B (user)            mode switch + context switch
```

### Voluntary vs Involuntary Context Switches

- **Voluntary:** the task **gives up the CPU** itself because it has to wait: it blocks on I/O, calls `sleep()`, waits on a lock or a pipe. Example: `read()` from disk → task goes into `TASK_INTERRUPTIBLE` (`S`) or `TASK_UNINTERRUPTIBLE` (`D`) → the scheduler picks someone else.
- **Involuntary (preemption):** the task **did not** want to stop, but the kernel said "your turn is over": its time slice ran out (detected on a timer interrupt), or a higher-priority task woke up.

```bash
$ grep ctxt /proc/$$/status
voluntary_ctxt_switches:        142     <- mostly waiting for your keystrokes
nonvoluntary_ctxt_switches:     3
```

Lots of *voluntary* switches = I/O-bound or lock contention. Lots of *involuntary* = CPU-bound tasks competing for cores.

### Indirect Costs (TLB Flush, Cache Pollution, PCID)

The direct cost (saving registers, running the scheduler) is ~1 µs. The **indirect** costs are often bigger:

- **TLB flush:** the new process has a different address space; old translations are useless. Without PCID, writing `CR3` flushes the TLB, and the new task starts with a burst of TLB misses (page walks).
- **Cache pollution:** the new task's data isn't in L1/L2. The previous task's data gets evicted. When it comes back, it's cold too.
- **Branch predictor state** is mostly trained for the other task (and is deliberately flushed on some switches as a Spectre mitigation).

**PCID** (Process-Context Identifier, 12 bits in `CR3`) tags each TLB entry with an address-space ID, so switching `CR3` doesn't require a flush: entries for other PCIDs simply don't match.

Switching between **threads of the same process** skips `switch_mm` entirely: same page tables, no TLB issue. That's why threads are "lighter" (5.1).

## 3.6 Scheduler

The scheduler decides which runnable task gets CPU time. It is the decision maker: which process/thread gets the CPU next.

Key ideas:

- fairness
- responsiveness
- priorities
- preemption

For normal tasks, Linux uses **EEVDF** (Earliest Eligible Virtual Deadline First) [before Linux 6.6 (2023): **CFS**, Completely Fair Scheduler]. It also supports real-time policies (`SCHED_FIFO`, `SCHED_RR`) and `SCHED_DEADLINE`.

### CFS and Virtual Runtime (`vruntime`)

CFS's core idea (which EEVDF keeps) is **virtual runtime**:

- Each task has a `vruntime` (in `task_struct->se`, the "scheduling entity").
- `vruntime` grows as the task runs, **scaled by its weight**: a nice −5 task's vruntime grows slower than a nice 0 task's, so it gets more real CPU time for the same vruntime.
- The lower the `vruntime`, the less CPU time it has gotten → **it should run sooner**.
- Runnable tasks are kept in a **red-black tree** (self-balancing binary search tree) ordered by vruntime; picking the leftmost node (the smallest vruntime) is O(1) with a cached pointer, inserting is O(log n).

```
                 red-black tree of runnable tasks, keyed by vruntime
                              [ 300 ]
                             /       \
                        [ 120 ]     [ 450 ]
                        /    \
         leftmost ->  [ 80 ]  [ 200 ]

pick_next: task with vruntime 80 runs, its vruntime grows to ~210,
           it's reinserted, and now the task with 120 is the leftmost
```

**What EEVDF adds:** each task also has a **virtual deadline** (eligible time + requested slice / weight). Among tasks that are *eligible* (haven't received more than their fair share, i.e. "lag" ≥ 0), it picks the one with the **earliest virtual deadline**. This gives latency-sensitive tasks (short slices) better responsiveness than CFS's heuristics did, without hurting fairness.

**What happens if all processes are I/O-bound?** They spend most of their time sleeping (`S`/`D`), so the run queue is often empty and the CPU runs the idle task (2.1). When one wakes up (an interrupt signals I/O completion), its vruntime is low compared to CPU hogs (it was sleeping), so it gets the CPU quickly. That's how CFS/EEVDF naturally favor interactive and I/O-bound tasks without special rules. (A sleeper's vruntime is clamped on wakeup so it can't hoard an unfair burst.)

### Scheduling Policies (`SCHED_OTHER`, `SCHED_BATCH`, `SCHED_IDLE`, `SCHED_FIFO`, `SCHED_RR`, `SCHED_DEADLINE`)

Scheduling classes are checked **in priority order**; a lower class runs only if no higher class has a runnable task:

| Class (highest first) | Policy | Behavior |
| :---- | :---- | :---- |
| stop | (internal) | CPU stop/migration tasks |
| deadline | `SCHED_DEADLINE` | Task declares runtime/period/deadline (e.g. 2 ms every 10 ms); kernel guarantees it (EDF + admission control) |
| real-time | `SCHED_FIFO` | Fixed priority 1–99; runs until it blocks or yields. A buggy `SCHED_FIFO` loop can starve everything (RT throttling keeps 5% for others by default). |
| real-time | `SCHED_RR` | Like FIFO, but round-robins with equal-priority tasks every time slice |
| fair | `SCHED_OTHER` (`SCHED_NORMAL`) | The default: EEVDF/CFS, weighted by nice |
| fair | `SCHED_BATCH` | Like OTHER but assumed CPU-bound: fewer preemptions, better cache use |
| idle | `SCHED_IDLE` | Runs only when nothing else wants the CPU |

```bash
$ chrt -p $$                          # show the policy of the current shell
pid 4120's current scheduling policy: SCHED_OTHER
$ sudo chrt -f 50 ./audio_engine       # run with SCHED_FIFO priority 50
$ nice -n 10 make -j8                  # SCHED_OTHER with nice 10 (see 4.6)
```

### Run Queues and Per-CPU Scheduling

There is no single global queue: **each CPU has its own run queue** (`struct rq`), each with its own red-black tree. This avoids lock contention and keeps tasks on CPUs where their data is cached.

- **Load balancing:** periodically (and when a CPU goes idle) the scheduler pulls tasks from busy CPUs, respecting the topology: prefer moving between hyperthreads of one core, then cores sharing L3, and only reluctantly across NUMA nodes (6.2).
- **CPU affinity:** you can pin tasks: `taskset -c 0,1 ./prog`, or `sched_setaffinity()`. cgroup `cpuset` (10.2) does the same for groups.
- **Wakeup placement:** a woken task is placed on a CPU close to its waker or where it last ran (warm cache).

### Kernel Preemption Models

The scheduler can always preempt a task running **user** code (on a timer interrupt, 3.4). But can it preempt a task that is in the middle of **kernel** code, say a long syscall? That's a build-time (and, with `PREEMPT_DYNAMIC`, boot-time) choice:

| Model | Kernel code can be preempted... | Trade-off | Typical use |
| :---- | :---- | :---- | :---- |
| `PREEMPT_NONE` | Only where it explicitly yields or blocks | Best throughput, worst latency | Servers |
| `PREEMPT_VOLUNTARY` | Also at extra explicit "may reschedule" points (`cond_resched()`) | Balanced | Many distro defaults |
| `PREEMPT` (full) | Almost anywhere, except while holding spinlocks or with interrupts disabled (5.3) | Lower latency, a bit less throughput | Desktops, low-latency |
| `PREEMPT_LAZY` (Linux 6.13) | Like full, but defers preemption of normal tasks to the next tick | Full-preemption latency for RT tasks, fewer switches otherwise | Newer kernels |
| `PREEMPT_RT` (mainline since **6.12**, 2024) | Even inside most critical sections: spinlocks become sleeping `rt_mutex`es (with priority inheritance, 5.2), interrupt handlers run as threads | **Bounded, predictable** latency | Audio, robotics, industrial control |

```bash
$ uname -v
#45-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 30 12:02:04 UTC 2024
$ cat /sys/kernel/debug/sched/preempt      # (as root, debugfs) the active model, e.g. "none (voluntary) full"
```

Real-time is about **predictability**, not speed: an RT kernel guarantees that a high-priority task starts within, say, 50 µs, every time.

### sched_ext: Schedulers Written in BPF

Since **Linux 6.12**, `sched_ext` lets you load a **scheduling policy written as a BPF program** at runtime, without rebuilding or rebooting the kernel. Tasks using the `SCHED_EXT` policy (or all normal tasks, if the loaded scheduler requests it) are scheduled by that program.

- If the BPF scheduler misbehaves (for example, leaves a runnable task waiting too long), the kernel **automatically unloads it** and falls back to the fair scheduler. A bad experiment can't hang the machine.
- Examples from the `scx` project: `scx_lavd` (latency-focused, developed with gaming on the Steam Deck in mind), `scx_rusty` (load balancing in user space), and `scx_bpfland`.
- It lets companies experiment with scheduling policies for their workloads in days instead of years.

### Load Average

```bash
$ uptime
 17:42:11 up 12 days,  3:04,  2 users,  load average: 3.12, 2.80, 1.95
                                                       1min  5min  15min
```

Load average is an **exponentially-damped moving average of the number of tasks that are runnable (`R`) or in uninterruptible sleep (`D`)**.

- On a 4-core machine, a load of 4.0 means "on average, exactly enough work to keep all cores busy".
- Linux is unusual in counting `D` tasks: a high load with idle CPUs often means processes are **stuck waiting on disk or NFS**, not burning CPU.
- For CPU vs I/O pressure split out properly, read **PSI** (Pressure Stall Information): `/proc/pressure/cpu`, `/proc/pressure/io`, `/proc/pressure/memory`.

> 🔗 **Connections:** The scheduler's inputs live in `task_struct` (policy, prio, `se.vruntime`), its trigger points are timer interrupts and wakeups (3.4), its output is a context switch (3.5), and cgroups (10.2) let you nest fair-share groups so one container can't starve another.

## 3.7 Boot Process: From Power-On to a Login Prompt

Now that we know what a kernel is, here is how it comes to life. Each stage's only job is to load the next, bigger stage. (Further reading: [the Linux boot process, detailed (PDF)](http://www.scott-a-s.com/files/linux_boot.pdf).)

```
POWER ON
   |
   v
FIRMWARE (BIOS or UEFI) -- POST: test CPU, RAM, devices
   |
   v
BOOT LOADER
   BIOS: MBR (first 512 bytes of disk) -> GRUB stage 1 -> GRUB core
   UEFI: reads GPT, runs \EFI\...\grubx64.efi (or shim, systemd-boot) from the EFI System Partition
   |
   v
KERNEL (vmlinuz) + INITRAMFS loaded into RAM, kernel command line passed
   |
   v
KERNEL INIT: decompress, set up page tables, IDT, memory zones, scheduler, drivers
   |
   v
INITRAMFS /init: load storage drivers, find & mount the real root fs (maybe decrypt LVM/LUKS)
   |  switch_root
   v
/sbin/init = systemd  (PID 1, first USER-SPACE process)
   |
   v
SERVICES: udev, journald, networking, sshd ...  and getty on each TTY
   |
   v
getty -> login -> your shell (bash)
```

### 1. Firmware and POST

When power is first applied, the CPU starts executing firmware from a flash chip on the motherboard (historically ROM/EEPROM). This firmware is the **BIOS** (Basic Input/Output System) on older machines or **UEFI** on modern ones.

The first thing it runs is **POST (Power-On Self Test)**, which ensures the hardware functions correctly:

1. **CPU:** runs basic instructions and checks registers. If this fails, the system halts.
2. **RAM:** writes patterns (all 1s, all 0s, alternating 1s and 0s) to memory and reads them back to verify data integrity. Failures are reported with beep codes or error messages.
3. **Firmware integrity:** calculates a checksum of the firmware code.
4. **Video:** initializes the graphics card so errors can be displayed.
5. **Storage and I/O devices:** detects disks, USB, network devices, and reads their metadata.

Then it builds tables describing the hardware for the OS (memory map via `e820`/UEFI memory map, ACPI tables) and finds a boot device according to the configured boot order.

### 2. Boot Loader: MBR (BIOS) vs GPT/ESP (UEFI)

**BIOS + MBR (legacy):** the BIOS reads the first sector of the boot disk, the **Master Boot Record** (512 bytes), into memory and jumps to it.

```
MBR (512 bytes)
+--------------------------------+------------------------------+-----------+
| bootstrap code (446 bytes)     | partition table (4 x 16 B)   | 0x55 0xAA |
| = GRUB stage 1                 | max 4 primary, max 2 TiB     | signature |
+--------------------------------+------------------------------+-----------+
```

446 bytes is too small for a real boot loader, so stage 1 loads GRUB's `core.img` (stored just after the MBR), which can read filesystems.

**UEFI + GPT (modern):** UEFI understands the **GPT** partition table (128+ partitions, huge disks) and FAT32 filesystems. It loads an EFI program from the **EFI System Partition** (ESP, usually mounted at `/boot/efi`), e.g. `shimx64.efi` → `grubx64.efi`. Secure Boot verifies each stage's signature.

**GRUB** (GRand Unified Bootloader) then shows the menu (`/boot/grub/grub.cfg`), lets you choose a kernel, and loads two files into RAM:

- the **kernel image**, e.g. `/boot/vmlinuz-6.8.0-45-generic`
- the **initramfs** (initial RAM filesystem), e.g. `/boot/initrd.img-6.8.0-45-generic`

It passes the **kernel command line** and jumps to the kernel:

```bash
$ cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-6.8.0-45-generic root=UUID=3f1e...a9 ro quiet splash
```

### 3. Kernel Initialization

The kernel decompresses itself and runs `start_kernel()`. It:

- sets up its own page tables and enables virtual memory (Part 6)
- builds `struct page` for every RAM frame and the memory zones (6.2)
- installs the IDT (3.4), initializes the scheduler (3.6), timers, and interrupt controllers
- initializes built-in drivers
- creates the first tasks: **PID 0** (the idle task, this boot thread itself), **PID 1** (`kernel_init`), and **PID 2** (`kthreadd`, the parent of all kernel threads)

### 4. initramfs: The Temporary Root

Problem: to mount the real root filesystem, the kernel may need drivers (NVMe, RAID, LVM, LUKS encryption, a network filesystem) that are modules **stored on that same root filesystem**. Chicken and egg.

Solution: the **initramfs**, a compressed `cpio` archive that the kernel unpacks into a RAM-based filesystem (tmpfs) and uses as a **temporary root**. It contains just enough: modules, `udev`, and a script/systemd called `/init`. It:

1. loads the needed storage drivers (3.1 modules)
2. waits for the root device to appear, unlocks it if encrypted
3. mounts the real root filesystem (read-only at first)
4. **`switch_root`**: makes the real root `/`, frees the initramfs, and `exec`s the real `/sbin/init`

```bash
$ lsinitramfs /boot/initrd.img-$(uname -r) | grep nvme     # Debian/Ubuntu
usr/lib/modules/6.8.0-45-generic/kernel/drivers/nvme/host/nvme.ko.zst
```

### 5. PID 1: init / systemd

`/sbin/init` is the **first user-space program**, PID 1. This point is where **user space** starts. On most distributions today it's **systemd** (older: SysV init with *runlevels*; systemd uses *targets*, e.g. `multi-user.target` ≈ runlevel 3, `graphical.target` ≈ runlevel 5).

PID 1 handles the overall initialization of the user-space environment:

- mounts the remaining filesystems (`/etc/fstab`), `/proc`, `/sys`, `/dev` (devtmpfs)
- starts `udevd` (7.12), logging (`journald`), networking, `sshd`, cron...
- starts a **getty** on each virtual terminal
- **adopts orphans** and reaps zombies for the rest of the system's life (4.5)
- is special: the kernel ignores signals **sent to PID 1 from user space** unless PID 1 installed a handler for them (even `SIGKILL`), and if PID 1 exits or crashes (a fault-generated `SIGSEGV` still kills it), the kernel **panics**

```bash
$ systemd-analyze                # how long each boot stage took
Startup finished in 4.1s (firmware) + 2.3s (loader) + 1.9s (kernel) + 3.2s (userspace)
$ systemd-analyze blame | head -3
$ ps -o pid,ppid,cmd -p 1,2
  PID  PPID CMD
    1     0 /sbin/init splash
    2     0 [kthreadd]
```

### 6. getty and Login

`agetty` opens a terminal device (`/dev/tty1`), makes it the process's stdin/stdout/stderr (fds 0, 1, 2), prints `login:`, and `exec`s `login` with what you typed. `login` checks your password (PAM, `/etc/shadow`), then forks; the child sets your UID/GID (7.11), `chdir`s to your home, and `exec`s your shell from `/etc/passwd`. Over SSH, `sshd` does the equivalent with a pseudo-terminal (`/dev/pts/N`, 4.9).

Notice that **getty → login is the same process** (same PID) transformed by `exec`, and the shell inherits the terminal as fds 0/1/2 through fork + exec (4.9). That's where every shell's stdin/stdout/stderr come from.

Why does Linux start several virtual terminals (`/dev/tty1`–`tty6`, switch with `Ctrl+Alt+F1..F6`)? So that one user can run a program on tty1, another can log in on tty2, and an admin can switch to tty3 to kill a stuck process even if the GUI (on tty1 or tty7, depending on the distro) is frozen.

> 🔗 **Connections:** Boot is a tour of everything: firmware tests the hardware of Part 2; the kernel builds `struct page` (6.2) and the IDT (3.4); initramfs exists because of kernel modules (3.1) and mounts (7.13); PID 1's special treatment explains orphans (4.5), zombies (4.9) and why containers need an init process (10.3); getty/login is fork + exec (4.9) plus credentials (7.11).

## 3.8 Time, Clocks and Timers

The kernel needs time for everything: scheduling slices (3.6), timeouts, `sleep()`, file timestamps (7.3), TCP retransmissions. Where does it come from?

### Hardware Clocks

| Source | What it is | Used for |
| :---- | :---- | :---- |
| **RTC** (Real-Time Clock) | Battery-powered clock chip on the motherboard; keeps time while powered off; low resolution | Read once at boot to set the initial wall-clock time |
| **TSC** (Time Stamp Counter) | A per-core 64-bit counter read with the `rdtsc` instruction; on modern CPUs it ticks at a constant rate regardless of DVFS (2.1) | Fast, high-resolution timekeeping: the usual Linux **clocksource** |
| **HPET**, ACPI PM timer | Older platform timers | Fallback clocksources |
| **Local APIC timer** | A per-core programmable timer | Generates the **timer interrupts** (3.4) that drive preemption |

```bash
$ cat /sys/devices/system/clocksource/clocksource0/current_clocksource
tsc
```

The kernel separates **clocksource** ("what time is it?", read a counter) from **clockevent** ("interrupt me at time T", program a timer). Modern kernels are **tickless** (`NO_HZ`): an idle CPU doesn't take periodic timer interrupts at all; it programs its timer for the next actual event, saving power (2.1). **hrtimers** provide nanosecond-resolution timers.

### Which Clock? `REALTIME` vs `MONOTONIC`

`clock_gettime()` offers several clocks, and picking the wrong one is a classic bug:

| Clock | Meaning | Can jump? | Use it for |
| :---- | :---- | :---- | :---- |
| `CLOCK_REALTIME` | Wall-clock time: seconds since 1970-01-01 UTC (the Unix epoch) | **Yes**: NTP corrections, an admin running `date -s`, VM resume | Timestamps for humans and logs |
| `CLOCK_MONOTONIC` | Time since some unspecified point (boot); **never goes backward** | No (NTP may only speed it up or slow it down slightly); stops during suspend | **Measuring durations**, timeouts |
| `CLOCK_BOOTTIME` | Like `MONOTONIC`, but includes time spent suspended | No | Timeouts that must count suspend |
| `CLOCK_MONOTONIC_RAW` | Raw hardware time, no NTP adjustment at all | No | Benchmarks, comparing clock drift |
| `CLOCK_PROCESS_CPUTIME_ID` / `CLOCK_THREAD_CPUTIME_ID` | CPU time consumed by this process / thread (`utime + stime` in `task_struct`, 4.2) | No | Profiling |

```c
// WRONG: if NTP steps the clock back 2 s during the work, "elapsed" is negative
time_t start = time(NULL);  do_work();  double elapsed = difftime(time(NULL), start);

// RIGHT
struct timespec a, b;
clock_gettime(CLOCK_MONOTONIC, &a);
do_work();
clock_gettime(CLOCK_MONOTONIC, &b);
double elapsed = (b.tv_sec - a.tv_sec) + (b.tv_nsec - a.tv_nsec) / 1e9;
```

These calls are served by the **vDSO** without entering the kernel (3.3), which is why reading the time costs ~20 ns.

### NTP: Keeping the Wall Clock Right

Crystal oscillators drift by seconds per day. **NTP** daemons (`chronyd`, `systemd-timesyncd`) compare with time servers and correct the kernel clock, normally by **slewing** (running it slightly faster or slower via `adjtimex()`, so time never jumps), and only **stepping** (jumping) when the error is large, typically at boot.

```bash
$ timedatectl
               Local time: Fri 2026-10-16 18:40:12 CEST
           Universal time: Fri 2026-10-16 16:40:12 UTC
System clock synchronized: yes
$ chronyc tracking | grep -E 'System time|Frequency'
System time     : 0.000012 seconds fast of NTP time
Frequency       : 4.105 ppm slow
```

**Time zones are a user-space concept.** The kernel keeps time in UTC. The C library converts it to local time using `/etc/localtime` or the `TZ` environment variable. Daylight saving time never changes the kernel clock.

### The Year 2038 Problem

Unix time was traditionally stored in a **signed 32-bit** `time_t`. Its maximum, 2³¹ − 1 seconds after 1970, is **2038-01-19 03:14:07 UTC**. One second later it overflows to −2³¹, which is **1901-12-13** (two's complement, 1.9). 64-bit Linux has always used a 64-bit `time_t`; 32-bit platforms were converted (kernel 5.6, glibc 2.34 with `_TIME_BITS=64`), but old binaries, embedded devices and on-disk formats (old ext3 timestamps, some databases) are still at risk.

### Timers

How a process waits for time:

- **`sleep()` / `nanosleep()` / `clock_nanosleep()`**: the task goes to state `S` (4.8) and an hrtimer wakes it.
- **`timerfd_create()`**: a timer as a **file descriptor** that becomes readable when it fires, so it fits into an `epoll` loop (7.8). Everything is a file again.
- **`alarm()`, `setitimer()`, POSIX `timer_create()`**: deliver a **signal** (`SIGALRM`, 8.2) when they expire.
- **Timer slack:** the kernel may delay a wakeup slightly (50 µs by default) so that several timers can fire together and the CPU can stay idle longer.

> 🔗 **Connections:** Time ties together the hardware (TSC, APIC timer, 2.1), interrupts and preemption (3.4–3.6), the vDSO (3.3), sleeping states (4.8), signals (8), fds and epoll (7.8), and containers: the **time namespace** (10.1) gives a container its own `CLOCK_MONOTONIC`/`BOOTTIME` offsets, but no namespace can change `CLOCK_REALTIME`.

---

# 4. Processes

> 🧪 **Interactive visuals:** [Part 4, visualized](visuals/part4-processes.html): an interactive page with a demo for each section.


## 4.1 Program vs Process

A **program** is a passive executable file stored on disk (such as an ELF file on Linux, Executable and Linkable Format). It contains instructions and data but is not executing.

Examples:

- `/bin/ls`
- `/usr/bin/python3`
- `firefox`

Think of a program as a recipe: it doesn't do anything until it's run.

A **process** is a running instance of a program. A process has its own:

- PID
- Virtual address space (code, data, heap, stacks)
- Open file descriptors
- Environment variables
- Current working directory
- Credentials (user/group IDs)
- Signal handlers
- One or more threads, each with its own registers and stack

When a program is executed, the kernel loads its binary instructions and data into virtual memory and creates process-tracking structures (assigning a PID, tracking state, memory mappings, user IDs, and open file descriptors).

A process is isolated in terms of memory, file descriptors, and scheduling context. It is the fundamental unit of resource ownership for user-space applications.

One program, many processes:

```bash
$ sleep 100 & sleep 100 &
$ pgrep -a sleep
5101 sleep 100
5102 sleep 100      <- same program file, two processes, two PIDs, two address spaces
```

## 4.2 `task_struct` (Process Control Block)

In Linux, a process is represented internally by `struct task_struct`, defined in `include/linux/sched.h`:

```
+----------------------+
|     task_struct      |
+----------------------+
|  PID: Process ID     |
|  State: Running, etc |
|  Registers: CPU info |
|  Memory: Mappings    |
|  Files: Open FDs     |
|  Parent PID: PPID    |
+----------------------+
```

Source code: [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)

The summarized version (see also the map in 0.2):

```
task_struct (one per process, and, later, one per THREAD)
 |- pid, tgid             identity
 |- __state               R / S / D / T / Z
 |- saved registers       restored on context switch          <- illusion 1
 |- parent, children      the process tree
 |- mm      --> mm_struct      (page tables, memory maps)      <- illusion 2
 |- files   --> files_struct   (fd table)                      \
 |- fs      --> fs_struct      (cwd, root)                      |
 |- cred    --> credentials    (uid, gid, capabilities)         | illusion 3
 |- signal/sighand -->         (signal handlers)                |
 `- nsproxy --> namespaces     (what makes a container)        /
```

And a more detailed (still simplified) view of the fields you'll meet in later parts:

```c
struct task_struct {
├── __state                 // Current state (running, sleeping, zombie...)          -> 4.8
├── pid                     // Process ID (actually the thread ID)                    -> 4.5
├── tgid                    // Thread group ID = what getpid() returns                -> 5.1
├── comm[16]                // Process name (what ps/top show; set by exec)
├── flags                   // PF_KTHREAD (kernel thread), PF_EXITING, ...

├── real_parent             // Who created me (fork)
├── parent                  // Who gets my SIGCHLD / wait() (usually == real_parent; differs under ptrace)
├── children, sibling       // Process tree links
├── group_leader            // Main thread of my thread group
├── thread_group            // Other threads in the same group
├── tasks                   // Link in the global list of all tasks

├── mm                      // Memory descriptor (heap, stack, mmap, ...)            -> 6.4
│   ├── pgd                 // Top-level page table (loaded into CR3)
│   ├── start_code, start_data, start_brk, start_stack
│   ├── mm_mt               // Maple tree of VMAs  [before 6.1: mmap linked list]
│   └── rss stats           // Resident set size info
├── active_mm               // Borrowed mm when mm is NULL (kernel threads)

├── thread                  // Arch-specific CPU state for context switching        -> 3.5
│   ├── sp                  // Saved kernel stack pointer
│   └── fpu                 // FPU/SIMD state
├── stack                   // This task's kernel stack (16 KiB on x86-64)

├── files                   // Open files (pointer to files_struct)                  -> 7.1
│   └── fdt -> fd[]         // Array of struct file* for each fd (0, 1, 2, ...)
├── fs                      // Filesystem info
│   ├── pwd                 // Current working directory
│   ├── root                // Root dir (changed by chroot / pivot_root)             -> 10.3
│   └── umask

├── cred                    // Credentials                                           -> 7.11
│   ├── uid, euid, suid, fsuid
│   ├── gid, egid, ...
│   └── cap_effective, cap_permitted, ...  // CAP_NET_ADMIN, ...                    -> 10.4

├── signal                  // Shared by all threads (signal_struct)                 -> 8.4
│   ├── shared_pending      // Process-wide pending signals
│   ├── rlim[]              // Resource limits (RLIMIT_NOFILE, RLIMIT_CPU, ...)
│   ├── pids[PIDTYPE_PGID]  // Process group,  pids[PIDTYPE_SID]: session            -> 4.5
│   └── tty                 // Controlling terminal
├── sighand                 // Signal handlers (shared among threads)
│   └── action[64]          // k_sigaction: handler, flags, mask
├── blocked                 // Mask of blocked signals (per thread)
├── pending                 // Thread-private pending signals
├── exit_signal             // Signal to send to parent on exit (usually SIGCHLD)

├── policy, prio, static_prio, rt_priority                                          -> 3.6
├── se                      // sched_entity: vruntime, deadline, load weight
├── on_cpu, cpus_mask       // where I'm running / allowed to run

├── utime, stime            // CPU time spent in user / kernel mode
├── start_time              // Process start time
├── exit_code, exit_state   // Exit status, EXIT_ZOMBIE / EXIT_DEAD                  -> 4.9

├── nsproxy                 // Namespaces: mnt, uts, ipc, net, pid_for_children...  -> 10.1
├── cgroups                 // cgroup membership                                    -> 10.2
└── ... and many more (the real struct is several KB)
};
```

**Does every process have its own `task_struct`, or is it a shared "class" that processes use?** Every process **and every thread** has its **own instance**. It's not a class that processes are created from; it's the kernel's per-task record, allocated (from a slab cache, 6.2) when the task is created and freed when it is reaped.

- The kernel links all of them in a **task list** (circular doubly linked list via `tasks`) and in a **PID hash table** for fast lookup (`find_task_by_vpid()`).
- Each CPU's run queue (3.6) links the runnable ones.
- The `current` macro gives the `task_struct` of the task running on this CPU. When your process makes a syscall, kernel code uses `current->files`, `current->mm`, etc. That's how `read(3, ...)` knows *whose* fd 3 you mean.

💡 **Note on C `struct`:** In C and C++, `struct` is a keyword used to define a custom structure data type that groups related variables under one name. In the Linux kernel, `struct task_struct` is the C structure that plays the role of the **Process Control Block (PCB)**.

💡 **Note on `/proc`:** `/proc` is not a filesystem on disk; it is a live view of kernel data structures. When you read `/proc/3950/status`, the kernel builds that text on the spot from 3950's `task_struct`. Nothing is stored on disk (7.10).

```bash
$ grep -E 'Name|State|Tgid|^Pid|PPid|Threads|VmRSS|ctxt' /proc/self/status
Name:   grep
State:  R (running)
Tgid:   8123
Pid:    8123
PPid:   8001
VmRSS:  2304 kB
Threads:        1
voluntary_ctxt_switches:        0
nonvoluntary_ctxt_switches:     1
```

## 4.3 Process Image

The **process image** is the complete set of information the OS needs to represent and run a process at a given moment. It's two things combined:

**1. The memory image** (what's inside the address space):

- **Text/code segment**: the compiled instructions, read-only, shared across all processes running the same binary
- **Data segment**: initialized global/static variables
- **BSS segment**: uninitialized global/static variables (zero-filled at load time)
- **Heap**: dynamically allocated memory (`malloc`/`new`), grows upward
- **Stack**: function call frames, local variables, return addresses, grows downward

**2. The execution context** (the CPU state, saved so the process can be paused and resumed):

- Register values (including the program counter, i.e. where execution was interrupted)
- Stack pointer
- Processor status flags

A distinction worth being precise about in an interview:

- The **process image** = the actual memory contents (code + data + heap + stack) that get loaded when you `exec()` a binary.
- The **`task_struct`** = the kernel's *metadata about* the process (PID, state, pointers to the memory descriptor, file table, parent). It doesn't contain the image itself; it *points to* it via `mm_struct`, which holds the page tables mapping the process image into virtual memory.

## 4.4 Process Memory Layout

A process's memory is logically divided into segments:

**1. Text/code segment:** contains the **machine code** generated by the compiler.

```c
int add(int a, int b) {
    return a + b;
}
```

The compiled instructions for `add()` live in `.text`. Usually readable + executable, but not writable.

**2. Data segment:** global/static variables that have an **explicit initial value**.

```c
int counter = 10;
static int x = 42;
```

These values need to exist when the program starts, so they're stored in the executable. Readable and writable.

**3. BSS segment:** global/static variables **without** an explicit initializer.

```c
int counter;
static int x;
```

When the program starts, these must be initialized to **zero** according to C semantics. The clever part is that the executable doesn't need to physically store thousands of zero bytes. Instead, the ELF file records something like:

> "I need 1 MB of zero-initialized memory."

The kernel/loader creates the corresponding virtual memory, initially zero-filled (anonymous memory, demand-paged: 6.5).

**4. Heap:** dynamically allocated memory (`malloc`/`new`), grows upward (via `brk()`, 6.6).

**5. Stack:** function call frames, local variables, return addresses, grows downward. Default limit 8 MiB (`ulimit -s`); overflowing it hits a guard gap → `SIGSEGV`.

### Memory-Mapped Region (Shared Libraries, Anonymous `mmap`)

Between the heap and the stack is a large area where `mmap()` places mappings (growing downward from below the stack):

- **shared libraries**: `libc.so.6`, `ld-linux-x86-64.so.2`, each mapped with separate r-x (code), r-- and rw- (data) regions
- **anonymous mappings**: large `malloc` allocations (≥128 KiB by default), thread stacks (5.1)
- **file mappings**: files a program `mmap`s directly (databases, `ripgrep`)
- **shared memory**: POSIX shm segments (9.3)

### Kernel Space (Upper Half of the Address Space)

On x86-64, the upper canonical half (`0xffff8000...` and up, 1.4) belongs to the kernel and is the **same in every process**: kernel code, kernel data, every task's kernel stack, and a **direct map** of all physical RAM (at `0xffff888000000000`, so the kernel can touch any frame by adding an offset). Its page-table entries have `U/S = supervisor`, so Ring 3 can't touch it (2.6, 6.3); with KPTI most of it is not even mapped while in user mode (3.3).

### Full Address Space Diagram

```
0xFFFFFFFFFFFFFFFF  +-------------------------------+
                    |  kernel space (shared by all) |  direct map of RAM, kernel code/data,
                    |  not accessible from Ring 3   |  kernel stacks, vmalloc, modules
0xFFFF800000000000  +-------------------------------+
                    |   non-canonical hole (1.4)    |
0x00007FFFFFFFFFFF  +-------------------------------+
                    |  [vvar] [vdso]                |  vDSO (3.3)
                    |  stack            (grows down)|  argv, envp, auxv at the top
                    |      v                        |  <- rsp ~ 0x7ffc...
                    |                               |
                    |      ^                        |
                    |  mmap region      (grows down)|  libc.so, ld.so, thread stacks,
                    |                               |  big mallocs   ~ 0x7f...
                    |                               |
                    |      ^                        |
                    |  heap             (grows up)  |  brk  (6.6)
                    +-------------------------------+
                    |  .bss   zero-initialized      |
                    |  .data  initialized globals   |  rw-
                    |  .rodata constants            |  r--
                    |  .text  code                  |  r-x   ~ 0x55... (PIE + ASLR)
                    +-------------------------------+
0x0000000000000000  |  unmapped (NULL page guard)   |  *NULL -> SIGSEGV
                    +-------------------------------+
```

See it live:

```bash
$ cat /proc/self/maps
55d0c1a00000-55d0c1a02000 r--p 00000000 103:02 1835123  /usr/bin/cat      <- ELF header, .rodata
55d0c1a02000-55d0c1a07000 r-xp 00002000 103:02 1835123  /usr/bin/cat      <- .text
55d0c1a0a000-55d0c1a0b000 rw-p 00009000 103:02 1835123  /usr/bin/cat      <- .data
55d0c2f4e000-55d0c2f6f000 rw-p 00000000 00:00 0         [heap]
7f2b1c600000-7f2b1c628000 r--p 00000000 103:02 1840211  /usr/lib/x86_64-linux-gnu/libc.so.6
7f2b1c628000-7f2b1c7bd000 r-xp 00028000 103:02 1840211  /usr/lib/x86_64-linux-gnu/libc.so.6
7ffd5b8e7000-7ffd5b908000 rw-p 00000000 00:00 0         [stack]
7ffd5b9d5000-7ffd5b9d9000 r--p 00000000 00:00 0         [vvar]
7ffd5b9d9000-7ffd5b9db000 r-xp 00000000 00:00 0         [vdso]
```

Each line is one VMA (`vm_area_struct`, 6.4). The columns are: range, permissions (`p` = private/COW, `s` = shared), file offset, device (major:minor), inode number (7.3), path.

### ASLR (Address Space Layout Randomization)

Run `cat /proc/self/maps` twice and the addresses change. That's **ASLR**: the kernel randomizes the base of the stack, the mmap region, the heap, the vDSO and (for **PIE** binaries, the default today) the program itself.

Why: most memory-corruption exploits need to know where something is (a function to jump to, a buffer). Randomizing the layout turns a reliable exploit into a guess.

```bash
$ cat /proc/sys/kernel/randomize_va_space
2      # 0 = off, 1 = stack/mmap/vdso, 2 = also heap (brk)
$ setarch $(uname -m) -R ./prog     # run one program with ASLR disabled (debugging)
```

The kernel has its own version, **KASLR**, which randomizes where the kernel image is loaded at boot.

## 4.5 Process Identity

Every running process gets a **PID** (process ID). The PID uniquely identifies a process **at that point in time** (PIDs are reused after a process exits; the maximum is in `/proc/sys/kernel/pid_max`). For example, PID 1 is the init process.

**PPID** is the Parent Process ID. When a process creates another process, the new process normally has the creator as its parent.

```
Shell  (PID 100)
  |
  `-- Python  (PID 200, PPID 100)
```

So:

```
PID 1
`-- PID 100 bash
    |-- PID 200 python
    `-- PID 300 sleep
```

```bash
$ pstree -p $$
bash(100)─┬─python(200)
          └─sleep(300)
$ ps -o pid,ppid,pgid,sid,tty,stat,cmd
```

### PID 0 vs PID 1 (and PID 2)

| Aspect | PID 0 (`swapper` / idle task) | PID 1 (`init` / `systemd`) | PID 2 (`kthreadd`) |
| :---- | :---- | :---- | :---- |
| **Created by** | Statically by the kernel at boot (`init_task`); it *is* the boot thread | PID 0, as a thread running `kernel_init()` (created with `user_mode_thread()` since Linux 6.0, `kernel_thread()` before), which then `exec`s `/sbin/init` | PID 0, as a kernel thread |
| **Lives in** | Kernel only | User space (very privileged process) | Kernel only |
| **Purpose** | Runs when a CPU has nothing else to do (`hlt`/`mwait`, 2.1) | First user-space process; starts services; adopts orphans | Parent of all kernel threads (`[kworker/...]`, `[ksoftirqd/0]`, `[kswapd0]`) |
| **Visible in `ps`** | No (it's the PPID of PIDs 1 and 2) | Yes | Yes, in `[brackets]` |
| **How many** | One per CPU | One (per PID namespace, 10.1) | One |
| **If it dies** | Can't | Kernel panic | Can't |

The name `swapper` is historical: on old Unix, PID 0 was responsible for swapping whole processes out. In Linux it's just the idle task.

### What Happens When the Parent Dies?

```
PID 100
`-- PID 200
```

Parent 100 dies:

```
PID 100 ❌
PID 200 <- orphan
```

Linux needs to give the orphan a new parent. It gets **adopted by the nearest subreaper ancestor, or otherwise by PID 1**. So you might end up with:

```
PID 1
`-- PID 200
```

This is one reason PID 1 is special: it must `wait()` for all these adopted children when they exit, or they'd stay zombies forever (4.9).

### Subreapers (`PR_SET_CHILD_SUBREAPER`)

A process can declare itself a **subreaper** with `prctl(PR_SET_CHILD_SUBREAPER, 1)`. Then orphaned *descendants* are reparented to **it** instead of PID 1.

Who uses it:

- `systemd --user` (your per-user service manager) so your user services' orphans stay under it
- container shims (`containerd-shim`) and tiny inits like `tini` / `dumb-init`, so a container's orphans don't escape
- build systems and supervisors that must track every process they spawned, even double-forked daemons

### Process Groups, Sessions and Controlling Terminals

Processes are grouped in two more levels, mostly for **job control** in shells:

```
SESSION (sid = 100, controlling terminal = /dev/pts/0)
 |
 |-- process group 100  [bash]                              <- session leader
 |
 |-- process group 200  [ls | grep txt]   foreground job    <- both processes share pgid 200
 |
 `-- process group 300  [sleep 100 &]     background job
```

- **Process group:** a job. A pipeline `ls | grep txt` is one group, so `Ctrl+C` kills both.
- **Session:** everything started from one login. Created with `setsid()`; the creator is the session leader.
- **Controlling terminal:** a session has at most one. Exactly one process group is the **foreground** group of the terminal.

How keyboard signals work: the **terminal driver** (in the kernel, not the shell!) sees `Ctrl+C` and sends `SIGINT` to **every process in the foreground process group**. `Ctrl+Z` sends `SIGTSTP`; `Ctrl+\` sends `SIGQUIT`. When the terminal hangs up (you close the SSH window), the session leader gets **`SIGHUP`**, which is forwarded to jobs; that's why `nohup` and `setsid` exist.

```bash
$ sleep 100 | sleep 200 &
$ ps -o pid,pgid,sid,tty,stat,cmd
  PID  PGID   SID TT       STAT CMD
 4120  4120  4120 pts/0    Ss   -bash          <- s = session leader
 5301  5301  4120 pts/0    S    sleep 100      <- same PGID: one job
 5302  5301  4120 pts/0    S    sleep 200
```

Daemons call `setsid()` (or are started by systemd) precisely to have **no** controlling terminal, so they never get terminal signals.

### pidfd

PIDs are reused. If you read a PID from a file, then `kill(pid, SIGTERM)`, the original process may have exited and **another process got the same PID**. You just killed the wrong process.

A **pidfd** is a file descriptor that refers to one specific process, forever:

```c
int pfd = pidfd_open(pid, 0);           // or clone(CLONE_PIDFD) at creation time
pidfd_send_signal(pfd, SIGTERM, NULL, 0);  // can't hit a recycled PID
// poll()/epoll() on pfd becomes readable when the process exits (7.8)
```

It's the "everything is a file" idea (7.4) applied to processes. systemd and modern container runtimes use it.

## 4.6 Process Priority and Niceness

Linux priorities:

- **0–99:** real-time priorities (`SCHED_FIFO`, `SCHED_RR`). This is the kernel's **internal** scale, where a **lower** number means **higher** priority. The user-facing real-time priority (`chrt -f 50`, `rt_priority`) runs 1–99 with **higher = more important**; internally it maps to `99 − rt_priority`, so RT priority 99 is internal priority 0.
- **100–139:** normal user processes. The priority is computed as **120 + nice value**, so nice −20 → 100 (highest) and nice +19 → 139 (lowest).

To set the niceness of a process, use the `nice` command (when starting it) or `renice` (for a running process). The range is **−20 (highest priority) to +19 (lowest priority)**. The default niceness is 0.

- Regular users can only **increase** niceness (lower their own priority).
- Lowering niceness (raising priority) requires root or `CAP_SYS_NICE`.
- Niceness only affects the normal policies, `SCHED_OTHER` (also called `SCHED_NORMAL`) and `SCHED_BATCH`.

Niceness is not a percentage; each step is a weight change of ~1.25×. Two CPU-bound tasks on one core, one at nice 0 and one at nice 5, split the CPU roughly 75% / 25%.

```bash
$ nice -n 10 tar czf backup.tgz /data       # start with nice 10
$ renice -n 5 -p 4321                        # change a running process
$ ps -o pid,ni,pri,cls,cmd -p 4321           # note: ps "PRI" uses its own scale
```

See 3.6 Scheduler for more.

## 4.7 Process Types

- **Foreground process**: interacts with the user through the terminal. `Ctrl+Z` **stops** it (`SIGTSTP`); `bg` then resumes it in the background, `fg` brings it back.
- **Background process**: runs without holding the terminal (e.g., started with `&`).
- **Daemon process**: runs for as long as the system is up or until explicitly stopped, detached from any terminal (e.g., `sshd`, `httpd`).
- **Zombie process**: has completed but not yet been `wait()`ed on by its parent.
- **Orphan process**: its parent terminated; it gets reparented to a subreaper or PID 1.
- **Kernel thread**: a task that runs only in kernel mode and has no user address space (`mm == NULL`). Shown in `[brackets]` by `ps`, children of `kthreadd` (PID 2).

## 4.8 Process States

You can see these in the `STAT` column of `ps` or the `S` column of `top`.

- **(R) Running:** either executing or in the run queue, waiting for a CPU.
- **(S) Interruptible sleep:** waiting **for an event** (e.g., user input, a socket). Can be woken by signals (like `SIGINT`, `SIGTERM`).
- **(D) Uninterruptible sleep:** waiting on hardware or kernel-level I/O that cannot be interrupted. Even `kill -9` doesn't take effect immediately: the signal stays pending until the sleep ends. (Newer kernel code often uses `TASK_KILLABLE`, which does respond to fatal signals.)
- **(T) Stopped:** process is **paused** (with `SIGSTOP` or `Ctrl+Z`). Resumed with `fg`/`bg` or `SIGCONT`.
- **(Z) Zombie:** the process has **finished executing**, but its **exit status has not yet been collected** by its parent. When a process dies, it doesn't fully disappear right away. The kernel keeps its PID and some info in the process table so the parent can call `wait()` and retrieve the exit status. Until then, it's a zombie, which is why they show up in `ps`.

**Orphan** is not a state: it describes a relationship (the parent exited first). An orphan can be in any of the states above.

```
                     fork()
                       |
                       v
          +-------> [R runnable] <---------- wakeup (I/O done, signal) ----+
          |         |   ^      |                                            |
 preempted|  picked |   |      | blocks on I/O, lock, sleep()               |
 (3.5)    |         v   |      v                                            |
          +------ [R running]  +--> [S interruptible] / [D uninterruptible]-+
                    |    |
       SIGSTOP/^Z   |    | exit() / fatal signal
                    v    v
               [T stopped]  [Z zombie] --parent wait()--> gone (task_struct freed)
                    |
               SIGCONT -> back to R
```

### Other States (`I` Idle Kernel Thread, `t` Tracing Stop, `X` Dead)

- **`I` (Idle):** a kernel thread sleeping uninterruptibly but *not* waiting on I/O (e.g. an idle `kworker`). Introduced so idle kernel threads don't count as `D` and inflate the load average (3.6).
- **`t` (tracing stop):** stopped by a debugger (`ptrace`, see the strace/ltrace/ptrace question in Part 12), e.g. at a `gdb` breakpoint.
- **`X` (dead):** being destroyed; you should never actually see it.

`ps` also appends modifiers: `s` session leader, `+` in the foreground process group, `l` multi-threaded, `<` high priority (negative nice), `N` low priority, `L` has locked pages in memory. So `Ss+` = sleeping session leader in the foreground.

## 4.9 Process Creation and Program Execution

This is the intersection point where most of the earlier topics meet. Let's follow what happens when you type `./my-program.sh` in bash.

### Step 0: Terminal Input Handling

Before bash sees anything:

1. Your terminal emulator (iTerm, GNOME Terminal) or `sshd` is connected to a **pseudo-terminal (PTY)** pair: a *master* side held by the emulator and a *slave* side, `/dev/pts/0`, which is bash's stdin/stdout/stderr (fds 0, 1, 2).
2. Each key you type is written to the PTY master. In the kernel, the **TTY line discipline** stores characters in a buffer. In **canonical (cooked) mode** it handles line editing (backspace) and only releases the line when you press Enter. Independently of that, with the `ISIG` setting on (the default), it catches `Ctrl+C`/`Ctrl+Z` and turns them into signals for the foreground process group (4.5).
3. Bash actually uses **raw mode** while you edit (via `readline`, so it can do arrow keys and tab completion), reading keys with `read(0, ...)` and echoing them itself.

### Step 1: Shell Parses and Resolves the Path

Inside the shell:

1. `readline()` returns the full line `./my-program.sh`.
2. The shell **tokenizes** it, handles quotes, expands variables (`$HOME`), globs (`*.txt`, 12), and finds redirections (`>`, `<`, `2>&1`) and pipes (`|`).
3. It checks what the first word is, in this order: **alias** → **shell function** → **builtin** (`cd`, `echo`, `export`, `exit`, `read`...) → **external command**.

Builtins must run inside the shell process itself when they change the shell's state: `cd` must change *bash's* current directory (`task_struct->fs->pwd`). If `cd` were an external program, it would change the directory of a child that immediately exits. (Note: `ls` is **not** a builtin; it's `/usr/bin/ls`.)

```bash
$ type cd ls ll
cd is a shell builtin
ls is /usr/bin/ls
ll is aliased to `ls -alF'
```

Since `./my-program.sh` contains a `/`, bash does **not** search `$PATH`; it treats it as a direct path relative to `$PWD`. (If you'd typed just `my-program.sh` with no slash, and it wasn't a function/alias/builtin, bash would search each directory in `$PATH`, caching the result in a hash table: see `hash`.)

Bash then forks a child, and the child calls `execve()`. If the file doesn't exist or isn't executable, it's the **`execve()` in the child** that fails: the child prints the error and exits with **127** (`No such file or directory`) or **126** (`Permission denied`). That's why `echo $?` shows 126/127 after these errors. The permission check that matters is the kernel's, during `execve()` (step 5), not anything the shell does.

### Step 2: `fork()` → `clone()`

On Linux, glibc's `fork()` doesn't use the `fork` syscall: it's a thin wrapper around the `clone()` syscall (historically `clone(SIGCHLD, 0)`; modern glibc uses `clone3()` when available, falling back to `clone()`). x86-64 still has a legacy `fork` syscall (number 57), but newer architectures like arm64 and riscv don't have one at all.

`clone()` is Linux's generalized process/thread creation primitive. Its signature is roughly:

```c
long clone(unsigned long flags, void *stack,
           int *parent_tid, int *child_tid, unsigned long tls);
```

The `flags` argument is what makes `clone()` so flexible. It's a bitmask controlling exactly what's shared between parent and child:

- `CLONE_VM`: share the same memory address space (this is what makes a thread a thread; `pthread_create` uses it)
- `CLONE_FILES`: share the file descriptor table
- `CLONE_FS`: share filesystem info (cwd, umask, root)
- `CLONE_SIGHAND`: share the signal handler table
- `CLONE_THREAD`: put the child in the same thread group
- `CLONE_NEWPID`, `CLONE_NEWNS`, `CLONE_NEWNET`, ...: put the child in **new namespaces** (this is how containers start, 10.1)

`fork()` provides traditional process semantics: the child gets a fresh `mm_struct`, a fresh fd table (but the fds themselves point to the same open file descriptions), fresh everything, just logically copied. That's the distinction:

- `fork()` = `clone()` with (almost) **nothing shared**
- `pthread_create()` = `clone()` with almost **everything shared**

Same underlying kernel mechanism, wildly different semantics depending on flags.

### Step 3: What the Kernel Does (`kernel/fork.c`, `kernel_clone()` → `copy_process()`) [before Linux 5.10: `_do_fork()`]

The CPU switches from user mode to kernel mode at the `syscall` instruction (3.3). Then:

1. Allocates a new `task_struct` (the kernel's process descriptor) and a new kernel stack.
2. Allocates a new PID via `alloc_pid()`.
3. Duplicates the parent's `mm_struct` (memory descriptor), but **does not copy the actual memory pages**. Instead it duplicates the page tables and marks all writable pages **copy-on-write (COW)**: both parent and child point to the same physical pages, marked read-only. The moment either process writes to one, a page fault fires, the kernel allocates a new physical page, copies the data, and remaps it: lazily, only for pages that actually get modified. This is why `fork()` is cheap even for a process with gigabytes of memory.
4. Duplicates the file descriptor table (a new table, but entries reference the same underlying open file descriptions, so file offsets are shared: if parent and child both read from the same fd, they advance a shared file position, 7.2).
5. Duplicates signal handler dispositions.
6. Sets up the parent/child relationship (`task_struct->parent`, added to the parent's `children` list).
7. Makes the child runnable (puts it on a run queue, 3.6).
8. Returns twice, conceptually: once in each process's context, resuming right after the syscall. Back in user space, `fork()` returns `0` in the child and the child's PID in the parent (or `-1` on failure). Bash's code branches on this return value; the child branch is what proceeds to exec.

**Copy-on-write (COW)** is an optimization strategy: if multiple callers ask for resources that are initially indistinguishable, give them pointers to the same resource. This continues until a caller tries to modify its "copy", at which point a true private copy is created so the change isn't visible to everyone else. COW works at the **page level** (usually 4 KB; 16 KB on Apple Silicon), not on the entire process memory.

```
after fork():                            after the child writes to page B:

parent PT     RAM      child PT          parent PT     RAM      child PT
  A ----->  [frame 7] <---- A              A ----->  [frame 7] <---- A
  B ----->  [frame 9] <---- B   (R/O)      B ----->  [frame 9]
                                                     [frame 12] <---- B   (copy, now R/W)
```

In summary:

- **Shared memory:** both processes initially share the same physical pages.
- **Memory write:** if either process writes to a shared page, a page fault allocates a new physical page, and the modification occurs there (6.5).
- **Efficiency:** this avoids unnecessary memory duplication when the child immediately calls `execve()`, which throws the memory away anyway.

|  | Parent (bash) | Child |
| :---- | :---: | :---: |
| `fork()` returns | 25111 | 0 |
| `getpid()` returns | 25110 | 25111 |
| `getppid()` returns | bash's own parent (e.g., the terminal) | 25110 |

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    int x = 1;
    pid_t pid = fork();
    if (pid == 0) {                 // child
        x = 100;                    // COW: child gets its own copy of this page
        printf("child:  pid=%d ppid=%d x=%d\n", getpid(), getppid(), x);
        return 0;
    }
    waitpid(pid, NULL, 0);          // parent
    printf("parent: pid=%d child=%d x=%d\n", getpid(), pid, x);   // x is still 1
}
```

#### `vfork()` and `posix_spawn()`

COW makes `fork()` cheap, but not free: the kernel still copies the page tables and VMAs. For a process using 50 GB of RAM, that's ~100 MB of page tables to copy just so the child can `exec` immediately.

- **`vfork()`**: the child **shares the parent's memory** (no copy at all), and the parent is **suspended** until the child calls `execve()` or `_exit()`. The child must do nothing else (not even return from the function). Dangerous but fast.
- **`posix_spawn()`**: a safe, high-level "fork + exec in one call" API, with a list of file actions (open/close/dup2) and attributes. glibc implements it with `clone(CLONE_VM | CLONE_VFORK)` on a separate small stack, getting `vfork` speed with safety.

Python's `subprocess`, Java's `ProcessBuilder` and Go's `os/exec` all use these on Linux.

### Step 4: The Child Calls `execve()`

`execve()` **is** the real syscall (not a wrapper). Its signature:

```c
int execve(const char *pathname, char *const argv[], char *const envp[]);

// e.g.
execve("/bin/cat", (char *[]){"cat", "test.txt", NULL}, environ);
```

All the other exec functions you see in `man 3 exec` (`execl`, `execlp`, `execle`, `execv`, `execvp`, `execvpe`) are **glibc library wrappers**, not syscalls. They differ only in argument-passing convenience:

- **`l`** (list): arguments passed one by one, `execl("/bin/ls", "ls", "-l", NULL)`
- **`v`** (vector): arguments passed as an array, `execv("/bin/ls", argv)`
- **`p`** (path): search `$PATH` for the program name
- **`e`** (environment): pass a custom environment array

If `execve()` succeeds, **it never returns**: there's no old program left to return to.

### Step 5: What `execve()` Does in the Kernel (`fs/exec.c`, `do_execve()` → `bprm_execve()`)

This is the interesting part. It's not "run a new process"; it's "**destroy and rebuild this process's memory image in place**":

1. Resolves the path through the VFS (7.4): walks `.` (the cwd from `task_struct->fs`), looks up the dentry and inode of `my-program.sh`.
2. Opens the target file and reads its first bytes into a buffer, `BINPRM_BUF_SIZE` = 256 bytes [before Linux 5.1: 128 bytes], as part of a `linux_binprm` struct.
3. Checks that it's a regular file, that the filesystem isn't mounted `noexec`, and the **execute permission bit** for your credentials (7.11); applies **setuid/setgid** bits if present. Failure here is the `EACCES` that becomes exit code 126.
4. Iterates through the registered **binary format handlers** (`binfmt` list), asking each "can you handle this?" by inspecting the magic bytes:
   - `binfmt_elf`: checks for the 4-byte ELF magic `\x7fELF`
   - `binfmt_script`: checks for `#!`
   - `binfmt_misc`: configurable; handles things like Java jars, Wine `.exe`, or other-architecture binaries (via QEMU) using registered magic/extension rules
5. Your file starts with `#!/bin/bash`, so `binfmt_script`'s handler (`load_script()`) matches.
6. `load_script()` parses the interpreter path from the `#!` line and **restarts the lookup** with `/bin/bash` as the file to execute and `./my-program.sh` as its argument. `/bin/bash` starts with `\x7fELF`, so this time **`binfmt_elf`** matches.

```bash
$ head -c 4 /bin/bash | xxd
00000000: 7f45 4c46                                .ELF
$ head -c 2 my-program.sh
#!
```

### Step 6: `binfmt_elf` Loading

Now the actual replacement happens:

1. The old memory mappings (the forked copy of the parent shell) are torn down. `exec_mmap()` discards the old `mm_struct`. (If `CLONE_VM` was in play, thread-group implications get hairy, but for a normal `fork` + `exec` there's just one user of that `mm_struct`, so it's freed cleanly.) Other threads of the process are killed: after exec, a process always has exactly one thread.
2. A fresh address space is built: new mappings (VMAs, 6.4) for `.text`, `.data`, `.bss`, the stack, and the dynamic linker `ld-linux.so` (since `/bin/bash` is dynamically linked). Most of this is demand-paged, not loaded eagerly: no file data is read yet (6.5).
3. `argv`, `envp`, and the auxiliary vector (`auxv`: things like `AT_ENTRY`, `AT_PHDR`, page size, etc.) are set up on the new stack.
4. **What survives the exec, despite the memory being wiped:**
   - PID (and TID, PPID)
   - Open file descriptors (unless marked `FD_CLOEXEC`)
   - Current working directory
   - Signals: *caught* signals are reset to default (the handler code no longer exists!), but *ignored* signals (`SIG_IGN`) stay ignored, and the signal mask and pending signals persist
   - Process group, session, controlling terminal
   - Resource limits (`ulimit`), nice value, cgroup membership, namespaces
   - Real/effective/saved UID and GID (except for setuid/setgid adjustments)
5. The kernel sets the instruction pointer to the ELF entry point (or to the dynamic linker's entry point, which then loads shared libraries and jumps to bash's actual `main`), and returns to user space, executing *new* code for the very first time. The first instruction immediately page-faults, and the kernel loads the code page from the page cache (6.5, 6.7).

At no point is there a "new process": same `task_struct`, same PID, same entry in `/proc`. It's an in-place metamorphosis.

```
              fork()                          execve("/bin/cat")
bash(100) ----------> bash(100) + bash(101) ----------------------> bash(100) + cat(101)
                      same code, COW memory                         same PID 101, new memory
```

#### Dynamic Linking (`ld-linux.so`, Shared Libraries, PLT/GOT)

Most binaries don't contain `printf`; they *reference* it in `libc.so.6`. (How ELF files, symbols and linking work in general: 4.10.) This is why `/lib` (7.14) contains shared libraries required by the binaries in `/bin` and `/sbin`.

1. The ELF file has a `PT_INTERP` header naming the **dynamic linker**: `/lib64/ld-linux-x86-64.so.2`. The kernel maps it and starts *it*, not `main`.
2. `ld.so` reads the binary's `DT_NEEDED` entries (`libc.so.6`, `libtinfo.so.6`...), finds them via `LD_LIBRARY_PATH`, `/etc/ld.so.cache` and default dirs, and `mmap`s each one (6.6).
3. It fills in addresses of external functions and variables (relocation) and jumps to the program's entry point.

```bash
$ ldd /bin/bash
        linux-vdso.so.1 (0x00007ffd...)                      <- vDSO (3.3), not a real file
        libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
        /lib64/ld-linux-x86-64.so.2
```

**PLT/GOT:** calls to library functions go through a small stub table (**PLT**, Procedure Linkage Table) that jumps through a pointer table (**GOT**, Global Offset Table). With lazy binding, the first call goes to the resolver, which patches the GOT entry; later calls jump straight to `printf`. Hardened binaries resolve everything at startup and then make the GOT read-only (**RELRO**) so attackers can't overwrite it.

Why shared libraries matter for memory: the code pages of `libc.so.6` are file-backed, read-only mappings of the **same page-cache pages** (6.7) in every process. 500 processes using libc share **one** copy of its code in RAM.

### Step 7: Process Termination and `wait()`

A process ends by calling `exit()` (or returning from `main`, which calls `exit()`), or by being killed by a signal. In the kernel, `do_exit()` releases its memory (`mm_struct`), closes its fds, detaches from namespaces, reparents its own children (4.5), and becomes a **zombie**: only the `task_struct` with the exit status remains. Then it notifies the parent with **`SIGCHLD`**.

The **`wait()`** system call lets the parent process wait for its child to finish and collect its exit status.

1. **Parent waits:** the parent calls `wait()` to block until one of its children exits.
2. **Child terminates:** the child exits; most of its resources (memory, open files) are released, but its `task_struct` and exit status stay behind.
3. **Exit status:** the parent retrieves the child's exit status, the kernel frees the remaining entry, and the parent resumes.

**IMPORTANT:** If the parent doesn't call `wait()`, the child stays a **zombie** until the parent collects its exit status (or the parent itself exits and the orphaned zombie is reaped by its new parent).

#### `waitpid()` and `SIGCHLD`

`wait()` waits for *any* child and blocks. `waitpid()` is more flexible:

```c
int status;
pid_t r = waitpid(pid, &status, 0);          // wait for one specific child
r = waitpid(-1, &status, WNOHANG);           // any child, but don't block: returns 0 if none exited

if (WIFEXITED(status))   printf("exit code %d\n", WEXITSTATUS(status));
if (WIFSIGNALED(status)) printf("killed by signal %d\n", WTERMSIG(status));
```

A server that spawns many children can't block in `wait()`. Instead it handles **`SIGCHLD`** (sent to the parent whenever a child exits or stops) and reaps in a loop, because several children may have exited but standard signals don't queue (8.4):

```c
void on_sigchld(int sig) {
    int saved = errno;
    while (waitpid(-1, NULL, WNOHANG) > 0)
        ;                       // reap every exited child
    errno = saved;
}
```

#### Auto-Reaping (Ignoring `SIGCHLD`)

If a parent doesn't care about exit statuses at all, it can tell the kernel to reap children automatically:

```c
signal(SIGCHLD, SIG_IGN);                           // POSIX: children never become zombies
// or: sigaction with SA_NOCLDWAIT
```

Note the subtlety: `SIGCHLD`'s *default* action is also "ignore", but explicitly setting `SIG_IGN` is what enables auto-reaping.

#### Exit Codes (0, 1–125, 126, 127, 128+N)

The exit status is **8 bits** (0–255): `exit(256)` is seen as 0.

| Code | Meaning |
| :---- | :---- |
| `0` | Success |
| `1` | General error |
| `2` | Misuse of shell builtin / bad usage (convention) |
| `126` | Found but **not executable** (`EACCES`) |
| `127` | **Command not found** (`ENOENT`) |
| `128 + N` | Killed by **signal N** (shell convention) |
| `130` | 128 + 2: `SIGINT` (`Ctrl+C`) |
| `137` | 128 + 9: `SIGKILL`, very often the **OOM killer** (6.8) or a container memory limit (10.2) |
| `139` | 128 + 11: `SIGSEGV`, segfault |
| `143` | 128 + 15: `SIGTERM`, e.g. `docker stop` |

```bash
$ sleep 100 & kill -9 $!; wait $!; echo $?
137
$ ./nonexistent; echo $?
127
```

> 🔗 **Connections:** One `./my-program.sh` touched the TTY driver (7.5), `fork` → COW page faults (6.5), the VFS path walk and inode permission bits (7.3, 7.11), the binfmt handlers, `mmap` of ELF segments and shared libraries (6.6) from the page cache (6.7), signals on exit (8), and the scheduler at every step (3.6). Part 11 traces a full pipeline the same way.

## 4.10 ELF Files and Linking

Section 4.9 used words like *ELF segment*, `PT_INTERP` and *dynamic linker*. This section explains where an executable file comes from and what's inside it.

### From Source to Executable

```
hello.c --[preprocessor]--> hello.i --[compiler]--> hello.s --[assembler]--> hello.o --[linker]--> hello
         #include, #define           C -> assembly             assembly ->       relocatable      executable
                                                               machine code     ELF object       ELF file
```

```bash
$ gcc -E hello.c -o hello.i      # preprocess only
$ gcc -S hello.c                 # compile to assembly (hello.s)
$ gcc -c hello.c                 # assemble to an object file (hello.o)
$ gcc hello.o -o hello           # link (gcc runs the linker, ld)
```

An **object file** (`.o`) contains machine code, but with **holes**: the call to `printf` doesn't know where `printf` will be. The **linker** fills in those holes (**relocation**) by matching **symbols** (named functions and variables) across object files and libraries.

```bash
$ nm hello.o
0000000000000000 T main          <- T: defined here, in .text
                 U puts          <- U: undefined, must come from somewhere else (libc)
```

### Inside an ELF File: Sections vs Segments

**ELF** (Executable and Linkable Format) is used for object files, executables, shared libraries (`.so`), kernel modules (`.ko`, 3.1) and core dumps. It has two views of the same bytes:

```
+------------------------+
| ELF header             |  magic 7f 45 4c 46 ("\x7fELF"), 32/64-bit, endianness (1.5),
|                        |  type (REL / EXEC / DYN), machine (x86-64), ENTRY POINT address
+------------------------+
| Program header table   |  SEGMENTS: what the KERNEL and the dynamic linker load (runtime view)
+------------------------+
| .text                  |  \
| .rodata                |   |  SECTIONS: what the LINKER and DEBUGGERS use (link-time view)
| .data                  |   |  several sections are packed into each loadable segment
| .bss  (no file bytes)  |   |
| .dynsym .dynstr .plt   |   |
| .got .symtab .debug_*  |  /
+------------------------+
| Section header table   |
+------------------------+
```

```bash
$ readelf -h /bin/ls | grep -E 'Class|Data|Type|Entry'
  Class:                             ELF64
  Data:                              2's complement, little endian
  Type:                              DYN (Position-Independent Executable file)
  Entry point address:               0x6aa0

$ readelf -lW /bin/ls | grep -E 'LOAD|INTERP|DYNAMIC|GNU_STACK|GNU_RELRO'    # (abbreviated)
  INTERP    0x000318 ... [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD      0x000000 0x0000000000000000 ... R   0x1000     <- headers, .rodata
  LOAD      0x004000 0x0000000000004000 ... R E 0x1000     <- .text  (readable + executable)
  LOAD      0x01e000 0x000000000001e000 ... RW  0x1000     <- .data, .bss
  DYNAMIC   ...                                            <- info for the dynamic linker
  GNU_STACK ...                             RW             <- stack NOT executable (NX, 6.3)
  GNU_RELRO ...                             R              <- made read-only after relocation
```

Each `LOAD` segment becomes one **VMA** (6.4) with those permissions when `execve` maps the file. That's exactly what you see in `/proc/<pid>/maps` (4.4): `r--p`, `r-xp`, `rw-p`. Note that `readelf` itself prints "2's complement, little endian" (1.5, 1.9).

| | Sections | Segments |
| :---- | :---- | :---- |
| Described by | Section header table | Program header table |
| Used by | Linker, debugger, `objdump` | Kernel (`binfmt_elf`) and `ld.so` |
| Granularity | Fine (`.text`, `.data`, `.symtab`, ...) | Coarse (one per permission set) |
| Needed to run? | No (`strip` removes symbol and debug sections) | Yes |

### Static vs Dynamic Linking

| | Static linking | Dynamic linking (the default) |
| :---- | :---- | :---- |
| Library code | **Copied into** the executable | **Referenced**; loaded at runtime by `ld.so` (4.9) |
| File size | Larger | Smaller |
| RAM | Each program has its own copy | One copy of each library's code in the page cache, **shared** by all processes (6.7) |
| Security updates | Rebuild every program | Update the library once; restart programs |
| Portability | Runs anywhere with a compatible kernel (great for `FROM scratch` containers) | Needs compatible library versions on the target |
| Startup | Slightly faster (no symbol resolution) | `ld.so` must load and relocate libraries |
| Interposition | Not possible | `LD_PRELOAD=./mylib.so prog` can override functions (debugging, profiling, hooks) |

Go builds static binaries by default; C programs usually link statically against **musl** libc rather than glibc. `file prog` tells you which kind you have.

### Position-Independent Code, PIE and ASLR

For a library to be loaded at **any** address (different in each process, thanks to ASLR, 4.4), its code can't contain absolute addresses. **Position-independent code** (`-fPIC`) addresses data relative to the current instruction (`rip`-relative addressing on x86-64) and reaches external symbols through the **GOT** (4.9).

A **PIE** (Position-Independent Executable) applies the same to the program itself, so ASLR can randomize where the executable is loaded too. That's why `readelf` says `DYN (Position-Independent Executable file)` and why the program's addresses in `/proc/<pid>/maps` look like `0x55...` (randomized), not a fixed `0x400000`.

### Symbol Versioning: "GLIBC_2.34 not found"

glibc keeps old behaviors available by **versioning** its symbols: a binary records that it needs `memcpy@GLIBC_2.14`. If you build on a new distro and run on an old one:

```bash
$ ./prog
./prog: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by ./prog)
$ objdump -T prog | grep GLIBC_ | sort -u -k5 | tail -2     # which versions does it need?
```

Build on the **oldest** system you need to support (or in a matching container), or link statically.

> 🔗 **Connections:** The ELF header declares its endianness (1.5); `LOAD` segments become VMAs with page-table permissions (6.3, 6.4); `GNU_STACK` enables NX for the stack (2.7); `PT_INTERP` names `ld.so`, which `mmap`s shared libraries from the page cache (6.6, 6.7); PIC/PIE makes ASLR possible (4.4); and kernel modules (3.1) and core dumps are ELF files too.

---

# 5. Threads and Concurrency

> 🧪 **Interactive visuals:** [Part 5, visualized](visuals/part5-threads.html): an interactive page with a demo for each section.


## 5.1 Threads

A **thread** is the smallest unit of execution that the OS kernel can schedule.

### Process vs Thread

- **Process** = unit of **resource ownership** (its own isolated virtual address space, file descriptor table, PID).
- **Thread** = unit of **execution / scheduling** (an execution path running inside a process).

| Concept | Description |
| :---- | :---- |
| **Process** | An independent unit of execution with its own memory, resources, and scheduling identity. |
| **Thread** | A lightweight execution unit **within a process**, sharing memory and resources, but **having its own control flow and registers**. |

A **thread** is like a person inside a house: they **share the rooms and keys** (memory, files), but have their **own thoughts and tasks** (registers, execution).

### What Threads Share vs What Is Private

| Shared across threads in a process | Private per thread |
| :---- | :---- |
| Virtual address space (text, data, BSS, heap) | Thread ID (TID) |
| Open file descriptors | Register state (PC/IP, SP, general registers) |
| Signal handlers & signal actions | Execution stack (function calls & local variables) |
| Current working directory & process credentials | Thread-local storage (TLS) & signal mask |

In Linux, both processes and threads are represented by `struct task_struct`. Threads are created with the `clone()` system call using flags like `CLONE_VM`, `CLONE_FILES`, `CLONE_SIGHAND` and `CLONE_THREAD`, sharing resources within a **thread group** (whose `TGID` equals the main thread's PID; this is what `getpid()` returns).

```
 process 4000 (thread group, TGID 4000)
 +----------------------------------------------------------+
 |  task_struct pid=4000   task_struct pid=4001   pid=4002  |   one task_struct per thread
 |       |                      |                   |       |
 |       +----------+-----------+-------------------+       |
 |                  v  (pointers to the SAME objects)        |
 |        mm_struct   files_struct   sighand   fs           |
 +----------------------------------------------------------+
```

```bash
$ ps -T -p $(pgrep -o firefox) | head -4     # -T shows threads (SPID = TID)
    PID    SPID TTY          TIME CMD
   3120    3120 ?        00:02:11 firefox
   3120    3135 ?        00:00:04 gmain
$ ls /proc/3120/task | wc -l                # one directory per thread
```

Each thread needs its own **stack**: `pthread_create` allocates one (8 MiB by default, with a guard page) with `mmap` in the mmap region (4.4), and passes it to `clone()`.

Context switching between threads of the **same process** is cheaper than between processes, because the address space is shared: there's no page-table switch and no TLB flush (3.5).

### User-Level vs Kernel-Level Threads (1:1, N:1, M:N Models)

How many *kernel* tasks back your *language's* threads?

| Model | Mapping | Examples | Pros | Cons |
| :---- | :---- | :---- | :---- | :---- |
| **1:1** | each user thread = one kernel task | Linux **NPTL** (pthreads), Java platform threads, Rust `std::thread` | True parallelism; a blocking syscall blocks only that thread | Each thread costs a kernel task + stack; ~10k threads gets heavy |
| **N:1** | many user threads on one kernel task | Old "green threads", early Python greenlets | Very cheap switches (no kernel) | No parallelism; one blocking syscall blocks all |
| **M:N** | many user threads multiplexed on a pool of kernel threads | **Go goroutines**, Erlang processes, Java **virtual threads** | Millions of cheap tasks + parallelism | Complex runtime scheduler; must avoid blocking kernel threads (uses `epoll`, 7.8) |

Linux itself only knows tasks. A Go program with 100,000 goroutines may only have ~10 kernel threads (`GOMAXPROCS` + some blocked in syscalls).

## 5.2 Concurrency Problems

Concurrency problems occur when multiple threads access shared data at the same time. A **critical section** is a piece of code that accesses a shared resource; if multiple threads enter it at the same time, things break: race conditions, data corruption, unpredictable bugs.

These problems are typical for threads because threads share the same memory, so one unsynchronized access can immediately create a race condition. You don't normally get a race condition just because two separate processes access their own variables, since processes have separate virtual address spaces.

Processes **normally don't share memory**, but they can explicitly share memory/resources (shared memory 9.3, `MAP_SHARED` 6.6, files), and then the same synchronization problems can occur.

### Race Condition

```python
balance = 0

def deposit():
    global balance
    for _ in range(100000):
        balance += 1
```

If two threads run this without protection, `balance` might end up < 200000. Why? `balance += 1` is not one operation; it's **read, add, write**:

```asm
mov  eax, [balance]   ; READ
add  eax, 1           ; ADD
mov  [balance], eax   ; WRITE
```

Suppose `balance = 100` and two threads execute at the same time:

```
Thread A: reads balance -> 100
Thread A: adds 1 -> 101
          (before it writes, Thread B runs: preemption or another core)
Thread B: reads balance -> still 100 (not updated yet!)
Thread B: adds 1 -> 101
Thread B: writes 101 to memory
Thread A: resumes, writes its 101 to memory too

❌ Final result: balance = 101
   Expected:     balance = 102
```

The interleaving can come from a context switch on one core (3.5) or from true parallel execution on two cores. (In CPython the GIL makes this specific example less likely, but the same code in C, Java or Go loses updates easily.)

### Deadlock (and the Four Coffman Conditions)

A **deadlock** is when two or more threads each wait for something another one holds, so none can ever continue.

```
Thread 1:  lock(A) ... lock(B)       Thread 2:  lock(B) ... lock(A)

  T1 holds A, wants B  ----->  B is held by T2
         ^                            |
         |                            v
  A is held by T1  <-----  T2 holds B, wants A          (both sleep forever)
```

A deadlock can only happen if **all four Coffman conditions** hold at once:

1. **Mutual exclusion:** the resource can be held by only one thread.
2. **Hold and wait:** a thread holds one resource while waiting for another.
3. **No preemption:** a resource can't be forcibly taken away.
4. **Circular wait:** a cycle of threads each waiting on the next.

Break **any one** and deadlock is impossible. The practical fix is breaking #4 with a **global lock ordering**: always take A before B, everywhere. The Linux kernel checks this at runtime in debug builds with **lockdep**, which records the order locks are taken and warns about potential cycles before they ever deadlock.

Other approaches: `pthread_mutex_trylock` and back off (breaks #2), timeouts, or one coarse lock.

### Livelock and Starvation

- **Livelock:** threads aren't blocked, they keep *reacting* to each other and make no progress. Like two people in a corridor both stepping aside the same way, forever. Example: two threads that each `trylock` the second lock, fail, release the first, and retry in perfect sync. Fix: randomized backoff.
- **Starvation:** a thread is runnable but **never gets** the resource because others always win. Examples: a writer that never gets a read-write lock because readers keep arriving (5.3), or a low-priority task under a stream of `SCHED_FIFO` tasks (3.6). Fix: fairness (FIFO queuing of waiters, aging priorities).

### Priority Inversion

```
priority:  HIGH  H  wants lock L ....... blocked ......................... runs
           MED   M        starts running, doesn't need L, runs for a long time
           LOW   L  holds lock L ... preempted by M ... can't release L
```

The high-priority task effectively waits for the **medium** one, which has nothing to do with the lock. This famously reset NASA's **Mars Pathfinder** rover in 1997.

Fix: **priority inheritance**. While L holds a lock that H is waiting for, L temporarily runs at H's priority, so M can't preempt it. In Linux: `PTHREAD_PRIO_INHERIT` mutexes, implemented with the kernel's `rt_mutex` and `FUTEX_LOCK_PI`. Note: this needs the lock to have an **owner**, which mutexes have and semaphores don't (5.3).

## 5.3 Synchronization Primitives

### Mutex (Mutual Exclusion)

A mutex prevents two threads from entering a critical section simultaneously.

```python
lock.acquire()
counter += 1
lock.release()
```

Now:

```
T1                 T2
|                  |
🔒 acquire         |
|                  |  (tries to acquire -> sleeps)
counter += 1       |
|                  |
🔓 release         |
                   🔒 acquire
                   |
                   counter += 1
                   |
                   🔓 release

Result: counter = 2
```

**Mutex = one thread can use this resource at a time.**

```c
#include <pthread.h>
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
long counter = 0;

void *worker(void *arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&m);
        counter++;                   // critical section
        pthread_mutex_unlock(&m);
    }
    return NULL;
}
```

### Futex (How Linux Implements Mutexes)

If every `lock()` were a system call, mutexes would be slow (3.3). Linux's answer is the **futex** ("fast userspace mutex"): the lock is just an integer in user memory, and the kernel is involved **only when there's contention**.

```
lock():
   CAS(&lock, 0 -> 1)            succeeded?  -> done. NO SYSCALL (fast path, ~20 ns)
   failed (someone holds it):
       set lock = 2 ("contended")
       futex(&lock, FUTEX_WAIT, 2)   -> kernel puts the thread to sleep (S state)
                                        on a wait queue keyed by the lock's address

unlock():
   old = atomic_exchange(&lock, 0)
   if old == 2:                       someone may be sleeping
       futex(&lock, FUTEX_WAKE, 1)    -> kernel wakes one waiter
```

Every `pthread_mutex`, `pthread_cond`, semaphore, Go `sync.Mutex` slow path, and Java lock on Linux sits on top of futexes. `strace` on a contended program shows many `futex(...)` calls; an uncontended one shows none.

> 🔗 **Connections:** Futexes combine everything: an atomic CAS instruction (below) on a cache line (MESI, 2.4), a system call (3.3) only on contention, a sleep in state `S` (4.8), and a wakeup that puts the thread back on a run queue (3.6). Because the kernel keys the wait queue by the *physical* page + offset for shared futexes, they also work across processes in shared memory (9.3).

### Atomic Operations

Instead of three separate steps:

```
READ
ADD
WRITE
```

you have an operation that the CPU guarantees happens atomically. Conceptually:

```c
atomic_increment(&counter);
```

Then:

```
T1 -> atomic +1
T2 -> atomic +1
result = 2
```

**Atomic = "don't let anyone observe this operation halfway through."**

Mutexes can protect larger pieces of code; atomic operations are particularly useful for simple shared state (counters, flags, reference counts like `struct page->_refcount`, 6.2).

```c
#include <stdatomic.h>
atomic_long counter = 0;
atomic_fetch_add(&counter, 1);       // compiles to: lock xadd [counter], 1
```

#### Compare-and-Swap (CAS) and the x86 `lock` Prefix

**CAS** is the building block of almost all lock-free code:

```c
// atomically: if (*addr == expected) { *addr = desired; return true; } else return false;
bool atomic_compare_exchange_strong(atomic_long *addr, long *expected, long desired);

// lock-free increment with a CAS loop:
long old = atomic_load(&counter);
while (!atomic_compare_exchange_weak(&counter, &old, old + 1))
    ;   // someone else changed it; 'old' now holds the new value, retry
```

On x86 it's the `lock cmpxchg` instruction. The **`lock` prefix** makes a read-modify-write instruction atomic across all cores: the core acquires the cache line in **Modified/Exclusive** state (2.4) and refuses to give it up until the instruction completes. That's why atomics are cheap when uncontended (~20 cycles) but expensive when many cores hammer the same variable (the line ping-pongs, like false sharing, 1.8). ARM uses `LDXR`/`STXR` (load-linked/store-conditional) or the newer `CAS`/`LDADD` instructions.

#### Memory Ordering and Memory Barriers

Even with atomic instructions, **the order in which other cores see your writes** is not necessarily program order. CPUs reorder memory operations (store buffers, out-of-order execution, 2.2) and compilers reorder too.

```c
// Thread 1 (producer)          // Thread 2 (consumer)
data  = 42;                     while (ready == 0) ;
ready = 1;                      print(data);       // can this print 0?
```

- **x86** is relatively strong (**TSO**, Total Store Order): only *store → later load* can be reordered. The example above works on x86 (if the compiler doesn't reorder).
- **ARM, POWER, RISC-V** are **weakly ordered**: `ready = 1` may become visible before `data = 42`, so the consumer can print 0.

Fix: tell both the compiler and the CPU about ordering with **barriers**, or with **acquire/release** atomics:

```c
// producer
data = 42;
atomic_store_explicit(&ready, 1, memory_order_release);   // all writes before this are visible first

// consumer
while (atomic_load_explicit(&ready, memory_order_acquire) == 0) ;
print(data);   // guaranteed 42
```

Hardware barriers: x86 `mfence`; ARM `dmb`. The kernel uses `smp_mb()`, `smp_wmb()`, `smp_rmb()`, `READ_ONCE()`/`WRITE_ONCE()`. Mutex lock/unlock include the necessary barriers, which is one more reason to prefer them unless you really know you need lock-free code.

### Semaphore

A semaphore is a counter that controls access to a limited number of resources. For example, a semaphore initialized to 3 allows up to three threads to access a resource simultaneously.

```
sem = 3        (3 database connections)

wait()/P():  if sem > 0: sem -= 1, continue      else: sleep
post()/V():  sem += 1, wake one sleeper

T1 wait -> 2    T2 wait -> 1    T3 wait -> 0    T4 wait -> sleeps
T1 post -> T4 wakes up and takes the slot
```

```c
#include <semaphore.h>
sem_t pool;
sem_init(&pool, 0, 3);   // 0 = shared between threads (1 = between processes, in shared memory)
sem_wait(&pool);  /* use a connection */  sem_post(&pool);
```

#### Mutex vs Binary Semaphore (Ownership)

A semaphore initialized to 1 *looks* like a mutex, but:

| | Mutex | Binary semaphore |
| :---- | :---- | :---- |
| Ownership | Has an **owner**: only the thread that locked it may unlock it | No owner: **any** thread may `post` |
| Typical use | Protect a critical section | **Signal** between threads ("an item is ready") |
| Priority inheritance | Possible (the kernel knows whom to boost) | Impossible (no owner to boost) |
| Recursive locking, error checking | Supported (`PTHREAD_MUTEX_RECURSIVE`, `ERRORCHECK`) | No |

Rule of thumb: **mutex for mutual exclusion, semaphore for counting/signaling.**

### Condition Variable

A condition variable lets a thread sleep until a condition becomes true.

**Without a condition variable:**

```
Queue = empty

Consumer:
    while queue is empty:
        keep checking 😐
```

The consumer wastes CPU constantly checking.

**With a condition variable:**

```
Consumer:
    lock mutex
    while queue is empty:
        wait()        # atomically releases the mutex and sleeps
    take item
    unlock mutex

Producer:
    lock mutex
    add item to queue
    signal()
    unlock mutex
```

The consumer waits instead of repeatedly checking the buffer. When the thread wakes up, it must **check the condition again** (hence `while`, not `if`), because waking up does not guarantee the condition is still true: another consumer may have taken the item first, or the wakeup may be spurious.

```c
pthread_mutex_lock(&m);
while (queue_empty(&q))
    pthread_cond_wait(&cv, &m);     // releases m, sleeps (futex), re-acquires m on wakeup
item = queue_pop(&q);
pthread_mutex_unlock(&m);
```

### Spinlock

A **spinlock** is a lock where a thread **keeps checking repeatedly** until the lock becomes available.

**Mutex:** "It's locked → I'll sleep." 😴 **Spinlock:** "It's locked → I'll keep checking." 🔄

Suppose Thread A has the lock:

```
Lock = LOCKED
Thread A -> using resource
Thread B -> wants resource
```

With a spinlock:

```
Thread B:
    while lock is locked:
        keep checking 🔄

Thread A finishes
      |
Lock = UNLOCKED
      |
Thread B gets lock
      |
Thread B works
```

Why would you do this? Imagine the lock will only be held for **1 microsecond**. With a mutex:

```
B -> sleep
  -> scheduler switches
  -> A releases lock
  -> B wakes up
```

The sleeping/waking and context switching can cost more time than simply waiting. With a spinlock:

```
B -> 🔄🔄🔄🔄
        |
     lock free
        |
     continue
```

So spinlocks make sense when the **wait is expected to be extremely short**, and only on multi-core systems (on a single core, the lock holder can't run while you spin).

Inside the Linux kernel, spinlocks are everywhere, because some code **cannot sleep at all**: interrupt handlers (3.4) have no task context to put to sleep. A kernel spinlock also disables preemption on that CPU while held (and `spin_lock_irqsave` disables local interrupts too, so an interrupt handler on the same CPU can't deadlock trying to take the same lock). In user space, plain spinlocks are rarely a good idea; glibc's adaptive mutexes and futexes already spin briefly before sleeping.

### Read-Write Locks

Many workloads read shared data far more often than they write it. A **read-write lock** allows:

- **many readers at once**, or
- **exactly one writer** (and no readers).

```
readers:  R1 R2 R3 ----reading together----|      |---- R4 R5
writer:                              W waits | W  |
```

```c
pthread_rwlock_rdlock(&rw);   /* read config */   pthread_rwlock_unlock(&rw);
pthread_rwlock_wrlock(&rw);   /* update config */ pthread_rwlock_unlock(&rw);
```

Pitfall: **writer starvation** (5.2). If readers keep arriving, the writer may never get in; most implementations can be configured to prefer writers.

The kernel goes further for read-mostly data:

- **seqlock:** readers don't lock at all; they read a sequence number before and after, and retry if a writer changed it (used for the time-of-day data the vDSO reads, 3.3).
- **RCU** (Read-Copy-Update): readers take no lock and never wait; writers make a **new copy**, publish it with one pointer swap, and free the old copy after all pre-existing readers are done. Used for routing tables, the dentry cache (7.3), and task lists.

> 🔗 **Connections:** Threads share `mm_struct`, which is why every concurrency problem here exists (illusion 2 is shared). The primitives are built bottom-up: atomic instructions (cache coherence, 2.4) → futex (syscall only on contention, 3.3) → mutex/condvar/semaphore → your code. The same problems reappear between processes with shared memory (9.3) and inside the kernel (spinlocks, RCU).

---

# 6. Memory Management

> 🧪 **Interactive visuals:** [Part 6, visualized](visuals/part6-memory.html): an interactive page with a demo for each section.


## 6.1 Physical Memory

Physical memory (RAM) is the actual hardware (DRAM chips). It's shared among all processes. The OS needs to ensure:

- **Isolation:** no process can access another's memory
- **Protection:** memory errors in one process don't corrupt others
- **Fair use:** memory is distributed efficiently

RAM is divided into **fixed-size blocks called page frames**.

- On most systems, a page frame is **4 KB** (16 KB on Apple Silicon).
- Physical memory is just a big array of these blocks:

```
RAM
+-------------+-------------+-------------+-------------+
| Frame #0    | Frame #1    | Frame #2    | Frame #3    | ...
+-------------+-------------+-------------+-------------+
```

- The kernel tracks which frames are free and which are used.
- When a process needs memory, the OS finds free frames and assigns them.
- For every frame, the kernel keeps a `struct page` descriptor that tracks its status (is it free, is it kernel code, does it belong to a user process, etc.).

Think of RAM as a cake cut into equal slices (frames). Each slice has a label (`struct page`) that says whether it's available, who it belongs to, its current state, and how many things are using it.

**Why is this design important?**

Because all core memory operations (**allocation, mapping, page faults, swapping, reclaiming, freeing, I/O**) require:

- **Locating the frame** via its PFN (Page Frame Number)
- **Accessing `struct page`** for bookkeeping and decisions
- **Updating `struct page`** based on usage (e.g., mapped, dirty, free)

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       5.1Gi       1.2Gi       412Mi       9.3Gi       9.8Gi
Swap:          2.0Gi          0B       2.0Gi
```

"free" is small and "buff/cache" is big. That's healthy: unused RAM is wasted RAM, so the kernel uses it as page cache (6.7). **"available"** is the number that matters: free + cache that can be dropped.

## 6.2 `struct page`: One Physical Page Frame

Every physical page has a corresponding `struct page` (simplified):

```c
struct page {
    unsigned long flags;           // Status flags (dirty, locked, reserved, LRU, ...)
    atomic_t _refcount;            // Reference count [before Linux 4.6: _count]
    atomic_t _mapcount;            // How many page tables map this page
    struct address_space *mapping; // File mapping if any (mmap, page cache)
    pgoff_t index;                 // Offset within the file
    struct list_head lru;          // For LRU or free lists
    unsigned long private;         // Private data for specific subsystems
    unsigned long compound_head;   // For compound/huge pages
    ...
};
```

Conceptually, all `struct page` instances form one big array indexed by physical frame number:

```c
phys_addr = 0x12345000  -> frame number (PFN) = 0x12345      // phys_addr >> 12
                        -> struct page *page = pfn_to_page(0x12345)
```

[The single flat `mem_map[]` array, where `&mem_map[pfn]` is the page, is the **FLATMEM** model used by older/simpler systems. Most x86-64 kernels today use **SPARSEMEM_VMEMMAP**, so code uses `pfn_to_page()` / `page_to_pfn()`, which work for every model.]

[Since Linux 5.16, much of the memory-management code works on **folios** (a group of one or more contiguous pages managed as a unit) instead of individual `struct page`s.]

When Linux boots:

1. It detects how much **physical RAM** is available (from the firmware memory map, 3.7).
2. It divides RAM into **page frames** (usually 4 KB).
3. For *each* physical page, it creates a **`struct page`**, a metadata structure in memory (~64 bytes each, so ~1.6% of RAM).
4. These `struct page`s track the **state** of every page: free, allocated, swapped, used by the slab cache, mapped to a file, etc.

**Important:** `struct page` is **not** stored inside `task_struct`. A process doesn't own frames directly; it owns *mappings* (page tables, 6.3) that point to frames. One frame can be mapped by many processes (shared libraries, COW after fork), and `_mapcount` counts them.

### Buddy Allocator

To manage free pages efficiently, the Linux kernel uses the **buddy system**. It hands out **physically contiguous** blocks of pages in power-of-two sizes.

- Physical memory is divided into **zones** (e.g., DMA, Normal).
- Each zone has **free lists** of page blocks grouped by **order** (power of 2 sizes):

```
Order 0  -> 1 page     (4 KB)
Order 1  -> 2 pages    (8 KB)
Order 2  -> 4 pages    (16 KB)
...
Order 10 -> 1024 pages (4 MB)
```

When you need memory:

- The buddy allocator tries to find the **smallest fitting block**.
- If no block of that order is available, it **splits** a higher-order block recursively.

```
need 1 page (order 0), only an order-2 block (4 pages) is free:

[        order 2: pages 0-3        ]
        split
[ order 1: 0-1 ][ order 1: 2-3 ]       2-3 goes to the order-1 free list
        split
[ o0: 0 ][ o0: 1 ]                     1 goes to the order-0 free list, 0 is returned
```

When a page is freed:

- The system checks if its "**buddy**" (the neighbor block of the same size it was split from; its address differs in exactly one bit) is also free.
- If so, it merges them (coalesces) into a bigger block, and repeats upward, to reduce fragmentation.

Buddy data is stored in:

- `struct zone → free_area[order]` for each order.
- Each entry has **linked lists** of free blocks (`struct page`, linked through `lru`).

```bash
$ cat /proc/buddyinfo
Node 0, zone      DMA      0      0      0      0      0      0      0      0      1      1      3
Node 0, zone    DMA32      5      4      6      5      7      6      4      5      4      5    410
Node 0, zone   Normal   3021   1810    902    411    210     95     41     12      4      2   1123
#                       ord0   ord1   ord2   ...                                           ord10
```

Few high-order blocks = **fragmented** memory: plenty free in total, but no contiguous 2 MB chunk for a huge page (6.9). The kernel runs **compaction** (moving pages around) to fix that.

### Slab / SLUB Allocator

The buddy allocator deals in pages (4 KB minimum). But the kernel constantly allocates **small objects**: a `task_struct` (~10 KB), an `inode` (~600 B), a `dentry` (~200 B), a `struct file` (~250 B). Using a whole page for each would waste memory, and initializing them from scratch each time would waste CPU.

The **slab allocator** (today's implementation is **SLUB**) takes pages from the buddy allocator and carves them into **caches of same-sized objects**:

```
buddy allocator  --pages-->  slab cache "task_struct"   [obj][obj][obj][obj] ...
                             slab cache "dentry"        [o][o][o][o][o][o][o] ...
                             slab cache "kmalloc-64"    generic 64-byte objects
```

- Allocation and free are very fast (per-CPU free lists, no locking on the fast path).
- `kmalloc(size)` picks the generic cache of the next power of two; `kmem_cache_alloc()` uses a dedicated cache.

```bash
$ sudo slabtop -o | head -8
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME
 912000 911873  99%    0.19K  43429       21    173716K dentry
 402116 401552  99%    1.15K  14897       27    476704K ext4_inode_cache
```

> 🔗 **Connections:** The "dentry cache" and "inode cache" of the VFS (7.3) are literally slab caches. That's why after `find /` your "used" memory goes up: the kernel cached millions of dentries and inodes. They're reclaimable (`SReclaimable` in `/proc/meminfo`), like the page cache.

### Memory Zones (DMA, Normal) and NUMA Nodes

Not all physical memory is equal, so RAM is divided into **zones**:

| Zone | Range (x86-64) | Why it exists |
| :---- | :---- | :---- |
| `ZONE_DMA` | first 16 MB | Ancient ISA devices could only DMA (2.5) to 24-bit addresses |
| `ZONE_DMA32` | below 4 GB | Devices with 32-bit DMA addressing |
| `ZONE_NORMAL` | everything else | Normal allocations |
| `ZONE_HIGHMEM` | (32-bit only) above ~896 MB | RAM that didn't fit in the kernel's 1 GB window (1.4) |
| `ZONE_MOVABLE` | configurable | Only movable pages, so memory can be hot-unplugged or compacted |

**NUMA** (Non-Uniform Memory Access): on multi-socket servers, each CPU socket has its own local RAM. Accessing the other socket's RAM is slower (maybe 1.5–2×). Linux models each socket's memory as a **node**, each with its own zones, and by default allocates memory on the node of the CPU that requested it ("first touch").

```bash
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0-15        node 0 size: 128000 MB
node 1 cpus: 16-31       node 1 size: 128000 MB
node distances:
node   0   1
  0:  10  21             <- remote access ~2.1x the cost
$ numactl --cpunodebind=0 --membind=0 ./db    # keep CPU and memory on one node
```

> 🔗 **Connections:** NUMA is why the scheduler is reluctant to move tasks across nodes (3.6): the task's memory stays behind on the old node.

## 6.3 Virtual Memory

The fundamental problem Linux is solving is:

**A process uses virtual addresses, but RAM uses physical addresses. How do we translate between them safely and efficiently?**

**Why do we need virtual memory if the kernel already tracks every physical frame?** The problem isn't what the **kernel** knows; it's what **processes** are allowed to see and do. Giving a process raw physical addresses is like giving a stranger a master key to every apartment in a building. Even if the building manager has a perfect list of who lives where, nothing stops that person from walking into your apartment. Without virtual memory, any process could read and write the memory of any other process, and even of the kernel. Constant crashes, zero security.

Imagine 3 processes all using RAM addresses directly. They'd be fighting over the same physical memory, and a buggy process could write into memory belonging to another process. That's why each process gets its own **illusion of memory** (illusion 2, 0.1). That's virtual memory.

Virtual memory enables process isolation, efficient allocation, shared libraries, memory-mapped files, copy-on-write, file caching, swapping, demand paging, and memory protection.

### Virtual to Physical Translation

Programs use virtual addresses. The CPU and Linux translate them into physical addresses through page tables:

```
Process virtual address -> page tables -> physical RAM frame
```

Each process normally has its own virtual address space. Two processes can use the same virtual address while referring to different physical memory. Page-table permissions prevent one process from accessing another process's memory. This illusion is managed by the Linux kernel together with the CPU's **MMU**.

```
process A: 0x601000 --A's page table--> frame #812
process B: 0x601000 --B's page table--> frame #77      same virtual address, different RAM
```

Benefits:

- Simplifies programming: every app assumes it has the full address space
- Allows memory to be non-contiguous physically (helps with fragmentation)
- Enables demand paging, swapping and shared memory

**My physical frames are scattered, but because my virtual addresses are contiguous, my process sees a single, unbroken block of memory.**

To make translation manageable, both spaces are divided into the same fixed-size blocks:

- **Pages (virtual memory):** the virtual address space is divided into **pages** (4 KB; also 2 MB / 1 GB huge pages, 6.9).
- **Frames (physical memory):** RAM is divided into **page frames** of the same size.
- **The split:** the MMU interprets a virtual address as two parts:
  - **Virtual Page Number (VPN):** the high bits; identifies *which* page.
  - **Offset:** the low 12 bits (for 4 KB pages, 2¹² = 4096); the byte *within* the page. **The offset is never translated**, it's copied as is.

```
Virtual Address -> (Page Number + Offset) -> Page Table -> (Frame Number + Offset) -> Physical Address

Virtual Page #0x123 -> Physical Frame #42
Virtual Page #0x124 -> Physical Frame #99
```

A (simplified, single-level) page table would look something like this:

| Virtual Address | Maps To |
| :---- | :---- |
| 0x00000000 | Physical Frame #58 |
| 0x00001000 (next 4 KB) | Physical Frame #59 |
| 0x00002000 (next 4 KB) | Physical Frame #120 |
| 0x00003000 (next 4 KB) | Physical Frame #800 |
| 0x00004000 (next 4 KB) | Physical Frame #801 |
| 0x00005000 (next 4 KB) | Physical Frame #802 |
| ... | ... |

In reality, x86-64 page tables are **multi-level** (4 levels, or 5 with LA57), and each level is one 4 KB page holding **512 entries** (4096 bytes / 8-byte entries).

### Multi-Level Page Tables (PGD → P4D → PUD → PMD → PTE)

**Why not one flat table?** A 48-bit address space with 4 KB pages has 2³⁶ pages. A flat table with 8-byte entries would be 2³⁶ × 8 = **512 GB per process**. But processes use only a tiny, sparse part of their address space. A **tree** only needs nodes for regions actually in use: a small process needs just a handful of 4 KB table pages.

```
VIRTUAL ADDRESS BREAKDOWN (48-bit VA, 4 levels)
| 63..48  | 47..39 | 38..30 | 29..21 | 20..12 |  11..0  |
| sign-ext| 9 bits | 9 bits | 9 bits | 9 bits | 12 bits |
|(1.4)    |  PGD   |  PUD   |  PMD   |  PTE   | Offset  |
```

- **PGD** → Page Global Directory
- **P4D** → Page 4th Directory (only real with 5-level paging, 57-bit VA, 128 PB; otherwise "folded" into the PGD)
- **PUD** → Page Upper Directory
- **PMD** → Page Middle Directory
- **PTE** → Page Table Entry

9 bits per level → 2⁹ = 512 entries per table → 512 × 8 bytes = exactly one 4 KB page per table. Everything fits together.

```
CR3 --> [ PGD ] --idx--> [ PUD ] --idx--> [ PMD ] --idx--> [ PTE table ] --idx--> frame base
                                                                                    + offset
                                                                                    = physical address
```

#### Worked Example: Walking `0x7f8010203456`

A standard 48-bit virtual address on x86-64 is written with 12 hex digits. First the MMU splits the address into its five parts according to the 9-9-9-9-12 rule.

**Virtual address:** `0x7f80'1020'3456`

**In binary:** `0111 1111 1000 0000 0001 0000 0010 0000 0011 0100 0101 0110`

Regrouped as 9 | 9 | 9 | 9 | 12 bits:

```
011111111 | 000000000 | 010000001 | 000000011 | 010001010110
   PGD         PUD         PMD         PTE          offset
```

| Part | Bits | Binary value | Hex | Decimal |
| :---- | :---- | :---- | :---- | :---- |
| PGD index | 47–39 | `011111111` | 0x0FF | **255** |
| PUD index | 38–30 | `000000000` | 0x000 | **0** |
| PMD index | 29–21 | `010000001` | 0x081 | **129** |
| PTE index | 20–12 | `000000011` | 0x003 | **3** |
| Offset | 11–0 | `0100 0101 0110` | 0x456 | **1110** |

(Bit 47 is `0`, so this is a user-space address, and the PGD index is in the lower half, 0–255. Kernel addresses use PGD entries 256–511, and those entries are the same in every process: that's how the kernel is mapped everywhere, 4.4.)

The MMU now performs the page walk in hardware. Let's assume the `CR3` register for our process points to the physical address `0x1000` (the start of our PGD table).

1. **To the PGD (level 4):**
   - The MMU starts at the PGD base address from `CR3`: `0x1000`.
   - It uses the **PGD index (255)** to read entry 255 (at `0x1000 + 255 × 8`).
   - Let's say this entry contains the physical base address of the next table (the PUD): **`0x8000`**.
2. **To the PUD (level 3):**
   - The MMU goes to the PUD base address it just found: `0x8000`.
   - It uses the **PUD index (0)** to read entry 0.
   - Let's say this entry contains the physical base address of the PMD: **`0x9A000`**.
3. **To the PMD (level 2):**
   - The MMU goes to the PMD base address: `0x9A000`.
   - It uses the **PMD index (129)** to read entry 129.
   - Let's say this entry points to the final PTE table at physical address: **`0x7B000`**.
4. **To the PTE (level 1):**
   - The MMU goes to the PTE table base address: `0x7B000`.
   - It uses the **PTE index (3)** to read entry 3. This is the Page Table Entry!
   - This entry contains the physical address of the data page frame we've been searching for. Let's say it holds the physical frame address **`0x12345000`** (plus flag bits, below).
5. **Final calculation:**
   - The MMU takes the physical frame address from the PTE: `0x12345000`.
   - It adds the **offset** from our original virtual address: `0x456`.
   - `0x12345000 + 0x456 = 0x12345456`

Four memory reads just to find where to do one memory read. That's why the **TLB** (2.4) caches the final result `VPN 0x7f8010203 → PFN 0x12345`. If any entry along the way is **not present**, the MMU raises a **page fault** (6.5).

Each process has its own PGD. **Changing `CR3` changes the whole address space**, and that's exactly what a context switch between processes does (3.5).

### Page Table Entry Flags (Present, Writable, User, NX, Dirty, Accessed)

A 64-bit PTE is a frame address plus control bits. These bits are where the hardware enforces everything the kernel decides:

```
 63  62..52  51 ................................. 12  11..9  8  7  6  5  4  3  2  1  0
[NX][ ... ][        physical frame number         ][ sw ][G][PS][D][A][..][..][U][W][P]
```

| Bit | Name | Meaning | Who uses it |
| :---- | :---- | :---- | :---- |
| 0 | **P** Present | Mapping is valid. If 0 → page fault. | Demand paging, swap (6.5, 6.8) |
| 1 | **R/W** Writable | 0 = read-only; a write → page fault | Copy-on-write (6.5), `.text` protection |
| 2 | **U/S** User/Supervisor | 0 = kernel only; Ring 3 access → fault | Protects kernel memory (2.6) |
| 5 | **A** Accessed | Set by hardware on any access | Reclaim: "was this page used recently?" (6.8) |
| 6 | **D** Dirty | Set by hardware on write | Writeback: "must this page be saved before freeing?" (6.7) |
| 7 | **PS** Page size | In a PMD/PUD entry: this is a 2 MB / 1 GB huge page | Huge pages (6.9) |
| 8 | **G** Global | Don't flush from TLB on `CR3` switch | Kernel mappings |
| 63 | **NX** No-execute | Code can't run from this page | Stack/heap can't execute injected code |

These bits map directly to the `rwx` permissions you see in `/proc/<pid>/maps` (4.4). A **segmentation fault** is just: a page fault where the kernel finds the access wasn't allowed by the VMA (6.5).

## 6.4 Connection to Processes: `task_struct` → `mm_struct` → VMAs

As explained in 4.2, a process is represented internally by `struct task_struct`. This struct stores **process-specific state**, including memory:

```
task_struct
 |-- mm -> mm_struct (memory descriptor)        // user-space memory
 |         |-- pgd   (pointer to the top-level page table)  -> loaded into CR3
 |         |-- mm_mt (maple tree of VMAs)       [before Linux 6.1: mmap, a linked list of VMAs + an rbtree]
 |         `-- other memory-related data
 `-- active_mm                                  // used when kernel threads borrow an mm

vm_area_struct (VMA)
 |-- vm_start
 |-- vm_end
 |-- vm_flags   (VM_READ, VM_WRITE, VM_EXEC, VM_SHARED, ...)
 |-- vm_file    (if file-backed)
 `-- vm_ops     (callbacks, e.g. how to fault in a page)
                                                [before Linux 6.1: also vm_next / vm_prev list pointers]
```

- Every process in Linux has a `task_struct`, the main process descriptor.
- Inside `task_struct`, there's a pointer to `mm_struct` (called `mm`).
- `mm_struct` holds all **memory-management info** for the process, including the **page tables** (via `pgd`) and the set of **VMAs**: each VMA describes one contiguous virtual memory region (e.g., the text segment, the heap, a mapped library) and its permissions.
- All threads of a process point to the **same** `mm_struct` (5.1). Kernel threads have `mm = NULL`.

### Inside `mm_struct`

This is the master memory descriptor: it describes the full memory layout of a process.

```c
struct mm_struct {
    struct maple_tree mm_mt;      // All VMAs  [before 6.1: struct vm_area_struct *mmap]
    pgd_t *pgd;                   // Top-level page table (Page Global Directory)
    unsigned long start_code, end_code;    // .text
    unsigned long start_data, end_data;    // .data
    unsigned long start_brk, brk;          // heap: start and current end  (6.6)
    unsigned long start_stack;
    atomic_t mm_users;            // How many threads use this mm
    ...
};
```

The page tables live in ordinary RAM frames; the CPU only knows the root, through `CR3`. On x86-64, when the kernel switches to another process, it loads that process's `mm->pgd` (converted to a physical address) into `CR3`.

### Inside `vm_area_struct`

Page tables are a raw, page-by-page translation map. They don't give a high-level view of the address space ("this range is the heap", "this range is libc's code"). That's what VMAs are for.

A VMA represents a **continuous region of virtual memory with the same properties**: start/end address, permissions, whether it's backed by a file, whether it's private or shared.

```c
struct vm_area_struct {
    unsigned long vm_start, vm_end;       // Virtual address range [start, end)
    struct mm_struct *vm_mm;              // Back-pointer to mm_struct
    unsigned long vm_flags;               // Access permissions (R/W/X), shared/private
    struct file *vm_file;                 // For file-backed mmap (NULL for anonymous)
    unsigned long vm_pgoff;               // Offset into that file, in pages
    const struct vm_operations_struct *vm_ops; // Callbacks (e.g. ->fault for lazy loading)
};
```

VMAs **break a process's virtual address space into meaningful, manageable regions**, allowing Linux to efficiently enforce permissions, handle page faults, and support dynamic memory operations.

The key distinction:

| | VMA | Page table |
| :---- | :---- | :---- |
| Who reads it | The **kernel** (software) | The **MMU** (hardware) |
| Granularity | Regions (any number of pages) | Individual pages |
| Describes | What is **allowed** and where data **comes from** | What is **currently mapped** in RAM |
| Created by | `execve`, `mmap`, `brk` | Lazily, by the page-fault handler |

A VMA can exist with **zero** page-table entries filled in. That's lazy allocation (6.5).

### `/proc/<pid>/maps` and `smaps`

`/proc/<pid>/maps` prints the VMA list (4.4 has a full example). `/proc/<pid>/smaps` adds per-VMA statistics from the page tables:

```bash
$ grep -A6 '\[heap\]' /proc/self/smaps
55d0c2f4e000-55d0c2f6f000 rw-p 00000000 00:00 0   [heap]
Size:                132 kB     <- VMA size (virtual)
Rss:                  12 kB     <- actually in RAM right now (resident)
Pss:                  12 kB     <- Rss with shared pages divided among sharers
Shared_Clean:          0 kB
Private_Dirty:        12 kB
Swap:                  0 kB
```

- **VSZ / Size** (virtual): how much address space is reserved. Can be huge and mean nothing.
- **RSS** (resident): pages in RAM. Counts shared libraries fully in *every* process, so summing RSS over-counts.
- **PSS** (proportional): shared pages divided by the number of sharers. The fairest "how much does this process cost".

`pmap -x <pid>` shows the same in a table.

## 6.5 Page Faults

A **page fault** is an exception (vector 14, `#PF`, a *fault* in the 3.4 sense: restartable) raised by the MMU when a process tries to access a virtual address that:

- has no corresponding valid **Page Table Entry** (`P = 0`), or
- violates the permission bits (write to read-only, user access to kernel page, execute an NX page), or
- maps to a non-resident page (e.g., swapped out or lazy-allocated).

Page faults are **normal**. Most of them are the kernel's lazy strategies working as designed. The CPU puts the faulting address in the `CR2` register and the kernel's handler (`exc_page_fault()` → `handle_mm_fault()`) decides:

```
CPU accesses virtual address X
        |
   TLB hit? --yes--> done
        | no
   page walk: PTE present & permissions OK? --yes--> fill TLB, done
        | no
   #PF exception -> kernel: find_vma(current->mm, X)
        |
   X inside a VMA?  --no-->  SIGSEGV  ("segmentation fault")
        | yes
   access allowed by vm_flags? (write to r-x?)  --no--> SIGSEGV
        | yes
   why wasn't it mapped?
     - never touched, anonymous     -> allocate a zeroed frame               (minor)
     - file-backed, in page cache   -> map the cached page                   (minor)
     - file-backed, not cached      -> read from disk, task sleeps           (MAJOR)
     - swapped out                  -> read from swap, task sleeps           (MAJOR)
     - write to a COW page          -> copy the page, map the copy R/W       (minor)
        |
   update the PTE, return from exception -> the SAME instruction is retried and now succeeds
```

### Minor vs Major Page Faults

- **Minor fault:** resolved **without I/O**: the data is already in RAM (page cache, a zero page, a COW copy). Costs ~1 µs.
- **Major fault:** requires **reading from disk** (file not in page cache, or swapped-out page). The task sleeps in `D` state; costs 10 µs (NVMe) to 10 ms (HDD) (2.4 latency table).

```bash
$ /usr/bin/time -v ./prog 2>&1 | grep -i fault
   Major (requiring I/O) page faults: 3
   Minor (reclaiming a frame) page faults: 1825
$ ps -o pid,min_flt,maj_flt,cmd -p 1234
```

A program whose **major** fault count keeps climbing is thrashing: its working set doesn't fit in RAM (6.8).

### Demand Paging

**Nothing is loaded until it's touched.** When you `execve` a 100 MB binary or `malloc` 1 GB, the kernel only creates **VMAs** (the promises). Physical frames are allocated one page at a time, when the first access faults.

```
User runs ./a.out
|
execve() loads ELF -> sets up mm_struct, VMAs    (no file data read yet)
|
No physical memory yet (lazy)
|
CPU starts executing -> accesses 0x00401000
|
MMU -> TLB miss -> page table walk -> not present -> PAGE FAULT
|
Kernel: inside the .text VMA, file-backed -> get the page from the page cache
        (reading it from disk if needed) -> update the page table
|
Return from fault -> instruction retried -> MMU caches it in the TLB -> execution continues
```

```c
char *p = malloc(1UL << 30);     // 1 GiB: RSS barely changes, just a VMA
for (size_t i = 0; i < (1UL << 30); i += 4096)
    p[i] = 1;                    // 262,144 minor faults; RSS grows to 1 GiB
```

Benefits: fast program startup, no RAM wasted on code paths never executed, and it's what makes **overcommit** (6.8) possible.

### Copy-on-Write Faults

After `fork()` (4.9), parent and child share all frames, with the PTEs marked **read-only** even though the VMA says writable. When either writes:

1. `#PF`: write to a read-only PTE.
2. The kernel sees the VMA allows writing → this is COW, not an error.
3. If the frame's `_mapcount` shows it's still shared: allocate a new frame, copy 4 KB, point *this* process's PTE at the copy, mark it writable.
4. If it's no longer shared (the other process already copied or exited): just mark it writable, no copy.
5. Retry the instruction.

That "is the VMA writable?" check is exactly how the kernel tells a COW fault apart from a real bug (a write to `.text`, which gets `SIGSEGV`).

## 6.6 Memory Mapping (`mmap`)

`mmap()` asks the kernel to create a new VMA in your address space:

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

- `prot`: `PROT_READ | PROT_WRITE | PROT_EXEC` → VMA flags → PTE bits
- `flags`: `MAP_PRIVATE` or `MAP_SHARED`, optionally `MAP_ANONYMOUS`
- `fd` + `offset`: which file and where in it (for file-backed mappings)

It returns immediately: no data is read, no frames are allocated. Everything happens later through page faults (6.5).

### File-Backed vs Anonymous Mappings

- **File-backed:** the VMA is backed by a file. A page fault fetches the page from the **page cache** (6.7), and the process accesses the file's bytes as plain memory, with no `read()` calls and no copying into a user buffer. Executables and shared libraries are mapped this way.
- **Anonymous:** not backed by any file (`MAP_ANONYMOUS`, `fd = -1`). Pages start as zeros. Heap, stack, BSS, big `malloc`s. If the system needs the RAM back, they can only go to **swap** (6.8).

### Private vs Shared Mappings

| | `MAP_PRIVATE` | `MAP_SHARED` |
| :---- | :---- | :---- |
| **File-backed** | Reads see the file; writes are **COW copies** visible only to you, never written to the file. Used for a program's `.text` and `.data`, and libraries. | Writes go to the **page cache** → visible to every process mapping the file, and written back to disk eventually. Used for databases, shared files, IPC. |
| **Anonymous** | Private zeroed memory. `malloc`, heap, stack. After `fork`, COW. | Memory shared between a parent and its children after `fork` (stays shared, no COW). |

```c
// Read a whole file without read(): map it.
int fd = open("data.bin", O_RDONLY);
struct stat st; fstat(fd, &st);
char *p = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
close(fd);                     // the mapping keeps its own reference to the file
printf("first byte: %c\n", p[0]);   // page fault -> page cache -> mapped
munmap(p, st.st_size);
```

Touching a mapped page beyond the end of the file gives **`SIGBUS`**, not `SIGSEGV`: the address is inside the VMA, but there's no file data to back it.

### `malloc()`, `brk()` and `mmap()`

- `malloc()` is a **library function** (part of the C library, like glibc). It lives in **user space**. Its goal is to be fast and efficient for requests of any size. It acts as a manager: it keeps free lists of chunks inside memory it already obtained, and only asks the kernel for more when needed:
  - **`brk()`/`sbrk()`** to grow the heap, for small allocations
  - **`mmap()`** for large allocations (≥ `M_MMAP_THRESHOLD`, 128 KiB by default, adjusted dynamically)
- `brk()` and `mmap()` are **system calls**. They are the actual requests to the kernel to change the process's address space.
  - `mmap()` **creates** a brand-new `vm_area_struct`.
  - `brk()` **moves the end** of the existing heap VMA (expands or shrinks it).
  - That's **all** they do at the time of the call: they update the blueprint of the address space. Physical memory arrives only on page fault.

So the hierarchy is: `task_struct` → `mm_struct` → `vm_area_struct`(s), and `mmap`/`brk` create or resize VMAs.

```bash
$ strace -e trace=brk,mmap,munmap python3 -c 'x = bytearray(10**6)' 2>&1 | tail -3
brk(0x55f1c7f4c000)                     = 0x55f1c7f4c000        <- small: grow the heap
mmap(NULL, 1003520, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f3a9c2fe000   <- 1 MB: mmap
munmap(0x7f3a9c2fe000, 1003520)         = 0
```

Why is a large block better from `mmap`? When `free()`d, it's returned to the kernel immediately with `munmap()`. Memory in the middle of the heap can't be returned while anything above it is still in use (heap fragmentation).

**Why not use `mmap` for everything?** Each `mmap`/`munmap` is a syscall (3.3) that creates/destroys a VMA, edits page tables, and on `munmap` may need a **TLB shootdown** across cores (2.4). Each mapping uses a VMA (limit: `vm.max_map_count`: 65,530 is the kernel default, though some distros raise it to 1,048,576), and each fresh mapping starts with page faults. `malloc` from the heap free list costs ~20 ns with none of that.

How they fit in the process life cycle:

1. **`execve()`** wipes the old layout, frees old VMAs and sets up a new `mm_struct` with VMAs for the ELF segments, the stack, and the dynamic linker. `ld.so` then `mmap`s each shared library (4.9).
2. When the program starts, the C runtime initializes (`_start` → `__libc_start_main`) and sets up the heap.
3. When you call **`malloc()`**: small allocations come from the heap (`brk` when it must grow), large ones from fresh anonymous `mmap`s.
4. The **first touch** of each new page page-faults and gets a real frame (6.5).

## 6.7 Page Cache

The **page cache** is the kernel's cache of **file contents in RAM**, organized in pages. Every regular file read or write goes through it (unless you use `O_DIRECT`).

```
read(fd, buf, 4096)
   |
   v
VFS -> is page N of this file's inode in the page cache?  (inode->i_mapping, an xarray of pages)
   |-- yes (cache hit):   copy 4 KB from the cached page into buf.   ~1 µs, no disk.
   `-- no  (cache miss):  allocate a page, ask the filesystem to read it from disk (DMA, 2.5),
                          task sleeps; on completion copy to buf. Also READAHEAD the next pages.
```

Key facts:

- The cache is keyed by **(inode, page offset)**, so it's shared by **every process** that reads the file, and by `mmap` (a shared file mapping maps the *same* page-cache pages directly into your page tables, no copy).
- That's why the second `grep -r` over a source tree is 10× faster than the first.
- It uses otherwise-free RAM and shrinks automatically under memory pressure (clean pages can just be dropped, 6.8). That's the big "buff/cache" in `free`.

```bash
$ echo 3 | sudo tee /proc/sys/vm/drop_caches   # drop clean page cache + dentries/inodes (for benchmarks)
$ time cat bigfile > /dev/null                 # first: from disk
$ time cat bigfile > /dev/null                 # second: from page cache, much faster
```

### Dirty Pages and Writeback

`write()` **does not write to disk**. It copies your data into page-cache pages and marks them **dirty**, then returns. This is the write-back caching idea from CPU caches (2.4), one level up.

```
write(fd, "hello", 5)  -> page cache page marked dirty -> return immediately (fast)
                                     |
                    later (seconds): flusher kernel threads (kworker/flush-*) write dirty
                                     pages to disk and mark them clean
```

When does writeback happen?

- After a page has been dirty for ~30 s (`vm.dirty_expire_centisecs`)
- When dirty data exceeds `vm.dirty_background_ratio` (10% of available memory) → background writeback starts
- When it exceeds `vm.dirty_ratio` (20%) → **the writing process itself is throttled** and blocks until it catches up
- When you call **`fsync(fd)`** / `fdatasync(fd)` (block until this file's data is on stable storage), `sync`, or `msync()` for mappings

```bash
$ grep -E 'Dirty|Writeback' /proc/meminfo
Dirty:              2048 kB
Writeback:             0 kB
```

Consequences:

- If the machine **loses power**, recently written data that wasn't `fsync`ed is **lost**, even though `write()` returned success. Databases call `fsync` on their logs at every commit for exactly this reason.
- A write error (disk full on a network fs, I/O error) may only be reported at `fsync()` or `close()` time.

## 6.8 Swapping and Memory Reclaim

When free memory runs low, the kernel must **reclaim** pages. What it does depends on the page type:

| Page type | How to reclaim it | Cost |
| :---- | :---- | :---- |
| Clean file page (page cache, program code) | Just drop it; it can be re-read from the file | Cheap |
| Dirty file page | Write it back to the file first, then drop | Disk write |
| Anonymous page (heap, stack) | Write it to **swap**, then drop | Disk write; later a major fault to bring back |
| Kernel slab objects (dentries, inodes) | Shrink the caches | Cheap |
| Anonymous page, **no swap configured** | **Cannot** be reclaimed | → OOM risk |

**Swap** is disk space (a partition or a file) used as backing store for anonymous pages. When a page is swapped out, its PTE is set to not-present and records the swap location; accessing it later causes a major fault (6.5).

```bash
$ swapon --show
NAME      TYPE  SIZE USED PRIO
/swap.img file    2G  12M   -2
$ cat /proc/sys/vm/swappiness      # 0..200: higher = more willing to swap anon vs drop file cache
60
$ vmstat 1                         # watch si/so (swap in/out per second)
```

### LRU Lists and Page Reclaim (`kswapd`)

Which pages to evict? Ones that **haven't been used recently** (temporal locality, 1.8). Linux keeps pages on **LRU lists**, separately for anonymous and file pages, each with an **active** and **inactive** part:

```
newly faulted page --> inactive list --(accessed again)--> active list
                             ^                                 |
                             |   (not accessed for a while)    |
                             +---------------------------------+
                             |
                       tail of inactive --> evicted (dropped / written back / swapped)
```

The hardware **Accessed bit** in the PTE (6.3) tells the kernel whether a page was touched since the last scan. Since Linux 6.1, many distros use **MGLRU** (multi-generational LRU), a more accurate version of this aging.

Who does the reclaiming:

- **`kswapd`** (one kernel thread per NUMA node) wakes up when free memory drops below the **low watermark** and reclaims in the background until it reaches the **high watermark**.
- **Direct reclaim:** if an allocation finds free memory below the **min watermark**, the *allocating process itself* has to reclaim before it gets its page. This shows up as unpredictable latency spikes in applications.

**Thrashing:** if the working set of all processes doesn't fit in RAM, pages get evicted and faulted back in constantly. The system spends its time on I/O; everything crawls. `si`/`so` in `vmstat` and the major fault counts explode.

### Overcommit

Because of demand paging (6.5), `malloc` only creates VMAs. So should the kernel allow you to reserve more memory than exists?

```c
// 16 GB of RAM, no swap
char *big = malloc(100UL << 30);       // 100 GB in ONE request: refused by the default
                                       // heuristic (larger than RAM + swap) -> NULL

char *p[10];
for (int i = 0; i < 10; i++)
    p[i] = malloc(10UL << 30);         // 10 x 10 GB: each request alone looks plausible,
                                       // so all succeed -> 100 GB promised on a 16 GB machine.
                                       // Trouble only starts when you TOUCH more than exists.
```

`vm.overcommit_memory` controls this:

| Value | Policy |
| :---- | :---- |
| `0` (default) | **Heuristic**: refuse a single request that obviously can't be satisfied (larger than total RAM + swap); otherwise allow overcommit, so many plausible requests can add up to far more than exists |
| `1` | **Always** allow (some scientific/Redis setups rely on this) |
| `2` | **Never** overcommit: total commitments limited to swap + `overcommit_ratio`% of RAM. `malloc` returns NULL properly, but much less memory is usable |

Why overcommit is the default: `fork()` of a 10 GB process would otherwise need a 10 GB reservation for the child, even though the child will `exec` right away and COW means almost nothing is copied. Most programs also allocate more than they ever touch.

The price: the kernel has promised more than it has. If everyone collects on the promise, something has to give...

### OOM Killer

When the kernel can't reclaim enough memory to satisfy a page fault, it invokes the **Out-Of-Memory killer**, which picks a victim process and kills it with **`SIGKILL`** (8.2), freeing its memory.

The victim is the process with the highest **`oom_score`**, mainly based on how much memory (RSS + swap + page tables) it uses, adjusted by **`oom_score_adj`** (−1000 to +1000):

```bash
$ cat /proc/$(pgrep -o postgres)/oom_score
42
$ echo -1000 | sudo tee /proc/1234/oom_score_adj    # never kill this process
$ sudo dmesg | grep -i 'killed process'
Out of memory: Killed process 31337 (java) total-vm:12GB, anon-rss:11GB, ... oom_score_adj:0
```

- The killed process's exit status shows **137** (128 + SIGKILL) (4.9). "My container randomly exits with 137" = OOM.
- With cgroups (10.2), a container hitting its `memory.max` triggers an OOM kill **inside that cgroup only**, even if the host has plenty of free memory.
- User-space OOM daemons (`systemd-oomd`, `earlyoom`) act earlier using PSI pressure metrics (3.6), before the system becomes unresponsive.

## 6.9 Huge Pages

With 4 KB pages, a process using 64 GB needs 16 million PTEs, and the TLB (~2,000 entries → 8 MB of reach, 2.4) misses constantly. **Huge pages** map a large region with **one** entry by stopping the page walk early:

- **2 MB page:** the PMD entry has the `PS` bit set and points directly at a 2 MB physical block (the 9 PTE bits + 12 offset bits = 21 bits of offset).
- **1 GB page:** the PUD entry points directly at a 1 GB block (30 bits of offset).

```
4 KB:  PGD -> PUD -> PMD -> PTE -> 4 KB frame       (4 levels)
2 MB:  PGD -> PUD -> PMD ------> 2 MB frame         (3 levels, 512x TLB reach per entry)
1 GB:  PGD -> PUD -------------> 1 GB frame         (2 levels)
```

Costs: they need **physically contiguous** memory (order-9 buddy blocks for 2 MB, 6.2), which gets hard on a fragmented system; and they waste memory if a process uses only a little of each 2 MB.

### Transparent Huge Pages (THP)

The kernel automatically uses 2 MB pages for **anonymous memory** when a VMA is big enough and aligned, without any application changes. A background thread, **`khugepaged`**, also collapses existing runs of 4 KB pages into huge pages.

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never        # madvise: only for regions the app marked with madvise(MADV_HUGEPAGE)
$ grep AnonHugePages /proc/meminfo
AnonHugePages:    614400 kB
```

Trade-off: big wins for large-heap workloads (JVMs, ML), but compaction and collapsing can cause **latency spikes**, and COW after `fork` may copy 2 MB instead of 4 KB. Some databases (Redis, MongoDB historically) recommend `never` or `madvise`.

### HugeTLB

The older, explicit mechanism: huge pages are **reserved in advance** into a pool and never swapped. Applications request them via `mmap(MAP_HUGETLB)` or files on a `hugetlbfs` mount.

```bash
$ echo 1024 | sudo tee /proc/sys/vm/nr_hugepages    # reserve 1024 x 2 MB = 2 GB
$ grep -i huge /proc/meminfo
HugePages_Total:    1024
HugePages_Free:     1024
Hugepagesize:       2048 kB
```

Used by databases (Oracle, PostgreSQL `huge_pages = on`), DPDK, and VMs (2.6: a TLB miss in a VM is extra expensive, so hypervisors back guest RAM with huge pages). Predictable, but the reserved memory is unusable for anything else.

## 6.10 Memory Compression and Deduplication (zswap, zram, KSM)

Disk-based swap (6.8) is slow. Many workloads have memory that **compresses well** (2–4×) or that is **duplicated** across processes. Linux can exploit both.

### zswap: A Compressed Cache in Front of Swap

When a page is about to be swapped out, **zswap** first compresses it (lz4, zstd) and keeps it in a RAM pool. Only if the pool fills up are the oldest pages written to the real swap device.

```
anonymous page evicted (6.8) -> compress -> zswap pool in RAM  --(pool full)--> swap partition/file
page needed again            <- decompress (~µs, a MINOR-ish fault instead of a MAJOR one, 6.5)
```

```bash
$ cat /sys/module/zswap/parameters/enabled
Y
```

### zram: A Compressed RAM Disk Used as Swap

**zram** creates a **block device** (7.12) backed by compressed RAM, e.g. `/dev/zram0`, and you use it as a swap device. There's no disk at all. It's the default swap on Fedora, Android and ChromeOS.

```bash
$ zramctl
NAME       ALGORITHM DISKSIZE  DATA COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram0 zstd          7.7G  1.2G  301M  316M       8 [SWAP]      <- 1.2 GB of pages stored in ~316 MB of RAM
```

zswap vs zram: zswap is a cache in front of a real swap device; zram *is* the swap device. Both trade CPU time (compressing) for RAM, which is almost always a good deal compared to disk I/O.

### KSM: Kernel Samepage Merging

Ten VMs running the same OS have thousands of **identical pages** (the same kernel code, the same zeroed pages). **KSM** finds them and merges them:

1. An application (typically QEMU/KVM, 2.6) marks memory as mergeable with `madvise(MADV_MERGEABLE)`.
2. The `ksmd` kernel thread scans those pages, hashing and comparing their contents.
3. Identical pages are replaced by **one shared, read-only page**.
4. If any process **writes** to it later, a normal **copy-on-write fault** (6.5) gives it a private copy again.

```bash
$ cat /sys/kernel/mm/ksm/pages_sharing     # how many page mappings are deduplicated
```

Trade-offs: scanning costs CPU, and merged pages can leak information between tenants through timing (a write to a merged page is slow because of the COW copy), so KSM is often disabled across security boundaries.

> 🔗 **Connections:** Memory management is where everything meets. `fork` relies on COW faults; `exec` builds VMAs that page-fault in data from the page cache; `malloc` sits on `brk`/`mmap`; files are read through the page cache that `mmap` shares; reclaim uses the hardware Accessed/Dirty bits; the OOM killer sends a signal (Part 8) whose exit code (137) you see in shells and containers; and cgroups (10.2) put a fence around all of it.

---

# 7. Files and I/O

> 🧪 **Interactive visuals:** [Part 7, visualized](visuals/part7-files.html): an interactive page with a demo for each section.


## 7.1 File Descriptors

A **file descriptor (fd)** is a small non-negative integer that identifies an open file (or more generally an open I/O resource: pipe, socket, device, eventfd, pidfd...) **within a process**.

When you write:

```c
int fd = open("data.txt", O_RDONLY);
```

it calls the `open()` system call (really `openat()`) to open `data.txt` read-only. Since it's `O_RDONLY` without `O_CREAT`, if the file doesn't exist it fails, returns `-1`, and sets `errno = ENOENT`. If the file exists, the kernel:

1. resolves the path to an inode through the VFS (7.3, 7.4) and checks permissions (7.11),
2. creates a `struct file` (an *open file description*, 7.2),
3. scans the process's file descriptor table for the **lowest unused number** and stores a pointer to that `struct file` there,
4. returns that number.

You **don't specify** `0`, `1`, or `2`: the kernel assigns the lowest available fd, which is usually `3`, because 0, 1, 2 are already taken by the standard streams (7.5):

- `0` = stdin
- `1` = stdout
- `2` = stderr

The fd is only an **index**. The real state lives in the kernel:

```c
// [1] Each process has a task_struct
struct task_struct {
    ...
    struct files_struct *files;     // <--- File descriptor management lives here
    ...
};

// [2] files_struct contains the descriptor table (fdtable)
struct files_struct {
    struct fdtable *fdt;            // <--- Pointer to fd table
    ...
};

// [3] fdtable is where the FD numbers map to struct file *
struct fdtable {
    unsigned int max_fds;
    struct file **fd;               // <--- Array: fd[0], fd[1], ..., fd[max_fds-1]
    ...
};

// [4] struct file represents an open file (system-wide)
struct file {
    struct path f_path;             // dentry + mount it was opened through
    struct inode *f_inode;          // Metadata about the file
    loff_t f_pos;                   // File offset
    unsigned int f_flags;           // O_RDONLY, O_APPEND, O_NONBLOCK, ...
    const struct file_operations *f_op;  // read/write/... for this file type
    atomic_long_t f_count;          // How many fds point at me
    ...
};
```

```
// THE CONNECTION
Process (task_struct)
  `-- files_struct
        `-- fdtable
              `-- fd[3] ------> struct file
                                   |-- f_pos    (file offset)
                                   |-- f_flags  (read/write flags)
                                   `-- f_inode ---> inode: file metadata, data block pointers
```

See a process's fds:

```bash
$ ls -l /proc/self/fd
lrwx------ 1 berat berat 64 ... 0 -> /dev/pts/0
lrwx------ 1 berat berat 64 ... 1 -> /dev/pts/0
lrwx------ 1 berat berat 64 ... 2 -> /dev/pts/0
lr-x------ 1 berat berat 64 ... 3 -> /proc/4321/fd
$ ulimit -n        # max fds per process (RLIMIT_NOFILE, in task_struct->signal->rlim)
1024
```

"Too many open files" (`EMFILE`) = you hit that limit; usually an fd leak (forgetting `close()`).

## 7.2 Open File Descriptions and the File Table

There are **three layers**, and understanding which one holds what explains a lot of surprising behavior:

```
 per-process fd tables            system-wide "file table"            inodes
 (files_struct)                   (struct file = open file            (struct inode = the file
                                   description)                        itself, permanent)
 process A
  fd 0 --------------------------> [ file: /dev/pts/0  pos=- ] -----> [ inode: pts/0 ]
  fd 3 --------------------------> [ file: log.txt  pos=100  ] --+
  fd 4 ---+                                                       +--> [ inode #1234 log.txt ]
          |                                                       |
          +----------------------> [ file: log.txt  pos=0    ] --+    (opened twice ->
                                                                       two files, one inode)
 process B (forked from A)
  fd 3 --------------------------> (same struct file as A's fd 3: SHARED offset pos=100)
```

1. **Per-process file descriptor table** (`files_struct`): fd number → pointer. Each process has its own.
2. **Global file table** (`struct file`, the *open file description*): created by each `open()` call. Holds the state of **one open session**: the **current offset**, the access mode and status flags (`O_APPEND`, `O_NONBLOCK`). It exists separately from the permanent state of the file on disk.
3. **Inode** (`struct inode`): the file itself: size, owner, permissions, timestamps, where the data blocks are. One per file, no matter how many times it's open.

What this explains:

| Operation | Effect |
| :---- | :---- |
| `open()` the same file twice | Two `struct file`s, **independent** offsets, same inode |
| `dup(fd)` / `dup2(old, new)` | Two fds → **same** `struct file`, **shared** offset |
| `fork()` | Child's fd table is a copy → fds point to the **same** `struct file`s: **shared offset** with the parent (4.9) |
| `close(fd)` | Removes the fd entry and decrements `f_count`; the `struct file` is freed when the last reference goes |
| `exec()` | fds survive unless `FD_CLOEXEC` (a flag on the fd, not on the file) |

This is why in `(echo a; echo b) > out.txt` both echoes append correctly: they share one `struct file` and its offset. And it's how a shell redirection works: open the file, `dup2` it onto fd 1, then `exec` (7.5).

> 🔗 **Connections:** The fd table is pointed to from `task_struct` (`files`), shared by threads (5.1, `CLONE_FILES`), copied by `fork`, preserved by `exec` (4.9). The inode at the end is the same object the page cache is keyed on (6.7).

## 7.3 Inodes and Directory Entries

### Filesystem Structure on Disk

A disk filesystem like ext4 lays out a partition roughly like this:

```
+------------+--------------------+--------------+--------------+----------------------------+
| Superblock | Group descriptors, | Inode bitmap | Inode table  | Data blocks                |
|            | block bitmap       |              | (fixed size) | (file contents, directory  |
|            |                    |              |              |  contents, indirect blocks)|
+------------+--------------------+--------------+--------------+----------------------------+
```

- **Superblock:** filesystem-level metadata: total size, block size (usually 4 KB), inode count, free counts, filesystem UUID, state (clean/dirty), features. Corrupted superblock = unmountable filesystem, so backups are kept in other block groups.
- **Inode table:** a fixed number of inodes, decided at `mkfs` time.
- **Data blocks:** file contents, and also **directory contents**.

```
   +----------------+
   |  Superblock    |  <- filesystem-level metadata (size, block size, inode count)
   +----------------+
           |
           v
   +----------------+
   |  Directory     |  (itself an inode whose data blocks are a list of entries)
   +----------------+
           |
           v
 +------------------+       +------------------+         +----------------+
 |     Dentry       |  -->  |    Filename: A   |   --->  |  Inode #123    |
 +------------------+       +------------------+         +----------------+
                                                                 |
                                                  +------------------------+
                                                  | File metadata (owner,  |
                                                  | permissions, size)     |
                                                  | Data block pointers    |
                                                  +------------------------+
                                                              |
                                                              v
                                                  +------------------------+
                                                  |    File Data Blocks    |
                                                  +------------------------+
```

In a Linux filesystem, the **superblock** stores metadata about the entire filesystem, a **directory entry** maps a filename to an inode number, and the **inode** contains metadata and pointers to the actual file data blocks.

```bash
$ sudo dumpe2fs -h /dev/nvme0n1p2 | grep -E 'Block size|Inode count|Inode size|Free inodes'
Inode count:              6553600
Free inodes:              5893121
Block size:               4096
Inode size:               256
```

### Index Node (inode)

Unix files consist of two parts: the **data part** and the **filename part**.

The data part is associated with an **inode**. The inode carries the map of where the data is, the file permissions, and so on. On ext4 they're usually 256 bytes each, stored in the inode table on disk; in memory the kernel uses `struct inode` (cached in the inode slab cache, 6.2).

What is inside an inode?

| Field | Example | Notes |
| :---- | :---- | :---- |
| Inode number | `1835123` | Unique **within one filesystem** only |
| Mode | `-rw-r--r--` | File **type** (regular, dir, symlink, device...) + permission bits (7.11) |
| UID / GID | `1000 / 1000` | Owner and group |
| Size | `14` | In bytes |
| Link count | `1` | Number of directory entries (hard links) pointing here |
| atime | | Last **access** (read) |
| mtime | | Last **modification** of the content |
| ctime | | Last **change** of the inode (permissions, owner, link count, or content). *Not* creation time! |
| btime (birth) | | **Creation** time, on filesystems that record it (ext4, xfs, btrfs). Shown as `Birth:` by `stat` (via the `statx()` syscall) |
| Block count / block size | | Allocated space (can be less than size: sparse files) |
| Block pointers / extents | | Where the data lives on disk |
| ACLs, xattrs | | Extended permissions, SELinux labels, capabilities (10.4) |

> **Inodes do not contain:** the **filename**, the **path**, or the actual file data (only pointers to blocks). Filenames belong to **directory entries**.

> **Directories are special files that map names → inode numbers.**

```
  [ Directory Entry ]
      |
      v
+---------------+
|   Filename    | ---> [ Inode Number ]
+---------------+
                         |
                         v
               +---------------------+
               |      Inode          |
               |---------------------|
               | Metadata: size,     |
               | timestamps, perms   |
               |---------------------|
               | Block pointers:     |
               |  [10][15][18][... ] |  --> data blocks
               +---------------------+

Each block pointer points to a data block on disk:

           [ Inode ]
             |
             v
       +------------------+
       | Block Pointers   |
       |------------------|
       |  [Block #1010]   |
       |  [Block #1011]   |
       |  [Block #1012]   |
       +------------------+
             |       |       |
             v       v       v
     +---------+ +---------+ +---------+
     | 4 KB    | | 4 KB    | | 4 KB    |
     | Data    | | Data    | | Data    |
     +---------+ +---------+ +---------+

   (Total: 12 KB = 3 blocks)
```

(Classic ext2/3 use 12 direct pointers plus single/double/triple **indirect** blocks for big files. ext4 uses **extents** instead: "blocks 1010 to 1012" as one record, which is much more compact for large contiguous files.)

```bash
$ stat notes.txt
  File: notes.txt
  Size: 14              Blocks: 8          IO Block: 4096   regular file
Device: 259,2   Inode: 1835123     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/   berat)   Gid: ( 1000/   berat)
Access: 2026-10-16 10:01:22
Modify: 2026-10-16 10:00:58
Change: 2026-10-16 10:00:58
$ ls -i notes.txt          # inode number
1835123 notes.txt
$ df -i /                  # inode usage per filesystem
```

**You can run out of inodes while having free space.** Millions of tiny files (mail spools, cache directories) can exhaust the inode table: `df -h` shows free space, but `touch` fails with `No space left on device`. Check `df -i`.

### Path Resolution and the Dentry Cache

To open `/home/berat/notes.txt`, the kernel walks the path one component at a time (`link_path_walk()`):

```
start at the process's root (task_struct->fs->root)  -> inode of "/"
lookup "home"  in "/"         -> read the directory's entries -> inode #2 of /home
                                  (permission: need x on "/")
lookup "berat" in "/home"     -> inode of /home/berat          (need x on /home)
lookup "notes.txt" in "/home/berat" -> inode #1835123          (need x on /home/berat)
at each step: is this dentry a MOUNT POINT? if so, jump to the root of the mounted fs (7.13)
              is it a SYMLINK? if so, read its target and continue resolving from there
```

Doing disk reads for every component would be slow, so the kernel caches the results in the **dentry cache** (dcache): `struct dentry` objects linking a name in a parent directory to an inode, including **negative dentries** ("this name doesn't exist") so repeated failed lookups (like `$PATH` searches) are cheap too.

### Hard Links vs Soft Links (Symlinks)

(Further reading: [Hard links and soft links explained, Linux Gazette](http://linuxgazette.net/105/pitcher.html).)

A **hard link** is just **another directory entry pointing to the same inode**. More than one filename can reference the same inode number; these names are said to be hard linked together. There's no "original": all names are equal.

```
    ! filename  ! inode # !
    +----------------------+
                    \
                     >--------------> ! permbits, etc ! addresses !
                    /                 +---------inode-------------+
    ! othername ! inode # !
    +----------------------+
```

A **symbolic (soft) link** is a **separate small file (its own inode) whose content is a path**. The kernel follows that path at access time.

```
[MY-HARD-LINK] --> [INODE #500] <-- [myfile.txt] <-- [my-soft-link]  (inode #501, content: "myfile.txt")
```

```bash
$ echo 'Hello, World!' > myfile.txt
$ ln myfile.txt my-hard-link          # hard link
$ ln -s myfile.txt my-soft-link       # soft link
$ ls -li
500 -rw-r--r-- 2 berat berat 14 ... my-hard-link            <- same inode, link count 2
501 lrwxrwxrwx 1 berat berat 10 ... my-soft-link -> myfile.txt   <- own inode, size 10 = len("myfile.txt")
500 -rw-r--r-- 2 berat berat 14 ... myfile.txt

$ rm myfile.txt
$ cat my-hard-link                    # still works: the inode still has 1 link
Hello, World!
$ cat my-soft-link                    # broken: the path it stores no longer exists
cat: my-soft-link: No such file or directory
```

If a symbolic link refers to a file that doesn't exist, it is said to be a **dangling link**.

| Feature | Symbolic link (soft link) | Hard link |
| :---- | :---- | :---- |
| **What it is** | A separate inode containing a **path** | Another **name** for the same inode |
| **Cross-filesystem** | **Yes**: it's just a path string, resolved at access time | **No**: inode numbers are only unique within one filesystem (`EXDEV`) |
| **Directories** | Yes | No (`EPERM`), except the automatic `.` and `..` |
| **If target is deleted** | Becomes a **dangling** link | Still works: the data stays until the last link (and last open fd) is gone |
| **If target is moved/renamed** | Breaks | Keeps working (it never referred to the name) |
| **Storage** | A small inode (short targets are stored inside the inode itself) | No extra inode; one more directory entry |
| **Permissions** | Always `lrwxrwxrwx`; the target's permissions are what count | Same inode → same permissions |

**The same inode number may appear on more than one filesystem.** Inode numbers are only unique per filesystem:

```bash
# /dev/sda1 mounted at /home, /dev/sdb1 mounted at /data
$ ls -i /home/file1 /data/file2
1024 /home/file1
1024 /data/file2        # same number, completely different files: (device, inode) is the real identity
```

That's exactly **why hard links can't cross filesystems**: a directory entry stores only an inode number, which means nothing on another filesystem. A symlink stores a path, and paths cross filesystems fine.

Why no hard links to directories? It could create **cycles** (`a/b` linking back to `a`), which would break `find`, `du`, and the guarantee that every directory has exactly one parent (`..`). It also explains directory link counts: a directory has `2 + number of subdirectories` links (its name in the parent, its own `.`, and each child's `..`).

```bash
# This will create a soft link
ln -s test/target-file.txt link-file.txt
# This will create a hard link
ln test/target-file.txt link-file.txt
# View inode number
ls -i file.txt
# View inode usage
df -i
# View detailed inode
stat file.txt
# Where does a symlink point (one level / fully resolved)?
readlink link-file.txt ; readlink -f link-file.txt
```

### What Happens on `rm`: `unlink()`

`rm file.txt` triggers the **`unlink()`** system call (really `unlinkat()`). There's no "delete file" syscall; you can only **remove a name**:

1. **Path resolution (VFS layer):** walk the path, find the parent directory's dentry and the target dentry.
2. **Permission check:** you need **write + execute permission on the parent directory**, *not* on the file itself (plus the sticky-bit rule in `/tmp`, 7.11). Deleting a name = modifying the directory.
3. **`vfs_unlink()`** calls the **filesystem-specific** unlink (e.g. `ext4_unlink()`), which removes the entry from the directory's data blocks.
4. The inode's **link count is decremented**.
5. **If the link count drops to 0 _and_ no process has the file open**, the inode and its data blocks are freed. If some process still has it open, the inode lives on (as an "orphan") until the last `close()`.
6. The dentry cache and inode cache are updated (the dentry becomes negative).
7. Return to user space: `0` for success, `-1`/`errno` on failure.

```bash
$ strace -e trace=unlinkat rm file.txt
unlinkat(AT_FDCWD, "file.txt", 0)       = 0
```

This is the root of the famous "`df` says the disk is full but `du` can't find the files" mystery (Part 12): a deleted log file that a running process still holds open.

```bash
$ sudo lsof +L1                          # open files with link count 0 (deleted but open)
COMMAND  PID USER  FD  TYPE DEVICE  SIZE/OFF NLINK  NODE NAME
java    2211 app   7w  REG  259,2  9.8G       0    1311 /var/log/app.log (deleted)
$ cp /proc/2211/fd/7 /tmp/recovered.log  # you can even recover it through /proc!
```

## 7.4 VFS (Virtual File System) and "Everything Is a File"

Linux supports dozens of filesystems: ext4, XFS, Btrfs, NFS, FAT, tmpfs, procfs, sysfs, overlayfs... Yet `cat` uses the same `open`/`read`/`write`/`close` on all of them. The **VFS** is the abstraction layer that makes this possible:

```
              user space:   open()  read()  write()  stat()  mmap()  ...
              --------------------------------------------------------------
              VFS:   generic objects + generic logic (path walk, permissions, page cache, fd tables)
                     struct super_block   one per mounted filesystem
                     struct inode         one per file (in use)
                     struct dentry        one per path component (cached)
                     struct file          one per open()
              --------------------------------------------------------------
              each object has an operations table of function pointers, filled in by the fs:
                 inode->i_op, file->f_op (read_iter, write_iter, mmap, poll...), sb->s_op
              --------------------------------------------------------------
   ext4    xfs    btrfs    nfs    tmpfs    procfs    sysfs    devtmpfs  pipefs  sockfs
    |       |       |       |                (generate data on the fly)
  block layer + device driver           network
```

It's object-oriented programming in C: `read()` ends up calling `file->f_op->read_iter()`, and *which* function that is depends on what the file is: ext4 reads from the page cache/disk, procfs formats kernel data into text (7.10), a pipe copies from its buffer (9.1), a TTY returns your keystrokes, a socket returns network data (9.5).

That's the meaning of the Unix philosophy **"everything is a file"**: more precisely, *everything is a file descriptor* with a common set of operations. Regular files, directories, devices (7.12), pipes, sockets, and even processes (pidfd, 4.5), timers (`timerfd`), signals (`signalfd`) and events (`eventfd`) are all accessed through fds, so the same tools (`read`, `write`, `poll`/`epoll`, `close`, redirection) work on all of them.

```bash
$ cat /proc/filesystems | head -5      # filesystem types this kernel knows
nodev   sysfs
nodev   tmpfs
nodev   proc
        ext4
```

`nodev` = a virtual filesystem not backed by a block device.

### FUSE: Filesystems in User Space

Normally a filesystem is kernel code (a module, 3.1). **FUSE** lets an ordinary **user-space program** implement one. The kernel's `fuse` module forwards VFS operations to that program through the character device `/dev/fuse` (7.12):

```
app: read("/mnt/remote/a.txt")
  -> syscall -> VFS -> fuse kernel module -> request queued on /dev/fuse
                                                 |
       fuse daemon (user space, e.g. sshfs) reads the request, fetches the data over SSH,
       writes the reply to /dev/fuse
                                                 |
  <- VFS <- fuse module <------------------------+     <- read() returns
```

Examples: `sshfs` (mount a remote directory over SSH), `rclone mount` / `s3fs` (cloud storage), `ntfs-3g`, `gocryptfs` (encryption), AppImage, and rootless container storage (`fuse-overlayfs`).

```bash
$ sshfs user@server:/var/log /mnt/logs     # mounted by a normal user, no root needed
$ fusermount -u /mnt/logs                  # unmount
```

The cost is performance: each operation needs extra context switches between the app, the kernel and the daemon (3.5), plus copying. The benefits are safety (a bug crashes a process, not the kernel; the microkernel idea from 3.1) and ease of development in any language.

## 7.5 Standard Streams and Redirection (stdin, stdout, stderr)

Every process conventionally starts with three open fds, inherited from its parent (ultimately from `getty`/`sshd`, 3.7):

| fd | Name | Purpose |
| :---- | :---- | :---- |
| `0` | **stdin** | Input stream where processes read data from the user or another process |
| `1` | **stdout** | Output stream for the expected, normal output |
| `2` | **stderr** | Output stream for error and diagnostic messages, kept separate so it isn't mixed into piped data |

They are **not special to the kernel**: just fds 0, 1, 2, pointing to whatever the parent set up: usually the terminal (`/dev/pts/0`), but possibly a file, a pipe, or `/dev/null`.

**Redirection is done by the shell, between `fork()` and `exec()`**, using `open` + `dup2`:

```bash
$ cmd > out.txt 2>&1
```

```c
// in the child, after fork(), before execve():
int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);   // fd 3
dup2(fd, 1);    // fd 1 now points to out.txt's struct file
close(fd);
dup2(1, 2);     // 2>&1: fd 2 points to the SAME struct file as fd 1 (shared offset, 7.2)
execve("/usr/bin/cmd", argv, envp);   // cmd just writes to 1 and 2, unaware
```

Order matters: `cmd 2>&1 > out.txt` makes fd 2 a copy of the *old* fd 1 (the terminal) first, then moves only fd 1 to the file.

```bash
cmd > f        # stdout to f (truncate)        cmd >> f      # append (O_APPEND)
cmd 2> err     # stderr only                    cmd &> f      # both (bash)
cmd < in       # stdin from file                cmd 2>/dev/null   # discard errors
cmd1 | cmd2    # stdout of cmd1 -> pipe -> stdin of cmd2 (9.1)
```

Terminal job control keys (handled by the TTY driver, 4.5):

- `Ctrl+C` → `SIGINT` to the foreground job
- `Ctrl+Z` → `SIGTSTP` (terminal stop) to the foreground job; then `fg` (resume in the foreground) or `bg` (resume in the background)
- `Ctrl+D` → not a signal: "end of input" (a `read()` on the TTY returns 0)

## 7.6 Buffered vs Unbuffered I/O

There are **two different buffers** between your `printf` and the disk, and confusing them causes real bugs:

```
printf("hi")                         user space
   |  copies into the stdio buffer (libc, inside YOUR process memory)
   |  flushed when: full / newline (if terminal) / fflush() / exit()
   v
write(1, "hi", 2)  ---- system call --------------------------------
   |  copies into the PAGE CACHE (kernel, 6.7), marks the page dirty
   |  returns immediately
   v
disk  <- written later by flusher threads, or when you call fsync()
```

**1. User-space (stdio) buffering:** `FILE*` functions (`printf`, `fputs`, `fwrite`) collect data in a buffer to make fewer system calls (3.3).

| stdout connected to | stdio buffering mode |
| :---- | :---- |
| a terminal | **line-buffered**: flushed at each `\n` |
| a file or a pipe | **fully buffered**: flushed when the buffer (4–8 KB) fills or at exit |
| stderr (always) | **unbuffered**: written immediately |

Classic bugs this causes:

```bash
$ ./progress_printer | tee log      # nothing appears for minutes: stdout is a pipe -> fully buffered
$ stdbuf -oL ./progress_printer | tee log      # force line buffering (or fflush / python -u)
```

```c
printf("hello ");      // sits in the stdio buffer (no newline, or stdout is a pipe)
fork();                // the buffer is copied into the child's memory (COW, 4.9)!
printf("\n");          // both flush -> "hello" is printed TWICE
```

And if a process is killed by `SIGKILL` or crashes, unflushed stdio data is simply lost, because it only existed in that process's memory.

**2. Kernel buffering:** the page cache (6.7). `write()` returning means "the kernel has it", not "it's on disk". Use `fsync()` for durability.

**Unbuffered** usually means using the raw syscalls (`read`/`write` on an fd) without stdio, so every call is a syscall. **`O_DIRECT`** goes further and bypasses the page cache entirely (requires aligned buffers, 1.6); databases with their own caches use it to avoid caching data twice.

## 7.7 Blocking vs Non-Blocking I/O

By default, fds are **blocking**: if there's no data yet, `read()` doesn't return; the kernel puts the task to sleep.

```
read(sock, buf, 100)   no data yet
   |
   kernel: add task to the socket's WAIT QUEUE, set state S (4.8), call schedule() (3.6)
   |                     ... other tasks run ...
   packet arrives -> NIC interrupt (3.4) -> TCP/IP softirq -> data queued on the socket
   |                     -> wake_up(wait queue) -> task becomes R, goes back on a run queue
   v
read() copies the data and returns
```

That's **voluntary context switching** (3.5) in action, and it's efficient for one connection. But a server handling 10,000 connections with blocking I/O needs 10,000 threads.

**Non-blocking** mode (`O_NONBLOCK`, set with `open` or `fcntl`): if the operation would block, it returns immediately with `-1` and `errno = EAGAIN` (also called `EWOULDBLOCK`).

```c
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

ssize_t n = read(fd, buf, sizeof buf);
if (n < 0 && errno == EAGAIN) {
    /* nothing yet: do something else, try later */
}
```

But looping over 10,000 fds calling `read()` just to get `EAGAIN` is a busy-wait. We need a way to ask the kernel "**which** of these fds are ready?" That's I/O multiplexing.

(Note: for **regular files**, `O_NONBLOCK` has no effect: disk reads always "block" in `D` state. Async file I/O needs `io_uring`, 7.9, or a thread pool.)

## 7.8 I/O Multiplexing (`select`, `poll`, `epoll`)

One thread waits on **many** fds at once, and the kernel tells it which ones are ready.

**`select()`** (1983): pass bitmaps of fds; the kernel checks all of them; returns modified bitmaps.

- Limited to `FD_SETSIZE` = 1024 fds.
- O(n) per call: the kernel scans every fd, every time, and the bitmaps are copied in and out on every call.

**`poll()`**: same idea with an array of `struct pollfd`, no 1024 limit, still O(n) per call.

**`epoll`** (Linux 2.6): the kernel **remembers** the interest list, and keeps a **ready list** that's filled by the wait-queue wakeups themselves:

```c
int ep = epoll_create1(0);                        // an fd representing the epoll instance

struct epoll_event ev = { .events = EPOLLIN, .data.fd = client };
epoll_ctl(ep, EPOLL_CTL_ADD, client, &ev);        // register ONCE

struct epoll_event ready[64];
for (;;) {
    int n = epoll_wait(ep, ready, 64, -1);        // sleep until something is ready
    for (int i = 0; i < n; i++)                   // only the READY fds come back: O(ready)
        handle(ready[i].data.fd);
}
```

```
epoll instance (in kernel)
  interest list (red-black tree): fd 5, fd 6, ... fd 10005
  ready list:                     fd 7, fd 912         <- appended by a callback that runs when
                                                          a socket's wait queue is woken (7.7)
epoll_wait() just hands back the ready list.
```

- **Level-triggered** (default): `epoll_wait` keeps reporting the fd as long as data is available.
- **Edge-triggered** (`EPOLLET`): reported only when new data **arrives**; you must read until `EAGAIN` or you'll miss it. Faster, easier to get wrong. Always used with non-blocking fds.

This is the engine behind nginx, Node.js (libuv), Redis, Go's network poller, Java NIO, Python `asyncio`. The macOS/BSD equivalent is `kqueue`.

> 🔗 **Connections:** epoll works on anything with a `poll` method in its `file_operations` (7.4): sockets, pipes, TTYs, eventfd, timerfd, signalfd (8.3), pidfd (4.5). That's "everything is a file" paying off: a single event loop can wait for network data, a timer, a child's exit and a signal at the same time.

## 7.9 `io_uring`

Even with epoll, each I/O is still: `epoll_wait` syscall + `read` syscall + `write` syscall... At millions of operations per second, the syscall cost (3.3, KPTI) dominates. And epoll doesn't help with regular files at all (7.7).

**`io_uring`** (Linux 5.1, 2019) replaces syscalls with **two ring buffers in memory shared between the process and the kernel** (set up with `mmap`, 6.6):

```
              user space                                   kernel
   +----------------------------------+
   | Submission Queue (SQ)            |  --->  kernel consumes SQEs: "read fd 5 into buf A",
   |  [read fd5][write fd9][accept 3] |        "write fd 9", "accept on 3" ... executes them
   +----------------------------------+        asynchronously (also for regular files)
   +----------------------------------+
   | Completion Queue (CQ)            |  <---  kernel posts CQEs: "read fd5 -> 4096 bytes"
   |  [res=4096][res=512]             |
   +----------------------------------+
   one io_uring_enter() syscall can submit AND reap many operations;
   with SQPOLL, a kernel thread polls the SQ and you may make no syscalls at all
```

- True async I/O for **files and sockets**, batching, zero-copy options, chains of dependent operations.
- Used by high-performance databases and storage engines (and via `liburing`).
- It has had many security bugs (it's a huge attack surface), so some environments (Android, some container runtimes via seccomp, 10.4) disable it.

## 7.10 Pseudo-Filesystems (`/proc`, `/sys`)

Some filesystems have **no disk at all**: their files are generated by the kernel when you read them (7.4). They let you inspect and tune the kernel with `cat` and `echo`.

**`/proc`** (procfs): process information and kernel statistics. It mimics kernel structures; the files exist only in memory and show the changing kernel data. That's why `ls -l` shows size 0 for most of them: the content doesn't exist until you read it.

```bash
/proc/<pid>/            one directory per process (built from its task_struct, 4.2)
   status  cmdline  environ  maps  smaps  fd/  fdinfo/  cwd  exe  root  ns/  limits  stack  oom_score
/proc/self              symlink to the reading process's own directory
/proc/meminfo           memory statistics (6.1)
/proc/cpuinfo           CPU details
/proc/loadavg           load average (3.6)
/proc/interrupts        interrupt counts per CPU (3.4)
/proc/mounts            mounted filesystems (symlink to self/mounts, 7.13)
/proc/sys/...           TUNABLE kernel parameters = what `sysctl` reads/writes

$ cat /proc/sys/vm/swappiness            # same as: sysctl vm.swappiness
$ echo 10 | sudo tee /proc/sys/vm/swappiness
$ tr '\0' ' ' < /proc/$$/cmdline; echo   # args are NUL-separated
$ readlink /proc/$$/exe /proc/$$/cwd
```

**`/sys`** (sysfs): similar to `/proc` in that it provides a device and system interface, but more structured: a view of the kernel's **device model**: buses, devices, drivers, and their attributes, one value per file.

```bash
/sys/class/net/eth0/address           MAC address
/sys/class/net/eth0/statistics/rx_bytes
/sys/block/nvme0n1/queue/scheduler    I/O scheduler
/sys/devices/system/cpu/cpu0/cpufreq/ CPU frequency (2.1)
/sys/fs/cgroup/                       cgroup v2 hierarchy (10.2)
/sys/kernel/mm/transparent_hugepage/  THP settings (6.9)
```

`udev` (7.12) builds `/dev` based on what appears in `/sys`.

## 7.11 File Permissions, Ownership and setuid

```bash
$ ls -l test.yaml
-rwxr-xr-x 1 berat staff 10 Oct 12 00:35 test.yaml
```

```
 -     rwx    r-x    r-x    1     berat  staff   10      Oct 12 00:35  test.yaml
 |     |      |      |      |     |      |       |       |             |
type  user  group  other  links owner  group   size    mtime         name
      (u)    (g)    (o)                        (bytes)
```

**File type** (first character): `-` regular file, `d` directory, `l` symlink, `c` character device, `b` block device, `p` named pipe (FIFO), `s` socket (7.12).

**Permission bits** (stored in the inode's mode, as octal: r=4, w=2, x=1):

| | On a file | On a directory |
| :---- | :---- | :---- |
| `r` | read the content | **list** the names in it (`ls`) |
| `w` | modify the content | **create, delete, rename** entries in it |
| `x` | execute it (4.9) | **traverse** it: use it in a path, access entries by name (`cd`) |

So `rm` needs `w`+`x` on the **directory**, not on the file (7.3). And a directory with `x` but no `r` lets you open files *if you know their names*, but not list them.

```bash
$ chmod 750 script.sh        # rwx r-x ---
$ chmod u+x,g-w file
$ chown berat:staff file
$ umask                       # bits REMOVED from new files' default mode (0666 files / 0777 dirs)
0022                          # -> new files 0644, new dirs 0755. Stored in task_struct->fs (4.2)
```

**How the kernel checks** (in `inode_permission()` during path walk and `open()`): it compares the process's **credentials** (`task_struct->cred`: effective UID, effective GID, supplementary groups) with the inode's owner and mode: if euid == owner → use the user bits; else if in the group → group bits; else → other bits. Only **one** class applies. root (euid 0, or more precisely `CAP_DAC_OVERRIDE`, 10.4) bypasses read/write checks (but still needs at least one `x` bit to execute).

### setuid, setgid and the Sticky Bit

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd
```

How can a normal user change their password, when that requires writing `/etc/shadow`, which only root can write? The answer is the **`s` in `rws`**: the **set-user-ID** bit. When this program is executed, it **doesn't run with the privileges of the user who ran it, but with the privileges of the file's owner** (root).

Mechanically, during `execve()` (4.9, step 5), if the file has the setuid bit, the kernel sets the process's **effective UID** to the file's owner, while the **real UID** stays yours (so `passwd` knows *who* you are and only lets you change your own password).

```
you (uid 1000) run /usr/bin/passwd (owner root, mode 4755)
   -> after execve: real uid = 1000, effective uid = 0, saved uid = 0
   -> open("/etc/shadow", O_RDWR) is checked against euid 0 -> allowed
```

| Bit | Octal | On a file | On a directory |
| :---- | :---- | :---- | :---- |
| **setuid** (`s` in user x) | `4000` | Run with the **owner's** UID | (ignored on Linux) |
| **setgid** (`s` in group x) | `2000` | Run with the file's **group** | New files inherit the **directory's group** (shared project dirs) |
| **sticky** (`t` in other x) | `1000` | (obsolete) | Only a file's **owner** (or the dir's owner/root) may delete or rename it, even if the dir is world-writable: `/tmp` is `drwxrwxrwt` |

Security notes: setuid programs are prime attack targets (any bug runs as root). Linux **ignores setuid on scripts** (`#!`), because of race conditions between the kernel checking the file and the interpreter opening it. Filesystems mounted with `nosuid` ignore the bit entirely. A safer modern alternative is to grant just one **capability** (10.4), e.g. `ping` with `cap_net_raw` instead of setuid root.

```bash
$ find / -perm -4000 -type f 2>/dev/null     # audit setuid binaries
$ chmod u+s ./tool ; chmod 4755 ./tool       # set it
```

## 7.12 Device Files, udev and devtmpfs

The kernel presents many device I/O interfaces to user processes as **files**. These **device files** (sometimes called **device nodes**) live in `/dev`.

```bash
$ ls -l /dev/sda /dev/sda1 /dev/null /dev/tty1 /dev/nvme0n1
brw-rw---- 1 root disk    8,   0 Oct 16 08:51 /dev/sda
brw-rw---- 1 root disk    8,   1 Oct 16 08:51 /dev/sda1
crw-rw-rw- 1 root root    1,   3 Oct 16 08:51 /dev/null
crw--w---- 1 root tty     4,   1 Oct 16 08:51 /dev/tty1
brw-rw---- 1 root disk  259,   0 Oct 16 08:51 /dev/nvme0n1
```

If the first character is `b` or `c`, the file is a device. Where a regular file shows its size, a device shows two numbers: **major, minor**. These are the **device numbers** the kernel uses to identify the device:

- **major** → which **driver** (8 = SCSI/SATA disks, 1 = memory devices, 4 = TTYs, 259 = NVMe/block extended)
- **minor** → which **instance** within that driver (sda = 0, sda1 = 1, ...)

A device file has **no data blocks**. Its inode just stores type + major/minor. When you `open()` it, the VFS sees the type and routes all `read`/`write`/`ioctl` calls to the driver registered for that major number. That's "everything is a file" (7.4) applied to hardware.

(On macOS, `ls -l /dev` shows the device number as one hex value, like `0x1000004`, instead of `major, minor`.)

| Type | Char | Description | Examples |
| :---- | :---- | :---- | :---- |
| **Block device** | `b` | Data accessed in **fixed-size blocks**, with **random access** to any block, through the kernel's **block layer** (and page cache). Total size is fixed and easy to index. | Disks and partitions: `/dev/sda`, `/dev/nvme0n1p2`, `/dev/loop0` |
| **Character device** | `c` | **Stream** of bytes; no fixed size; reads/writes go (mostly) straight to the driver. | `/dev/null`, `/dev/zero`, `/dev/urandom`, `/dev/tty*`, `/dev/pts/*`, `/dev/input/*`, `/dev/kvm` |
| **Named pipe** | `p` | Not really a device: a FIFO with **another process** at the other end of the I/O stream instead of a kernel driver (9.1) | `mkfifo mypipe` |
| **Socket** | `s` | Not a device either: a Unix domain socket file used for IPC (9.2) | `/run/docker.sock`, `/dev/log` |

Other letters you'll see in `ls -l`: `d` directory, `l` symlink, `-` regular file.

Some handy character devices:

```bash
$ cmd > /dev/null                      # discard output (writes always succeed, reads return EOF)
$ head -c 16 /dev/urandom | xxd        # random bytes from the kernel CSPRNG
$ dd if=/dev/zero of=img bs=1M count=100   # 100 MB of zeros
$ echo hi > /dev/pts/3                 # write to another terminal!
```

### udev

Devices come and go (USB sticks, hot-plugged disks, network cards). Who creates `/dev/sdb` when you plug in a USB drive, loads the driver, and sets permissions?

The kernel sends a notification to a user-space daemon called **`udevd`** (part of systemd: `systemd-udevd`) whenever it detects a device change. `udevd` examines the new device's characteristics, performs device initialization, and can run actions. It works as follows:

1. The kernel sends `udevd` a notification event, called a **uevent**, through an internal network link (a netlink socket, 9.5).
2. `udevd` loads all of the attributes in the uevent (from the event and from `/sys`, 7.10).
3. `udevd` parses its **rules**, filters and updates the uevent based on those rules, and takes actions or sets more attributes accordingly. The rule files are in **`/lib/udev/rules.d`** (distribution defaults) and **`/etc/udev/rules.d`** (your overrides).

Typical actions: load a kernel module (`modprobe` by `MODALIAS`, 3.1), set owner/group/permissions on the device node, create **stable symlinks**, rename network interfaces (`eth0` → `enp3s0`), or run a script.

```bash
$ udevadm monitor --environment --udev      # then plug in a USB stick:
UDEV  [1234.56] add      /devices/pci0000:00/.../block/sdb (block)
ACTION=add
DEVNAME=/dev/sdb
ID_SERIAL=SanDisk_Ultra_4C53...
ID_FS_TYPE=vfat
$ udevadm info /dev/sda                     # all attributes udev knows about a device
```

```
# /etc/udev/rules.d/99-mydevice.rules: give a specific USB serial adapter a stable name
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", SYMLINK+="arduino", MODE="0666"
```

### devtmpfs

Historically, `udevd` created device files itself, which raised a problem at boot: you need `/dev` entries to start `udevd`. Today the **kernel creates device nodes itself**, in **devtmpfs**, a RAM filesystem mounted on `/dev`, as soon as a driver registers a device.

The kernel creates device files as necessary, and also notifies `udevd` that a new device is available. Upon receiving this signal, `udevd` does **not** create the device files, but it does perform device initialization, set permissions, notify other processes that new devices are available, and create a number of **symbolic links** in `/dev` to further identify devices:

```bash
$ ls -l /dev/disk/by-uuid/ /dev/disk/by-id/ | head -4
lrwxrwxrwx 1 root root 15 ... 3f1e...a9 -> ../../nvme0n1p2      <- created by udev
lrwxrwxrwx 1 root root 13 ... nvme-Samsung_SSD_980_... -> ../../nvme0n1
```

That's why `/etc/fstab` and the kernel command line (3.7) use `UUID=...` rather than `/dev/sda2`: device names depend on detection order; UUIDs don't.

## 7.13 Mounting, Bind Mounts and the Mount Table

Linux has **one directory tree**. Every filesystem (each disk partition, a USB stick, tmpfs, procfs...) has its **own** tree of inodes, starting at its own root inode. **Mounting** attaches a filesystem's root onto a directory (the **mount point**) of the existing tree:

```
/  (ext4 on nvme0n1p2, root inode 2)
|-- etc
|-- home  <-- mount point: path walk jumps to the ROOT of xfs on /dev/sdb1 (also inode 2 there!)
|    `-- berat
|-- proc  <-- procfs (no device)
|-- tmp   <-- tmpfs (RAM)
`-- mnt
     `-- usb <-- vfat on /dev/sdc1
```

```bash
$ sudo mount /dev/sdc1 /mnt/usb              # attach; the old contents of /mnt/usb are hidden, not deleted
$ findmnt                                     # tree of all mounts
$ cat /proc/self/mounts                       # the mount table as the kernel sees it (per mount namespace, 10.1)
$ stat -c '%d %i %n' / /home                  # device id + inode: (device, inode) is the true identity
2050 2 /
2065 2 /home                                  # both are inode 2: root inodes of different filesystems
$ sudo umount /mnt/usb                        # "target is busy" = some process has a cwd or open fd in it (lsof +D)
```

During path resolution (7.3), whenever the kernel reaches a dentry that's a mount point, it **crosses into the mounted filesystem's root**. This is exactly why:

- **hard links can't cross mount points** (`EXDEV`): different filesystems, different inode number spaces;
- **`rename()` can't cross filesystems** either: `mv` across filesystems silently falls back to **copy + delete**, which is slow and gives the file a new inode;
- `..` at the root of a mounted filesystem goes back to the parent filesystem.

Mount options change behavior for everything under the mount: `ro`, `noexec` (execve fails, 4.9), `nosuid` (setuid ignored, 7.11), `nodev` (device files don't work), `noatime` (don't update atime on reads).

### Bind Mounts

A **bind mount** makes an existing directory (or file) **appear at a second place in the tree**. It's not a copy and not a link file: it's another **mount** of the same underlying dentry/inodes.

```bash
$ sudo mount --bind /data/projects /home/berat/projects
$ ls -i /data/projects/a.txt /home/berat/projects/a.txt
1311 /data/projects/a.txt
1311 /home/berat/projects/a.txt          # same inode: the same file, two paths
$ sudo mount -o remount,bind,ro /home/berat/projects    # the second view can even be read-only
```

| | Bind mount | Symlink |
| :---- | :---- | :---- |
| Level | **Kernel mount table** (VFS) | A **file** containing a path, stored in a filesystem |
| Resolved | Once, when mounted: points at the actual directory | **Every access**: the path is re-resolved |
| Inside chroot / container | **Works**: it's part of the mount tree the container sees | An absolute target like `/data/x` is resolved **inside** the container's root and usually points nowhere (10.3) |
| Who can create | root / `CAP_SYS_ADMIN` (or inside a user namespace) | Anyone with write access to the directory |
| Visible as | Looks exactly like the original; `ls -l` shows a normal directory | `l` type, `-> target` |
| Survives reboot | Only if in `/etc/fstab` | Yes, it's a file |
| Can be read-only on its own | Yes | No (target's permissions apply) |

This is how Docker volumes (`-v /host/path:/container/path`), Kubernetes `hostPath`, and `/etc/resolv.conf` inside containers work (10.3).

## 7.14 Linux Directory Hierarchy (FHS)

```
/
|-- bin     -> usr/bin     ready-to-run programs: ls, cp, grep, ps, ping
|-- sbin    -> usr/sbin    system administration programs: fdisk, iptables, reboot, ip
|-- lib     -> usr/lib     shared libraries (libc.so.6, ld-linux) and kernel modules (/lib/modules)
|-- boot                   kernel (vmlinuz), initramfs, GRUB config           (3.7)
|-- dev                    device files (devtmpfs)                             (7.12)
|-- etc                    system-wide configuration
|-- home                   users' personal directories
|-- root                   root's home directory
|-- proc                   process & kernel info (procfs, virtual)             (7.10)
|-- sys                    device model & kernel tunables (sysfs, virtual)     (7.10)
|-- run                    runtime data since boot (tmpfs): PID files, sockets
|-- tmp                    temporary files (often tmpfs; cleaned at boot), sticky bit (7.11)
|-- usr                    second-level hierarchy: the bulk of installed software
|   |-- bin, sbin, lib     programs and libraries
|   |-- share              architecture-independent data: man pages, docs
|   |-- include            C headers
|   `-- local              software installed locally by the admin
|-- var                    data that changes over time
|   |-- log                log files
|   |-- lib                persistent state of programs (databases, package manager)
|   |-- cache              caches that can be regenerated
|   `-- spool              queues (mail, print, cron)
|-- opt                    add-on, self-contained third-party packages
|-- mnt, media             mount points: temporary / removable media           (7.13)
`-- srv                    data served by this system (web, ftp)
```

- **`/bin`**: ready-to-run programs (executables) needed by everyone, e.g. `ps`, `ls`, `ping`, `grep`, `cp`.
- **`/dev`**: device files (7.12).
- **`/etc`**: the core system configuration directory: user database (`/etc/passwd`, `/etc/shadow`, `/etc/group`), boot, device, and networking setup (`/etc/fstab`, `/etc/hosts`, `/etc/resolv.conf`), and service configuration (`/etc/ssh/sshd_config`, `/etc/logrotate.conf`, systemd units in `/etc/systemd/system`). Historically also start/stop scripts (`/etc/init.d`).
- **`/home`**: home (personal) directories for regular users.
- **`/lib`**: library files that support the binaries in `/bin` and `/sbin` (4.9, dynamic linking), plus kernel modules in `/lib/modules/$(uname -r)` (3.1).
- **`/proc`**: system statistics through a browsable directory and file interface: information about running processes and some kernel parameters. A virtual filesystem; files exist only in memory (7.10).
- **`/run`**: runtime data specific to this boot: PID files, socket files (`/run/docker.sock`, 9.2), status records. tmpfs, so it's empty after reboot.
- **`/sys`**: similar to `/proc` in that it provides a device and system interface (7.10).
- **`/sbin`**: system executables related to system management, e.g. `iptables`, `reboot`, `fdisk`, `ip`, `swapon`.
- **`/tmp`**: temporary files; cleaned at boot (and often a RAM-backed tmpfs).
- **`/usr`**: a large directory hierarchy with binaries, libraries, documentation and source code for second-level programs. On modern distros ("usr merge"), `/bin`, `/sbin` and `/lib` are just symlinks into `/usr`.
- **`/var`**: where programs record information that changes over time. Files expected to grow (logs, databases, caches, mail spools) live here.

## 7.15 The Storage Stack: Block Layer, I/O Schedulers and Journaling

What happens below the page cache when data actually goes to disk?

```
write() / fsync()                                               user space
-------------------------------------------------------------------------------
VFS (7.4)
page cache: dirty pages (6.7)
filesystem (ext4, xfs, btrfs): decides WHICH blocks; JOURNALS metadata
-------------------------------------------------------------------------------
BLOCK LAYER
  bio  ("write these sectors from these pages")
   -> merged/sorted into requests
   -> blk-mq: per-CPU software queues -> I/O scheduler -> hardware queues
-------------------------------------------------------------------------------
device driver (nvme, ahci/sata, virtio-blk, dm/LVM, md/RAID)
-------------------------------------------------------------------------------
device: DMA from RAM (2.5) -> device's own volatile write cache -> flash / platters
        completion -> INTERRUPT (3.4) -> wake the waiting task
```

### The Block Layer and blk-mq

The filesystem describes I/O as **bios** (block I/O structures: a sector range plus memory pages). The block layer merges adjacent bios into larger requests and dispatches them. Modern drives can process many commands in parallel (NVMe supports up to 64K queues), so **blk-mq** (multi-queue) gives each CPU its own submission queue, avoiding a single global lock. Same idea as per-CPU run queues (3.6).

### I/O Schedulers

An I/O scheduler decides the **order** in which requests are sent to the device:

| Scheduler | Strategy | Good for |
| :---- | :---- | :---- |
| `none` | No reordering; pass requests straight through | Fast NVMe SSDs (the default for them): the device reorders better than the kernel can |
| `mq-deadline` | Sort and merge requests, but guarantee each one a deadline so none starves (5.2) | SATA SSDs, HDDs, many servers |
| `bfq` | Budget Fair Queueing: fair share of disk time per process/cgroup; honors `ionice` | Desktops, slow disks, interactive responsiveness |
| `kyber` | Token-based latency targets for reads vs writes | Fast multi-queue devices |

```bash
$ cat /sys/block/nvme0n1/queue/scheduler
[none] mq-deadline kyber bfq
$ echo bfq | sudo tee /sys/block/sda/queue/scheduler
$ iostat -x 1          # per-device: r/s, w/s, await (ms per request), %util
```

On HDDs, reordering matters enormously: a seek costs ~5–10 ms (2.4), so serving requests in disk order instead of arrival order can multiply throughput. On SSDs there's no seek, which is why `none` is often best.

### Journaling: Surviving a Crash

Creating one file touches **several blocks** in different places: the inode bitmap, the inode itself, the directory's data block, the block bitmap, and the data. If the power fails halfway, the filesystem is **inconsistent**: for example, an inode is marked used but no directory entry points to it, or worse, a block is owned by two files. Older filesystems (ext2) needed a full `fsck` scan of the disk after every crash, which could take hours.

A **journal** (write-ahead log) fixes this:

```
1. write the planned metadata changes to the JOURNAL (a dedicated area of the disk)
2. write a COMMIT record                          <- the transaction now "happened"
3. write the changes to their real locations ("checkpoint")
4. mark the journal space free

crash before 2:  the transaction is incomplete -> ignored on mount: as if nothing happened
crash after 2:   on mount, REPLAY the committed transaction from the journal -> consistent
```

Mounting after a crash takes seconds instead of hours. ext4 has three modes (`data=` mount option):

| Mode | What's journaled | Guarantee |
| :---- | :---- | :---- |
| `journal` | Metadata **and** data | Strongest; data is written twice (slow) |
| `ordered` (default) | Metadata only, but data blocks are written **before** the metadata that points to them is committed | A file never points at garbage blocks after a crash |
| `writeback` | Metadata only, no ordering | Fastest; a crash can leave old data inside recently written files |

Other designs: XFS also journals metadata; **copy-on-write** filesystems (**Btrfs**, **ZFS**) never overwrite blocks in place. They write new versions elsewhere and atomically switch a root pointer, which also gives cheap **snapshots**.

**Device caches matter too:** the drive has its own volatile write cache. To make data durable, the kernel sends **FLUSH** / **FUA** (Force Unit Access) commands; that's part of what `fsync()` does (6.7), and why it can take milliseconds.

### The Safe Way to Update a File

Journaling protects the filesystem's **structure**, not your file's **contents**. If you rewrite `config.json` in place and crash halfway, you get a half-written file. The standard pattern combines `fsync` (6.7) with the atomicity of `rename()` (Part 12, `mv`):

```c
int fd = open("config.json.tmp", O_WRONLY | O_CREAT | O_TRUNC, 0644);  // 1. temp file, SAME directory/fs
write(fd, data, len);
fsync(fd);                                // 2. the new content is durable on disk
close(fd);
rename("config.json.tmp", "config.json"); // 3. atomic swap: readers see old OR new, never half
int dir = open(".", O_RDONLY | O_DIRECTORY);
fsync(dir);                               // 4. make the rename itself (a directory change) durable
close(dir);
```

Skip step 2 and a crash may leave a renamed but **empty** file (the rename's metadata reached the disk before the data did). Skip step 4 and after a crash you may still have the **old** file. Databases, package managers, and editors (`vim`'s `writebackup`) do this dance.

> 🔗 **Connections:** Files tie the system together: the fd table hangs off `task_struct` (4.2); every read/write is a syscall (3.3) through the VFS into the page cache (6.7); permission checks use credentials (7.11) that setuid and capabilities (10.4) modify; devices are files thanks to the VFS and udev; and mounts plus bind mounts are the raw material containers are made of (10.3).

---

# 8. Signals

> 🧪 **Interactive visuals:** [Part 8, visualized](visuals/part8-signals.html): an interactive page with a demo for each section.


## 8.1 What Signals Are

A **signal** is a kind of (usually software) interrupt, used to announce **asynchronous events** to a process. It's the kernel's way of tapping a process on the shoulder: "something happened". A signal carries almost no data: just its **number** (and, with `sigaction`'s `SA_SIGINFO`, a little info like the sender's PID).

- **Synchronous** signals are caused by the process itself, by the instruction it's executing: `SIGSEGV` on an invalid memory access, `SIGFPE` on an integer divide-by-zero, `SIGILL`, `SIGBUS`. These come from CPU **exceptions** (3.4).
- **Asynchronous** signals are sent from outside, at any moment: `kill`, `Ctrl+C`, a child exiting (`SIGCHLD`), a timer expiring (`SIGALRM`), a broken pipe.

### Signal Lifecycle

```
GENERATION          ->  PENDING (in the kernel)    ->  DELIVERY             ->  ACTION
user: Ctrl+C            bit set in the task's          when the task next        default action,
kernel: SIGSEGV,        pending set; if blocked,       returns to user mode      ignore, or run
        SIGCHLD         it waits there (8.4)           (or is woken from S)      the handler
process: kill()
```

1. **Generation:** a signal is generated when an event occurs. This can be initiated by:
   - **The user:** pressing certain key combinations (e.g., `Ctrl+C` for `SIGINT`), via the TTY driver (4.5).
   - **The kernel:** notifying the process of a system event (e.g., illegal memory access causing `SIGSEGV`, a child terminating causing `SIGCHLD`, writing to a closed pipe causing `SIGPIPE`).
   - **Another process:** using system calls like `kill()`, if it has permission: the sender's real **or** effective UID must match the target's real **or** saved UID, or the sender has `CAP_KILL`.
2. **Pending:** the kernel records the signal in the target's **pending set** (a bitmask in `task_struct`). If the same standard signal is already pending, it's **not queued twice**: the bit is just set. (Real-time signals *can* queue.)
3. **Delivery:** the signal is acted on when the target is about to **return to user mode** (after a syscall, interrupt, or being scheduled), unless it's **blocked** (8.4). If the target is sleeping in state `S`, the kernel wakes it up; the interrupted system call returns `EINTR` (or is restarted with `SA_RESTART`). A task in state `D` (4.8) is not woken.
4. **Action:** the process receives the signal and takes one of the actions below.

How delivery really works, by the way: before returning to user space, the kernel checks the `TIF_SIGPENDING` flag. If a handler is installed, it **builds a signal frame on the user stack** (saving the interrupted registers), sets `rip` to the handler, and returns to user mode. So the handler runs *as if the interrupted code had called it*. When the handler returns, it lands in a tiny trampoline that calls the **`rt_sigreturn`** syscall, which restores the saved registers, and the original code resumes as if nothing happened.

> 🔗 **Connections:** Signal delivery piggybacks on **mode switches** (3.5): signals are checked on every return to user mode. That's why a process stuck in an uninterruptible kernel wait (`D`) can't die even from `SIGKILL`: it never gets to that return path.

## 8.2 Common Signals (`SIGINT`, `SIGTERM`, `SIGKILL`, `SIGSTOP`, `SIGCHLD`, `SIGSEGV`, `SIGPIPE`)

Each signal is defined with a number. Linux has **64 signal numbers**: 1–31 are the standard signals, named with `SIG` followed by a description; 32–64 are **real-time signals** (`SIGRTMIN`..`SIGRTMAX`; glibc reserves the first two internally for threads). Numbers below are for x86/ARM; some differ on other architectures, so prefer names (`kill -TERM`) over numbers.

| Signal | No. | Default action | Meaning / typical use |
| :---- | :---- | :---- | :---- |
| `SIGHUP` | 1 | Terminate | Terminal hung up (4.5). Daemons conventionally use it as **"reload config"** (`nginx -s reload`) |
| `SIGINT` | 2 | Terminate | Interrupt from keyboard (`Ctrl+C`): politely stop the foreground job |
| `SIGQUIT` | 3 | Core dump | `Ctrl+\`: quit and dump core (for debugging) |
| `SIGILL` | 4 | Core dump | Illegal instruction |
| `SIGTRAP` | 5 | Core dump | Breakpoint/trace trap (debuggers) |
| `SIGABRT` | 6 | Core dump | `abort()`, failed `assert()` |
| `SIGBUS` | 7 | Core dump | Bus error: misaligned access on strict CPUs (1.6), or touching an mmap past end of file (6.6) |
| `SIGFPE` | 8 | Core dump | Arithmetic error (integer divide by zero) |
| `SIGKILL` | 9 | Terminate | **Kill. Cannot be caught, blocked or ignored.** Ungraceful. Also used by the OOM killer (6.8) |
| `SIGUSR1` / `SIGUSR2` | 10 / 12 | Terminate | User-defined: apps choose (e.g. `dd` prints progress on `SIGUSR1`) |
| `SIGSEGV` | 11 | Core dump | Segmentation fault: invalid memory access (6.5) |
| `SIGPIPE` | 13 | Terminate | Wrote to a pipe/socket with **no reader** (9.1) |
| `SIGALRM` | 14 | Terminate | Timer from `alarm()` expired |
| `SIGTERM` | 15 | Terminate | **Termination request**: the polite, catchable "please exit". Default of `kill`, `systemctl stop`, `docker stop` |
| `SIGCHLD` | 17 | Ignore | A child stopped or exited (4.9) |
| `SIGCONT` | 18 | Continue | Resume a stopped process |
| `SIGSTOP` | 19 | Stop | **Pause. Cannot be caught, blocked or ignored.** |
| `SIGTSTP` | 20 | Stop | Terminal stop (`Ctrl+Z`): catchable version of `SIGSTOP` |
| `SIGTTIN` / `SIGTTOU` | 21 / 22 | Stop | Background job tried to read from / write to the terminal |
| `SIGWINCH` | 28 | Ignore | Terminal window resized (`vim`, `less` redraw) |

```bash
$ kill -l                    # list all signals
$ kill 1234                  # SIGTERM (15) by default
$ kill -TERM 1234            # same, by name
$ kill -9 1234               # SIGKILL: last resort
$ kill -0 1234               # signal 0: sends nothing, just checks "does it exist & may I signal it?"
$ pkill -HUP nginx           # by name
```

**Why `SIGTERM` first, `SIGKILL` last:** `SIGTERM` lets the process flush buffers (7.6), finish requests, remove temp files and lock files, and tell its children. `SIGKILL` gives it no chance at all. `docker stop` / Kubernetes send `SIGTERM`, wait (10 s / 30 s by default), then `SIGKILL`.

## 8.3 Signal Dispositions and Handlers

For each signal, a process has a **disposition**: what to do on delivery. There are 3 possibilities:

1. **Default action (`SIG_DFL`):** what the kernel does by default unless the process sets something else. Depending on the signal: **terminate**, **terminate + core dump**, **ignore**, **stop**, or **continue**.
   - `SIGTERM` → terminate
   - `SIGKILL` → terminate
   - `SIGCHLD` → ignore
   - `SIGSTOP` → stop the process
2. **Ignore (`SIG_IGN`):** the process can choose to ignore most signals: they're discarded on arrival.

   ```c
   signal(SIGINT, SIG_IGN);  // Ignore Ctrl+C
   ```

3. **Custom handler:** a function the programmer writes, which runs when the signal is delivered, if and only if the disposition is set to that handler. The process can then gracefully shut down, clean up resources, reload configuration, etc.

**`SIGKILL` and `SIGSTOP` cannot be caught, blocked, or ignored.** They're handled entirely by the kernel and bypass user space. That's the system's guarantee that any process can always be stopped or killed (except one stuck in `D` state until its I/O finishes, and PID 1, 3.7).

Use **`sigaction()`** rather than the old `signal()` (whose semantics vary between systems):

```c
#include <signal.h>
#include <unistd.h>

static volatile sig_atomic_t stop = 0;

static void on_term(int sig) {
    stop = 1;                          // just set a flag: keep handlers tiny
}

int main(void) {
    struct sigaction sa = {0};
    sa.sa_handler = on_term;
    sigemptyset(&sa.sa_mask);          // extra signals to block while the handler runs
    sa.sa_flags = SA_RESTART;          // auto-restart interrupted syscalls like read()
    sigaction(SIGTERM, &sa, NULL);
    sigaction(SIGINT,  &sa, NULL);

    while (!stop) {
        /* do work */
        sleep(1);
    }
    /* graceful cleanup: flush, close, remove pid file */
    return 0;
}
```

**Handlers run asynchronously**, interrupting your code at any instruction, possibly in the middle of `malloc` updating its free lists. If the handler calls `malloc` or `printf` too, it can corrupt that state or deadlock on a lock that the interrupted code is holding. So handlers may only call **async-signal-safe** functions (`write`, `_exit`, `kill`, `waitpid`... see `man 7 signal-safety`). The safe patterns are:

- set a `volatile sig_atomic_t` flag and check it in the main loop (above)
- the **self-pipe trick**: write one byte to a pipe that the main `epoll` loop watches (7.8)
- **`signalfd()`**: block the signals and read them as data from an fd, in your normal event loop (everything is a file, 7.4)

Where it lives in the kernel:

```
Process (task_struct)
|-- pending                // Pending signals for this thread
|-- blocked                // Mask: which signals are blocked (per thread)
|-- sighand (shared by all threads)
|     `-- action[64]       // One k_sigaction per signal: handler / SIG_DFL / SIG_IGN, flags, mask
`-- signal (shared by all threads)
      |-- shared_pending   // Process-wide pending signals (thread group level)
      |-- rlim, timers     // Resource limits, timers, etc.
      `-- job control info // fg/bg state, orphaned process group status, etc.
```

Across `fork` and `exec` (4.9): `fork` copies dispositions; `exec` resets **handled** signals to default (the handler code is gone) but keeps **ignored** ones ignored. That's how `nohup` works: it sets `SIGHUP` to `SIG_IGN` and then execs your command.

## 8.4 Signal Masks and Pending Signals

A process (each thread, actually) can **block** signals: a blocked signal isn't lost, it stays **pending** until unblocked, then is delivered. Blocking ≠ ignoring.

```c
sigset_t set, old;
sigemptyset(&set);
sigaddset(&set, SIGINT);

sigprocmask(SIG_BLOCK, &set, &old);     // Ctrl+C now stays pending
/* critical section: update a config file atomically */
sigprocmask(SIG_SETMASK, &old, NULL);   // unblock -> pending SIGINT is delivered NOW
```

```bash
$ grep -E 'Sig(Pnd|Blk|Ign|Cgt)|ShdPnd' /proc/$$/status
SigPnd: 0000000000000000     <- pending for this thread
ShdPnd: 0000000000000000     <- pending for the whole process
SigBlk: 0000000000010000     <- blocked mask: bit 16 = signal 17 (SIGCHLD)
SigIgn: 0000000000380004     <- ignored: bit 2 = signal 3 (SIGQUIT), bits 19-21 = SIGTSTP/TTIN/TTOU
SigCgt: 000000004b817efb     <- caught (handler installed)
```

Each mask is a 64-bit number; **bit (N−1)** represents signal N (1.1).

**Standard signals don't queue.** The pending set is a bitmask: if `SIGCHLD` arrives 5 times while blocked, it's delivered **once**. That's why a `SIGCHLD` handler must reap in a loop (4.9). **Real-time signals** (`SIGRTMIN+n`) are queued, delivered in order, and can carry a value (`sigqueue()`).

## 8.5 Signals and Threads

Signals were designed before threads, so the rules are a bit special:

- **Dispositions are per process:** all threads share one handler table (`sighand`). A thread can't have its own `SIGINT` handler.
- **Masks are per thread:** each thread has its own `blocked` set (`pthread_sigmask()`).
- **Process-directed signals** (`kill(pid, ...)`, `Ctrl+C`) go to the process's shared pending set and are delivered to **any one thread that doesn't block it**.
- **Thread-directed signals** go to one specific thread: synchronous ones like `SIGSEGV`/`SIGFPE` go to the **thread that caused** them; `pthread_kill()`/`tgkill()` target a specific thread.
- A **fatal** signal (like `SIGKILL`, or an unhandled `SIGSEGV`) kills the **whole process**, all threads.
- `SIGSTOP`/`SIGCONT` stop/continue all threads.

The common multithreaded pattern: **block all signals in every thread** (set the mask before creating threads, since new threads inherit it), and have **one dedicated thread** call `sigwait()` (or read a `signalfd`) to receive them synchronously, in normal code, without async-safety worries.

> 🔗 **Connections:** Signals connect almost everything: CPU exceptions (3.4) become `SIGSEGV`/`SIGFPE`; the TTY driver and process groups (4.5) turn keystrokes into `SIGINT`/`SIGTSTP`; process exit (4.9) sends `SIGCHLD` and leaves zombies until `wait()`; the OOM killer (6.8) uses `SIGKILL` (exit 137); pipes (9.1) send `SIGPIPE`; and PID 1 in a container (10.3) ignores signals it has no handler for.

---

# 9. Inter-Process Communication (IPC)

> 🧪 **Interactive visuals:** [Part 9, visualized](visuals/part9-ipc.html): an interactive page with a demo for each section.


Processes are isolated by design (illusion 2: separate address spaces). So when they *need* to talk, they must go through the kernel. Linux offers many mechanisms; which one fits depends on how closely related the processes are (parent/child or unrelated), how fast they need to talk, how much data they exchange, and whether they're on the same machine.

| Mechanism | Data? | Related processes only? | Across machines? | Speed | Typical use |
| :---- | :---- | :---- | :---- | :---- | :---- |
| Signals (Part 8) | No (just a number) | No | No | Fast | Notifications: stop, reload, child exited |
| Anonymous pipe | Byte stream, one-way | **Yes** (inherited fd) | No | Fast | Shell pipelines |
| Named pipe (FIFO) | Byte stream, one-way | No (has a path) | No | Fast | Simple producer/consumer |
| Unix domain socket | Stream or datagrams, two-way, can pass fds | No (path or abstract name) | No | Fast | Local services: Docker, X11/Wayland, systemd, databases |
| Shared memory | Raw memory | No | No | **Fastest** (no copy) | Large data, high throughput |
| Message queue | Discrete messages | No | No | Medium | Structured messaging |
| Semaphores / futexes | No (synchronization) | No | No | Fast | Coordinate shared memory access |
| TCP/UDP socket | Stream / datagrams | No | **Yes** | Slower (network stack) | Everything networked |
| Files | Bytes | No | (via network fs) | Slow | Simple, persistent exchange |
| D-Bus, gRPC, ZeroMQ | Structured messages, RPC | No | gRPC/ZMQ: yes | Varies | Higher-level service communication |

## 9.1 Pipes and Named Pipes (FIFOs)

### Anonymous Pipes (`|`)

A **pipe** is a one-way byte stream with a buffer in the kernel (64 KiB by default). `pipe()` returns **two fds**: one for reading, one for writing.

```c
int fds[2];
pipe(fds);          // fds[0] = read end, fds[1] = write end
```

```
writer process                  kernel pipe buffer (64 KiB ring)              reader process
write(fds[1], ...)  ------>  [ h e l l o \n . . . . . . . . . ]  ------>  read(fds[0], ...)
```

Because a pipe has **no name** in the filesystem, the only way to share it is to **inherit its fds** through `fork()`. So anonymous pipes work only between **related processes** (parent and child, or siblings).

```bash
ls | grep txt
```

Behind the scenes, a pipe connects `ls`'s stdout to `grep`'s stdin. The shell does `pipe()`, two `fork()`s, and `dup2()` in each child (full walkthrough in Part 12).

Pipe semantics that matter:

- **Reader blocks** when the pipe is empty (state `S`, 7.7); **writer blocks** when it's full. This automatically throttles a fast producer to the speed of a slow consumer (**backpressure**).
- `read()` returns **0 (EOF)** only when the buffer is empty **and all write ends are closed** in every process. Forget to `close()` an unused write end and the reader waits forever.
- Writing when **all read ends are closed** → the writer gets **`SIGPIPE`** (default: terminate), or `EPIPE` if it ignores the signal. That's why `yes | head -1` terminates: `head` exits, `yes` gets `SIGPIPE`.
- Writes of up to `PIPE_BUF` (4 KiB) are **atomic**: multiple writers won't interleave within such a write.

✅ Good for simple streams. ❌ One-way, no message boundaries (just bytes), related processes only.

### Named Pipes (`mkfifo`)

A **FIFO** is a pipe with a **name** in the filesystem (a special file of type `p`, 7.12), so **unrelated processes** can open it by path. The data still goes through a kernel buffer, never to disk.

```bash
$ mkfifo mypipe
$ ls -l mypipe
prw-r--r-- 1 berat berat 0 Oct 16 18:00 mypipe
$ echo "hello" > mypipe &      # blocks in open() until a reader opens the other end
$ cat < mypipe
hello
```

## 9.2 Unix Domain Sockets

**Unix domain sockets** (`AF_UNIX`) use the socket API (9.5), but for processes **on the same machine**. They're identified by a **file path** (socket file, type `s`) or by an abstract name. Data goes through the kernel without any network stack: no TCP, no checksums, no routing.

Examples: `/run/docker.sock` (the Docker CLI talks to `dockerd` through it), `/var/run/postgresql/.s.PGSQL.5432`, `/tmp/.X11-unix/X0`, `/run/systemd/journal/socket`, `/tmp/nginx.socket`.

```bash
$ ls -l /run/docker.sock
srw-rw---- 1 root docker 0 Oct 16 08:52 /run/docker.sock      <- permissions = access control (7.11)
$ curl --unix-socket /run/docker.sock http://localhost/version
$ ss -xlp | head -3                                           # list listening unix sockets
```

What makes them more than a local TCP:

- **Bidirectional**, with both `SOCK_STREAM` (byte stream) and `SOCK_DGRAM`/`SOCK_SEQPACKET` (message boundaries preserved).
- **Faster** than TCP over loopback.
- **Credential passing** (`SO_PEERCRED`): the server learns the client's PID/UID/GID, verified by the kernel. That's how `dockerd`, `systemd` and PostgreSQL "peer" authentication know who's calling.
- **File descriptor passing** (`SCM_RIGHTS`): send an **open fd** to another process. The receiver gets a new fd pointing to the **same `struct file`** (7.2). Used by systemd socket activation, Wayland, privilege-separated daemons.
- Access is controlled by **file permissions** on the socket path; being in the `docker` group = being allowed to write to `docker.sock` = effectively root.

## 9.3 Shared Memory

The **fastest** IPC: two processes map **the same physical pages** into their address spaces (6.6). After setup, communication is just memory reads and writes: **no system calls, no copying through the kernel**.

```
process A virtual 0x7f10...  --+
                               +--> same physical frames (page cache of a tmpfs file)
process B virtual 0x7fa3...  --+          (different virtual addresses, same RAM)
```

POSIX shared memory (modern API): it's literally a file in `/dev/shm` (a tmpfs) mapped with `MAP_SHARED`:

```c
// writer
int fd = shm_open("/mycounter", O_CREAT | O_RDWR, 0600);   // creates /dev/shm/mycounter
ftruncate(fd, 4096);
long *p = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
*p = 42;

// reader (another, unrelated process)
int fd = shm_open("/mycounter", O_RDWR, 0);
long *p = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
printf("%ld\n", *p);    // 42
```

The older System V API uses `shmget`, `shmat`, `shmdt` with integer keys (`ipcs -m` lists segments).

✅ Fastest IPC. ❌ **Requires synchronization**: it brings back every race condition from Part 5 between processes. Use process-shared mutexes (`PTHREAD_PROCESS_SHARED`, which work because futexes can be keyed by the shared physical page, 5.3), semaphores, or atomics in the shared region. PostgreSQL's shared buffers, Chrome's renderer ↔ GPU process communication, and many databases use shared memory.

## 9.4 Message Queues

**Kernel-managed queues** where processes send and receive **discrete messages** (boundaries preserved, unlike pipes), optionally with types or priorities.

- **System V** message queues: identified by keys; `msgget`, `msgsnd`, `msgrcv`. Messages have a numeric type, so a receiver can pick specific types. `ipcs -q` lists them.
- **POSIX** message queues: `mq_open("/name")`, `mq_send`, `mq_receive`, with priorities. They're visible as files under `/dev/mqueue` and the queue descriptor works with `epoll` (7.8).

```c
mqd_t q = mq_open("/jobs", O_CREAT | O_RDWR, 0600, NULL);
mq_send(q, "resize img42", 13, /*priority*/ 5);
char buf[8192]; unsigned prio;
mq_receive(q, buf, sizeof buf, &prio);
```

✅ Structured messaging, persists until removed (independent of the processes). ❌ Slower than pipes or shared memory for bulk data (each message is copied into and out of the kernel); size limits (`/proc/sys/fs/mqueue/`).

### Semaphores and Mutexes (for IPC)

Not for data exchange directly: used for **synchronization**, to ensure only one process accesses shared memory (or a critical section) at a time. Same concepts as 5.3, but process-shared: POSIX named semaphores (`sem_open("/name")`, also in `/dev/shm`), System V semaphore sets (`semget`), or process-shared pthread mutexes living inside the shared memory segment.

### Files

Processes can simply write to and read from a shared file (with `flock()`/`fcntl()` locks to coordinate). Easy and persistent, but slow and without real-time notification (you'd have to poll, or use `inotify` to be notified of changes). ✅ Easy. ❌ Inefficient for frequent updates.

### D-Bus, gRPC, ZeroMQ, etc.

Higher-level abstractions and libraries built **on top of** the primitives above (mostly sockets): **D-Bus** is the desktop/system message bus (systemd, NetworkManager, and desktop apps talk through it, over a Unix socket); **gRPC** does RPC over HTTP/2; **ZeroMQ** offers messaging patterns (pub/sub, push/pull) over TCP, IPC or in-process transports.

## 9.5 Network Sockets

A **socket** is an endpoint for communication, created with `socket()` and represented as an fd (7.4). **TCP/UDP sockets** (`AF_INET`, `AF_INET6`) are used for IPC across **networks**, or on the same machine via `localhost`. Most flexible and scalable.

```
TCP server                                      TCP client
fd = socket(AF_INET, SOCK_STREAM, 0)            fd = socket(AF_INET, SOCK_STREAM, 0)
bind(fd, 0.0.0.0:8080)   (htons! 1.5)
listen(fd, backlog)
conn = accept(fd)    <---- 3-way handshake ---- connect(fd, server:8080)
read(conn) / write(conn) <-- byte stream -->     write(fd) / read(fd)
close(conn)                                     close(fd)
```

- **TCP** (`SOCK_STREAM`): connection-oriented, reliable, ordered **byte stream** (no message boundaries: one `write` of 100 bytes may arrive as two `read`s of 60 + 40).
- **UDP** (`SOCK_DGRAM`): connectionless **datagrams**; boundaries preserved, but they can be lost, duplicated or reordered.

Under the hood: `write()` on a TCP socket copies data into the socket's **send buffer** in the kernel; the TCP/IP stack splits it into segments, adds headers (big-endian, 1.5), and hands them to the NIC driver, which DMAs them out (2.5). Incoming packets arrive by DMA + interrupt + `NET_RX` softirq (3.4), land in the socket's **receive buffer**, and wake the reader's wait queue (7.7) or mark the socket ready in epoll (7.8).

```bash
$ ss -tlnp                  # listening TCP sockets with their processes
State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
LISTEN 0      511          0.0.0.0:80         0.0.0.0:*     users:(("nginx",pid=812,fd=6))
$ ls -l /proc/812/fd/6
lrwx------ 1 root root 64 ... 6 -> socket:[48213]          <- a socket is an inode in sockfs (7.4)
```

Other socket families: `AF_NETLINK` (kernel ↔ user-space communication: `ip`, `udev` uevents (7.12), `ss`), `AF_PACKET` (raw frames: `tcpdump`), `AF_VSOCK` (VM ↔ host).

✅ Powerful, bidirectional, works across machines. ❌ Slightly more complex setup, and every byte goes through the network stack.

### The Path of a Packet (`sk_buff`, NAPI, netfilter)

What happens between the network card and your `read()`:

```
RECEIVE
NIC  --DMA-->  RX ring buffer in RAM (descriptors prepared by the driver, 2.5)
     --IRQ-->  hard IRQ handler: disable further NIC interrupts, schedule NAPI    (3.4)
NET_RX softirq (NAPI poll): process up to a budget of packets per run
     each packet wrapped in an sk_buff (metadata + pointers into the data)
     GRO: merge consecutive TCP segments into one big sk_buff
IP layer:   netfilter hooks PREROUTING -> routing decision -> INPUT  (iptables/nftables)
TCP layer:  checksum, sequence numbers, send ACK, find the socket by its 4-tuple
            (src IP, src port, dst IP, dst port)
socket receive buffer  -> wake the reader's wait queue / mark ready in epoll (7.7, 7.8)
read() copies the data into user memory

SEND
write() -> copy into sk_buffs in the socket send buffer
TCP: segment (or let the NIC do it: TSO), keep a copy until ACKed (retransmission timers, 3.8)
IP:  routing, netfilter OUTPUT -> POSTROUTING (where container NAT/MASQUERADE happens, 10.3)
qdisc (traffic-control queue, shaping) -> driver TX ring -> NIC DMAs it out -> TX-complete IRQ
```

Key ideas:

- **`sk_buff`** is the kernel's packet structure. Moving between layers means adjusting pointers to the headers (Ethernet → IP → TCP), not copying the data.
- **NAPI** switches from "one interrupt per packet" to **polling** under load. At 1 million packets per second, per-packet interrupts would leave no CPU time for anything else (an "interrupt storm"); NAPI handles packets in batches and re-enables interrupts only when the queue is drained.
- **Multi-queue NICs + RSS** (Receive Side Scaling) hash each flow to one of several RX queues, each with its own IRQ and CPU, so network processing scales across cores (and packets of one connection stay in order on one core).
- **netfilter** hooks are where `iptables`/`nftables` rules run: firewalls, and the **NAT** that Docker uses to give containers outbound access and published ports (10.3).
- **XDP** (eXpress Data Path) runs a BPF program in the driver, before an `sk_buff` is even allocated: used for DDoS filtering and load balancers at tens of millions of packets per second.

```bash
$ cat /proc/net/softnet_stat | head -2     # per-CPU: processed, dropped, time_squeeze (budget ran out)
$ ethtool -S eth0 | grep -i drop           # NIC-level drops (ring buffer full)
$ ss -tim                                  # per-socket TCP internals: rtt, cwnd, retransmits
```

> 🔗 **Connections:** Every IPC mechanism is a kernel object behind an fd or a shared mapping: pipes and sockets are files in pseudo-filesystems (pipefs, sockfs, 7.4), shared memory is a tmpfs file mapped with `MAP_SHARED` (6.6), POSIX queues live in mqueuefs. They're inherited across `fork` (4.9), waited on with `epoll` (7.8), and isolated per container by the **IPC** and **network namespaces** (10.1).

---

# 10. Namespaces and cgroups (Containers)

> 🧪 **Interactive visuals:** [Part 10, visualized](visuals/part10-containers.html): an interactive page with a demo for each section.


A container is **not** a virtual machine (2.6). There is no "container" object in the Linux kernel. A container is just **a normal process tree** that the kernel has been told to:

1. **show a restricted view of the system** → **namespaces** ("what can I see?")
2. **limit resource usage** → **cgroups** ("how much can I use?")
3. **restrict privileges** → **capabilities, seccomp**, LSMs ("what am I allowed to do?")

...plus a different root filesystem. Everything in this part is a field of `task_struct` (`nsproxy`, `cgroups`, `cred`, `seccomp`, 4.2).

## 10.1 Namespaces (PID, Mount, Network, UTS, IPC, User, Cgroup, Time)

A **namespace** wraps a global resource so that processes inside it see their **own isolated instance** of it. This is illusion 3 (0.1) taken to its logical end.

| Namespace | Clone flag | Isolates | Inside a container you see... |
| :---- | :---- | :---- | :---- |
| **Mount** | `CLONE_NEWNS` | Mount table (7.13) | Its own `/`, its own mounts |
| **PID** | `CLONE_NEWPID` | Process IDs | Itself as **PID 1**; only its own processes in `ps` |
| **Network** | `CLONE_NEWNET` | Interfaces, IPs, routes, iptables, ports, `/proc/net` | Its own `eth0`, `lo`, can bind port 80 independently |
| **UTS** | `CLONE_NEWUTS` | Hostname, domain name | Its own hostname |
| **IPC** | `CLONE_NEWIPC` | System V IPC, POSIX message queues (9.4) | Its own shm/queues |
| **User** | `CLONE_NEWUSER` | UIDs/GIDs, capabilities | "root" (UID 0) that maps to an unprivileged UID outside |
| **Cgroup** | `CLONE_NEWCGROUP` | View of the cgroup tree | Its own cgroup as the root `/` |
| **Time** | `CLONE_NEWTIME` | `CLOCK_MONOTONIC`/`BOOTTIME` offsets | Its own uptime (useful for checkpoint/restore) |

Three syscalls manage them:

- **`clone(flags)`** with `CLONE_NEW*`: create a child in new namespaces (4.9)
- **`unshare(flags)`**: move the *calling* process into new namespaces
- **`setns(fd)`**: join an existing namespace (via its file in `/proc/<pid>/ns/`). That's what `docker exec` and `nsenter` do.

```bash
$ ls -l /proc/$$/ns
lrwxrwxrwx ... mnt -> 'mnt:[4026531841]'
lrwxrwxrwx ... net -> 'net:[4026531840]'
lrwxrwxrwx ... pid -> 'pid:[4026531836]'           <- same number = same namespace
...
$ sudo unshare --uts --pid --fork --mount-proc bash   # new UTS + PID namespaces
# hostname container-demo        # changes the hostname only inside
# ps aux
USER  PID %CPU %MEM ... COMMAND
root    1  0.0  0.0 ... bash     <- we are PID 1 here!
root    8  0.0  0.0 ... ps aux
```

**PID namespaces are nested.** The same process has a PID in each namespace it's visible in: PID 1 inside the container, maybe PID 31337 on the host. The host sees everything; the container sees only its subtree.

**The container's PID 1 inherits PID 1's special rules** (3.7, 4.5): it adopts orphans in the container, must reap zombies, and the kernel won't deliver signals to it that it has no handler for. So if your app runs as PID 1 and doesn't handle `SIGTERM`, `docker stop` does nothing until the `SIGKILL` 10 s later. That's why tiny inits like `tini` (`docker run --init`) exist. And when the container's PID 1 exits, the kernel kills every other process in that namespace.

**Network namespaces** start with only a `lo` interface. Container runtimes create a **veth pair** (a virtual cable): one end goes into the container as `eth0`, the other stays on the host attached to a bridge (`docker0`), with NAT via iptables/nftables for outgoing traffic.

**User namespaces** let an unprivileged user be "root" inside: UID 0 in the namespace maps to, say, UID 100000 on the host (`/proc/<pid>/uid_map`). Inside, it has capabilities over objects *in its namespaces*; outside, it's a nobody. This is the basis of **rootless containers** (Podman, rootless Docker).

## 10.2 cgroups (v1 vs v2)

**Control groups** organize processes into a hierarchy and apply **resource limits, prioritization and accounting** to each group. Namespaces limit what you *see*; cgroups limit what you *use*.

They're managed through a pseudo-filesystem (7.10), mounted at `/sys/fs/cgroup`: **directories are groups, files are settings**.

```bash
$ cat /proc/self/cgroup                       # which cgroup am I in?
0::/user.slice/user-1000.slice/session-3.scope
$ ls /sys/fs/cgroup/system.slice/docker-<id>.scope/
cgroup.procs  cpu.max  cpu.stat  memory.current  memory.max  memory.events  pids.max  io.max ...
```

Main controllers (v2):

| Controller | Example file | Effect |
| :---- | :---- | :---- |
| **cpu** | `cpu.max` = `50000 100000` | At most 50 ms of CPU per 100 ms period = 0.5 CPU (CFS/EEVDF bandwidth control, 3.6). `cpu.weight` for relative shares |
| **cpuset** | `cpuset.cpus` = `0-3` | Which CPUs / NUMA nodes may be used (3.6, 6.2) |
| **memory** | `memory.max` = `512M` | Hard limit: reclaim within the group, then **OOM-kill inside the group** (6.8). `memory.high` throttles first |
| **io** | `io.max` = `8:0 rbps=10485760` | Disk bandwidth/IOPS limits per device (major:minor, 7.12) |
| **pids** | `pids.max` = `100` | Max number of tasks: stops fork bombs |

Doing it by hand:

```bash
$ sudo mkdir /sys/fs/cgroup/demo
$ echo "+memory +cpu +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
$ echo 100M  | sudo tee /sys/fs/cgroup/demo/memory.max
$ echo "20000 100000" | sudo tee /sys/fs/cgroup/demo/cpu.max     # 20% of one CPU
$ echo $$ | sudo tee /sys/fs/cgroup/demo/cgroup.procs             # move this shell in
$ python3 -c 'x = bytearray(500*1024*1024)'                        # exceeds 100M
Killed                                                              # exit status 137
$ cat /sys/fs/cgroup/demo/memory.events
oom_kill 1
```

**v1 vs v2:**

| | cgroup v1 | cgroup v2 (unified, default on modern distros) |
| :---- | :---- | :---- |
| Hierarchies | **One per controller** (`/sys/fs/cgroup/memory/...`, `/sys/fs/cgroup/cpu/...`); a process could be in different places in each | **One single tree**; each process is in exactly one cgroup |
| Processes | Can be in any node | Only in **leaf** nodes ("no internal processes" rule) |
| Memory + I/O | Accounted separately → writeback I/O couldn't be charged correctly | Coordinated: dirty page writeback (6.7) is charged to the right cgroup |
| Extras | | PSI per cgroup (3.6), `memory.high` throttling, delegation to unprivileged users |

systemd puts every service in its own cgroup (`systemctl status nginx` shows the CGroup; `systemd-cgtop` shows usage). That's how `systemctl stop` reliably kills **all** of a service's processes, even double-forked ones: it kills the whole cgroup.

## 10.3 How Containers Are Built from Namespaces + cgroups

What `docker run -it --memory=512m alpine sh` boils down to:

```
docker CLI --(HTTP over /run/docker.sock, 9.2)--> dockerd --> containerd --> containerd-shim --> runc

runc:
 1. image layers -> OVERLAYFS root filesystem (read-only layers + one writable layer)
 2. clone(CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWUTS | CLONE_NEWIPC
          [| CLONE_NEWUSER | CLONE_NEWCGROUP])                         (10.1, 4.9)
 3. create a cgroup, write memory.max = 512M, cpu.max, pids.max ..., add the child   (10.2)
 4. network: veth pair, one end into the new net namespace as eth0; bridge + NAT on the host
 5. in the child (inside the new mount namespace):
      mount /proc, /sys, /dev (tmpfs + a few device nodes), /dev/pts, /dev/shm
      bind-mount volumes and /etc/resolv.conf, /etc/hostname, /etc/hosts       (7.13)
      pivot_root() into the overlay rootfs, unmount the old root   (task_struct->fs->root, 4.2)
      sethostname()
 6. drop capabilities to a small default set                                   (10.4)
 7. install the seccomp filter                                                 (10.4)
 8. set the user, then execve("/bin/sh")  -> becomes PID 1 in the container    (4.9)
```

**OverlayFS** stacks directories: the image's read-only **lower** layers and a container-specific writable **upper** layer are merged into one view. Reading a file finds it in the topmost layer that has it; **modifying** a lower-layer file first **copies it up** into the upper layer (copy-on-write at file level, same idea as 6.5). Deleting creates a "whiteout" marker. 100 containers from the same image share the lower layers on disk **and in the page cache** (6.7).

Why **`pivot_root`** and not `chroot`? `chroot` only changes the process's root for path resolution, and a root process can escape it (e.g. with a second `chroot` + `..`). `pivot_root` inside a new mount namespace actually swaps the root **mount** and lets the old root be unmounted completely, so there's nothing to escape to.

And remember: it's still the **host kernel**. `uname -r` inside a container prints the host's kernel version. A kernel exploit inside a container is a host exploit, which is why capabilities and seccomp matter so much.

```bash
$ docker run -d --name web nginx
$ docker inspect web | jq '.[0].State.Pid'
31337                                   # just a normal host process
$ sudo nsenter -t 31337 -n ip addr      # join only its network namespace and look around
$ cat /proc/31337/cgroup
0::/system.slice/docker-8f2c....scope
```

## 10.4 Capabilities and seccomp

### Capabilities

Traditionally, Unix had two kinds of process: **root** (UID 0, bypasses all checks) and **everyone else**. **Capabilities** split root's power into ~40 separate privileges, stored in `task_struct->cred` (4.2):

| Capability | Allows |
| :---- | :---- |
| `CAP_NET_BIND_SERVICE` | Bind to ports < 1024 |
| `CAP_NET_ADMIN` | Configure interfaces, routes, iptables |
| `CAP_NET_RAW` | Raw sockets (`ping`, `tcpdump`) |
| `CAP_SYS_ADMIN` | "The new root": mount, many namespace ops, lots more. Avoid granting it |
| `CAP_SYS_PTRACE` | Trace/debug other users' processes |
| `CAP_SYS_MODULE` | Load kernel modules (3.1) |
| `CAP_SYS_NICE` | Raise priority, set real-time policies (4.6, 3.6) |
| `CAP_KILL` | Signal any process (8.1) |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permission checks (7.11) |
| `CAP_CHOWN`, `CAP_SETUID`, `CAP_SETGID` | Change file owners, switch UIDs/GIDs |

Each thread has several sets (permitted, effective, inheritable, bounding, ambient). Files can carry capabilities too (in an xattr on the inode, 7.3), a safer alternative to setuid root (7.11):

```bash
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep                       # ping can open raw sockets, nothing else
$ sudo setcap cap_net_bind_service=+ep ./myserver  # bind :80 without running as root
$ grep Cap /proc/$$/status
CapEff: 0000000000000000                           # a normal shell: no capabilities
$ capsh --decode=00000000a80425fb                  # docker's default set, decoded
```

Docker's default keeps ~14 harmless capabilities (`CHOWN`, `NET_BIND_SERVICE`, `KILL`, `SETUID`...) and drops the dangerous ones (`SYS_ADMIN`, `SYS_MODULE`, `NET_ADMIN`, `SYS_PTRACE`...). `--privileged` gives all of them back (plus all devices): it essentially disables the container boundary.

### seccomp

**seccomp** (secure computing) filters **system calls** (3.3). A process installs a small **BPF program** that the kernel runs on **every syscall entry**; it inspects the syscall number and arguments and returns *allow*, *return an error* (`EPERM`), *kill the process*, or *notify a supervisor*.

```
user code: syscall(nr=165 mount, ...)
   |
   entry_SYSCALL_64 -> seccomp filter (BPF): nr == mount? -> return EPERM
   |                                         nr == read?  -> allow -> sys_call_table[0]
```

- Filters are **inherited** across `fork` and `exec` and can never be removed, only made stricter.
- Docker's default profile blocks ~40–60 of the ~350+ syscalls: `mount`, `reboot`, `kexec_load`, `init_module`, `ptrace` (depending on version), `keyctl`, `bpf`, and others with a history of kernel exploits. Blocking a syscall removes its kernel code as attack surface.
- Browsers (Chrome renderer sandboxes), systemd services (`SystemCallFilter=` in unit files), Android apps, and OpenSSH use seccomp too.

```bash
$ grep Seccomp /proc/$$/status
Seccomp:        0                      # 0 = off, 1 = strict, 2 = filter
$ docker run --rm alpine grep Seccomp /proc/self/status
Seccomp:        2
```

Together with **LSMs** (Linux Security Modules: **AppArmor**, **SELinux**), which add mandatory access control rules on files, sockets and capabilities, these layers form **defense in depth**: namespaces hide, cgroups limit, capabilities shrink root, seccomp shrinks the syscall surface, and LSMs constrain whatever is left.

> 🔗 **Connections:** A container is the whole book in one command: `clone()` with namespace flags (4.9), a private mount table with bind mounts and `pivot_root` (7.13), an overlay filesystem using copy-up, cgroup limits enforced by the scheduler (3.6) and the OOM killer (6.8), a veth pair and Unix sockets (9), PID 1's signal rules (8, 3.7), capabilities checked against `task_struct->cred`, and a seccomp filter on the syscall entry path (3.3), all on the **same host kernel**.

---

# 11. Putting It All Together: One Command, Every Layer

> 🧪 **Interactive visuals:** [Part 11, visualized](visuals/part11-all-together.html): an interactive page with a demo for each section.


Let's trace one ordinary command from keystroke to disk and back, and name the section behind each step:

```bash
$ cat notes.txt | grep inode > out.txt
```

```
 TIME ──────────────────────────────────────────────────────────────────────────────────────────►

 keyboard ─► PTY ─► bash: parse ─► pipe() ─► fork() ──► child 1: dup2, execve(cat) ─► open, read ─► write(pipe) ─► exit
                                          └─ fork() ──► child 2: open(out.txt), dup2, execve(grep) ─► read(pipe) ─► write(file) ─► exit
                    bash: close pipe fds, set foreground job, waitpid() ...................................... SIGCHLD ─► reap ─► prompt
                                                                                    later: flusher ─► out.txt pages to disk
```

### 1. Keystrokes Reach bash

- Each key you press reaches the PTY master (from the terminal emulator, or from `sshd` reading a network socket, 9.5). The TTY line discipline in the kernel buffers it (4.9 step 0).
- bash was **sleeping** in `read(0, ...)` in state `S` on the TTY's wait queue (4.8, 7.7). The new input **wakes** it: it becomes runnable, and the scheduler (3.6) soon gives it a CPU through a context switch (3.5).
- When you press Enter, `readline` returns the line.

### 2. bash Parses the Line

- Tokens: `cat`, `notes.txt`, `|`, `grep`, `inode`, `>`, `out.txt`. A **pipeline** of two commands, with an output **redirection** on the second (7.5).
- `cat` and `grep` aren't aliases, functions or builtins, so bash searches `$PATH` and finds `/usr/bin/cat` and `/usr/bin/grep` (with `stat()`, a syscall that walks the dentry cache, 7.3).

### 3. bash Creates the Pipe and Two Children

- `pipe([3, 4])`: the kernel creates a pipe inode in pipefs with a 64 KiB buffer and two `struct file`s; bash gets fds 3 (read end) and 4 (write end) (9.1, 7.1, 7.2).
- `fork()` twice (really `clone()`, 4.9): two new `task_struct`s, new PIDs, **copied fd tables** (so both children also hold fds 3 and 4) and **copy-on-write** memory (no data copied, 4.9 step 3).
- Both children are put into a **new process group** (the job), and bash makes it the terminal's **foreground** group, so `Ctrl+C` will hit both of them and not bash (4.5).

### 4. Child 1 Becomes `cat`

- `dup2(4, 1)`: stdout now points to the pipe's write end. `close(3)`, `close(4)` (7.5).
- `execve("/usr/bin/cat", ["cat", "notes.txt"], envp)` (4.9 steps 4–6):
  - path walk + execute permission check against the child's credentials (7.11)
  - `binfmt_elf` recognizes `\x7fELF`, tears down the COW copy of bash's memory, builds new VMAs for cat's segments and stack (6.4), maps `ld-linux.so`
  - `ld.so` `mmap`s `libc.so.6`; its code pages are **already in the page cache** because every process uses them, so they're shared with hundreds of other processes (6.6, 6.7)
- The first instructions **page-fault** (minor faults: the pages are cached) and the kernel fills in page-table entries; the TLB caches the translations (6.5, 2.4).

### 5. Child 2 Becomes `grep`, Writing to `out.txt`

- `dup2(3, 0)`: stdin is the pipe's read end. `close(3)`, `close(4)`.
- `open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0666 & ~umask)`: the path walk needs **write + execute on the current directory** to create the entry (7.11); ext4 allocates a new **inode** from the inode bitmap and adds a **directory entry** "out.txt → inode #N" (7.3). Then `dup2(fd, 1)`.
- `execve("/usr/bin/grep", ["grep", "inode"], envp)`.

### 6. bash Closes Its Copies and Waits

- bash `close(3)` and `close(4)`. **This matters:** if bash kept fd 4 open, the pipe would always have a writer, and grep would never see EOF (9.1).
- bash calls `waitpid()` and **sleeps** (state `S`).

### 7. `cat` Reads the File

- `openat(AT_FDCWD, "notes.txt", O_RDONLY)` → path walk through the dentry cache, permission check (`r` for your UID), a new `struct file` with `f_pos = 0`, fd 3 (7.1–7.3).
- `read(3, buf, 131072)` is a **system call**: `syscall` instruction → Ring 0 (2.6, 3.3) → VFS → ext4's `read_iter` → **page cache** lookup by (inode, offset) (6.7):
  - **Hit:** copy into cat's buffer. Microseconds.
  - **Miss:** allocate a page, the block layer sends a request to the NVMe driver, the SSD **DMAs** the data into RAM (2.5), cat sleeps in state **`D`** (4.8). Other tasks run meanwhile. The SSD raises an **interrupt** (3.4); the handler completes the I/O and **wakes** cat. Readahead also fetches the next pages, betting on sequential access (1.8).
- The first write into cat's `buf` was itself a **page fault** on fresh anonymous memory (demand paging, 6.5).

### 8. Data Flows Through the Pipe

- cat's `write(1, buf, n)` copies into the pipe buffer. If grep is slow and the buffer fills, cat **blocks**, which is automatic backpressure (9.1).
- grep was sleeping in `read(0, ...)` on the pipe's wait queue. The write **wakes** it; the scheduler picks it up, maybe on another core (3.6). On a multi-core machine, cat and grep genuinely run in parallel.
- grep searches each line. Its stdout is a **file**, so stdio is **fully buffered** (7.6): matches accumulate in grep's own memory and are flushed with `write(1, ...)` in 4 KiB+ chunks.
- grep's `write()` copies the data into **out.txt's page cache pages** and marks them **dirty**; it returns immediately (6.7).
- All along, the **timer interrupt** fires every few ms on each core. If another task is waiting, the scheduler may preempt cat or grep: an involuntary context switch (3.5).

### 9. EOF, Exit and Reaping

- cat's `read()` returns **0** (end of file) → cat calls `exit(0)` → the kernel closes its fds. That was the **last write end** of the pipe.
- grep's next `read(0)` returns **0** (EOF on the pipe) → grep flushes its stdio buffer → `exit(0)` if it found a match (1 if not).
- Each child becomes a **zombie** and the kernel sends **`SIGCHLD`** to bash (4.9, 8.2). bash wakes from `waitpid()`, **reaps** both (their `task_struct`s are freed), and sets `$?` to grep's status (`${PIPESTATUS[@]}` holds both).
- bash takes the terminal's foreground back and prints the prompt, then sleeps in `read(0)` again.

### 10. Later: Durability

- ~5–30 seconds later, a **flusher** kernel thread writes out.txt's dirty pages to the SSD (DMA again), and ext4 commits the metadata (new inode, directory entry, mtime) through its **journal** (6.7).
- If the power fails before that, `out.txt` may be empty or missing, even though `grep` "succeeded". Only `fsync()` (or `sync`) would have guaranteed it.

### What If You Pressed `Ctrl+C` Halfway?

The **TTY driver** (not bash) sees `^C` and sends **`SIGINT`** to every process in the **foreground process group**: cat and grep, but not bash (4.5). On their next return to user mode, the signal is delivered (8.1); the default action terminates both. bash reaps them and reports status **130** (128 + 2, 4.9). Any matches still sitting in grep's stdio buffer are lost, because they only existed in grep's memory (7.6).

### The Same Story as a Table

| Step | What happened | Parts |
| :---- | :---- | :---- |
| Key press → bash wakes | TTY line discipline, wait queues, scheduler | 4.9, 7.7, 3.6 |
| Parse, `$PATH` lookup | shell logic, `stat()`, dentry cache | 4.9, 7.3 |
| `pipe`, `fork`, `dup2` | fd tables, open file descriptions, COW, process groups | 7.1, 7.2, 9.1, 4.5 |
| `execve` | binfmt, VMAs, dynamic linking, demand paging | 4.9, 6.4–6.6 |
| `open` / `read` | syscalls, rings, VFS, permissions, page cache, DMA, interrupts, `D` state | 2.6, 3.3, 7.4, 7.11, 6.7, 2.5, 3.4 |
| Pipe flow | blocking I/O, backpressure, context switches, multi-core | 9.1, 7.7, 3.5 |
| Output | stdio buffering, dirty pages, writeback | 7.6, 6.7 |
| Exit | zombies, `SIGCHLD`, `wait`, exit codes | 4.9, 8 |
| Under it all | bits, caches, TLB, privilege levels | 1, 2 |

If you can tell this story end to end, and explain *why* each step works the way it does, you understand system internals.

---

# 12. Questions and Answers

> 🧪 **Interactive visuals:** [Flashcards for every question below](visuals/part12-quiz.html): flip cards, track what you know, shuffle.


Each answer points to the sections behind it. Try answering before reading.

## Files, Links and Filesystems

### Let's assume we have two different filesystems. Can a symlink point to a file on the other filesystem?

**Yes.** A symlink is a tiny file whose content is just a **path string** (7.3). The kernel doesn't resolve it when you create it; it resolves it **every time you access it**, using the normal path walk, which crosses mount points freely (7.13).

```bash
# / is ext4 on nvme0n1p2, /mnt/usb is vfat on sdc1
$ ln -s /mnt/usb/photos ~/photos        # symlink on ext4 -> directory on vfat: fine
$ ls ~/photos                            # works
$ sudo umount /mnt/usb
$ ls ~/photos                            # now dangling: the path leads to an empty mount point
ls: cannot access '/home/berat/photos': No such file or directory
```

Things to keep in mind:

- The link can **dangle** if the other filesystem isn't mounted, or is mounted somewhere else.
- **Absolute** targets (`/mnt/usb/x`) depend on where things are mounted. **Relative** targets (`../x`) depend on where the link itself lives.
- Inside a **container or chroot**, an absolute target is resolved against the **container's** root, so a symlink to `/data/x` usually points at nothing there. Use a bind mount instead (7.13, 10.3).

### Can a hard link point to another filesystem?

**No.** `ln /mnt/usb/a.txt ~/a.txt` fails with `Invalid cross-device link` (`EXDEV`). A hard link is a directory entry that stores only an **inode number**, and inode numbers are only unique **within one filesystem**: inode 1024 on `/home` and inode 1024 on `/data` are unrelated files (7.3). A directory entry on filesystem A simply can't name an inode on filesystem B.

### Can two files have the same inode number?

Yes, if they're on **different filesystems**. The true identity of a file is **(device, inode)**, which is what `stat` reports and what tools like `find -samefile`, `rsync -H`, and `tar` use to detect hard links. Within one filesystem, the same inode number means the same file (hard links). The root of almost every ext4 filesystem is inode 2, which is why `/` and `/home` can both be inode 2 if they're separate filesystems (7.13).

### Why can't I create a hard link to a directory?

It would allow **cycles** (`a/b` linking back to `a`) and give a directory **several parents**, which would make `..` ambiguous and send `find`, `du`, `rm -r` and backup tools into infinite loops. The kernel refuses with `EPERM`. The only directory hard links are the automatic `.` and `..`, which is why on ext4 a directory's link count is `2 + number of subdirectories` (7.3). Symlinks to directories are allowed because tools can recognize a symlink and choose not to follow it.

### What does `mv` do to the inode? Same filesystem vs different filesystem?

- **Same filesystem:** `mv` calls **`rename()`**. Only directory entries change: the data isn't touched, the **inode number stays the same**, it's instant even for a 100 GB file, it's **atomic** (no moment where the file is missing), and open fds keep working.
- **Different filesystems:** `rename()` fails with `EXDEV` (7.13), so `mv` **copies** the data into a **new inode** on the target filesystem and then **unlinks** the original. It's slow, not atomic, and a process with the old file open still writes to the old (now deleted) inode.

```bash
$ ls -i big.iso; mv big.iso sub/; ls -i sub/big.iso      # same inode number
$ mv big.iso /mnt/usb/; ls -i /mnt/usb/big.iso           # different inode, took minutes
```

This is also why "write to a temp file, then `rename()` over the original" is the standard way to update a file atomically, but only if the temp file is on the **same filesystem**.

### What happens when you remove a file that a process still has open?

`rm` only calls `unlink()`, which removes the **name** and decrements the inode's link count (7.3). The inode and data blocks are freed only when **both** the link count is 0 **and** no `struct file` references it (7.2). So:

- The process keeps reading and writing the file normally, through its fd.
- No new process can open it (no name).
- The space is **not** freed until the process closes it or exits.
- You can still get the content back through `/proc/<pid>/fd/<n>`.

```bash
$ sudo lsof +L1                        # deleted-but-open files
$ cp /proc/2211/fd/7 /tmp/rescued.log  # recover it
$ : > /proc/2211/fd/7                  # or truncate it in place to free the space without restarting
```

### `df -h` says the disk is full, but `du -sh /` shows much less. What are all the possible causes, and how would you investigate?

The two tools measure different things:

- **`df`** asks the filesystem itself (`statfs()` on each mount, reading counters from the superblock, 7.3): "how many blocks are allocated?" It counts **everything**, including data of deleted-but-open files.
- **`du`** walks the **directory tree** and sums the sizes of the files it can **reach by name**.

So anything allocated but not reachable by a name shows up in `df` but not in `du`:

1. **Deleted files still held open** (the most common cause, previous question): a log rotated or deleted while the app still writes to it.
   ```bash
   $ sudo lsof +L1      # or: sudo lsof | grep '(deleted)'
   # fix: restart/reload the process (logrotate's postrotate usually sends SIGHUP, 8.2),
   #      or truncate through /proc/<pid>/fd/<n> if you can't restart (e.g. a prod database)
   ```
2. **Files hidden under a mount point**: data written to `/mnt/data` *before* a filesystem was mounted on top of it is still on the root filesystem but invisible to `du` (7.13). Check with a bind mount of `/` somewhere else: `mount --bind / /tmp/root && du -sh /tmp/root/mnt/data`.
3. **Reserved blocks**: ext4 reserves 5% for root by default, so `df` reports 100% for normal users before the disk is truly full (`tune2fs -l` → "Reserved block count").
4. **du can't read some directories** (permission denied), so it under-counts. Run it as root.
5. **Filesystem-level features**: snapshots (Btrfs/ZFS/LVM), metadata, and journal space are allocated but not files.
6. And the inverse case, **`df` shows free space but writes fail with "No space left on device"**: you're out of **inodes** (`df -i`), not blocks (7.3).

### What does `rm` do to a symlink? And `chmod`?

`rm link` unlinks **the symlink itself**; the target is untouched. `chmod` on a symlink changes the **target's** permissions, because symlink permissions are meaningless on Linux (always `lrwxrwxrwx`). `chown -h` / `lchown()` change the link's own ownership. Most syscalls follow symlinks by default; the "l" variants (`lstat`, `lchown`) and flags like `O_NOFOLLOW` / `AT_SYMLINK_NOFOLLOW` don't.

### What happens with a symlink loop (`a → b`, `b → a`)?

Path resolution follows symlinks for you, but gives up after **40** symlinks in one lookup and fails with **`ELOOP`** ("Too many levels of symbolic links"). The kernel can't detect a cycle cheaply, so it just bounds the work.

### How does `mount --bind` work? How is it different from a symlink?

- **`mount --bind`** is a **kernel mount-table entry**: the same directory (same dentries and inodes) appears at a second place in the tree. It looks and behaves exactly like the original, works inside chroots and containers, and can even be made read-only on its own. Used for container volumes, isolation, and remounts.
- **A symlink** is a **file containing a path**, resolved at each access. Lightweight, visible (`l` type), creatable by anyone, but it can break or misbehave in chroots and containers, where absolute paths are resolved against a different root.

Full comparison table in 7.13.

### Why do files in `/proc` show a size of 0?

Their content doesn't exist until you read it: `read()` on `/proc/<pid>/status` calls a function that **generates the text on the fly** from kernel structures like `task_struct` (7.10, 4.2). There's no stored data, so there's no size to report. (Tools that trust `st_size`, like some old copy utilities, misbehave on them.)

### Why can `ls -l` show a file as 10 GB while `du` says 4 KB?

It's a **sparse file**. `ls -l` shows the inode's **size** (the highest offset written); `du` shows **allocated blocks**. If a program `lseek`s far ahead and writes, the gap in between ("holes") has no blocks allocated and reads back as zeros. VM disk images, database files and core dumps are often sparse.

```bash
$ truncate -s 10G sparse.img
$ ls -lh sparse.img; du -h sparse.img
-rw-r--r-- 1 berat berat 10G ... sparse.img
0       sparse.img
```

### Can you explain what a device file in `/dev` is, and how it differs from a regular file?

A device file has **no data blocks**: its inode stores only a type (`b`/`c`) and a **major:minor** number. When you open it, the VFS routes every `read`/`write`/`ioctl` to the **driver** registered for that major number, instead of a filesystem (7.12, 7.4). Writing to `/dev/sda` writes raw disk sectors; reading `/dev/urandom` calls the kernel's random-number generator; writing to `/dev/null` calls a function that discards everything. Same syscalls, completely different code behind them. That's "everything is a file".

### `/lib` contains shared libraries like `libc`. How do these relate to the executables in `/bin`? How does a program use `libc` at runtime?

Binaries in `/bin` (really `/usr/bin`) are **dynamically linked**: they don't contain `printf` or `open`, only references to them. At `execve`, the kernel sees the ELF's `PT_INTERP` entry and starts the **dynamic linker** `/lib64/ld-linux-x86-64.so.2` first. It `mmap`s the needed libraries (`/lib/x86_64-linux-gnu/libc.so.6`, ...), resolves symbol addresses (PLT/GOT), then jumps to the program's entry point (4.9, dynamic linking). The library's code pages come from the page cache and are **shared** by every process using them (6.6, 6.7). `ldd /bin/ls` lists the libraries; a missing one makes the program fail before `main` even runs.

### Why does `sudo echo hello > /root/file` fail with "Permission denied"?

Because the **redirection is done by your shell, not by sudo** (7.5). Your (non-root) shell tries to `open("/root/file")` *before* running `sudo echo hello`, and that open fails. Only `echo` would have run as root. Fix: make a root process do the open:

```bash
$ echo hello | sudo tee /root/file > /dev/null
$ sudo sh -c 'echo hello > /root/file'
```

## Shell and Processes

### What happens when you run `echo *` in an empty directory?

- If the directory is **not empty**, the **shell** expands `*` into the list of filenames (sorted) before running anything, so `echo` receives `file1 file2 file3` as arguments. `echo` itself never sees the `*`.
- If the directory **is empty**, the glob matches nothing, and by default bash passes the **literal `*`** to `echo`, which prints:

```bash
$ mkdir empty && cd empty && echo *
*
```

(With `shopt -s nullglob`, an unmatched glob expands to nothing and `echo` prints an empty line; with `failglob`, bash reports an error. Hidden files starting with `.` aren't matched by `*` unless `dotglob` is set.) This is a good reminder that **globbing is a shell feature**, done in user space before any `fork`/`exec` (4.9).

### How does the shell parse and execute `ls *.txt | grep foo`? What syscalls are involved?

**1. Parsing and globbing (lexical analysis/tokenization).** Before anything is executed, the shell:

- expands `*.txt` to the list of matching files (e.g. `file1.txt file2.txt`), using `openat` + `getdents64` on the current directory
- splits the line at `|` into two commands: `ls file1.txt file2.txt` (left) and `grep foo` (right)

**2. Set up a pipe.** `pipe()` returns **two file descriptors**: `[read_fd, write_fd]` (9.1).

**3. Fork two children.** The shell calls `fork()` (`clone()`) **twice**: one child for `ls`, one for `grep`. Now there are three processes: the parent shell, child 1 (`ls`), child 2 (`grep`).

**4. Redirection setup (in each child, before exec):**

- In `ls`: `dup2(write_fd, STDOUT_FILENO)` sends its output into the pipe, then it closes both `read_fd` and `write_fd`.
- In `grep`: `dup2(read_fd, STDIN_FILENO)` reads its input from the pipe, then it closes both originals.

**5. Replace the process image.** Both children call `execve()`: `/usr/bin/ls file1.txt file2.txt` and `/usr/bin/grep foo` (4.9). This is where the shell lets go: the commands take over.

**6. The parent** closes **its own** copies of both pipe ends (otherwise `grep` would never get EOF) and calls `wait4()`/`waitpid()` for both children, woken by `SIGCHLD`.

| Syscall | Purpose |
| :---- | :---- |
| `openat()`, `getdents64()` | Shell reads the directory to expand `*.txt` |
| `pipe2()` | Creates the pipe (two fds) |
| `clone()` | Duplicates the shell process (`fork`) |
| `setpgid()`, `ioctl(TIOCSPGRP)` | Put the job in its own process group and give it the terminal (4.5) |
| `dup2()` | Redirects stdin/stdout to pipe ends |
| `close()` | Closes unused pipe fds |
| `execve()` | Executes the `ls` and `grep` binaries |
| `mmap()`, `openat()` | Dynamic linker loads `libc`; `ls` may `stat`/open files |
| `read()` / `write()` | `ls` writes into the pipe, `grep` reads from it and writes to the terminal |
| `exit_group()` | Each child exits |
| `wait4()` | Parent reaps both children |

Watch it all: `strace -f -e trace=process,pipe2,dup2,close,execve bash -c 'ls *.txt | grep foo'`.

### What's the difference between `exec()` and `fork()`? Why does a shell use both to launch a program?

- **`fork()`** creates a **new process** by duplicating the caller: same code, same (COW) memory, same open files. Two processes now run the same program.
- **`exec()`** **replaces the program** running in the current process: same PID, new code and memory. No new process.

The shell itself is a process running its own code, and it needs to **stay alive** after your command finishes. If it called `exec()` directly, bash would *become* `ls` and vanish when `ls` exits. So it `fork()`s a child, the **child** `exec()`s the program, and the parent shell `wait()`s. The gap between `fork` and `exec` is also exactly where the child sets up redirections, pipes, process groups and environment (7.5): the separation of the two calls is what makes Unix shells so flexible (4.9).

### Why is `cd` a shell builtin, and not a program like `ls`?

The current directory is per-process state (`task_struct->fs->pwd`, 4.2). An external `cd` would run in a **child process** after `fork()`, change the **child's** directory, and exit, leaving the shell exactly where it was. Builtins exist for anything that must modify **the shell's own process**: `cd`, `export`, `ulimit`, `umask`, `exit`, `source`. For the same reason, running a script with `./script.sh` can't change your shell's directory or variables, but `source script.sh` (run it in the current shell) can.

### What is PID 0?

PID 0 is the **idle task** (historically called `swapper` or `sched`). It's not a normal process: it's created statically by the kernel at boot (it *is* the boot thread), lives only in kernel space, has one instance per CPU, and runs whenever a CPU has nothing else to do, putting the core into a low-power C-state (2.1). It created PID 1 (`init`/`systemd`, the first user-space process, parent of all user processes) and PID 2 (`kthreadd`, parent of all kernel threads). You can't see it in `ps`, but you see it as the PPID of PIDs 1 and 2. Full comparison table in 4.5.

### Why does Linux start multiple virtual TTYs (`/dev/tty1`–`tty6`) by default? Isn't one enough?

Several independent login sessions on one physical screen and keyboard: one user runs a program on tty1, another (or the same user) logs in on tty2, and an admin can switch to tty3 (`Ctrl+Alt+F3`) to kill a stuck process even if the GUI or the first session is frozen. The GUI usually takes tty1 or tty7 depending on the distro. Each tty is a separate character device with its own `getty` (3.7).

### Do all processes have a `task_struct`, or is it like one main class that processes are created from?

**Every process, and every thread, has its own `task_struct` instance.** It's not a shared class; it's the kernel's per-task record, allocated (from a slab cache, 6.2) by `copy_process()` when the task is created (4.9) and freed when it's reaped. The kernel links all of them in a task list and a PID hash table; each CPU's run queue links the runnable ones; and the `current` macro gives the one running on this CPU (4.2).

### Explain zombie and orphan processes. How are they created, and how do you clean up zombies?

A **zombie** is a process that has **exited**, but whose parent hasn't yet collected its exit status with `wait()`. Its memory and fds are already freed; only the `task_struct` (PID, exit status, resource usage) remains (4.9). An **orphan** is a process that is **still running** but whose parent has exited; it's adopted by the nearest subreaper or PID 1, which will reap it when it exits (4.5).

```bash
$ ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/'      # find zombies (STAT starts with Z)
```

| Feature | Zombie | Orphan |
| :---- | :---- | :---- |
| Alive? | ❌ Dead (exited) | ✅ Alive (still running) |
| Parent alive? | ✅ Usually yes | ❌ Parent is dead |
| Reaping needed? | ✅ Yes, the parent must `wait()` | ✅ Will be reaped by its new parent (PID 1 / subreaper) |
| Causes problems? | A few are harmless; thousands can exhaust **PIDs** (`pid_max`) | ❌ No, if PID 1 behaves properly |
| Removed by | Parent's `wait()`, or the new parent after the old parent dies | Its new parent when it exits |

**You can't kill a zombie**: it's already dead, so signals do nothing. To clean it up: get the parent to call `wait()` (send it `SIGCHLD`, fix the bug), or **kill the parent**: the zombie is then reparented to PID 1, which reaps it immediately. The real fix is in the parent's code: handle `SIGCHLD` with a `waitpid(..., WNOHANG)` loop, or set `SIGCHLD` to `SIG_IGN` for auto-reaping (4.9).

### Can `kill -9` kill a process stuck in `D` state?

**Not immediately.** `SIGKILL` is marked pending, but signals are only acted on when the task returns toward user mode (8.1), and a task in uninterruptible sleep (`D`) isn't woken by signals (4.8). It dies as soon as the I/O it's waiting for completes. If that I/O never completes (a dead NFS server, a failing disk), the process is unkillable until the device recovers or you reboot. Newer kernel code uses `TASK_KILLABLE` sleeps that **do** wake for fatal signals, which has made this rarer.

### After `fork()`, the parent and the child both write to the same fd. Do they overwrite each other?

They **share the file offset**, because `fork()` copies the fd *table*, but both entries point to the **same `struct file`** (open file description), which holds `f_pos` (7.2). So their writes are appended one after another rather than overwriting each other (interleaving is still possible). If instead each process had called `open()` separately, they'd have independent offsets and would overwrite each other's data, unless they used `O_APPEND`.

### If a multithreaded process calls `fork()`, how many threads does the child have?

**One**: a copy of the thread that called `fork()`. The other threads simply don't exist in the child. But their **memory** was copied, including any **mutexes they were holding at that instant**, which are now locked forever with no owner to unlock them. If the child then calls `malloc` (whose internal lock another thread held), it can deadlock. Hence the rule: in a multithreaded program, the child of `fork()` should only call async-signal-safe functions until it calls `execve()` (or use `posix_spawn`, 4.9).

### A script does `export FOO=bar`. Why doesn't my shell see `FOO` afterwards?

The environment is copied **from parent to child** at `fork`/`execve` (it's passed as `envp` and lives in the new process's stack memory, 4.9). The script ran in a **child** process; changing its own copy can't affect the parent's memory (separate address spaces, 6.3). Information only flows parent → child. Use `source script.sh` to run it inside your current shell.

### What does exit code 137 mean? And 139, 143, 127?

The shell reports **128 + N** when a process is killed by signal N (4.9):

- **137** = 128 + 9 = **`SIGKILL`**: very often the **OOM killer** (6.8) or a container hitting its cgroup `memory.max` (10.2). Check `dmesg` or `memory.events`.
- **139** = 128 + 11 = **`SIGSEGV`**: a segmentation fault (6.5).
- **143** = 128 + 15 = **`SIGTERM`**: graceful stop, e.g. `docker stop` (8.2).
- **127** = command **not found**; **126** = found but **not executable**.

### How does a background job survive when I log out?

When the terminal hangs up, the kernel sends **`SIGHUP`** to the session leader (your shell), and the shell forwards it to its jobs, whose default action is to terminate (4.5, 8.2). Ways to survive:

- `nohup cmd &`: sets `SIGHUP` to **ignored** before `exec` (ignored dispositions survive `exec`, 8.3), and redirects output to `nohup.out`.
- `setsid cmd`: starts it in a **new session** with no controlling terminal, so it never gets the hangup.
- `disown` (bash): removes the job from the shell's job table so bash doesn't forward `SIGHUP`.
- `tmux`/`screen`, or a systemd service: the process doesn't depend on your terminal at all.

## Signals

### How does a process handle `SIGTERM`? How can you prevent a process from being killed with `Ctrl+C`?

`SIGTERM` is the **polite** way to ask a process to terminate. The kernel marks it pending (sent by `kill()` or similar); on the target's next return to user mode it's delivered (8.1):

- If the process **installed a handler** (`sigaction(SIGTERM, ...)`), the handler runs: typically set a flag, then the main loop finishes current work, flushes buffers and exits cleanly.
- If **no handler** is installed, the default action applies: **terminate**.

`Ctrl+C` sends **`SIGINT`** to the foreground process group (4.5). To prevent it from killing a process, either **ignore** it with the special disposition `SIG_IGN` (it's a disposition, not a signal):

```c
signal(SIGINT, SIG_IGN);           // Ctrl+C now does nothing
```

or **catch** it with a handler that does something else (like asking "really quit?"), or **block** it temporarily with `sigprocmask()` (it stays pending, 8.4). From the shell: `trap '' INT`. What you can **never** ignore, catch or block are **`SIGKILL`** and **`SIGSTOP`** (8.3).

### What happens when you run `kill -9` on a process? Can it be intercepted or handled?

Signal 9 is **`SIGKILL`**: immediately and unconditionally terminate the target. It **cannot be caught, blocked, or ignored**: the kernel never consults the process's handlers or mask for it. It **bypasses user space entirely**: no handler runs, no stdio buffers are flushed (7.6), no temporary files or lock files are cleaned up, no child processes are told. The kernel still releases the process's memory, fds and other kernel resources in `do_exit()`, and the parent still gets `SIGCHLD` and must reap it (exit status 137). The only things that delay it: uninterruptible sleep (`D`, see above), and PID 1, which ignores `SIGKILL` sent from inside its own PID namespace. That's why `kill -9` should be the **last resort** after `SIGTERM`.

### Why does `docker stop` take 10 seconds for some containers?

Because the app runs as **PID 1** in its PID namespace (10.1), and the kernel doesn't deliver a signal to PID 1 unless it **installed a handler** for it. Many programs never install a `SIGTERM` handler (they rely on the default action, which doesn't apply to PID 1), so `docker stop`'s `SIGTERM` is silently dropped; after the 10 s grace period Docker sends `SIGKILL`, which is sent from *outside* the namespace and therefore works (exit 137). Fixes: handle `SIGTERM` in the app, use `exec` in entrypoint scripts so the app *is* PID 1 rather than a shell that doesn't forward signals, or run a tiny init (`docker run --init` = `tini`) that forwards signals and reaps zombies.

### Why does `yes | head -n 1` terminate, even though `yes` loops forever?

`head` exits after one line, closing the pipe's **only read end**. The next `write()` by `yes` hits a pipe with no readers, so the kernel sends it **`SIGPIPE`**, whose default action is terminate (9.1, 8.2). Without `SIGPIPE`, `yes` would get an `EPIPE` error from `write()` and would have to check for it. That's also why `bash` sometimes reports exit status 141 (128 + 13) for the left side of a pipeline.

### In a multithreaded program, which thread receives a signal?

- A **process-directed** signal (`kill`, `Ctrl+C`) goes to **any one thread that isn't blocking it**; you can't predict which.
- A **synchronous** signal caused by an instruction (`SIGSEGV`, `SIGFPE`) goes to **the thread that caused it**.
- `pthread_kill()`/`tgkill()` target a **specific** thread.

Handlers are shared by all threads; masks are per thread. The robust pattern is to block signals in all threads and handle them in one dedicated thread with `sigwait()` or a `signalfd` (8.5).

## Memory

### Two different processes print the same pointer value (e.g. `0x55d0c1a02000`). Do they share that memory?

**No.** Each process has its **own page tables** (its own `CR3`, 6.3), so the same **virtual** address maps to different **physical** frames in each process. The address is only meaningful inside one address space. (They *may* map to the same frame by design: shared library code, or shared memory, 9.3. But then it's because the kernel set up both mappings that way, not because the numbers match.)

### Why doesn't `fork()` of a process using 10 GB take 10 GB of extra RAM or seconds of copying?

**Copy-on-write** (4.9, 6.5). `fork()` copies the page **tables** and marks writable pages read-only in both processes; the data pages are shared. A page is copied only when one side **writes** to it. If the child calls `execve()` right away, almost nothing is ever copied. What *is* proportional to memory size is copying the page tables themselves (~20 MB for 10 GB), which is why `vfork()`/`posix_spawn()` exist for huge processes.

### How can a program allocate 100 GB on a 16 GB machine?

**Overcommit + demand paging** (6.5, 6.8). `malloc` only asks the kernel for a **VMA** (`mmap`): a promise of address space, not RAM. Physical frames are allocated one page at a time when you first **touch** them. So the kernel can promise more than it has:

- With the default heuristic (`vm.overcommit_memory = 0`), a **single** 100 GB request is refused (it's larger than RAM + swap, so `malloc` returns `NULL`), but **ten 10 GB requests** each look plausible and all succeed.
- With `vm.overcommit_memory = 1` (always allow), even the single 100 GB request succeeds.
- With `vm.overcommit_memory = 2` (strict accounting), allocations fail with `NULL` once total commitments exceed swap + `overcommit_ratio`% of RAM.

If the program then actually writes to more pages than RAM + swap can hold, the **OOM killer** picks a process and kills it with `SIGKILL` (exit 137).

### What's the difference between a page fault and a segmentation fault?

A **page fault** is a CPU exception (`#PF`) raised whenever a translation fails, and most are **completely normal**: demand paging, copy-on-write, loading a file page, swapping in. The kernel fixes the page table and **retries** the instruction; the program never notices (6.5).

A **segmentation fault** is what happens when the kernel investigates a page fault and finds the access is **illegal**: the address isn't inside any VMA (`NULL` pointer, freed memory, stack overflow into the guard gap), or the VMA's permissions forbid it (writing to `.text`, executing the stack). Then it sends **`SIGSEGV`** (8.2). So every segfault starts as a page fault, but very few page faults are segfaults.

### Why do user-space addresses look like `0x7ffc...` or `0x55...`, and kernel addresses like `0xffff...`?

x86-64 uses **48-bit canonical addresses** (1.4): user space is the lower half (`0x0000...` to `0x00007fffffffffff`), the kernel is the upper half (`0xffff800000000000` and up), with a non-canonical hole between. Within user space, the **stack** is placed near the top (`0x7ffc...`), **mmap**/shared libraries just below it (`0x7f...`), and **PIE executables + heap** around `0x55...`/`0x56...`, all randomized by **ASLR** (4.4).

### `free` shows almost no free memory. Is the system running out of memory?

Probably not. Linux uses otherwise idle RAM as **page cache** for file data (6.7), shown as `buff/cache`. Clean cache pages can be dropped instantly when a program needs memory. Look at **`available`**, not `free`. Signs of real memory pressure are: `available` near zero, swap in/out activity (`vmstat` `si`/`so`), rising **major** page faults, `kswapd` CPU usage, high `/proc/pressure/memory`, and OOM kills in `dmesg` (6.8).

### Does every memory access go through the page tables?

Logically yes, physically almost never. The **TLB** caches recent virtual → physical translations, and a TLB hit takes about a cycle (2.4). Only on a TLB **miss** does the MMU walk the 4-level table in RAM (6.3), and only if that walk finds no valid entry does the kernel get involved (page fault, 6.5). This is why huge pages help large-memory workloads: one TLB entry covers 2 MB instead of 4 KB (6.9).

### Can a user process read kernel memory?

Architecturally, **no**: kernel pages have the **U/S** bit set to supervisor in the page-table entries, so any Ring 3 access faults (2.6, 6.3). But **Meltdown** showed that on vulnerable CPUs, speculative execution could read kernel data *before* that check took effect, and leak it through cache timing (2.2). The fix, **KPTI**, removes almost all kernel mappings from the page tables used while in user mode (3.3). Legitimate access goes through system calls, `/proc`, and `/sys`, which copy data out after checking permissions.

### What happens on a stack overflow (e.g. infinite recursion)?

Each call pushes a frame; the stack grows down toward the **guard gap** below the stack VMA (limit `ulimit -s`, 8 MiB by default). When a push touches an address that is outside any VMA, the page fault handler finds no valid mapping and sends **`SIGSEGV`** (6.5). Since the stack is full, the handler couldn't even run, unless one was installed on an alternate signal stack (`sigaltstack`). Thread stacks (5.1) are smaller and have their own guard pages.

## Kernel, CPU and Scheduling

### Is every system call a context switch?

**No.** A system call is a **mode switch**: the same task moves from Ring 3 to Ring 0 and back (3.5). A **context switch** happens only if the kernel decides to run a *different* task, for example because the syscall had to wait (`read()` on an empty pipe → sleep → `schedule()`) or because a higher-priority task became runnable. Many syscalls (`getpid`, `write` to a pipe with space) complete with no context switch at all, and vDSO calls like `clock_gettime` don't even need the mode switch (3.3).

### How does the kernel schedule processes? What happens if all processes are I/O-bound?

Normal tasks are scheduled by **EEVDF** (CFS before Linux 6.6): each task accumulates weighted **virtual runtime**, and the scheduler picks the eligible task with the earliest virtual deadline, roughly "whoever has received the least CPU relative to its fair share" (3.6). Real-time and deadline policies are checked first. Each CPU has its own run queue, and tasks are load-balanced between them.

If **all processes are I/O-bound**, they spend most of their time **sleeping** (`S`/`D`) waiting for devices; run queues are usually empty, and the CPU runs the **idle task** in a low-power state (2.1). When an I/O completes, the device interrupt **wakes** the waiting task (3.4, 7.7); because it slept, its vruntime is low compared to any CPU hog, so it's scheduled almost immediately. The result: excellent responsiveness, low CPU utilization, and (in Linux's definition) a load average that counts the `D` tasks (3.6).

### What's the difference between `strace`, `ltrace`, and `ptrace`?

`ptrace` is not a tool: it's the **system call** that the tools (and debuggers) are built on.

| | Traces | Layer | How | Privileges |
| :---- | :---- | :---- | :---- | :---- |
| **`strace`** | **System calls** + signals | The user/kernel boundary (3.3) | Uses `ptrace(PTRACE_SYSCALL)` to stop the target at every syscall entry/exit | Your own processes as a normal user (subject to `kernel.yama.ptrace_scope`); others need root / `CAP_SYS_PTRACE` |
| **`ltrace`** | **Library calls** (`malloc`, `strlen`, `printf`) | User space, between the program and shared libraries (4.9 PLT/GOT) | Uses `ptrace` to set breakpoints on PLT entries | Same as strace |
| **`ptrace()`** | Anything: read/write the target's memory and registers, single-step, intercept syscalls and signals | Kernel API | The tracer *attaches*; the target stops in state `t` (4.8) | Same rules; used by `gdb`, `strace`, `ltrace`, and some sandboxes |

`strace` is heavy: every syscall causes two ptrace stops (entry and exit), each a round trip to the tracer process, so several context switches per syscall. For low-overhead production tracing, use eBPF-based tools (`bpftrace`, `perf trace`).

### Why is `read()`-ing a file one byte at a time so slow?

Each `read()` is a **system call**: a mode switch, entry checks, the VFS path, a page cache lookup, and the return (~100–300 ns, more with KPTI, 3.3), just to copy one byte. Reading 1 MB costs a million syscalls instead of a few dozen. That's why stdio buffers (`fgetc()` reads 4 KB at once and hands you bytes from user-space memory, 7.6), and why every fast I/O API (`readv`, `mmap`, `io_uring`) is about crossing the user/kernel boundary less often.

### The load average is 20 on an 8-core machine, but `top` shows the CPUs mostly idle. What's happening?

Linux's load average counts tasks in **`R` (runnable) and `D` (uninterruptible sleep)** (3.6). Idle CPUs + high load = many tasks stuck in **`D`**, waiting on **disk or network filesystem I/O**: a saturated disk, a hung NFS mount, heavy swapping. Check `ps -eo stat,pid,cmd | grep '^D'`, `iostat -x 1` (disk utilization), `vmstat 1` (the `b` column and `wa` CPU %), and `/proc/pressure/io`.

### Why does a `printf` before `fork()` sometimes print twice?

`printf` writes into the **stdio buffer in the process's own memory** (7.6). If it hasn't been flushed (no newline, or stdout is a pipe/file so it's fully buffered), `fork()` copies that memory (COW, 4.9), so **both** processes have the pending text and both flush it when they exit. Fix: `fflush(stdout)` before `fork()`, or have the child `_exit()` (which skips flushing stdio).

### What's the real difference between a container and a virtual machine?

A **VM** runs its **own kernel** on virtualized hardware: the CPU's VMX mode, EPT for memory (2.6). Strong isolation, but each VM needs a full OS, its own memory, and a boot.

A **container** is a group of ordinary **host processes** with **namespaces** (restricted views), **cgroups** (resource limits), and **capabilities/seccomp** (restricted privileges), sharing the **host kernel** (10.1–10.4). It starts in milliseconds and has almost no overhead, but a kernel vulnerability is shared by every container on the host. `uname -r` inside a container shows the host's kernel; inside a VM, its own.

### Can a process in one container see or signal processes in another container?

Not normally. Each container has its own **PID namespace**, so the other container's processes simply don't exist in its view: no PIDs to signal (10.1). The **host** (the root PID namespace) sees all of them, with host PIDs. If two containers are started sharing a PID namespace (`docker run --pid=container:other`, or pods in Kubernetes with `shareProcessNamespace`), they can.

### When one thread calls `close(fd)`, is the file closed for the other threads too?

**Yes.** Threads share the fd table (`CLONE_FILES`, 5.1, 7.1), so `close()` removes the fd for **everyone**. Worse, the number is immediately reusable: if another thread opens a file, it may get the same fd number, and a third thread that still uses the old number now reads or writes the **wrong file**. That's a classic multithreading bug, and one of the reasons the fd table is called shared state (5.2).

## Numbers, Time and Storage

### Why is `0.1 + 0.2` not equal to `0.3`?

Because 0.1, 0.2 and 0.3 have **no exact binary representation**, just like 1/3 has no exact decimal one. A `double` stores the closest value with a 53-bit significand, so each number carries a tiny error, and the sum of the two errors doesn't equal the error of 0.3: `0.1 + 0.2 = 0.30000000000000004` (1.10). It's not a bug in any language; it's IEEE 754. Compare with a tolerance, and use integers (cents) or decimal types for money.

### Why does `exit(-1)` give exit status 255, and `exit(256)` give 0?

The exit status passed to the parent is only the **low 8 bits** (4.9). `-1` in two's complement is all ones (1.9), so its low byte is `0xFF` = 255. `256` is `0x100`, whose low byte is `0x00` = 0, which means "success". A script that does `exit(errors)` with 256 errors reports success!

### Why should you measure elapsed time with `CLOCK_MONOTONIC` instead of `time()` or `gettimeofday()`?

Those return **wall-clock** time (`CLOCK_REALTIME`), which can **jump**, backward or forward, when NTP steps the clock, an admin runs `date -s`, or a VM is resumed (3.8). A duration computed across such a jump can be negative or hours long, causing broken timeouts and nonsense metrics. `CLOCK_MONOTONIC` never goes backward. Use wall-clock time only for timestamps that humans read.

### What is the Year 2038 problem?

A **signed 32-bit** `time_t` counts seconds since 1970 and overflows at **2³¹ − 1 = 2038-01-19 03:14:07 UTC**, wrapping to a negative value that means December 1901 (1.9, 3.8). 64-bit Linux uses a 64-bit `time_t` and is safe. 32-bit systems need kernel ≥ 5.6 plus a C library built with 64-bit time, and every file format, database column and protocol field that stores time in 32 bits must be fixed too.

### How do you update a file so that a crash never leaves it half-written?

Write the new content to a **temporary file in the same directory**, `fsync()` it, `rename()` it over the original, then `fsync()` the directory (7.15). `rename()` within one filesystem is **atomic** (readers see either the old file or the new one, never a mix), and the two `fsync`s make sure the data and the rename are actually on disk before you rely on them (6.7). The temp file must be on the same filesystem, or `rename()` fails with `EXDEV` (7.13).

### After a power failure, why can a file that was written successfully be empty or missing?

`write()` returning success only means the data reached the **page cache** (6.7). It's written to disk seconds later by flusher threads, and the drive may hold it in its own volatile cache even longer. The **journal** (7.15) guarantees the filesystem's structure is consistent after a crash, but not that your recent writes survived. Only `fsync()`/`fdatasync()` (which also flushes the drive's cache) gives that guarantee, which is why databases call it on every commit.

### A program built on my laptop fails on a server with "GLIBC_2.34 not found". Why?

glibc **versions its symbols** (4.10). Your binary was linked against a newer glibc and recorded that it needs, say, `__libc_start_main@GLIBC_2.34`. The server's older glibc doesn't provide that version, so the dynamic linker refuses to start it, before `main` ever runs. Fix: build on (or in a container of) the oldest distro you target, or link statically (Go does this by default; C can use musl).
