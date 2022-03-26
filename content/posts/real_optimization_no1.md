---
title: "Optimization in Real Code No. 1"
date: 2022-03-24T00:38:02+08:00
draft: false
---

Just a note on some optimization found in the wild.

<!--more-->

## $2^n-1$

The idea is that if $k = 2^n - 1$ then $x = x & k (mod k)$
