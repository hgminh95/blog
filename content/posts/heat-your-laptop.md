---
title: "Heat Up Your Laptop Efficiently"
date: 2023-12-21T18:44:50+08:00
thumbnail: laptop-burning.png
draft: false
---

Have you ever been in a long-haul flight, nestled in your seat with your laptop, only to find that the chilly cabin air is making it impossible to sleep? With no access to a charging outlet, and your laptop's battery life is limited, you are indeed in a desperate situation. In this article, we will explore an efficient way to heat up your laptop while minimizing power usage, ensuring that you stay cozy throughout the flight.

The final tool can be found [here](https://github.com/hgminh95/heat).

<!--more-->

## Where does the heat come from?

Before we delve into the solution, let's first pinpoint the primary component responsible for generating heat on your laptop - the Central Processing Unit (CPU). It is one of the components that [consume the most power](https://static.aminer.org/pdf/PDF/000/525/706/power_consumption_breakdown_on_a_modern_laptop.pdf), and the fact that there are fans or cooling system dedicated to it makes this at least a good guess.

But by what mechanism that heat is generated in the CPU. After minutes of research in [Stack](https://lemire.me/blog/2023/03/21/counting-cycles-and-instructions-on-arm-based-apple-systems/) [Overflow](https://stackoverflow.com/questions/12715461/what-cpu-instructions-use-the-most-power), it looks like heat is generated when a transistor changes state; which roughly means the more hardware is activated at a time, the more heat we can produce.

## Setup the test environment

_Note that below is only for Apple M2 MacBook Air_

First is how we measure the amount of heat generated and the power consumed. Both are hard to measure directly (with my limited tooling). After some testing, I arrive at the below solutions that seem to work reliably and well enough:

For **temperature**, there are various sensors measuring different components in your laptop. You can read some values from [SMC](https://en.wikipedia.org/wiki/System_Management_Controller) via [IOKit](https://developer.apple.com/documentation/iokit). The sensor name varies between different models. I cannot really find any official source documenting this, but luckily, there are quite many open source libraries, applications that extract this. In the end, I settle with [smctemp](https://github.com/narugit/smctemp), which seems to work well with my laptop. The library reads the temperature values from sensors placed at all CPU cores (both P-cores and E-cores) and average the values.

In Linux, you can get them from [/sys/class/thermal](https://docs.kernel.org/driver-api/thermal/sysfs-api.html).

Note that there is cooling system that actively tries to drive down the temperature. And if the laptop is overheat, the CPU frequency and workload can be throttled. So this metric is actually more like a lower bound of heat produced.

I measure the temperature differences in 5s, 10s, 15s after the code run (after 15s, the temperature tend to stay constant).

For **power usage**, this [answer](https://superuser.com/a/1700413) in Stack Overflow provides a great way to do it, by plugging your laptop to a power outlet, and measuring the amount of energy drawing into the laptop; you can get a pretty accurate number. You can extract the data from [ioreg](https://www.manpagez.com/man/8/ioreg/). The number is not real time, and seems to be refreshed in the 30s interval, so you will need to wait for around 1 minute after your code run to get the correct number. This is the whole laptop power input, so you might want to disable everything that might consume power: screen, bluetooth, wifi, and kill every possible process you can.

For completeness, I also measure IPC (instruction per cycle) to see how many instructions are performing together (higher IPC ~ higher hardware utilization ~ more heat). The code is borrowed from [Daniel Lemire's blog](https://lemire.me/blog/2023/03/21/counting-cycles-and-instructions-on-arm-based-apple-systems/).

Now, what kind of code do we benchmark? I can think of several things:

- Memory Access
- Arithmetic (integer, floating point)
- Inter-core communication (multi-thread basically)
- Branch predictor
- SIMD

I don't really benchmark all of them though, as some components seem a lot more lightweight than others (e.g. branch predictor), or leave CPU idle most of the time (e.g. memory access), or due to pure laziness.

Lastly, for the code itself, I end up writing a small framework where you can easily benchmark a piece of code. For example, you can add this [code](https://github.com/hgminh95/heat/blob/main/heat/methods/math.cpp) and run the benchmark with below command

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

I wouldn't say this is the most accurate benchmark, but at least we can get some useful conclusions from it (hopefully!).

## Results

I tested below approaches:

- A code that does some simple [arithmetic](https://github.com/hgminh95/heat/blob/main/heat/methods/math.cpp) calculation
- A simple [SIMD](https://github.com/hgminh95/heat/blob/main/heat/methods/simd.cpp) code

The benchmark is run multiple times to make sure the result are stable as there are many external environmental factor that could affect it. Here are some of the representative result.

| Code                | # cores | Temp Diff (5s) | Temp Diff (10s) | Power Consumption (Wh - p90) | IPC |
| ------------------- | ------- | -------------- | --------------- | ---------------------------- | --- |
| arithmetic_integer  | 1       | +13            | +15             | 10.7                         | 4.7 |
| arithmetic_integer  | 2       | +25            | +29             | 16.3                         | -   |
| arithmetic_integer  | 3       | +32            | +37             | 19.1                         | -   |
| arithmetic_integer  | 5       | +45            | +55             | 18.6                         | -   |
| arithmetic_float    | 1       | +11            | +13             | 10.1                         | 3.4 |
| arithmetic_float    | 2       | +21            | +24             | 12.2                         | -   |
| arithmetic_float    | 3       | +28            | +33             | 15.6                         | -   |
| SIMD                | 1       | +12            | +13             | 6                            | 2.9 |
| SIMD                | 3       | +17            | +20             | 10.3                         | -   |
| SIMD                | 4       | +20            | +22             | 11.6                         | -   |
| SIMD                | 5       | +9             | +10             | 8.8                          | -   |

The full logs can be found [here](https://github.com/hgminh95/heat/tree/main/logs).

Some notes:

- You can see that not all the methods/# cores combination are listed here. It is mostly because I am lazy to re-run the experiment due to mistake on initial run, e.g. the initial temperature is too high.
- The initial temperature does not have a lot of affect on temperature changes as long as they are < 40 degree celcius. In most of the test, they are usually 35 degree.
- I can achieve highest IPC with integer operation, and unsurprisingly, we generate the most heat with it.
- The result with SIMD will look weird if I use more than 4 cores (which is the number of P-cores in my laptop). Not really sure why, but my guess is probably frequency slow down if we run too many simd operations at a time?
- What is the best one depends on various factors, e.g. how warm you want it to be, how much battery you have left, etc. But in term of heat/power ratio, SIMD seems to be the winner here.

In conclusion, staying warm during a flight when conserving your laptop's battery power is indeed possible. Just remember to keep an eyes on your laptop to avoid overheating issues. Safe travel and happy computing!
