## Consistency

Lots of features in HW and SW are intended to increase performance. But some of them have non-deterministic behavior.

Ideally when doing benchmarking we try to disable all the potential sources of performance non-determinism in a system.

1) Disable turboboost
2) Disable hyper threading
3) Set scaling_governor to ‘performance’
4) Set cpu affinity
5) Set process priority
6) Drop file system cache
7) Disable address space randomization
8) Use statistical methods to process measurements

This is useful and I should take some notes: 
- https://temci.readthedocs.io/en/latest/temci_exec.html#plugins
- https://llvm.org/docs/Benchmarking.html

## Ideal conditions

Microbenchmarking is a form of science. Think about the physicist trying to measure the speed of light in some medium. It usually calls for idealized conditions. Why would you want to use idealized conditions when you know that they will not incur in the normal course of system operations?

1) Idealized conditions makes it easier to reason about the results. You are much more likely to know about the factors that influence your code if you have far fewer factors.
2) Idealized conditions are more consistent and reproducible. Real systems are not stationary. Their performance tends to fluctuate. And to reproduce even your own results can be a challenge when working with real systems. A small idealized scenario is much easier to reproduce.

Conditions:
- As much as possible, all the data is readily available to the processor. We often try to keep the data in cache.
- We focus on single-core performance.
- We avoid comparing functions that process vastly different memory layouts.
- We make sure that the processor runs at a flat clock frequency. In real systems, processors can run slower or faster depending on the system load. As far as I know, it is flat out impossible to know how many CPU cycles were spent on a given task if the CPU frequency varies on Intel processors. Let me quote Wikipedia on time-stamp counters (TSC):
- Constant TSC behavior ensures that the duration of each clock tick is uniform and makes it possible to use of the TSC as a wall clock timer even if the processor core changes frequency. This is the architectural behavior for all later Intel processors.
- Your processor can run faster or slower than the advertized frequency, unless you specifically forbid it.
- If the code performance depends on branching, then we make sure that the branch predictor has had the chance to see enough data to make sound predictions. An even better option is to avoid branching as much as possible (long loops are ok though). The counterpart of this point is that you want to also provide sufficiently large inputs so that branching does not become artificially free. See Benchmarking is hard: processors learn to predict branches.
- When using a programming language like Java, we make sure that the just-in-time compiler has done its work. Ideally, you’d want to avoid the just-in-time compiler entirely because it is typically not deterministic: you cannot count on it to compile the code the same way each time it runs.
- You keep memory allocation (including garbage collection) to a minimum.
- You keep system calls to a minimum.
- You must benchmark something that lasts longer than ~30 cycles, but you can use loops.
- You have to examine the assembly output from your compiler to make sure that your compiler is not defeating your benchmark. Optimizing compilers can do all sorts of surprising things. For example, your compiler could figure out that it does not need to compute the trigonometric functions you are calling very accurately, and so it could fall back on a cheaper approximation. If you can’t examine the assembly output, you should be very careful.
