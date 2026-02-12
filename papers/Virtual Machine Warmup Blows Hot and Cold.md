---
link: https://dl.acm.org/doi/pdf/10.1145/3133876
---

# TLDR;

The traditional view of JIT compiled VMs is that program execution is slow during the warmup phase, and fast afterwards, when a steady state of peak performance is reached. This traditional view underlies nearly all JIT compiler benchmarking methodologies, with reported measurements focusing only on peak performance.

The paper uses changepoint analysis to automatically analyze VM benchmarking data and classify each process execution as either: no steady state, flat (no detectable change in performance over the benchmark), warmup, or slowdown. Changepoint analysis detects shifts in the nature of time series data, such as when a benchmark switches from 'slow' to 'fast' modes after JIT compilation, allowing for the automatic identification and classification of different performance patterns.

---

Virtual Machines (VMs) with Just-In-Time (JIT) compilers are traditionally thought to execute programs in two phases: the initial warmup phase determines which parts of a program would most benefit from dynamic compilation, before JIT compiling those parts into machine code; subsequently the program is said to be at a steady state of peak performance.

Measurement methodologies almost always discard data collected during the warmup phase such that reported measurements focus entirely on peak performance.

## H1: Small, deterministic programs reach a steady state of peak performance

Of the seven VMs we studied, none consistently reached a steady state of peak performance. These results are much worse than reported in previous works.

Our results suggest that much real-world VM benchmarking, which nearly all relies on assuming that benchmarks do reach a steady state of peak performance, is likely to be partly or wholly misleading.

## Background

Benchmarking of JIT compiled VMs typically focusses on reporting steady state numbers based on two assumptions: first, that warmup is both fast and inconsequential to users; second, that the steady state is also peak performance.

## Ensuring determinism

User programs that are deliberately non-deterministic are unlikely to warm-up in the traditional fashion. We therefore wish to guarantee that our benchmarks are, to the extent controllable by the user, deterministic, taking the same path through the Control Flow Graph (CFG) on all process executions and in-process iterations.

## Measuring Computation Rather than File Performance

By their very nature, microbenchmarks tend to perform computations which can be easily opti- mised away. While this speaks well of optimising compilers, benchmarks whose computations are entirely removed are rarely useful

## H2: Moderately different hardware and operating systems have little effect on warmup

Turbo boost allows CPUs to temporarily run in an higher-performance mode; if the CPU deems it ineffective, or if its safe limits (e.g. temperature) are exceeded, turbo boost is reduced.

Turbo boost can thus substantially change one’s perception of performance. Hyper-threading gives the illusion that a single physical core is two logical cores. Programs and threads that would otherwise run on separate physical cores are thus inter-leaved on a single core, leading to less predictable performance.

Modern systems have various temperature-based limiters built in: CPUs, for example, lower their frequency if they get too hot.

## Outliers

Defining an outlier as one that, within a sliding window of 200 in-process iterations, lies outside the median ±3 × (90%ile − 10%ile). In order that we avoid classifying slow warmup iterations at the start of an execution as outliers (when they are in fact likely to be important warmup data), we ignore the first 200 in-process iterations.

## H3: Non-warmup process executions are largely due to JIT compilation or GC events

The relatively few results we have with GC and JIT compilation events, and the lack of a clear message from them, means that we feel unable to validate or invalidate Hypothesis H3.
