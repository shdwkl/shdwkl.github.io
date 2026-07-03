---
layout: post
title: Address Translation Maps Virtual Addresses to Physical Addresses
date: 2026-07-02 21:29 +0000
type: mechanism
source: "OSTEP Ch. 15: Mechanism: Address Translation"
tags:
  - virtualization
  - memory
  - address-translation
related:
  - "[[The Address Space Abstracts Physical Memory for Each Process]]"
  - "[[Memory Virtualization Goals - Transparency Efficiency and Protection]]"
  - "[[Base-and-Bounds Provides Dynamic Relocation]]"
  - "[[Hardware Registers Enable Efficient Address Translation]]"
  - "[[Virtualization MOC]]"
categories: [books, OSTEP]
---


> Address translation is the hardware mechanism that converts every virtual address issued by a process into the corresponding physical address in RAM. This interposition happens on every memory reference, enabling the OS to control where each process's memory actually resides while maintaining the illusion of a private address space starting at zero.
{: .prompt-tip }

## The Core Idea: Interposition

When a process executes an instruction like `load 0x1000`, it references a **virtual address** within its The Address Space Abstracts Physical Memory for Each Process address space. The hardware must convert this to a **physical address**—the actual location in DRAM where the data lives. This conversion is called **address translation**.

The key insight is that the hardware interposes on every memory reference. The CPU never sends raw virtual addresses to the memory bus. Instead, a hardware unit (later formalized as the **Memory Management Unit** or MMU) transforms the address before it reaches physical memory. The process runs unmodified, believing its addresses are real.

## Why Hardware-Based Translation

Software-only translation would be too slow. Programs access memory billions of times per second; adding even a few instructions per access would devastate performance. Instead, the hardware performs translation with zero additional cycles in the common case.

The OS provides **control** (setting up translation parameters), while the hardware provides **speed** (performing the actual translation on every access). This OS/hardware partnership mirrors the User Mode and Kernel Mode Provide Dual Protection Levels split for CPU virtualization: the OS sets policy, the hardware enforces it efficiently.

## From Virtual to Physical: The Formula

In the simplest form of address translation (Base-and-Bounds Provides Dynamic Relocation):

```
physical_address = virtual_address + base
```

The hardware adds a **base register** value to every virtual address. If the process's code lives at physical address 32 KB but its virtual addresses start at 0, then `base = 32 KB`. A virtual reference to address 0x1000 becomes physical address 32 KB + 0x1000.

This relocation is **dynamic**—it happens at runtime, not at compile time or load time. The process can be placed anywhere in physical memory without modification.

## Address Translation Enables Three Goals

Translation is the mechanism through which the OS achieves its Memory Virtualization Goals - Transparency Efficiency and Protection:

- **Transparency:** The process sees addresses starting at 0 regardless of physical placement.
- **Efficiency:** Hardware performs the translation with negligible overhead per access.
- **Protection:** The hardware can check that the translated address falls within legal bounds, preventing out-of-range accesses.

Every subsequent memory virtualization technique—segmentation, paging, TLBs—is a more sophisticated form of this same fundamental operation: converting virtual addresses to physical addresses while maintaining transparency, efficiency, and protection.
