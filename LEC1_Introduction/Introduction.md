# Introduction

## Why distributed?

- 为了连接物理上相互分离的实体
- 为了通过隔离(isolation)获得安全性
- 为了通过复制(replication)实现容错
- 为了使得 CPUs / mem / disk / net 可以实现扩容


## Main Topics

- Storage
- Communication
- Computation


### Topic: implementation

RPC, threads, cocurrency control.

### Topic: performance

- dream: scalable throughput, 更多的机器意味着能够处理更多的负载
- Scaling gets harder as N grows: 负载均衡问题、straggler 问题（其他Worker都已经完成了任务，只剩下一个异常慢的机器）、无法并行化的代码（初始化过程、交互）、共享资源导致的瓶颈（网络）


### Topic: fault tolerance

- hide failures from the application.

We want:

- Availability - can make progress despite failures
- Durability - can come back to life when failures are repaired

Big idea: replicated servers (If one server crashe, client can proceed using the other)

### Topic: consistency

Achieving good behavior is hard!

- Replica servers are hard to keep identical. （难以保持相同）
- Clients may crash midway through multi-step update.（client 在多步更新中的中途崩溃）
- Servers crash at any time. （如已经执行操作但还未回复）
- Network may make live servers look dead; "split brain" problem（当有状态系统中，部分节点之间不可达，分裂为两部分独立节点，开始争抢共享资源，导致系统混乱）

Consistency and performance are enemies.

- Consistency requires communication.
- "Strong consistency" often leads to slow systems.
- High performance often imposes "weak consistency" on applications.

# MapReduce

## Overview

defines Map and Reduce functions.

## Abstract view of MapReduce

map(k1, v1) -> list(k2, v2)

reduce(k2, list(v2)) -> list(k2, v3)

## Hides many painful details

- starting s/w on servers
- tracking which tasks are done
- data movement
- recovering from failures


## MapReduce scales well:

Map()s can run in parallel, since they don't interact. Same for Reduce()s.
The only interaction（交互）is via the "shuffle" in between maps and reduces.（这是指map的结果转移给reduce）

## What will likely limit the performance?

In 2004 autors were limited by "network cross-section bandwidth", So they care about minimizing movement of data over the network.

但是如今数据中心的内网速度要比当时快多了，因此如今的答案可能是磁盘了，新的架构试图减少数据持久化到磁盘的次数，更多的利用内存甚至网络（也就是 Spark 的设计理念）

## How does detailed design reduce effect of slow network?

通过 GFS 来存放 input data，尽量分配 map 任务到数据所在机器或同机房机器上执行。


Map worker writes to local dist, not GFS.

为什么不把中间数据通过 TCP 直接传输给 Reducer？

我觉得有两个方面：

1. reduce 需要处理的数据可能很大，即使通过网络直接给 reduce，也需要存在reduce本地上。
2. 考虑到 reduce 可能 down 掉，需要重新传输数据。


## How do they get good load balance?

bad for N-1 servers to wait for 1 to finish. (Critical to scaling)

Solution: many more tasks than workers


## Details of worker crash recovery:

Map worker crashes.

- master sees worker no longer responds to ping （意味着 worker 的 Map output 丢失了）
- master re-runs, spreads tasks over other GFS replicas of input.
- Map 需要是无状态、幂等的
- 如果 Reduce 已经获取了所有中间数据，那么master need not re-run Map


Reduce worker crashes.

- finshed tasks are OK -- stored in GFS, with replicas.
- master re-starts worker's unfinished tasks on other workers.


Reduce worker crashes in the middle of writing its output.

- GFS has atomic rename that prevents output from begin visible until complete.
- It's safe for the master to re-run the Reduce tasks somewhere else.


## Other failures/problems:

- what if the master gives two workers the same Map() task?

    the master incorrectly thinks one worker died. it will tell Reduce workers about only one of them.

- what if the master gives two workers the same Reduce() task?

    they will both try to write the same output file on GFS. atomic GFS rename prevents mixing.

- what if a single worker is very slow -- "straggler" problem ?

    master starts a second copy of last few tasks.

- what if the master crashes?

    recover from check-point, or give up on job


## Conclusion

Not the most efficient or flexible / Scales well / Easy to program.
