---
link: https://link.springer.com/article/10.1007/s10664-021-10037-x
---

# TLDR;

The best test case prioritization (TCP) techniques achieve mean APFD-P values between 0.54 and 0.71 and mean Top-3 values between 29% and 66%, depending on the project. Techniques using the total strategy outperform the ones with the additional strategy, and dynamic-coverage enables more effective techniques compared to static-coverage.

The total strategy is a test case prioritization technique that ranks benchmarks based on their total coverage. This strategy has been found to outperform the additional strategy in terms of effectiveness in prioritizing benchmark executions.

---

we empirically study traditional coverage-based prioritization techniques along multiple dimensions: 
1) greedy prioritization strategies that rank benchmarks either by their total coverage or additional coverage that is not covered by already ranked benchmarks
2) benchmark granularity on either method or parameter level
3) coverage information with method granularity extracted either dynamically or statically
4) different coverage-type-specific parameterizations

In addition, our TCP techniques also consider the code change between the old and the new version. They perform the ranking based on the coverage information and the strategy and then split the ranked benchmarks into two sets, i.e., the ones that are affected and the ones that are unaffected by the code change. The affected ones are executed before the unaffected ones according to the ranking.

If the analysis time is too expensive, the benefits of earlier performance change detection might not outweigh the temporal overhead compared to just running the benchmarks in random order.

We have found that the most effective techniques in terms of APFD-P and Top-3 use dynamic-coverage. The best dynamic-coverage technique uses the total strategy, benchmark-parameter, and dc-benchm and has an analysis time overhead of approximately 11%.

However, if an 11% overhead is too expensive, a technique relying on static-coverage might be an attractive alternative. The most effective static-coverage technique, for both APFD-P and Top-3, in our study uses the total strategy, benchmark-parameter, sc-algoRTA, sc-roMAX, and sc-eps. This technique is also efficient with a mean analysis overhead of below 3%.

It is important to keep in mind that TCP can be less effective than a random ordering, depending on the project and the parameterization of the technique. However, on average across all studied projects all techniques are superior to random.

Practitioners who are keen on applying TCP for their microbenchmark suites should carefully evaluate whether they would benefit from it, by answering the following questions:

1) Is the suite runtime too long to wait for its completion, and can we, therefore, benefit from prioritization?
2) Which analysis overhead is acceptable (in relation to the suite runtime)?
3) Which technique is effective and efficient for our project?

There is a conceptual difference between dynamic TCP for unit tests and dynamic TCP for benchmarks: coverage information (for unit tests) is usually acquired during the test executions of the previous version.

TCP is considerably less effective for benchmarks than for unit tests, if we assume that values for APFD and APFD-P are comparable.
