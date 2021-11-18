---
layout: post
title:  "What is an atomic increment?"
date:   2021-11-17 14:12:06 +0000
categories: welcome
---
Why do we call ourselves Atomic Increment?
One of the myriad ways of improving code performance
is to use multithreaded code to lower latency. Every modern computer has
many cores which can run one or more threads at the same time.

The "atomic increment" operation allows us to safely share a counter between
two threads. Why is this necessary? This is because two CPUs running the same
code increment a counter using these three operations:

```
   load from memory
   increment
   store back to memory
```

If two threads are running, then we may get these following sequences
The good one:

```
    Processor 1               Processor 2
   |------------------------|------------------------|
   | load from memory       |                        |
   | increment              |                        |
   | store back to memory   |                        |
   |                        | load from memory       |
   |                        | increment              |
   |                        | store back to memory   |
   |------------------------|------------------------|
```

This adds 2 to the memory.

And the bad one:

```
    Processor 1               Processor 2
   |------------------------|------------------------|
   | load from memory       |                        |
   |                        | load from memory       |
   | increment              |                        |
   |                        | increment              |
   |                        | store back to memory   |
   | store back to memory   |                        |
   |------------------------|------------------------|
```

The second sequence is bad because we only add 1 to the memory
not 2! This is because Processor 1 overwrites the result of
Processor 2 - a situation known as a "Race Condition".

To solve this we use special instructions that allow the CPUs
to "lock" the memory while the increment occurs. How this
is implemented depends on the CPU.

For more information about concurrent programming, get in touch
with us through andy@atomicincrement.com
