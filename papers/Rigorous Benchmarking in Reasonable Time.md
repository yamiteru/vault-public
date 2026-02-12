---
link: https://kar.kent.ac.uk/33611/45/p63-kaliber.pdf
---

# TLDR;

The statistically rigorous methodology for repetition and summarizing results involves capturing experimentation cost with a novel mathematical model to identify the necessary and sufficient number of repetitions at each level of an experiment to obtain a given level of precision.

The novel mathematical model captures experimentation cost by considering the trade-off between the cost of running an experiment and the precision of the results obtained.

- Out of 122 papers presented in 2011 at PLDI, ASPLOS, and ISMM, or published in TOPLAS, 71 failed to provide any measure of variation for their results.
- The paper reports a median performance improvement of 10% in the field.
- Running SPEC INT benchmarks with 30 different linking orders takes 3 days on a recent platform.
- Running the DaCapo 2006 and 2009 benchmarks to compare two systems, using 20 executions and 10 iterations, takes almost 3 days.
- Researchers in the field often face the task of running ever more experiments to deal with non-determinism, requiring multiple iterations and executions for each benchmark.

---

Because modern systems are complex and non-deterministic, good experimental methodology demands that researchers account for uncertainty. To obtain valid results, they are expected to run many iterations of benchmarks, invoke virtual machines (VMs) several times, or even rebuild VM or benchmark binaries more than once. All this repetition costs time to complete experiments.

We have seen how variation can be introduced at several stages of a benchmark experiment (iteration, execution, compilation and so on). Three kinds of variables influence the outcomes of exper- iments. Values of controlled variables (such as the platform we choose, the heap size or compiler options) and how they impact the results are of interest for the evaluation. Random variables (such as the time between hardware interrupts or scheduling or- der on a multi-processor) change frequently in a random or non- deterministic manner.

Through repetition we get a number of measurements and typ- ically we calculate a confidence interval. The more repetitions made, the narrower (‘more precise’) is the interval. At the very least, repetition must be done at the highest level that has random variation to avoid bias, but sometimes repeating at lower levels can reduce experimentation time without sacrificing precision.

Random factors, such as context switches, scheduling order or Java heap layout, can affect performance, so repetition at the iter- ation level or higher is needed. Repeating iterations is experimen- tally cheaper since there is no need to wait for a new execution (or higher level operation) to reach a steady state.

Rigorous performance evaluation requires benchmarks to be built, executed and measured multiple times in order to deal with random variation in execution times. Researchers should provide measures of variation when reporting results.
