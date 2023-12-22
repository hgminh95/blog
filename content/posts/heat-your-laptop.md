---
title: "Heat Your Laptop (draft)"
date: 2023-12-21T18:44:50+08:00
thumbnail: laptop-burning.png
draft: false
---

Have you ever been in a long-haul flight, nestled in your seat with your laptop, only to find that the chilly cabin air is making it impossible to sleep? With no access to charging outlet, and your laptop's battery life is limited, you are indeed in a desperated situation. In this article, we will explore an efficient way to heat up your laptop while minimizing power usage, ensuring that you stay cozy throughout the flight.

I also wrote a tool to heat up your laptop, you can find it [here](https://github.com/hgminh95/heat).

<!--more-->

## Where does the heat come from?

Before we delve into the solution, let's first pinpoint the primary component responsible for generating heat on your laptop - the Central Processing Unit (CPU). It is one of the components that [consume the most power](https://static.aminer.org/pdf/PDF/000/525/706/power_consumption_breakdown_on_a_modern_laptop.pdf), and the fact that there are fan or cooling system dedicated to it making this at least a good guess.

But what mechanism that generate the heat in CPU. After minutes of research in Stack Overflow ([link1](https://electronics.stackexchange.com/questions/160249/why-does-a-processor-get-hot), [link2](https://stackoverflow.com/questions/12715461/what-cpu-instructions-use-the-most-power), TODO... it looks like the more transistor you can activate at a time, the more heat you can generate.

There are various techniques in hardware design to minimize heat generation such as TODO..... As for our purpose, we basically try to undo their effort.

My initial hypothesis is TODO...

## Setup the test environment

_Note that below is only for Apple M2 Macbook Air_

First is how do we measure amount of heat generated and the power consumed. Both are hard to measured directly (with my limited tooling), I arrive at below solutions that seems to work reliably and well enough:

For **temparature**, there are various sensors measuring different components in your laptop. You can read some values from [SMC](https://en.wikipedia.org/wiki/System_Management_Controller) via [IOKit](https://developer.apple.com/documentation/iokit). The sensor name varies between different models. I cannot really find any official source documenting this, but luckly, there are quite many open source library, applications that extract this. In the end, I settle with [smctemp](https://github.com/narugit/smctemp), which seems to work well with my laptop. The library read the temperature values from all the sensor placed at all CPU cores (both P-cores and E-cores) and average the values.

Note that there are cooling system that actively trying to drive down the temperature. And if the laptop is overheat, the CPU frequency and workload can be throttled. So this metric is actually more like a lower bound of heat produced.

I measure the temperatures differences in 5s, 10s, 15s after the code run (after 15s, the temperature tend to stay constant).

For **power usage**, this [answer](https://superuser.com/a/1700413) in Stack Overflow provides a great way to do it, by plugging your laptop to a power charger, and measuring the amount of power coming into the laptop. The number is not real time, and seems to be refresh on the 30s interval. This is the whole laptop power input, so you might want to disable everything that might consume power: screen, bluetooth, wifi, and kill every possible process you can.

Now, what kind of code do we benchmark? I can think of serveral things:

- Memory Access (L1, L2, L3)
- Arithmetics (integer, floating point, etc)
- CPU interconnect (multi-thread basically)
- SIMD

TODO: Explain a bit more here

Lastly, for the code itself, I end up write a small framework where you can easily add a piece of code you want to benchmark in the github repos. For example, you can add this [code](https://github.com/hgminh95/heat/blob/main/heat/methods/math.cpp) and run the benchmark with below command

```
$ bazel-bin/heat/heat --method math --cores 4 --bm
...] Emit heat using method math in 1 core(s)
...] Running in benchmark mode
...] Getting baseline temperature and power usage. Please check to make sure the numbers are stable.
...] Will be finished in 2min
...] Baseline temp: sample=24 avg=38.702347 std=3.649732 min=36.057220 max=53.969673
                    p10=36.253639 p50=37.664368 p90=41.612141
...] Baseline watt: sample=24 avg=4.013780 std=2.496656 min=1.516578 max=6.510436
                    p10=1.516578 p50=6.510436 p90=6.510436
...] Run #0
...] Initial Temp: 35.2926
...]   5s:  +7.83518
...]   10s: +8.84517
...]   15s: +10.1611
...] Temps: sample=30 avg=47.651833 std=1.432894 min=43.127800 max=48.967751
            p10=45.620243 p50=48.366695 p90=48.770966
...] Watts: sample=9 avg=5.409431 std=0.019220 min=5.373477 max=5.419704
            p10=5.373477 p50=5.419704 p90=5.419704
```

I wouldn't say this is the most accurate benchmark to measure this, but at least we can some useful conclusion from it (hopefully!).

## Results

I tested below:

- A [empty](https://github.com/hgminh95/heat/blob/main/heat/methods/empty.cpp) while loop that does nothing
- A code that does some simple [arithmetics](https://github.com/hgminh95/heat/blob/main/heat/methods/math.cpp) calculation
- A simple [SIMD](https://github.com/hgminh95/heat/blob/main/heat/methods/simd.cpp) code

The benchmark is run multiple times to make sure the result are stable as there are many external environmental factor that could affect it. Here are some of the representative result.

| Code         | Initial Temp | Temp Diff (5s) | Temp Diff (10s) | Temp Diff (15s) | Power Consumption (Wh - p90) |
| ------------ | ------------ | -------------- | --------------- | --------------- | ---------------------------- |
| empty        | 37           | +8             | +11             | +11             | 6.7                          |
| arithmetics  | 37           | +7             | +8              | +10             | 5.4                          |
| SIMD         | 40           | +14            | +16             | +18             | 8.3                          |
| more...      |

The full logs can be found [here](TODO).

Some results can be a bit of surprise:

- TODO...

Now, what is the best one depends on various factors, e.g. how warm you want it to be, how much battery you have left, etc. But in term of heat/power ratio, SIMD seems to be the winner here (again, the benchmark is not that accurate, so don't put too much trust on this).

In conclusion, staying warm during a flight when conserving your laptop's battery power is indeed possible. Just remember to keep an eyes on your laptop to avoid overheating issues. Safe travel and happy computing!
