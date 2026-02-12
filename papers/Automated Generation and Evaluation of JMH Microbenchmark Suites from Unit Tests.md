---
link: https://users.encs.concordia.ca/~shang/pubs/MOSTAFA_TSE_2022.pdf
---

# TLDR;

The study reveals that automatically generated microbenchmarks exhibit promising performance in comparison to manually-written benchmarks and that the AutoJMH framework provides an effective means of generating microbenchmarks by simplifying the process for developers.

---

Performance is a crucial non-functional requirement of many software systems. Despite the widespread use of performance testing, developers still struggle to construct and evaluate the quality of performance tests. To address these two major challenges, we implement a framework, dubbed ju2jmh, to automatically generate performance microbenchmarks from JUnit tests and use mutation testing to study the quality of generated microbenchmarks.

Performance microbenchmarking is executed at a similar level of granularity as unit tests, e.g., method level or even a statement level. The goal of performance microbenchmarking frameworks is to detect performance bugs as early as possible by, for example, checking and testing each system build.

The challenges of microbenchmark design and configuration (e.g., microbenchmarks may be “over-optimized” by JIT, resulting in muddled time measures.) hinder developers from effectively exploiting performance microbenchmarks

ju2jmh benchmarks are more effective in detecting performance bugs than JUnit tests and AutoJMH benchmarks. ju2jmh benchmarks cover more mutants than manually-written JMH benchmarks and JMH benchmarks necessitate proper enhancements, such as higher mutant overage, to achieve better results. Furthermore, we discover that different mutation operators can have various effects on benchmarks. In general, ju2jmh benchmarks can detect a higher proportion of mutants exclusively than other tests.

