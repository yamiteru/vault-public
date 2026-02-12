---
link: https://icl.utk.edu/files/publications/2019/icl-utk-1182-2019.pdf
---

# TLDR;

Dynamic reconfiguration is an alternative approach to static benchmark configuration in Java microbenchmarking. It leverages stability criteria to automatically determine the end of the warmup phase at runtime, potentially reducing execution time with low impact on results quality. It employs techniques such as coefficient of variation (CV), relative confidence interval width (RCIW), and Kullback-Leibler divergence (KLD) to dynamically estimate the end of the warmup phase and improve warmup estimation accuracy.

---

The Continuous Integration (CI) framework of Ginkgo is also extended to automatically run a benchmark suite on predetermined HPC systems, store the state of the machine and the environment along with the compiled binaries, and collect results in a publicly accessible performance data repository based on Git.

The motivation for such an automated / continuous benchmarking system is:

1. Reproducibility. Reproducibility is a central pillar of natural science that is often left out in high-performance benchmark studies. An automated performance evaluation system archiving all performance data along with execution parameters, plot scripts, machine settings and produced binary code would enable us to reproduce benchmark results at any point in the future.
2. Consistency. Archiving performance results over the lifetime of a software library can also detect performance degradations and track down the modifications that triggered a performance loss. Coupling this with the Git Workflow and applying continuous benchmarking to all merged requests allows us to tie every code change and feature addition to performance results. This is an important consideration in the development process of high-performance software.
3. Usability. An automated performance evaluation workflow allows launching benchmark tests with a few clicks in the web interface.
