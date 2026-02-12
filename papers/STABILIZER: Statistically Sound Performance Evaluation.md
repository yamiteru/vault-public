---
link: https://people.cs.umass.edu/~emery/pubs/stabilizer-asplos13.pdf
---

# TLDR;

STABILIZER is a system designed to enable statistically sound performance evaluation on modern processor architectures by repeatedly randomizing the placement of code, stack, and heap objects at runtime.

The paper highlights the unpredictable and substantial impact of layout changes on performance: changing the size of environment variables can cause performance degradation by as much as 300%, while simply changing the link order of object files can lead to a performance decrease of up to 57%.

---

Software developers use automatic performance regression tests to discover when changes improve or degrade performance.

Statistically sound evaluation requires multiple samples to test whether one can or cannot (with high confidence) reject the null hypothesis that results are the same before and after. However, caches and branch predictors make performance dependent on machine-specific parameters and the exact layout of code, stack frames, and heap objects.

STABILIZER forces executions to sample the space of memory configurations by repeatedly re-randomizing layouts of code, stack, and heap objects at runtime. STABILIZER thus makes it possible to control for layout effects.

The problem is due to the interaction between software and modern architectural features, especially caches and branch predictors. These features are sensitive to the addresses of the objects they manage. Because of the significant performance penalties imposed by cache misses or branch mispredictions (e.g., due to aliasing), their reliance on addresses makes software exquisitely sensitive to memory layout. Small changes to code, such as adding or removing a stack variable, or changing the order of heap allocations, can have a ripple effect that alters the placement of every other function, stack frame, and heap object.

The impact of these layout changes is unpredictable and sub- stantial: Mytkowicz et al. show that just changing the size of environment variables can trigger performance degradation as high as 300%.

Failure to control for layout is a form of measurement bias: a systematic error due to uncontrolled factors.

Normally distributed execution times allow researchers to evaluate performance using parametric hypothesis tests, which provide greater statistical power by leveraging the properties of a known distribution (typically the normal distribution). Statistical power is the probability of correctly rejecting a false null hypothesis. Parametric tests typically have greater power than non-parametric tests, which make no assumptions about distribution.
