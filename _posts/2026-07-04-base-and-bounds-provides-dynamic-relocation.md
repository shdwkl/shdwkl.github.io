---
layout: post
title: Base-and-Bounds Provides Dynamic Relocation
date: 2026-07-04 06:32 +0000
type: mechanism
source: "OSTEP Ch. 15: Mechanism: Address Translation"
tags:
  - virtualization
  - memory
  - base-and-bounds
  - dynamic-relocation
related:
  - "[[Address Translation Maps Virtual Addresses to Physical Addresses]]"
  - "[[Hardware Registers Enable Efficient Address Translation]]"
  - "[[The OS Manages Memory at Process Creation and Context Switch]]"
  - "[[The Address Space Abstracts Physical Memory for Each Process]]"
  - "[[Virtualization MOC]]"
prerequisites:
  - "[[Address Translation Maps Virtual Addresses to Physical Addresses]]"
categories: [books, OSTEP]
---

> Base-and-bounds (also called dynamic relocation) is the simplest hardware mechanism for address translation. Each process gets two registers: a base (starting physical address) and a bounds (size limit). The hardware adds the base to every virtual address and checks that the result stays within bounds, providing both relocation and protection in a single mechanism.
{: .prompt-tip }

## The Mechanism

Dynamic relocation places the entire address space of a process contiguously in physical memory. Two hardware registers define the placement:

- **Base register:** The physical address where the process's address space begins.
- **Bounds register:** The size of the address space (or equivalently, the maximum valid virtual address).

On every memory reference, the hardware computes:

```
physical_address = virtual_address + base

if virtual_address >= bounds:
    raise protection fault
```

If the virtual address exceeds the bounds, the hardware traps to the OS, which typically terminates the offending process. This bounds check is what provides Memory Virtualization Goals - Transparency Efficiency and Protection|protection - a process cannot reference memory outside its own address space.

## Physical Memory Layout

Consider three processes (A, B, C) each with a 16 KB address space placed in 64 KB of physical memory:

```
Physical Memory (64 KB)
+------------------+ 0 KB
|   OS (kernel)    |
+------------------+ 16 KB
| Process A        |  <- base_A = 16 KB, bounds_A = 16 KB
| code|heap|stack  |
+------------------+ 32 KB
| Process B        |  <- base_B = 32 KB, bounds_B = 16 KB
| code|heap|stack  |
+------------------+ 48 KB
|     (free)       |
+------------------+ 56 KB
| Process C        |  <- base_C = 56 KB, bounds_C = 8 KB
| code|heap|stack  |
+------------------+ 64 KB
```

When Process A references virtual address 0x1000 (4 KB), the hardware computes: 16 KB + 4 KB = 20 KB physical. When Process B references the same virtual address 0x1000, it gets: 32 KB + 4 KB = 36 KB physical. Each process sees the same virtual layout but occupies different physical regions.

## Why "Dynamic"

Earlier approaches relocated programs at **load time** by rewriting addresses in the binary. Dynamic relocation happens at **runtime**—the binary is unchanged, and the hardware adds the base on every access. This means:

- A process can be **moved** in physical memory (just update its base register).
- The same binary can run at any physical location without modification.
- The OS can relocate processes during Context Switch Saves and Restores Process State|context switches for compaction.

## Limitations

Base-and-bounds is simple but wasteful. The entire address space—including the unused gap between heap and stack—must be allocated contiguously. If a process has a 16 KB address space but only uses 3 KB of heap and 1 KB of stack, the remaining 12 KB of its physical allocation is **internal waste** (later called internal fragmentation in the paging context). This waste motivates [[Segmentation Generalizes Base-and-Bounds with Multiple Segments|segmentation]], which assigns separate base-bounds pairs to each logical segment.
