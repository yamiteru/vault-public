---
link: https://arxiv.org/pdf/2001.05811.pdf
---

# TLDR;

Duet benchmarking is a measurement procedure introduced to improve accuracy in shared resource environments, such as virtual machine instances in the cloud. It involves executing the measured artifacts in parallel to maximize the likelihood of equal impact from performance fluctuations, and then filtering out the fluctuations by considering the relative performance of the measured artifacts together. 

This procedure is aimed at isolating synchronized interference and addressing the challenges of performance testing in cloud environments where resources are shared with other users.

The potential risks of duet benchmarking, compared to executing a benchmark only once in a single thread, are the introduction of competition on resources between the paired workloads, and uneven resource utilization patterns. 

However, the research suggests that these effects are either negligible or bounded and do not prevent the detection of performance regressions.

---

At the heart of various performance comparison activities is a measurement experiment, whose statistical nature involves an inherent trade off between execution time and sensitivity to differences in performance.

Longer experiment times average over noise in the measurement data and provide more accurate results, but are also expensive both in terms of time and computing resources. Conversely, shorter execution times may cause the loss of sensitivity or report false alarms. This is a problem when automating performance test execution and evaluation.

A specific hurdle in this context is the fact that the cloud does not necessarily provide the performance stability required for performance testing. Performance measurements in the cloud are noisy, in part due to lack of control over hardware configuration, in part due to overhead of virtualization, but most importantly due to interference from colocated workloads of other tenants.

Essential to performance regression testing is robust performance change detection.

Cloud providers offer abstract virtual machine types that can run on different types of physical hosts, resulting in different execution times even for the same code.

The work of Abedi and Brecht shows how the ordering of trials can impact the experiment conclusions. Utilizing A/A testing, the authors show that possible regularity in performance interference can be incorrectly interpreted as actual difference in performance between alternatives. Randomized ordering of trials is proposed as a remedy. Our duet measurements similarly randomize the assignment of workloads to processors.

Some authors propose mechanisms that help detect the presence of performance interference. Joshi et al. measure an application in controlled conditions, constructing a throughput-vs-utilization curve. In real deployment, significant departure from that curve is interpreted as a sign of performance interference.

## Duet Measurement Procedure

To account for the probabilistic nature of the interference, we have to repeat the measured operation enough times to obtain a representative sample of measurements, and then calculate confidence intervals for any values derived from the measurements.

The two workloads are executed in parallel, inside a virtual machine with two virtual cores, with each workload restricted to one virtual core.

The workloads are synchronized using a shared memory barrier, so that their measured operations always start at the same time.

This setting ensures that any external interference on the virtual machine impacts both workloads simultaneously, which equalizes the probability of interference between the workloads for each paired measurement and thus avoids the bias immediately—rather than only for a long enough experiment.
