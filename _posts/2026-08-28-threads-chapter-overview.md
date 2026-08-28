---
layout: post
title: Threads - Chapter Overview
date: 2026-08-28 14:32 +0300
tags:
  - concurrency
  - threads
  - overview
related:
  - "[[Concurrency MOC]]"
  - "[[Processes - Chapter Overview]]"
prerequisites: []
source: "OSTEP Ch. 26: Concurrency and Threads"
categories: [books, OSTEP]
---


This chapter introduces the concept of **threads**, a fundamental abstraction for concurrency. Unlike processes, which are isolated execution units, threads share the same address space, enabling them to access the same data. This shared-memory model provides powerful capabilities for parallelism and responsiveness but introduces new challenges such as race conditions and the need for synchronization.

## Key Concepts

*   **[[Thread vs Process - Shared Address Space]]**: Understanding the key difference: processes have private memory, while threads share the address space (specifically the heap and code, but have private stacks).
*   **[[Why Use Threads - Parallelism and Responsiveness]]**: The two main motivations for multi-threading: speeding up execution on multi-core hardware and keeping applications responsive during blocking I/O operations.
*   **[[Thread Creation and Execution Model]]**: How threads are spawned and managed by the OS, including the unique layout of a multi-threaded address space with multiple stacks.
*   **[[Race Conditions lead to Non-Deterministic Behavior]]**: The danger of uncontrolled access to shared data, where the outcome of a program depends on the unlucky timing of thread execution.
*   **[[Critical Sections Require Mutual Exclusion]]**: The solution to race conditions—identifying code segments that access shared resources and ensuring only one thread executes them at a time.

## Summary

> The transition from a single-threaded process to a multi-threaded one changes the programming model significantly. While the CPU scheduler treats threads much like processes (scheduling them independently), the shared state requires careful coordination. The lack of protection between threads means a bug in one thread can corrupt the state of another, making debugging difficult. Understanding the underlying mechanics of thread execution, context switching (which now involves saving thread-private registers but keeping the address space), and the non-determinism of scheduling is a prerequisite for mastering concurrency.
{: .prompt-tip }