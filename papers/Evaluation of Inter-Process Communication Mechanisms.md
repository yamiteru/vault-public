---
link: https://pages.cs.wisc.edu/~adityav/Evaluation_of_Inter_Process_Communication_Mechanisms.pdf
---

# TLDR;

The paper suggests that for small message sizes, pipes and shared memory offer similar latencies, but for larger messages, shared memory offers significantly lower latencies compared to pipes and sockets. 

Throughput-wise, pipes and shared memory offer similar throughput for small message sizes, while shared memory offers a significantly higher transfer rate beyond 16KB. 

Sockets, on the other hand, offer the lowest average transfer rate relative to the other two mechanisms, especially as the message size increases. 

These findings suggest that for smaller messages or bidirectional communication, pipes or shared memory may be suitable, while for larger messages or communication with different machines, shared memory or sockets may be preferred.

---

Shared memory provides the lowest latency and highest throughput, followed by kernel pipes and lastly, TCP/IP sockets.

Processor caches play a significant role in reducing memory access latency by fetching an entire block of ad- dresses for each cache miss.

## Pipes

A pipe is a unidirectional communication device that permits serial transfer of bytes from the writing process to the reading process.

## Sockets

A socket is a bidirectional communication mechanism that can be used to communicate with another process on the same machine or on a different machine.

## Shared Memory

Shared memory allows two or more processes to access the same memory region, which is mapped onto the address space of all participating processes.
