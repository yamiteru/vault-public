---
link: https://users.cs.northwestern.edu/~robby/courses/322-2013-spring/mytkowicz-wrong-data.pdf
---

# TLDR;

- The paper surveyed 133 recent papers from ASPLOS, PACT, PLDI, and CGO and found that none of them addressed measurement bias satisfactorily.
- Out of 88 papers with dedicated sections to experimental methodology and evaluation, 36 used simulations, and the median speedup reported was 10%.
- The paper used 12 benchmarks and still saw significant measurement bias, indicating that larger benchmark suites might not fully eliminate the effects of measurement bias.
- They present data showing contradictory conclusions about the speedup of O3 due to different experimental setups, making measurement bias a significant and commonplace issue.

---

What appears to be an innocuous aspect in the experimental setup may in fact introduce a significant bias in an evaluation. This phenomenon is called measurement bias in the natural and social sciences.

Researchers in computer systems either do not know about measurement bias or do not realize how severe it can be. For example, we found that none of the papers in APLOS 2008, PACT 2007, PLDI 2007, and CGO 2007 address measurement bias satisfactorily.

Many researchers use simulations because simulators enable them to try out hypothetical architectures. 36 of the 88 papers we reviewed used simulations. Even simulations suffer from measurement bias.

If the ideas in a paper result in huge (e.g., many-fold) improvements, then one may argue that the improvements are a result of the ideas and not artifacts of measurement bias. However, we found that the median speedup reported by these papers was 10%

If we use a sufficiently diverse set of workloads along with careful statistical methods, most measurement bias should get factored out. Unfortunately, there is no reason to believe that our benchmark suites are diverse.

Causal analysis is a general technique for determining if we have reached an incorrect conclusion from our data. The conclusion may be incorrect because our data is tainted or because we arrived at the conclusions using faulty reasoning.

Causal analysis takes the following steps:
1. Intervene: We devise an intervention that affects X while having a minimal effect on everything else.
2. Measure: We change our system with the intervention and measure the changed system.
3. Confirm: If Y changed as we would expect if X had caused Y then we have reason to believe that our conclusion is valid.
