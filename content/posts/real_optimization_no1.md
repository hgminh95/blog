---
title: "Optimization in Real Code No. 1"
date: 2022-03-24T00:38:02+08:00
draft: true
---

Just some notes on optimizations found in the wild.

In this article, we will discuss about the capacity of the container. How to choose the new capacity once the container grow.

<!--more-->

Most containers usually have 2 attributes: size (number of the elements in the container) and capacity (maximum number of element it can hold before resizing to). You can see this in C++ std, Java, Go, etc.

When the number of the element reach the capacibility, it must grow to be able to hold more elements, or to prevent performance degration when the container is nearly full (e.g. hash table). The question is, by how much?

## C++ vector

## Boost unordered_map

## flat_hash_map

## SPSC queue
