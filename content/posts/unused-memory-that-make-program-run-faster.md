---
title: "Unused Memory That Make Program Run Faster"
date: 2022-03-25T23:09:57+08:00
tags: ["memory"]
draft: true
---

They are more common than you might think.

<!--more-->

## Not All Memory Access Are Equal

Most of the time, optimization made in generic device or general framework is a trade off that will favor the most common use cases. It is the same thing with memory access, certain memory access pattern will be more favored.

## Unaligned Memory Access



## False Sharing

Say you have a multi-thread application that need to read/write to a small memory region that is close together (i.e. high chance of in the same cache line). When you write to a cache line, even just a little bit.
