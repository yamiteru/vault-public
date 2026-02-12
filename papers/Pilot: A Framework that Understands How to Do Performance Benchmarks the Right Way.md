---
link: https://escholarship.org/content/qt3pk548f5/qt3pk548f5.pdf?t=qaxqxc
---

# TLDR;

Pilot is a software framework designed to automate the process of performance benchmarking. It uses analytical methods and heuristic algorithms to improve statistical validity in a short time. Pilot includes methods to measure and correct sample autocorrelation, estimate optimal test duration, detect warm-up and cool-down phases, and identify shifts in sample means. Additionally, it offers two new algorithms: a linear performance model and a method to detect and measure system performance bottlenecks while managing overhead. Pilot automates performance evaluation tasks based on real-time time series analytical results and is implemented in C++. It aims to provide reliable and statistically valid results, particularly for those with limited statistics knowledge.

---

Carrying out even the simplest performance bench- mark requires considerable knowledge of statistics and computer systems, and painstakingly following many error-prone steps, which are distinct skill sets yet essential for getting statistically valid results. As a result, many performance measurements in peer-reviewed publications are flawed. Among many problems, they fall short in one or more of the following requirements: accuracy, precision, comparability, repeatability, and control of overhead. This is a serious problem because poor performance measurements misguide system design and optimization. We pro- pose a collection of algorithms and heuristics to automate these steps. They cover the collection, storing, analysis, and comparison of performance measurements. We also implement these methods as a readily-usable open source software framework called Pilot, which can help to reduce human error and shorten benchmark time. Evaluation of Pilot with various benchmarks show that it can reduce the cost and complexity of running benchmarks, and can produce better measurement results.

Hoefle and Belli analyzed 95 papers from HPC-related conferences and discovered that most papers are flawed in experimental design or analyses.

Basically, if any published number is derived from fewer than 20 samples, or is presented as a mean without variance or confidence interval (CI), the authors are likely doing it wrong.

Performance measurement is concerned with how to measure the performance of a running program or system, while bench- marking is more about issuing a certain workload on the system in order to measure the performance.

People usually do not allocate a lot of time to performance evaluation or tuning.

## Statistical concepts

- Accuracy: reflects whether the results actually measure what the user wants to measure. A benchmark usually involves many components of the system. When we need to measure a certain property of the system, such as I/O bandwidth, the benchmark needs to be designed in such a way that no other components, such as CPU or RAM, are limiting the measured performance. This requires carefully designing the experiment, measuring the usage of related components while the benchmark is running, and checking which component is limiting the overall performance.
- Precision: is related to accuracy but is a different concept. Precision is the difference between the measured value and the real value the user needs to measure. In statistical terms, precision is the difference between a sample parameter and its corresponding population parameter. Precision can be described by confidence interval (CI). The CI of a sample parameter describes the range of possible population parameter at certain likelihood. For instance, if the CI of a throughput mean (μ) is C at the 95% confidence level, we know that there is a 95% chance that the real system’s throughput is within interval [ μ − C , μ + C ]. In practice, CIs are typically stated at 22 the 90% or 95% confidence level. We can see that the common practice of presenting a certain performance parameter using only one number, such as saying the write performance of a disk drive is 100 MB/s, is misleading.
- Repeatability: is critical to a valid performance mea- surement because the goal of most performance benchmark is to predict the performance of future workloads, which exactly means that we want the measurement results to be repeatable. In addition to the accuracy and precision, errors in the measurement can have a negative impact on repeatability. There are two kinds of errors: the systematic error and random error. Systematic error means that the user is not measuring what he or she plans to measure. For instance, the method of benchmarking is wrong or the system has some hidden bottleneck that prevents the property to be measured from reaching its maximum value. In these cases, even though the results may look correct, it may well be not repeatable in another environment. Random errors are affected by “noise” outside our control, and can result in non-repeatable measurements if the sample size is not large enough or samples are not independent and identically distributed.

## Terms

