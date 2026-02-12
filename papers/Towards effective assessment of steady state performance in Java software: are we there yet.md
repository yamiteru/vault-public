---
link: https://link.springer.com/article/10.1007/s10664-022-10247-x
---

A microbenchmark repeatedly executes a small chunk of code while collecting measurements related to its performance.

Due to Java Virtual Machine optimizations, microbenchmarks are usually subject to severe performance fluctuations in the first phase of their execution (also known as warmup).

Developers estimate the end of the warmup phase based on their expertise, and configure their benchmarks accordingly. Unfortunately, this approach is based on two strong assumptions:

1) benchmarks always reach a steady state of performance
2) developers accurately estimate warmup

The Java Virtual Machine (JVM) uses just-in-time compilation to translate “hot” parts of the Java code into efficient machine code at run-time, leading to (often severe) performance fluctuations and potentially unstable results.

To tackle this problem, practitioners rely on the assumption that microbenchmarking is characterized by two distinct phases. During an initial warmup phase, the JVM determines which parts of the software under test would most benefit from dynamic compilation, then, in a subsequent phase, the benchmark reaches a steady state of performance.

Relevant portion of benchmarks never hit the steady state.

We then classify each benchmark as either:

1) steady state, if all the forks reach a steady state of performance
2) no steady state, if none of the forks reach a steady state
3) inconsistent, if the execution involves both steady state and no steady state forks
