---
link: https://www.semanticscholar.org/paper/WebAssembly-versus-JavaScript%3A-Energy-and-Runtime-Macedo-Abreu/5f24eb714cb8324af3a4b6dcb61acf02281c9a59
---

# TLDR;

Significant differences were observed in both the energy consumption and runtime performance of Wasm and JS between real-world applications and micro-benchmarks. In micro-benchmarks, on average, Wasm was 9.84% slower than JS, but the opposite was observed in real-world applications, where Wasm outperformed JS. Additionally, Wasm's behavior was noted to be different between micro and real applications, indicating varying performance.

---

We used micro-benchmarks and also real applications in order to have more realistic results. Preliminary results show that WebAssembly, while still in its infancy, is start- ing to already outperform JavaScript, with much more room to grow. A statistical analysis indicates that WebAssembly produces significant performance differences compared to JavaScript. However, these differences differ between micro-benchmarks and real-world benchmarks. Our results also show that WebAssembly improved energy efficiency by 30%, on average, and showed how different WebAssembly behaviour is among three popular Web Browsers: Google Chrome, Microsoft Edge, and Mozilla Firefox. Our findings indicate that WebAssembly is faster than JavaScript and even more energy-efficient. Additionally, our benchmarking framework is also available to allow further research and replication.

Wasm is not a new language to directly write our applications. Instead, it is a compilation target that allows C/C++, Rust or TypeScript developers to build their applications, compile to Wasm and execute it on a browser.

## Findings

While the difference between Wasm and JS performance is irrelevant in micro-benchmarks, there is a clear and significant gain in both energy and run-time effi- ciency of Wasm using real applications.
