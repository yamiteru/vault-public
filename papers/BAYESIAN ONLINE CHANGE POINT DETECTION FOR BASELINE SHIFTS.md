---
link: https://arxiv.org/pdf/2201.02325.pdf
---

# TLDR;

The extension of the original BOCPD algorithm, BOCPD-BLS, includes a mechanism that uses feedback on detected change points to guide subsequent detection attempts. This allows the algorithm to reinitialize the underlying baseline distribution after each change point, enabling it to maintain high detection sensitivity regardless of the offset of the data values compared to the original baseline.

---

Online change point detection refers to the real-time detection of an acute change in a sequential time series data set.

According to a comprehensive study of the categorization of change point detection algorithms, the keys to useful online change point detection algorithms can be summarized as follows: 1) minimal delay, 2) minimal false positives, 3) computational efficiency, and 4) robustness.

If there is no guarantee that the long-term expectation of a time series is approximately constant nor that an anomalous state will regress to a fixed known baseline value, the original BOCPD algorithm struggles to maintain its change point detection sensitivity.