- Performance index (PI) is one property we want to measure. For instance, the throughput of a storage device is a PI, and its latency is another PI.
- Session is the context for doing one measurement. We can measure multiple PIs in one session. One session can include multiple rounds of benchmarks, and each round can have a different length.
- Work amount is the amount of work involved in one round of benchmark. The work amount is related to the length of the workload. For instance, in a sequential write I/O workload round, we write 500 MB data using 1 MB writes, the work amount of this round is 500.
- Work unit is a smallest unit of work amount that we can get a measurement from. Using the above sample, if the I/O size is 1 MB, we can measure the time of each I/O syscall and calculate the throughput of each I/O operation. Here the work unit is 1MB, and we have 500 work units in that workload round. Not all workloads should be divided into units. Pilot expects the work unit to be reasonably homogeneous. So, for instance, reading one 1 MB from different locations of a device can be thought as homogeneous because the difference in performance is small and mostly normally distributed. But shifting from sequential I/O to random is not homogeneous because that would result in huge difference in I/O performance. In general, the user should only divide the workload into units when one expects them to have similar performance. If not the user should not use the work unit-related analytical methods of Pilot and should stick with readings (see next term) only. We leave heterogeneous work units as a future work.
- Reading is a measurement of a PI of a round. Each benchmark round generates one reading for each PI at the end of the round. In the sample above, when PI is the throughput of the device, we can get one throughput reading for each round.
- Unit reading (UR) is a measurement of a PI of a work unit. In the sample above, we would have 500 throughput Unit readings for Round 1 because it contains 500 work units, and these 500 unit readings would be the throughput of each 1 MB I/O operation.
- Work-per-second (WPS) is the calculated speed at which the workload consumes work. This is usually the desired PI for many simple workloads.

## Warm-up and cool-down

Performance results are often used to predict the run time of future workloads, and it is a common practice to use one number to express the performance of a PI. For example, people usually say “the write throughput of this device is X”. Using only one number assumes that the device’s performance follows a linear model. Linear models (work amount = duration × speed) are simple, but using only one number can only state the device’s stable performance and is not adequate when the performance of the PI can be significantly affected by a long warm-up or cool-down phase.

## Change-point detection

Multiple change-point detection is a challenging research question, especially when we cannot make any assumption about the distribution of the error or the underlying process. The method we use also has to be fast to calculate and should support online update.

After evaluating many change-point detection methods, we found that the E-Divisive with Medians (EDM), which is a new method published by Matteson and James in 2014, best fits our requirements. EDM is non-parametric (works on mean and variance) and robust (performs well for data drawn from a wide range of distributions, especially non-normal distributions). EDM’s initial calculation is O(n log n) and can do update in O(log n) time.

EDM outputs a list of all the change-points in the time series. It is common to see many change-points at the start and end of the workload. These change-points divide the test data into multiple segments. Pilot uses a heuristic method to determine which segment is the stable segment: it has to be the longest segment and also dominate the test data (containing more than 50% of the samples). This method can effectively remove any number of non-stable phases at the beginning and the end.

## Optimal session length

We treat each unit reading as one measurement. One workload round can usually provide hundreds or thousands of measurements, making it faster to reach the required sample size. For instance, if a sequential write workload writes 500 MB data using 1 MB I/O, we can get 500 throughput measurements if the workload saves the duration of each write. Pilot sends the unit reading results through the non-stable phases removal, performs auto- correlation reduction, and uses the rest of the unit readings to calculate the CI. Pilot keeps running new rounds of the workload until the desired width of the CI is reached

## Comparing benchmarks

We use the Welch’s unequal variance t-test (an adaptation of Student’s t-test ) to compare the benchmark results, A and B. Welch’s t-test is more reliable when the two samples have unequal variance and unequal sample sizes, which are true for most system benchmarks.

This test can effectively tell us the probability of rejecting a hypothesis and the required sample size. Here the null hypothesis (the hypothesis we want to reject) is that there is no statistical significant difference between result A and B (A = B). We compute the probability (p-value) of getting results A and B if the null hypothesis is true.

The comparing result algorithm runs until:

1) There are enough data for calculating the CIs,
2) Each adjacent CI pair is either non-overlap or their p-value
of the null hypothesis (A = B) is less than the predefined
threshold (usually 0.01),
3) Every CI is tighter than the required width (this step is
optional but recommended because a narrower CI makes
it easier to compare with new results in the future).

## Problems

- Imprecise: the result may not be a good approximation of the “real” value; often caused by failing to consider the width of CI and not collecting enough samples
- Inaccurate: the result may not reflect what you need to measure; often caused by hidden bottleneck in the system
- Ignoring the overhead: not measuring or documenting the measurement and instrumentation overhead
- Presenting improvements or comparisons without provid- ing the p-value, making it impossible to know how reliable the improvements are

Benchmarks must meet several requirements to be useful. The measurement results must reflect the performance property one plans to measure (accuracy). The measurement must be a good approximation of the real value with quantified error (precision). Multiple runs of the same benchmark under the same condition should generate reasonably similar results (repeatability). The overhead must be measured and documented. If the results are meant for publication, they also must include enough hardware and software information so that other people can compare the results from a similar environment (comparability) or replicate the result (replicability).
