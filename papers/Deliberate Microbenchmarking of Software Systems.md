---
link: https://web.archive.org/web/20210517195601id_/https://www.zora.uzh.ch/id/eprint/203225/1/cl-phd-thesis-final.pdf
---

# TLDR;

To decrease benchmark cost, one approach is to dynamically stop software microbenchmark executions when their results are sufficiently stable. This can be achieved with statistical stoppage criteria, which can reduce benchmark suite execution times by 48.4% to 86.0% while retaining the same result quality for 78.8% to 87.6% of the benchmarks, compared to executing the suite for the default duration. This method provides an automated mechanism for dynamic reconfiguration, making it highly effective and efficient in decreasing benchmark cost.

The biggest challenges of reliable benchmarks are: 

1. Long benchmark suite runtimes, which are often longer than 3 hours in real-world software microbenchmark executions.
2. Extensive result variability, reaching up to 100% in some cases, impacting the consistency and reliability of benchmark results.
3. Difficulty in reliably detecting small performance changes, as benchmarks often only reliably detect large performance changes of 60% or more.

To reduce variability in benchmark results, best practices include repeating measurements at different levels to address measurement uncertainty, and randomizing factors that influence the measurements. Novel techniques, such as randomly interleaving benchmarks, executing benchmarks in parallel on different CPUs of the same machine, and utilizing source code features as proxies for performance variability, can also help in reducing variability.
