---
link: https://www.crysys.hu/publications/files/setit/cpaper_szte_AntalHTFGy19scam.pdf
---

# TLDR;

The paper evaluates and compares the performance of five static JavaScript call graph extraction tools based on quantitative, qualitative, and performance analyses on benchmark programs and real-world Node.js modules. It also provides a manually validated dataset of call edges found by these tools.

The performance and capabilities of the five different static analysis based call graph building approaches were compared by quantitatively and qualitatively evaluating their results on 26 benchmark programs and 6 real-world Node.js modules. The evaluation included aspects such as precision, recall, runtime performance, and memory consumption. The study found variances in the number, precision, and type of call edges reported by the individual tools, with some tools exhibiting higher precision or recall in detecting true call edges. Additionally, the study examined the tools in combination, assessing their abilities to detect true call edges collectively. Additionally, runtime performance and memory consumption were considered, with Closure and TAJS excelling in runtime performance, and ACG and Closure utilizing more memory for realistic input sizes.

---

## Algorithms

- npm call graph, 
- IBM WALA 
- Google Closure Compiler
- ACG (Approximate Call Graph)
- TAJS (Type Analyzer for JavaScript)

## Conclusion

ACG and Google Closure Compiler are the best ones.
