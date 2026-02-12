---
link: https://courses.engr.illinois.edu/cs241/sp2014/lecture/27-IPC.pdf
---

# TLDR;

Shared memory is a technique that allows multiple processes to access common data by mapping a portion of a process's virtual address space to a shared physical memory segment. In the context of interprocess communication, shared memory allows processes to share data without the need for explicit communication through message passing, making it a fast and efficient means of communication between cooperating processes.

---

Treats file I/O like memory access rather than read(), write() system calls.

Several processes can map the same file - saves memory 
