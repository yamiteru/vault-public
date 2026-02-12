---
link: https://ieeexplore.ieee.org/ielx7/12/9984045/09812514.pdf
---

# TLDR;

To minimize system noise in a Linux setup, you can consider adjusting the hardware configuration to achieve the best trade-off between performance and determinism. This includes adjusting processor speed, power savings setup, and disabling features that could cause hardware-induced latencies, such as system management interrupts (SMIs). Additionally, partitioning the system into isolated and housekeeping CPUs, configuring NFV threads with real-time priorities, and enabling fully preemptive mode in the kernel (using the PREEMPT_RT patch-set) can help reduce system noise. Furthermore, using tools like the osnoise tracer and conducting synthetic workloads can aid in understanding and minimizing system noise.

---

As modern network infrastructure moves from hardware-based to software-based using Network Function Virtualization,
a new set of requirements is raised for operating system developers. By using the real-time kernel options and advanced CPU isolation features common to the HPC use-cases, Linux is becoming a central building block for this new architecture that aims to enable a new set of low latency networked services. Tuning Linux for these applications is not an easy task, as it requires a deep understanding of the Linux execution model and the mix of user-space tooling and tracing features. This paper discusses the internal aspects of Linux that influence the Operating System Noise from a timing perspective. It also presents Linux’s osnoise tracer, an in-kernel tracer that enables the measurement of the Operating System Noise as observed by a workload, and the tracing of the sources of the noise, in
an integrated manner, facilitating the analysis and debugging of the system. Finally, this paper presents a series of experiments demonstrating both Linux’s ability to deliver low OS noise (in the single-digit ms order), and the ability of the proposed tool to provide precise information about root-cause of timing-related OS noise problems.

The limitation of workload- based tools is that they do not provide any insight into the root cause of the noise. Conversely, trace-based methods show potential causes of latency spikes, but they cannot account for how the workload perceives the noise.

osnoise uses a hybrid approach, leveraging both the workload and a trac- ing mechanism synchronized together to account for the operating system noise while still providing detailed information on the root causes of OS noise spikes, and also reducing the tracing overhead using in-kernel tracing features. While the tool was developed with extreme isolation cases in mind, targeting the detection of single-digit ms noise occurrences, it is not limited to this use case.

Since Linux kernel version 5.17, the tracer can be used as an user-space tool available via the rtla (Real-Time Linux Analysis) toolset, becoming easily accessible both by practitioners to test their systems and developers to extend it.

Because the osnoise tracer uses the most basic building blocks of the Linux tracing sub-system, it can be combined with many other existing tracing tools, such as performance counters provided via perf tool, or to be used with graphi- cal interfaces provided by LTTng and KernelShark.
