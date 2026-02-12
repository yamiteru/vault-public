---
link: https://arxiv.org/pdf/1809.06270.pdf
---

# TLDR;

Nested Performance Profiles are a method for comparing and benchmarking mathematical software solvers. They involve using consecutive performance profiles achieved by eliminating the best solver, which generates nested systems of solvers-problems, and then taking the mean performance over all the reduced systems to give a single graph to rank the solvers. This approach aims to overcome the potential bias of regular performance profiles, especially when comparing more than two solvers.

A performance profile is a graph that shows the cumulative distribution function for the performance of different solvers on a set of problems

---

Dolan and More introduced performance profiles - cumulative distribution function of a performance metric

They used the ratio of the computing time of the solver versus the best time of all of the solvers as the performance metric. They showed that performance profiles eliminate the influence of a few of problems on the benchmarking process and the sensitivity of results associated with the ranking of solvers

## Conclusion

Performance profile provides a measure to compare multiple solvers. For a binary comparison it is a strong tool for a selected range of τ. However, if performance profile is used to compare multiple solvers we can determine which one has a higher probability of being within a factor τ of the best solver but we can not evaluate the performance of one solver relative to another one that is not the best. To address this problem we introduced the nested performance profile that uses consecutive performance profiles achieved by eliminating the best solver and calculates the mean performance over all of the runs. This algorithm combines the relative features of the solvers and gives a reliable criteria to compare all the solvers together. This can be useful if we don’t have access to the first solver or if we are interested in determining second or third best solvers out of a large set of solvers. The proposed method is a practical approach to deal with the fine tuning problem which can be seen as a benchmarking problem.
