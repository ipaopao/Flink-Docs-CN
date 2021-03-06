# 生产准备检查表

## 生产准备检查表

此生产准备清单的目的是提供重要的配置选项概述，如果您计划将Flink作业投入**生产**，则需要**仔细考虑**。对于大多数这些选项，Flink提供了开箱即用的默认设置，以便更轻松地使用和采用Flink。对于许多用户和场景，这些默认值是开发的良好起点，并且完全足以用于“一次性”作业。

然而，一旦您计划将Flink应用程序投入生产，需求通常会增加。例如，您希望您的作业具有\(重新\)可伸缩性，并为您的作业和新的Flink版本提供良好的升级体验。

在下面的内容中，我们将提供一组配置选项，您应该在作业投入生产之前检查这些选项。

### 明确地为操作符设置最大并行度

最大并行度是Flink 1.2中新引入的一个配置参数，它对Flink作业的\(重新\)可伸缩性有重要影响。这个参数可以在每个作业和/或每个操作符粒度上设置，它决定了可以缩放操作符的最大并行度。重要的是要理解\(到目前为止\)，在作业启动之后，除了完全从头重新启动作业\(即使用新的状态，而不是从以前的检查点/保存点\)之外，**没有任何方法**可以更改此参数。即使Flink将来能够提供某种方法来更改现有保存点的最大并行度，也可以假设对于大型状态，这可能是您希望避免的长时间运行的操作。此时，你可能想知道为什么不使用一个非常高的值作为这个参数的默认值。这背后的原因是，高最大并行度会对应用程序的性能甚至状态大小产生一定的影响，因为Flink必须维护某些元数据，以使其具有重新伸缩的能力，而这种能力会随着最大并行度的增加而增加。通常，应该选择一个最大并行度，该并行度要足够高，以满足未来在可伸缩性方面的需求，但是尽可能地保持较低的并行度可以带来稍微更好的性能。特别是，高于128的最大并行度通常会导致来自键控后端的稍微更大的状态快照。

请注意，最大并行度必须满足以下条件：

`0 < parallelism <= max parallelism <= 2^15`

可以通过`setMaxParallelism(int maxparallelism)`设置最大并行度。默认情况下，Flink将在作业首次启动时选择最大并行度作为并行度的函数：

* `128` ：对于所有并行度&lt;= 128。
* `MIN(nextPowerOfTwo(parallelism + (parallelism / 2)), 2^15)` ：对于所有并行度&gt; 128。

### 为操作符设置UUID

正如[保存点](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/state/savepoints.html)文档中提到的，用户应该为操作人员设置uid。这些操作符uid对于Flink将操作符状态映射到操作符非常重要，而这些操作符对于保存点来说又是必不可少的。默认情况下，操作符uid是通过遍历JobGraph和散列某些操作符属性生成的。虽然从用户的角度看这很舒服，但它也非常脆弱，因为JobGraph的更改\(例如交换操作符\)将导致新的uuid。为了建立一个稳定的映射，我们需要用户通过setUid\(String uid\)提供稳定的操作符uid。

### 选择状态后段

目前，Flink的局限性在于它只能从保存点恢复保存点的相同状态后端的状态。例如，这意味着我们不能使用带有内存状态后端的保存点，然后将作业更改为使用RocksDB状态后端并进行恢复。虽然我们计划在不久的将来使后端可互操作，但它们还没有。这意味着在投入生产之前，您应该仔细考虑使用哪个后端来完成作业。

通常，我们建议使用RocksDB，因为它是目前唯一支持大状态\(即超过可用主内存的状态\)和异步快照的状态后端。根据我们的经验，异步快照对于大型状态非常重要，因为它们不会阻塞操作符，Flink可以在不停止流处理的情况下编写快照。然而，与基于内存的状态后端相比，RocksDB的性能可能更差。如果您确信您的状态永远不会超过主内存，并且阻塞流处理来编写它不是问题，那么您可以考虑不使用RocksDB后端。但是，在这一点上，我们强烈建议使用RocksDB进行生产。

### 配置JobManager高可用性（HA）

JobManager协调每个Flink部署。它负责_调度_和_资源管理_。

默认情况下，每个Flink群集都有一个JobManager实例。这会产生_单点故障_（SPOF）：如果JobManager崩溃，则无法提交新程序并且运行程序会失败。

使用JobManager High Availability，可以从JobManager故障中恢复，从而消除_SPOF_。我们**强烈建议**您为生产配置[高可用性](https://ci.apache.org/projects/flink/flink-docs-release-1.7/ops/jobmanager_high_availability.html)。



