---
link: https://d3s.mff.cuni.cz/files/publications/bulej_initial_2019.pdf
---

# TLDR;

The duet benchmarking method demonstrated significant improvements in measurement accuracy, with an average accuracy improvement ranging from 114% to 683% over standard sequential measurements, depending on the platform used. This indicates that duet benchmarking can be highly effective in improving accuracy in virtualized environments, making performance testing in the cloud more effective.

---

Accurate performance testing may require many measurements and therefore many machines to execute on. When many machines are needed, the cloud offers a tempting solution, however, measurements conducted in the cloud are generally considered unstable.

Depending on the platform used, experiments show average accuracy improvement ranging from 114% to 683%.

Performance testing can also be effectively parallelized, especially if there are multiple tests to be executed with each version, because these can be built and executed independently. However, neither test reduction nor test parallelization entirely removes the problem of infrastructure capacity limits.

Different sources of variability can influence the observed execution times at different granularities, and the performance testing procedure must ensure that significant sources of variability are sufficiently represented in the measured data.

To avoid excess measurement variability, benchmarks are typically executed on dedicated machines configured to disable disruptive features such as advanced power management.

With the randomized interleaving of workloads, external performance interference impacts the measured workloads independently and individually. With a long enough execution, the interference should eventually impact all the measured workloads similarly, avoiding possible bias but still increasing variance across measurements.

With duet measurements, both measured workloads execute in parallel in one virtual machine, with synchronized benchmark runs and task iterations. Any external performance interference that impacts the virtual machine as a whole is therefore encountered simultaneously in both workloads.

Where the randomized interleaving of workloads eliminates systematic bias only across long enough execution (such that any external performance interference has enough opportunities to hit all workloads equally), the duet measurements prevent such bias already with individual measurements.

Additionally, the duet measurement procedure naturally provides paired measurements, that is, measurements collected on the two workloads at the same time. For some types of external performance interference – such as linear slowdown due to resource contention – this may help separate the variance due to the interference from the variance inherent to the measured workloads and therefore further improve accuracy.

If a cloud platform were to exhibit systematic performance difference between the two virtual cores with duet measurements, it would likely exhibit similarly unwarranted performance difference in many common concurrent workloads. Such behavior would be considered a bug and likely remedied.

We have seen that when the virtual cores provided by the platform are served by two hardware threads of the same host processor core, the synchronized performance interference that the duet measurements target is more likely to occur than in other configurations.

The duet measurement procedure requires concurrent execution of the pair workloads.
