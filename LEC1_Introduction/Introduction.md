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
