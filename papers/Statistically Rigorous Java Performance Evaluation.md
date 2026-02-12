---
link: https://dri.es/files/oopsla07-georges.pdf
---

# TLDR;

Benchmarks for managed runtime systems can have high variance and instability due to non-deterministic factors such as Just-In-Time (JIT) compilation, thread scheduling, garbage collection, and system effects like interrupts. These factors can result in varying execution times and interactions between threads, affecting overall performance.

- 16 out of 50 papers surveyed did not specify the methodology used for their performance evaluation, making it difficult to reproduce and interpret results.
- Approximately one third of the research papers (16 out of 50) did not specify the methodology used for their performance evaluation.
- From 2000 to 2003, many papers did not specify or only partially specified their performance evaluation methodology, but more recent papers tend to have more detailed descriptions.
- Best and average approaches are the most prevalent in reporting performance numbers, with confidence intervals being reported in only a small minority of research papers (4 out of 50).

To make performance evaluation statistically rigorous, the paper advocates using statistics theory to compute confidence intervals and compare multiple alternatives. Additionally, the authors provide methodologies for quantifying startup and steady-state performance, recommend taking multiple measurements and using JavaStats software to monitor variability and collect information for statistically rigorous analysis.

---

Java performance is far from being trivial to benchmark because it is affected by various factors such as the Java application, its input, the virtual machine, the garbage collector, the heap size, etc.

In addition, non-determinism at run-time causes the execution time of a Java program to differ from run to run. There are a number of sources of non-determinism such as Just-In-Time (JIT) compilation and optimization in the virtual machine (VM) driven by timer- based method sampling, thread scheduling, garbage collection, and various system effects.

We provide the JavaStats software to automatically obtain performance numbers in a rigorous manner.

A non-rigorous methodology may skew the overall picture, and may even lead to incorrect conclusions. And this may drive research and development in a non-productive direction, or may lead to a non-optimal product brought to market.

Managed runtime systems are particularly challenging to benchmark because there are numerous factors affecting overall performance, which is of lesser concern when it comes to benchmarking compiled programming languages such as C.

In only a small minority of the research papers (4 out of 50), confidence intervals are reported to characterize the variability across multiple runs.
