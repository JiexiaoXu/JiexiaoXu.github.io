---
title: "Work Stealing in OS"
status: "In progress"
excerpt: "A short note on why microsecond-scale services make OS scheduling difficult, and where work stealing fits among ZygOS, Shenango, and Caladan."
---

## Background

Current applications do not live on a single machine. To handle large amounts of user data and high request volume, online applications distribute work across multiple machines and expose them as services. After receiving a client request, a gateway sends remote procedure calls (RPCs) over the network to multiple services.

To manage time budgets and service performance, developers often use service-level objectives (SLOs). An SLO defines the expected response time for a request. In systems such as memcached, many requests are expected to finish within only a few microseconds.

Maintaining per-request SLOs under load while also keeping CPU efficiency high is hard. The difficulty comes from both the workload itself and the mismatch between operating system efficiency and modern network bandwidth.

## Problem 1: Workload

The latency of each service is susceptible to multiple factors. Request arrival rates can follow different patterns, such as bursty traffic or diurnal demand. Once the arrival rate exceeds server capacity, queues build up. Requests that arrive later then suffer from head-of-line blocking and risk violating their SLOs unless more resources are provisioned.

One straightforward response is to overprovision resources so queues are less likely to build. However, this wastes resources. If developers provision enough capacity so overload never happens, utilization during normal periods can be poor. It is expensive to maintain a large cluster of underutilized machines.

Service latency can also change because of the application itself. Events such as cache misses or reconnections may trigger longer latencies than expected. Tasks such as garbage collection, which are less time-sensitive, may also contend for the same resources needed by latency-sensitive tasks. Again, the common response is to add more resources, which leads back to the same underutilization problem.

## Problem 2: Kernel Efficiency

The operating systems widely used in production are not always well optimized for current network hardware. From queueing theory, a single queue shared by multiple processors can be better than one queue per processor, and processor sharing can handle highly variable tasks well. In practice, however, real systems do not match the theory neatly.

As Amy Ousterhout discusses in her talk, network bandwidth has increased dramatically over the past decades. The network is now fast enough that the operating system can become the bottleneck. Receiver-side scaling (RSS) is often used to achieve high throughput by steering requests from the network interface card (NIC) directly to CPU cores rather than relying on one centralized queue.

This improves throughput, but it also departs from the theoretically ideal shared-queue design. Linux without kernel bypass keeps the single-queue model that looks best in theory, yet it can incur high overheads for microsecond-scale tasks because scheduling is too coarse-grained. Kernel-bypassing approaches improve throughput, but they introduce the drawbacks of one queue per core: poor work conservation and head-of-line blocking.

From queueing theory, the per-core strategy is only attractive for short, low-dispersion tasks. Otherwise, it is common for one worker to stay idle while another remains overloaded.

Work stealing is a natural idea for maintaining high CPU utilization. Each worker keeps a local queue of pending tasks, and idle workers steal tasks from the opposite end of another worker's queue. In principle, this makes the system work-conserving while requests are in flight.

The intuition is simple, but work stealing is not immediately useful here because its overhead is often on the millisecond scale, including context switching and stack management.

## References

These systems are useful anchors for thinking about the design space:

- [ZygOS: Achieving Low Tail Latency for Microsecond-scale Networked Tasks](https://marioskogias.github.io/docs/zygos.pdf) studies a work-conserving scheduler for microsecond-scale networked tasks.
- [Shenango: Achieving High CPU Efficiency for Latency-sensitive Datacenter Workloads](https://www.usenix.org/conference/nsdi19/presentation/ousterhout) reallocates cores at microsecond granularity to improve CPU efficiency for latency-sensitive applications.
- [Caladan: Mitigating Interference at Microsecond Timescales](https://www.usenix.org/conference/osdi20/presentation/fried) uses fast core allocation and interference-aware scheduling to improve quality of service.
