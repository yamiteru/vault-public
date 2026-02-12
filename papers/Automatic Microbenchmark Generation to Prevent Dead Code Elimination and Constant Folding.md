---
link: https://inria.hal.science/hal-01343818/file/technical-paper-main.pdf
---

# TLDR;

Common mistakes made in microbenchmarks include failing to prevent the JIT from performing dead code elimination (DCE) and constant folding/-constant propagation (CF/CP), choosing poorly initialization values, and reaching a steady state measuring an undesired behavior.

The proposed technique aims to address and prevent these mistakes by automatically generating payloads for Java microbenchmarks that are guaranteed to be free of dead code and CF/CP. It does this by performing a static slicing to automatically extract the critical code segment and its dependencies, preventing the JIT from "over-optimizing" the payload, initializing the payload's input with relevant values, and maintaining the payload in a steady state. Additionally, the technique uses novel transformations such as sink maximization and turning some local variables into fields in the payload to mitigate the risks of over-optimization.

---

Microbenchmarking consists of evaluating, in isolation, the performance of small code segments that play a critical role in large applications. The accuracy of a microbenchmark depends on two critical tasks: wrap the code segment into a payload that faithfully recreates the execution conditions that occur in the large application; build a scaffold that runs the payload a large number of times to get a statistical estimate of the execution time.

Dead Code Elimination (DCE) is one of the most common optimizations engineers fail to detect in their microbenchmarks.

Constant Folding and Constant Propagation (CF/CP) is another JIT optimization that removes all computations that can be replaced by constants.
