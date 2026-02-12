---
link: https://dl.acm.org/doi/pdf/10.1145/3473575
---

# TLDR;

Hopc's performance is close to that of JIT compilers on many new tests, suggesting that an AoT compiler could compete with the fastest JIT compilers.

---

- no type annotation
- same expression can mean many things
- JIT requires code AND data at the same time
- AoT compilers can allocate conceptually infinite resources for analyzing and optimizing the program because they run before execution
- AoT compilers are efficient even for brief executions while JIT compilers need the execution to last sufficiently long to benefit from gathered profiled data
- New techniques [Serrano 2018] have been proposed to adapt JIT-style heuristics-based ap- proach to AoT compilation. These studies have shown that if indeed a JavaScript expression involves many different interpretations, typically only one is used over and over execu- tion [Würthinger et al. 2017] and guessing it before the execution is not too difficult

## Hopc

- AOT compiler
- Compiles to Scheme which compiles to C
- Focuses on JS strict mode
- Scheme gives less freedom and flexibility
- JS to Scheme takes more than 50 passes
- It uses the Boehm-Weiser collector [Boehm and Weiser 1988] hence it can exchange Scheme data and C data without restriction and without any conversion

## Observation

If JIT compilers suffer from a slow start with a ramp up period, it is unnoticeable when the executions last half a second or more. These JIT compilers are so efficient that all the curves of their execution times appear to be continuous functions.

Apart from almabench.js and moment.js, Hopc delivers performance comparable
to that of JIT compilers, among the fastest on some tests, among the slowest on others.

Hopc uses at least 3× less memory than V8, the most memory-efficient JIT.
