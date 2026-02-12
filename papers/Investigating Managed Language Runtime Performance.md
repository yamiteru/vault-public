---
link: https://www.usenix.org/system/files/atc22-lion.pdf
---

# TLDR;

JavaScript is considered slower compared to C++ primarily due to the dynamic type checking it performs at runtime, which adds overhead, and its event-driven concurrency model with a single-threaded event loop. Additionally, the V8/Node.js runtime exhibits high overheads and poor scalability, leading to slower execution times.

---

Runtime performance is shrouded in mystery as it involves complex interactions of different components, such as interpreter, just-in-time (JIT) compiler, thread library, and Garbage Collection (GC) system.

Stream abandoned Python for Go, as Python would spend 10ms creating objects from data that Cassandra took 1ms to fetch, noting that “We’ve been optimizing Cassandra, PostgreSQL, Redis, etc. for years, but eventually, you reach the limits of the language you’re using.” 

Discord switched from Go to Rust claiming that “Rust was able to outperform the hyper hand-tuned Go version.” 

Performance issues are also cited as the main reason Twitter was forced to switch from Ruby on Rails to Scala and Java

We found that the high-level abstractions provided by the runtimes can, in some cases, result in better performance and scalability. This is counter-intuitive given the conventional wisdom that abstractions generally come at the expense of performance:

1) object relocations in OpenJDK’s moving GC can result in better cache locality
2) Go’s scheduler automatically maps user threads to kernel threads, and hence abstracts away the direct usage of kernel threads, reducing the number of context switches and the number of kernel threads used
3) abstracting away the low-level I/O operations allows runtimes to use the optimal I/O system call configurations
