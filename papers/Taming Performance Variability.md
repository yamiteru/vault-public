---
link: https://www.usenix.org/system/files/osdi18-maricq.pdf
---

# TLDR;

The paper identifies various sources of performance variability, including variability of the same physical system under repeated experiments and variance between supposedly identical physical systems. It specifically mentions variability due to temperature, variations in timings and orderings, remapped storage blocks or memory cells, variance in manufacture, and "fail-slow" hardware as potential causes of variability in hardware performance.

The paper discusses both parametric and nonparametric statistical methods. It emphasizes the use of nonparametric techniques for analyzing performance data, especially in cases where the normality assumption does not hold. Nonparametric statistics like the median and confidence intervals are recommended for comparing performance measurements. Additionally, the paper introduces a new resampling-based technique for estimating the number of experiment repetitions needed for nonparametric models.

The paper emphasizes the importance of running enough repetitions to achieve tight confidence intervals for performance measurements. It specifically mentions the target of achieving a confidence level of 95% with a relative difference of 1%, denoted as Ě(X). Additionally, it highlights the use of a standard score (or z-score) of 1.96 for the commonly-used confidence level of α = 95%.

---

The performance of compute hardware varies: software run repeatedly on the same server (or a different server with supposedly identical parts) can produce performance results that differ with each execution.

Variability is an unavoidable aspect of computer systems performance. In the research community, rigorous com- parison of systems requires understanding, analysis, and control of system variability.

Two types of variability: vari- ability of the same physical system under repeated ex- periments, and variance between different physical sys- tems that are supposedly identical.

Hardware can exhibit variability due to temperature, variations in timings and orderings, remapped storage blocks or memory cells, variance in manufacture, “fail-slow” hardware, and many more causes.

## The Statistics of Performance Variability

The fundamental way variability impacts systems research is that it affects our confidence in the statistical power of our results and the correctness of conclusions that we draw.

For a chosen confidence level α, a CI defines a range in which we are α% sure that the population mean lies. For example, a sample mean of 10.0, with a CI of 9.9 − 10.1 at 95% confidence indicates a 95% confidence that the true mean lies within r = 1% of our estimate 10.0.

Statistical methods fit into two broad classes: para- metric and nonparametric techniques. The former class, which is more well-known, relies on the assumption that the analyzed data stems from known probability distri- butions, typically the Normal/Gaussian distribution.

In contrast, nonparametric techniques are used when the probability distributions are unknown, and require fewer assumptions.

Many studies suggest that the normality assumption does not hold for the data obtained in computer systems experiments, especially when the data includes measurements of performance.


## Unavoidable Variability

Some degree of variation in hardware performance is unavoidable, no matter what steps the facility provider takes to provide consistent hardware. Coefficients of Variance of up to 10% may be attributed to hardware variability and considered expected, while higher values may indicate room for improvement from the measurement standpoint.

Many computer systems performance results have skewed distributions (longer tails on one side); nonparametric confidence intervals are simple to compute, and work for these distributions (as well as normally-distributed results).

## Checking Stationarity

Most statistical tests—including confidence intervals— assume stationarity: that is, that the properties of the underlying distribution (such as median and variance) do not change over time. In addition to affecting data analysis, non-stationary distributions would harm reproducibility: if performance is not stable over time, future experiments cannot reliably be compared to past ones.

## How Many Measurements Are Enough?

given a set of measurements and a desired confidence level (such as 95%), we can compute a confidence interval (CI) for the mean or median. A standard procedure is to “invert” this cal- culation, and for a given desired confidence level and CI width, estimate how many repetitions are likely necessary to achieve the desired confidence.

The higher the performance variance of the underlying hardware, the more repetitions must be run to establish statistical significance; conversely, if not enough repetitions are run, there is a greater chance that the conclusions are incorrect.


