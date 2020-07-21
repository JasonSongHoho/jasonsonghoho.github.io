---
title: Flink 架构核心概念
date: 2020-04-11 21:51:07
tags:
- 大数据
- 中间件

---


### 任务和算子链


先介绍一下：算子(Operator)

Logical Graph 是一种描述流处理程序的高阶逻辑有向图。边代表输入/输出关系、数据流和数据集之一。算子是 Logical Graph 的节点，执行某种操作，该操作通常由 Function 执行。Source 和 Sink 是数据输入和数据输出的特殊算子。

分布式计算中，Flink 将算子（operator）子任务（subtask） 链接成一个个的任务（task）。每个任务占一个单独的线程。把多个算子链接成一个任务减少了线程间切换和缓冲的开销，并且在降低延迟的同时提高了整体吞吐量。

下图的示例数据流由五个子任务执行，因此由五个并行线程执行。

{% asset_img operator  operator.jpg %}

#### Job Managers、Task Managers 和 Clients

Flink 运行环境包含两类进程：
JobManagers（“主管” master）主要用来协调分布式计算。它们负责调度任务、协调检查点（checkpoints，参见下面）、协调故障恢复等。至少得有一个 JobManager（必须得有个安排任务的），高可用环境下可以有多个 JobManagers，其中一个正常工作，其它的作为“备胎”。
TaskManagers（“搬砖工” woker）主要用来执行任务（确切来说是子任务），缓存产生的数据并交换数据流。TaskManager 也至少得有一个（也不能没有打工的...）。

JobManagers 和 TaskManagers 有多种启动方式：
直接在机器上启动称为 standalone 集群；
在容器或资源管理框架 中启动，如 YARN 或 Mesos，TaskManagers 会连接到 JobManagers，通知后者已经可用，然后开始工作。

客户端（Client）虽然不是运行时（runtime）和程序执行时的一部分，但它被用来准备数据流并向  JobManager  提交。提交完之后客户端就可以断开连接，或者保持连接来接收进度报告。客户端既可以作为 Java / Scala 程序启动，也可以在命令行中运行，如 ./bin/flink run ...。

{% asset_img runtime  runtime.jpg %}

#### Task Slots 和资源

每个 worker（TaskManager）都是一个 JVM 进程，子任务通过其中不同的线程执行。每个 worker 可以接收任务的数量与它拥有的 task slots （可译为任务槽，至少一个）有关。

每个 task slot 可以认为是 TaskManager 的一份内存资源。例如，具有三个 slot 的 TaskManager 会将其管理的内存资源分成三等份，每个一份。划分资源可以避免子任务之间竞争资源，当然这也意味着它们拥有的资源大小是不可变的。不过 CPU 并没有隔离，只是平分了 woker 的内存资源。

用户可以通过调整 slot 的数量，调整子任务的隔离方式。若每个 TaskManager 只有一个 slot ，意味着每组任务占一个单独的 JVM 进程（例如，在一个单独的容器中启动）。如果一个 worker 有多个 slot ，则意味着多个子任务共享同一个 JVM。同一个 JVM 中的任务会共享 TCP 连接（通过多路复用技术）和心跳信息，还可以共享数据集和数据结构，从而降低整体开销。

{% asset_img taskSlots  taskSlots.jpg %}

默认情况下，Flink 允许来自同一个 job 的子任务共享 slot，即使它们是不同 task 的子任务。因此，一个 slot 可能会负责这个 job 的一整条路径（结合图1理解）。允许 slot 共享有两个好处：

Flink 集群需要的 slot 与 job 中使用的最高并行度恰好一样多。这样不需要计算程序总共包含多少个任务（任务可能具有多种并行度）。

资源利用率更高。slot 不共享时，简单的子任务（如：source/map()）将会占用和复杂的子任务（如：window）一样多的资源。通过共享 slot，将示例中的并行度从 2 增加到 6 可以充分利用 slot 的资源，还可以确保繁重的子任务能在多个 TaskManagers 之间平均分配。

{% asset_img slots  slots2.jpg %}

APIs 中包含了 resource group 机制，可以用来避免不必要的 slot 共享。

根据经验，slot 数量最好与 CPU 核数一致。使用超线程（hyper-threading）时，每个 slot 可以占用 2 个或更多的硬件线程。

#### Checkpoint

Flink 中的每个方法或算子都可以是有状态的。有状态的方法会在处理每个元素/事件的时候记录状态，从而使各种算子更加准确（因为可恢复）。Flink 使用 checkpoint（检查点）机制来保存状态。Checkpoint 能够恢复状态以及在数据流中的位置，从而保证无故障执行。

#### State Backends

不同类型的 state backend 会影响 key/values 索引存储时的数据结构。一种是将数据存储在基于内存的 HashMap 中，另一种会使用 RocksDB 存储。state backend 定义了数据结构的保存状态（state），定义了如何创建 key/values 的快照，并将该快照存储为 checkpoint 的一部分。

{% asset_img  stateBackend  stateBackend.jpg %}

#### Savepoints

用 Data Stream API 编写的程序可以从 savepoint 恢复执行。Savepoints 可以保证在更新、升级代码和 Flink 集群配置时，不丢失任何状态。

Savepoints 可以理解为手动触发的 checkpoints，类似常规的 checkpoint 机制，它会对程序创建一个快照并将其保存到 state backend。程序会定期在 worker 上创建快照并生成 checkpoints。Flink 只需要最后一个完整的 checkpoint 来确保恢复，一旦创建好了新的 checkpoint，旧的就可以丢弃。

Savepoints 类似于 checkpoints，只不过是手动触发的，并且在新的 checkpoint 创建好后不会自动过期。你可以通过命令行来创建 Savepoints，或者在取消一个 job 时通过 REST API 来创建。



*翻译自[官方文档](https://ci.apache.org/projects/flink/flink-docs-release-1.10/concepts/runtime.html)*

