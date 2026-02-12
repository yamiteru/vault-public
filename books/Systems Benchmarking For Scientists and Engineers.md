1. Relevance:Abenchmarkshouldmeasuretheperformanceofthetypicaloperation within the problem domain.
2. Portability:Abenchmarkshouldbeeasytoimplementonmanydifferentsystems and architectures.
3. Scalability: A benchmark should be scalable to cover small and large systems.
4. Simplicity: A benchmark should be understandable to avoid lack of credibility.

---

1. Relevance: A benchmark has to reflect something important.
2. Repeatable: It should be possible to reproduce the results by running the bench-
mark under similar conditions.
3. Fair and Portable: It should be possible for all systems compared to participate
equally.
4. Verifiable:Thereshouldbeconfidencethatthedocumentedresultsarereal.This
can, for example, be assured by reviewing results by external auditors.
5. Economical: The cost of running the benchmark should be affordable.

---

A benchmark is a test, or set of tests, designed to compare the performance of one computer system against the performance of others.

---

The major issue with synthetic benchmarks is that they do not capture the impact of interactions between operations caused by specific execution orderings. Furthermore, such benchmarks often fail to capture the memory-referencing patterns of real applications. Thus, in many cases, synthetic benchmarks fail to provide representative workloads exhibiting similar performance to real applications from the respective domain. However, given their flexibility, synthetic benchmarks are useful for tailored system analysis allowing one to measure the limits of a system under different conditions.

Microbenchmarks are small programs used to test some specific part of a system (e.g., a small piece of code, a system operation, or a component) independent of the rest of the system.

Kernel benchmarks (also called program kernels) are small programs that capture the central or essential portion of a specific type of application. A kernel benchmark typically executes the portion of program code that consumes a large fraction of the total execution time of the considered application. The hope is that since this code is executed frequently, it captures the most important operations performed by the actual application.

---

In classical performance benchmarking, three different benchmarking strategies can be distinguished: 

1) fixed-work benchmarks, which measure the time required to perform a fixed amount of work
2) fixed-time benchmarks, which measure the amount of work performed in a fixed period of time
3) variable- work and variable-time benchmarks, which vary both the amount of work and the execution time

---

If a timer’s counter rolls over during an activity whose duration is being measured using the timer, then the value (c2 − c1 )Tc reported by the timer will be negative. Therefore, applications that use a timer must either ensure that roll over can never occur when using the timer or they should detect and correct invalid measurements caused by roll over.

The accuracy of measurements obtained through an interval timer generally depends on two factors: the timer resolution and the timer overhead.
