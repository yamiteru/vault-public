---
link: https://link.springer.com/article/10.1007/s10664-021-09996-y
---

# TLDR;

The research paper proposes a method to predict if a software benchmark is stable or unstable before execution, using 116 static source code features. They find that machine learning models, particularly Random Forest, are effective at predicting benchmark stability. The paper also investigates the impact of different stability thresholds, numbers of measurement iterations, pre-processing steps, and variability measures on prediction performance. The study concludes that source code features can effectively predict benchmark stability, with potential applications in software development and performance testing.

The study found that Random Forest exhibited good prediction performance, with an area under the curve (AUC) ranging from 0.79 to 0.90 and Matthew's Correlation Coefficient (MCC) ranging from 0.43 to 0.68.

---

Software benchmarks are only as good as the performance measurements they yield. Unstable benchmarks show high variability among repeated measurements, which causes uncertainty about the actual performance and complicates reliable change assessment. 

Running more iterations leads to narrower confidence intervals of the results and, hence, to more stable benchmark results.
