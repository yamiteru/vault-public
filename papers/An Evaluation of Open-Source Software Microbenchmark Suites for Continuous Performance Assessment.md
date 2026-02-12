---
link: https://www.zora.uzh.ch/id/eprint/159079/1/msr18-author-version.pdf
---

# TLDR;

1. Assess Benchmark Suite Quality: Use the proposed methodology in the paper to assess the quality of the microbenchmark suite, especially focusing on variability and stability of the results.

2. Select Tests for Continuous Integration: Utilize the ABS metric to select a subset of tests to be run as part of continuous integration, ensuring quick performance feedback to developers.

3. Generate Additional Benchmarks: Identify currently untested parts of the API and use the information to generate new benchmark stubs to improve performance tests.

---

Continuous integration (CI) emphasizes quick feedback to developers. This is at odds with current practice of performance testing, which predominantely focuses on long-running tests against entire systems in production-like environments. Alternatively, software microbenchmarking attempts to establish a performance baseline for small code fragments in short time.

Improving the build time, i.e., the time it takes for a CI server to check out, compile, and test the build, is a constant concern for CI and CD projects, and the foundation of continuous performance assessment.

This is primarily for two reasons: 

1) Quick builds allow for fast feedback to the developer, reducing the time that is “wasted” waiting for builds that ultimately fail.
2) State of practice CI platforms restrict builds to a certain maximum runtime. A prominent example is TravisCI, which at the time of writing employs a build job timeout of maximum 120 minutes5. Hence, we first study how large (in number of benchmarks) the microbenchmark suites of our study subjects are, and how long they take to execute.

both Go and Java projects produce more consistent benchmarking results on bare-metal than in in the cloud. This is due to the performance instability of public cloud instances themselves, as widely reported in literature.
