---
layout: post
title: Thread vs Process - Shared Address Space
date: 2026-08-28 14:33 +0300
type: concept
source: "OSTEP Ch. 26: Concurrency: An Introduction"
tags:
  - concurrency
  - threads
  - processes
  - abstraction
related:
  - "[[Threads - Chapter Overview]]"
  - "[[A Process is a Running Program]]"
  - "[[The Address Space Abstracts Physical Memory for Each Process]]"
prerequisites:
  - "[[A Process is a Running Program]]"
  - "[[The Address Space Abstracts Physical Memory for Each Process]]"
categories: [books, OSTEP]
---


> A **thread** is often described as a "lightweight process." Like a process, a thread has its own program counter (PC) that tracks where the program is fetching instructions from, and its own set of private registers for computation. However, the fundamental difference lies in memory usage.
{: .prompt-tip }

## Single-Threaded vs. Multi-Threaded

In a traditional **single-threaded process**, there is one program counter and one stack. The address space contains a single stack region (usually growing downward) and a single heap region (usually growing upward). The process owns this entire memory image, and the OS protects it from other processes.

In a **multi-threaded process**, multiple execution streams exist within the *same* address space. This means they share:
*   **Code Segment**: All threads execute instructions from the same program text.
*   **Heap**: Memory allocated via `malloc()` is accessible to all threads.
*   **Global Data**: Static and global variables are shared.
*   **Open Files**: File descriptors are shared across threads.

## The Multi-Threaded Address Space

The most visible change in the memory layout is the **stack**. Instead of a single stack, a multi-threaded address space contains **one stack per thread**. 

```
+---------------------------+
| Code (Shared)             |
+---------------------------+
| Heap (Shared)             |
| ...                       |
+---------------------------+
| Stack (Thread 1)          |
|    | v                    |
+---------------------------+
| Stack (Thread 2)          |
|    | v                    |
+---------------------------+
```

Each thread needs its own stack because function calls, local variables, and return addresses are inherently private to an execution stream. If two threads tried to use the same stack, one thread calling a function would overwrite the return address or local variables of the other, leading to chaos.

## Context Switching

Context switching between threads is faster than between processes. When switching processes, the OS must switch the page table (changing the entire address space). When switching threads within the same process, the address space remains the same (the page table pointer doesn't change); only the registers (including the PC and stack pointer) need to be saved and restored. This is why threads are "lightweight."

> [!SUMMARY]
> A thread is an independent execution context within a process. Unlike processes which have separate address spaces, threads share the code, heap, and global data of their parent process. The key distinction is that each thread maintains its own **private stack** and register state (including Program Counter and Stack Pointer) to track its independent execution flow. This sharing enables efficient communication via memory but requires synchronization, while the separate stacks allow independent function call chains.
