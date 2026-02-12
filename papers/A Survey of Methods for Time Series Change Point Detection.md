---
link: https://eecs.wsu.edu/~cook/pubs/kais16.2.pdf
---

# TLDR;

The "Inertial Hidden Markov Models" algorithm is often considered the best non-parametric online change detection algorithm for detecting changes in software benchmarks. It is often used for modeling change in multivariate time series.

---

Change points are abrupt variations in time series data. Such abrupt changes may represent transitions that occur between states.

Change point detection algorithms are traditionally classified as “online” or “offline”. Offline algorithms consider the entire data set at once, and look back in time to recognize where the change occurred. The goal of this scenario is generally to identify all of a sequence’s change points in batch mode. In contrast, online, or real-time, algorithms run concurrently with the process they are monitoring, processing each data point as it becomes available, with a goal of detecting a change point as soon as possible after it occurs, ideally before the next data point arrives.

Change detection methods need to be designed in a computationally efficient manner so that they can scale to massive data sizes.

Distinguishing between parametric and nonparametric approaches is important because nonparametric approaches have demonstrated greater success for massively large datasets.

A successful algorithm must trade off decision quality for deliberation cost.
