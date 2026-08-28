---
layout: post
title: Locks - Chapter Overview
date: 2026-08-28 14:36 +0300
type: overview
source: "OSTEP Ch. 28: Locks"
tags:
  - concurrency
  - locks
  - synchronization
  - overview
related:
  - "[[Concurrency MOC]]"
  - "[[Critical Sections Require Mutual Exclusion]]"
prerequisites:
  - "[[Critical Sections Require Mutual Exclusion]]"
categories: [books, OSTEP]
---

# Locks - Chapter Overview

> Locks are the fundamental mechanism for ensuring **mutual exclusion** in concurrent programs. By wrapping a **critical section** with a lock, we ensure that only one thread can execute that code at a time, preserving atomicity and preventing race conditions.
{: .prompt-tip }


This chapter explores how to build locks from the ground up, starting with hardware primitives and moving to OS-supported implementations.

## Key Concepts

*   [Lock Design Goals - Mutual Exclusion Fairness and Performance]({% post_url 2026-08-28-lock-design-goals-mutual-exclusion-fairness-and-performance %}) - What makes a good lock?
*   [[Spin Locks rely on Busy Waiting]] - The simplest lock, but wasteful of CPU.
*   [[Test-and-Set Instruction Enables Atomic Locking]] - The hardware support needed for basic spin locks.
*   [[Compare-and-Swap Provides Powerful Atomic Update]] - A more general synchronization primitive.
*   [[Ticket Locks Ensure Fairness with Fetch-and-Add]] - How to guarantee every thread gets a turn.
*   [[Yielding Locks Avoid Spinning but Incur Context Switches]] - Cooperating with the OS scheduler.
*   [[Queue-Based Locks Put Waiting Threads to Sleep]] - Explicitly managing waiting threads (Solaris/Linux implementation).
*   [Two-Phase Locks Combine Spinning and Sleeping]({% post_url 2026-08-28-two-phase-locks-combine-spinning-and-sleeping %}) - A hybrid approach used in Linux.