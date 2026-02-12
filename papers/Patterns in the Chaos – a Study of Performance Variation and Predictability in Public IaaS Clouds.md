---
link: https://arxiv.org/pdf/1411.2429.pdf
---

# TLDR;

The performance variation in cloud instances can be affected by factors such as hardware heterogeneity for CPU-bound applications, the behavior of co-located tenants for IO-bound applications, and temporal factors such as time of day, day of the week, and the selected region of the cloud provider. These factors can impact both the absolute performance and the performance predictability of cloud instances.

---

In an Infrastructure-as-a-Service (IaaS) cloud, computing resources are acquired and released as a service, typically in the form of virtual machines with attached virtual disks.

IaaS clouds often serve requests for identical instance types with differing underlying hardware (hardware heterogeneity).

In IaaS, multiple customers typically own instances running on the same physical machine (multi-tenancy).

Most computing resources (e.g., network or disk I/O, but usually not CPU cores) are shared among all instances on a machine. This can lead to the “noisy neighbour” problem, when one instance experiences a slowdown due to the behavior of a co-located other tenant.

## H1: Performance Predictability

It is evident that cloud instances with different configurations (e.g., m1.small versus m3.large in EC2) provide different performance.

This makes it hard for cloud customers to predict the performance of their instances in advance.

The performance of CPU-bound applications or benchmarks (i.e., those that depend strongly on processing speed or memory access) varies primarily due to the randomness of which physical hardware is used for an instance (hardware heterogeneity).

Contrary, the performance of IO-bound applications or benchmarks (i.e., those that primarily depend on disk IO or networking bandwidth) varies primarily due to the behavior of other co-located tenants

## H2: Variability Within Instances

Even the performance of a single instance may vary over time. For IO- bound applications (e.g., disk IO and network throughput), existing research has established that even multiple benchmarks taken in quick succession, within the timeframe of minutes or even seconds, can lead to substantially varying results.

## H3: Temporal and Geographical Factors

Some authors have speculated that there may be additional temporal and geographical factors that influence cloud performance and predictability, in addition to hardware heterogeneity and multi-tenancy. One such potential factor of influence is the time of the day when a benchmark was launched

Multiple earlier studies have shown that the region an instance is launched in (e.g., us-east-1 or eu-west-1 in EC2) has implications, both on the performance and the expected performance variability. 

Different regions typically map to different physical data centers, which differ in the hardware used to serve requests, as well as in the data center utilization, which impacts how many tenants are typically located on each physical machine.

## H4: Instance Type Selection

The ratio of performance and costs of more expensive instance types tends to be lower than of cheaper instance types. Essentially, this means that there are diseconomies of scale when selecting larger instances.

However, one advantage of larger instance types is a higher predictability of performance

# Results

Our data indeed shows that controlling for the CPU model further reduces the ob- served relative standard deviations in CPU-bound benchmarks. 

Controlling for the served CPU model also reduces the relative standard deviations in IO-bound benchmarks, but to a much smaller degree.

In terms of IO, Azure and EC2 exhibit fluctuating performance for most instance types, while both, GCE and SL, are remarkably predictable.

More concretely (considering that for the CPU and Java benchmarks, lower is better) our data seems to indicate that us-east-1 is typically worse-performing as well as more unpredictable (i.e., have higher standard deviation) than eu-west-1. Incidentally, this may explain why providers are, at the time of our research, typically pricing the European regions slightly higher than the North Amer- ican regions.
