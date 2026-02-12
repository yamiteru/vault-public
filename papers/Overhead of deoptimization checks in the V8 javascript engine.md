---
link: https://users.soe.ucsc.edu/~renau/docs/iiswc16.pdf
---

# TLDR;

1. Characterizing the performance overhead of deoptimization checks for the Chrome V8 JavaScript engine on different machines.
2. Showing that performance overhead does not always correlate with instruction count overhead, but rather depends on a combination of the workload and the system microarchitecture.
3. Categorizing the types of deoptimization checks and characterizing their execution frequency when running optimized code.

---

Just-in-time compilation for dynamic languages uses type speculation to generate optimized code for frequently executed code sections. However, even the high-performance code requires checks to ensure that assumptions made during code generation are not violated. If the assumptions are violated then the program reverts to executing unoptimized code. This process is called deoptimization, and the checks are called deoptimization checks.

After type information has been collected, frequently executed code is recompiled with an optimizing compiler, which assumes that variable types will not change even though this is not guaranteed by the language semantics. In order to guarantee that these assumptions are not violated, the compiler inserts checks to ensure that the assumptions about types still hold. If the condition expected by the checks fails to hold then the function stops running the optimized code and switches back to the unoptimized version.

The cost of each conditional deoptimization check is rel- atively small, typically ranging from 1 to 3 instructions.

## Types of checks

There are 64 different types of checks.

These are the most common ones:
- type checks
- small integer checks
- bounds checks
- overflow checks

## Findings

The performance impact of the checks varies significantly depending on the microarchitecture of the CPU that is running the code. On average the conditional branch used by the checks accounts for over 6.2% of the benchmark’s dynamic instructions. Removing these checks provides only a 2.2% performance improvement when running on a high performance Intel CPU, but a 4.6% performance improvement on an Intel CPU optimized for low power consumption.
