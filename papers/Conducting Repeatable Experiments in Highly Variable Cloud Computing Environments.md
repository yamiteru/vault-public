---
link: https://cs.uwaterloo.ca/~brecht/papers/icpe-rmit-2017.pdf
---

# TLDR;

The main challenges outlined in the research paper include the need for obtaining repeatable performance measurements in cloud computing environments, where variations in performance can occur due to factors such as being assigned different physical systems, sharing resources with other applications, and variations in network latencies or throughput. Another challenge is ensuring that any differences in performance between competing alternatives are due only to the differences in the alternatives and not because of external factors such as a more favorable environment.

The research paper suggests using a technique called Randomized Multiple Interleaved Trials (RMIT) to obtain repeatable and fair performance measurements in cloud computing environments. This technique aims to address the challenges by enabling fair comparisons between multiple alternatives over an extended period. Additionally, the paper proposes the use of traces collected and analyzed by an independent research group for evaluating commonly used methodologies, and discusses the shortcomings of existing techniques for comparing competing alternatives.

---

Previous work has shown that benchmark and application performance in public cloud computing environments can be highly variable.

Unfortunately, performance measurements in cloud computing environments can vary significantly. This may be due to being assigned different physical systems across different experiments at different times. Or, the performance may be impacted due to sharing, in this case variations occur due to other applications that are executing on the same physical hosts (e.g., affecting disk throughput) or elsewhere in the same cloud environment (possibly affecting network latencies or throughput).

## OVERVIEW OF METHODOLOGIES

- Single Trial: In this approach, an experiment consists of only a single trial. Figure 1-A shows an example of this approach with three alternatives. The performance results obtained from single trials are compared directly. This is the easiest methodology for running an experiment to compare multiple alternatives.
- Multiple Consecutive Trials: All trials for the first alternative are run, followed by the second alternative and each of the remaining alternatives. Figure 1-B shows the Multiple Consecutive Trials technique for 3 alternatives.
- Multiple Interleaved Trials: One trial is conducted using the first alternative, followed by one trial with the second, and so on until each alternative has been run once. When one trial has been conducted using each alternative we say that one round has been completed. Rounds are repeated until the appropriate number of trials has been conducted (Figure 1-C).
- Randomized Multiple Interleaved Trials: If the cloud computing environment is affected at regular intervals, and the intervening period coincides with the length of each trial, it is possible that some alternatives are affected more than others. Therefore, the randomized multiple interleaved trials methodology randomly reorders alternatives for each round (Figure 1-D). In essence, a randomized block design is constructed where the blocks are intervals of time (rounds) and within each block all alternatives are tested, with a new random ordering of alternatives being generated for each block.
