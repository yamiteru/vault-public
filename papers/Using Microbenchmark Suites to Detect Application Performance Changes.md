---
link: https://arxiv.org/pdf/2212.09515.pdf
---

# TLDR;

- Performance issues in software systems should be identified early.
- Benchmarking is essential for validating performance properties.
- Using application benchmarks for large projects is costly and time-consuming.
- Optimized microbenchmark suites can be a faster alternative for detecting performance changes in certain situations.
- A combination of optimized microbenchmark suites and well-designed application benchmarks is recommended for a continuous benchmarking strategy.

The execution time of micro-benchmarks can be reduced by:

1. Prioritizing test cases through techniques such as test case prioritization.
2. Selecting micro-benchmarks based on code change indicators and information from prior benchmark runs.
3. Stopping micro-benchmark runs when the system under test is stable and is unlikely to produce different results with more load or repetitions.

---

It is, however, unclear whether microbenchmarks and application benchmarks detect the same performance problems and one can be a proxy for the other.

Our results show that it is possible to detect application performance changes using an optimized microbenchmark suite if frequent false-positive alarms can be tolerated.

Microbenchmarks are unable to reliably detect all problems, as they do not take the integration of the respective functions or modules into the overall (production) system into account. For a single (standalone) component, however, they might be used as a proxy for a complex application benchmark.

Application call graphs represent the individual methods and functions of an application as nodes and their respective calls to each other as edges. For this study, we use an approach that removes redundancies in a microbenchmark suite based on an application benchmark call graph and includes only practically relevant microbenchmarks, i.e., microbenchmarks evaluating functions that are actually used in production.

A wide CI of an experiment implies that the concrete performance change of a (micro-)benchmark cannot be clearly quantified and that the individual benchmark is unstable. Thus, we refer to the width of a confidence interval as instability. The smaller this instability is, the better and more precisely it is possible to detect performance changes. On the other hand, a wide CI implies that it is only possible to detect large performance changes, because overlapping CIs of the respective experiments do not allow us to draw precise conclusions. Thus, we classify the detected performance changes as (99%) definite (no overlap) and potential (overlapping CIs) performance changes.

Moreover, because we do not consider small performance changes as relevant in our evaluated projects, we also set a minimum threshold of 1%, similar to what best practice suggests.

Running optimized microbenchmark suites using bootstrapping analysis and dynamic thresholds will detect multiple potential and definite performance changes for several microbenchmarks. These results, however, cannot be directly linked to a request type in the application benchmark or allow other direct conclusions. For example, if there is a definite performance drop of 5% in a microbenchmark, this drop cannot be directly linked to application performance (e.g., slower queries). To link a respective microbenchmark that detected a performance change to the application benchmark, we therefore use a reference impact value which is the sum of the execution durations of the overlapping functions in the reference application benchmark (see Figure 1). The key idea behind this is that a microbenchmark whose covered functions in the application benchmark have a smaller total execution duration (e.g., 10 for MB1) will have less impact on overall application performance than another microbenchmark covering functions with a larger total application benchmark execution duration (e.g., 15 for MB3). We call this metric reference impact because it refers to the recorded application benchmark call graph and not to the performed microbenchmark experiment.

A strict policy that, for example, rejects a commit when a performance issue is detected by a microbenchmark would in many cases be incorrect.
