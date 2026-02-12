---
link: https://wwang.github.io/papers/PT4Cloud.pdf
---

# TLDR;

The key challenges and factors contributing to the difficulty of obtaining accurate performance testing results on cloud applications as highlighted in the paper include:

1. **Performance Uncertainty**: Considerable and unpredictable performance fluctuation of cloud applications due to factors such as hardware resource contentions and VM scheduling.
2. **Black-Box Environment**: Cloud services are provided as black boxes, making it difficult to know which uncertainty factors are covered by a test run and when enough runs have been conducted.
3. **Testing Cost**: The high cost of executing tests for extended periods to produce accurate results, while minimizing testing costs.

---

To fully realize the cost benefits of cloud services, users usually need to reliably know the execution performance of their applications. However, due to the random performance fluctuations experienced by cloud applications, the black box nature of public clouds and the cloud usage costs, testing on clouds to acquire accurate performance results is extremely difficult.

In this paper, we present a novel cloud performance testing methodology called PT4Cloud. By employing non-parametric statistical approaches of likelihood theory and the bootstrap method, PT4Cloud provides reliable stop conditions to obtain highly accurate performance distributions with confidence bands.

To obtain accurate results, performance testing on a cloud should cover all the uncertainty factors. Even more importantly, it should cover the uneven impacts of all these factors proportional to their frequencies experienced on the clouds.

While ensuring high accuracy, our second goal is to reduce the number of test runs to cut down on the cost of performance testing.

A statistical approach that is proper for cloud performance testing should satisfy three requirements. First, the approach should be able to handle non-typical distributions, as performance distributions of cloud applications usually do not follow known distributions. Second, this approach should be able to handle distributions acquired from experiments, as it is practically impossible to make a hypothesis about the exact theoretical distribution for a cloud application’s performance. Third, the comparison result from this approach should be intuitive so that the ordinary user can understand.

The distribution comparison approach that we eventually adopted is Kullback-Leibler (KL) divergence, which can handle any types of distributions. More specifically, KL-divergence measures how one distribution diverges from a second distribution.

Because the performance distributions of cloud applications do not necessarily follow known distributions, we chose to compute point-wise confidence bands (CB) using bootstrap. Point-wise confidence bands are commonly used to describe non-parametric distributions, and bootstrap is a statistical method for treating non-parametric distributions.
