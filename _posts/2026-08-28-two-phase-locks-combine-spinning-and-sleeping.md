---
layout: post
title: Two-Phase Locks Combine Spinning and Sleeping
date: 2026-08-28 14:38 +0300

type: mechanism
source: "OSTEP Ch. 28: Locks"
tags:
  - concurrency
  - locks
  - linux
  - futex
related:
  - "[[Spin Locks rely on Busy Waiting]]"
  - "[[Queue-Based Locks Put Waiting Threads to Sleep]]"
prerequisites:
  - "[[Lock Design Goals - Mutual Exclusion Fairness and Performance]]"
categories: [books, OSTEP]
---

> Two-phase locks describe a hybrid strategy that combines the low latency of spinning with the efficiency of sleeping. A thread first spins for a short duration, hoping the lock becomes free quickly; if not, it enters a second phase where it sleeps. This approach works well for both short and long critical sections.
{: .prompt-tip }


# Two-Phase Locks Combine Spinning and Sleeping

Choosing between spinning (good for short waits) and sleeping (good for long waits) is difficult because the OS doesn't know how long a lock will be held. Two-phase locks (or hybrid locks) attempt to get the best of both worlds.

## The Strategy
1.  **Phase 1 (Spinning):** When a thread tries to acquire a lock, it spins for a fixed amount of time (or a fixed number of loops). If the lock is released during this time, the thread acquires it cheaply without a context switch.
2.  **Phase 2 (Sleeping):** If the spin phase times out and the lock is still held, the thread puts itself to sleep (using a mechanism like queues/futexes) and waits to be woken up.

## Real-World Example: Linux Futex
Linux uses "futexes" (fast userspace mutexes) which embody this principle.
*   **Fast Path:** An atomic instruction attempts to grab the lock in user space. If successful (no contention), the kernel is never involved.
*   **Slow Path:** If the lock is held, the thread calls into the kernel to sleep.

Older Linux kernels used a strict two-phase lock that spun for a set number of cycles before sleeping. This heuristic avoids the "yield storm" of yielding locks and the CPU waste of pure spin locks, making it a robust general-purpose solution.
