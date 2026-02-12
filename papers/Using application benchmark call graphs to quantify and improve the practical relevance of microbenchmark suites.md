---
link: https://pdfs.semanticscholar.org/e045/cbb0a0f5470b3377b35157b54afb42552e45.pdf
---

# TLDR;

The algorithm for removing redundancies in the microbenchmark suite works by creating a minimal subset of microbenchmarks that structurally covers the application benchmark graph to the same extent. This approach significantly reduces the overall runtime of the microbenchmark suite, with the potential to reduce the number of microbenchmarks by 77% to 90%, depending on the application and benchmark scenario.

---

Thus, it is often reasonable to use one or more application benchmarks as the next accurate proxy to simulate and evaluate a representative artificial production system. The execution of these benchmarks for each code change is very expensive and time-consuming, but a light-weight microbenchmark suite that has proven to be practically relevant could replace them to some degree.

We analyze the called functions of a reference run, which can be (an excerpt from) a production system or an application benchmark, and compare them with the functions invoked by microbenchmarks to determine and quantify a microbenchmark suite’s practical relevance. If every called function of the reference run is also invoked by at least one microbenchmark, we consider the respective microbenchmark suite as 100% practically relevant as the suite covers all functions used in the baseline execution.

If there are many microbenchmarks in a suite, they are likely to have redundancies and some functions will be benchmarked by multiple microbenchmarks. By creating a new subset of the respective microbenchmark suite without these redundancies, it is possible to achieve the same coverage level with fewer microbenchmarks, which significantly reduces the overall runtime of the microbenchmark suite.
