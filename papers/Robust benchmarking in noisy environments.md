---
link: https://arxiv.org/pdf/1608.04295.pdf
---

# TLDR;

To mitigate noise in benchmarks, the paper suggests several strategies:

1. Designing a language- and platform-agnostic benchmarking methodology suitable for continuous integration (CI) pipelines and manual user workflows.
2. Developing a robust benchmarking strategy that accounts for timer error, OS jitter, and other environmental fluctuations.
3. Constructing a model to explain non-ideal statistics arising from environmental fluctuations and justifying the proposed benchmarking strategy.
4. Implementing the benchmarking strategy in a specific software package (BenchmarkTools Julia package) for use in production continuous integration pipelines for the Julia language and its ecosystem.

---

Unimodal = data visually forms single line
Bimodal = data visually forms two lines

Authors of high performance applications rely on benchmark suites to detect and avoid program regressions. However, many developers often run benchmarks and interpret their results in an ad hoc manner with little statistical rigor. This ad hoc interpretation wastes development time and can lead to misguided decisions that worsen performance.

Consecutive timing measurements can fluctuate, possibly in a correlated manner, in ways which depend on a myriad of factors such as environment temperature, workload, power availability, and network traffic, and operating system (OS) configuration.

Many factors stem from OS behavior, including CPU frequency scaling, address space layout randomization (ASLR), virtual memory management, differences between CPU privilege levels , context switches due to interrupt handling, activity from system daemons and cluster managers, and suboptimal process and thread-level scheduling. 

Even seemingly irrelevant configuration parameters like the size of the OS environment can confound experimental reproducibility by altering the alignment of data in memory.

Linkers for many languages are free to choose the binary layout of the library or executable arbitrarily, resulting in non-deterministic memory layouts. This problem is exacerbated in languages like C++, whose compilers introduce arbitrary name mangling of symbols. Overzealous compiler optimizations can also adversely affect the accuracy of hardware counters, or in extreme cases eliminate key parts of the benchmark as dead code. Yet another example is garbage collector performance, which is influenced from system parameters such as heap size.

Defining a serious regression as a 30% or greater increase in a benchmark’s minimum execution time.

## Repeated benchmark execution is often necessary but not always sufficient

One cannot always reliably eliminate bias due to external variations simply by executing the benchmark many times.

## Statistics of timing measurements are not i.i.d.

The existence of many sources of performance variation result in timing measurements that are not necessarily independent and identically distributed (i.i.d.).

As a result, many textbook statistical approaches fail due to reliance on the central limit theorem, which does not generally hold in the non-i.i.d. regime.

## Justifying the minimum estimator

Choosing the smallest timing measurement will choose the sample with the smallest magnitude of error.

The distribution of the minimum across all experimental trials is unimodal in all cases we have observed.

Thus for our purposes, the minimum is a unimodal, robust estimator for the location parameter of a given benchmark’s timing distribution.

## Conclusion

Virtually all timing variations are delays caused by flushing cache lines, task switching to background OS processes, or similar events.

Simply running a benchmark for longer, or repeating its execution many times, can render the effects of external variation negligible, even as the error due to timer inaccuracy is amortized.
