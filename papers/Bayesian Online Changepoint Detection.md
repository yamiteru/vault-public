---
link: https://arxiv.org/pdf/0710.3742v1.pdf
---

# TLDR;

Bayesian online changepoint detection uses a recursive message-passing algorithm to estimate the probability distribution over the current "run length", or time since the last changepoint, given the observed data. This algorithm focuses on predicting the next unseen datum in a sequence, based on past data, and is designed for real-time applications like modeling and prediction in finance, biometrics, and robotics.

---

Changepoints are abrupt variations in the generative parameters of a data sequence.

Online detection of changepoints is useful in modelling and prediction of time series in application areas such as finance, biometrics, and robotics.
