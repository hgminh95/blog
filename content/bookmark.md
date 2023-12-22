---
title: "Bookmark"
date: 2020-03-11T20:34:35+08:00
---

<!--more-->

## Sources

- [Julia Evans's blog](https://jvns.ca/) - Short posts on various topics, concise and to the point.
- [easyperf](https://easyperf.net/notes/) - General CPU Performance Tuning
- [1024 cores](https://www.1024cores.net/) - Concurrency, Parallelism
- [Daniel Lemire's blog](https://lemire.me/blog/) - Division, parsing stuff, etc
- [Bartosz Ciechanowski's blog](https://ciechanow.ski/) - Very details, and well-presented
- [CPPCon](https://cppcon.org/) - C++, and performance stuff
- [Hacker News](https://news.ycombinator.com/) - Mostly not about technical stuff, but sometimes interesting things pop up over there
- [Erik Rigtorp's blog](https://rigtorp.se/) - Low latency (no longer updated)
- [Brendan Gregg's website](https://www.brendangregg.com/) - Performance monitor and analysis
- [JabPerf's blog](www.jabperf.com) - Low Latency

## Data Structure and Algo

### Hash Table

- probablydance's series - https://probablydance.com/?s=hash 
- Consistent Hashing - https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf
- Google's Swiss Table - https://abseil.io/about/design/swisstables

### Memory Allocator

- tcmalloc - https://github.com/google/tcmalloc/blob/master/docs/design.md
- jemalloc - https://github.com/jemalloc/jemalloc/wiki

### Timer

- Hashed and Hierarchical Timing Wheels: Data Structures for the Efficient Implementation of a Timer Facility - http://www.cs.columbia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf

### Multi-Thread

- rigtorp ring buffer - https://rigtorp.se/ringbuffer/
- 1024cores MPMC queue - https://www.1024cores.net/home/lock-free-algorithms/queues/bounded-mpmc-queue

### Indexing

- The Ubiquitous B-Tree - http://carlosproal.com/ir/papers/p121-comer.pdf
- Log Structured Merge Tree (LSM Tree) - https://www.google.com/search?q=log+structured+merge+tree&oq=log+structured+merge+tree&aqs=chrome..69i57j35i39l2j0i512l7.2934j0j7&sourceid=chrome&ie=UTF-8

### Large Data System

- The Google File System - https://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf
- MapReduce: Simplified Data Processing on Large Clusters - https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf

### Data Aggregation

- Hyperloglog: The analysis of a near-optimal cardinality estimation algorithm - http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf
- Bloom filter - https://en.wikipedia.org/wiki/Bloom_filter
- HdrHistogram: A High Dynamic Range (HDR) Histogram - https://github.com/HdrHistogram/HdrHistogram
- The PageRank Citation Ranking: Bringing Order to the Web - http://ilpubs.stanford.edu:8090/422/1/1999-66.pdf

### Consensus

- Paxos - The Part Time Paliarment - http://lamport.azurewebsites.net/pubs/pubs.html#lamport-paxos
- Raft - https://raft.github.io/

### Small Tricks

- fastrange: A fast alternative to the modulo reduction - https://github.com/lemire/fastrange
- Small String Optimization - https://github.com/elliotgoodrich/SSO-23

### General Hardwares

- Branch Prediction - https://danluu.com/branch-prediction/
- An Overview of On-Chip Cache Coherence Protocol - https://www.researchgate.net/profile/Zainab-Alwaisi/publication/362068592_An_Overview_of_On-Chip_Cache_Coherence_Protocols/links/5ad71029a6fdcc2935835926/An-Overview-of-On-Chip-Cache-Coherence-Protocols.pdf

### Intel

- How to Benchmark Code Execution Times on Intel® IA-32 and IA-64 Instruction Set Architectures - https://www.intel.de/content/dam/www/public/us/en/documents/white-papers/ia-32-ia-64-benchmark-code-execution-paper.pdf
