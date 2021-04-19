---
title: 正确理解flink流处理中exactly once
description: 正确理解flink流处理中exactly once
categories:
  - big-data
tags:
  - flink
---

当提到flink流处理中的“exactly once”，通常我们的第一反应是，flink在处理事件流的过程中，即使任务失败，也可以保证单个事件只被处理，并且仅仅被处理一次。实际上这个理解是错误的。流处理中常说的“exactly once”通常有一定的误解和不准确性。下面我们从“exactly once”实现的原理出发，来解释为什么这样理解是不准确的。

流处理中的应用通常可以被描述为一个有向无环图(DAG)，在图中，边表示数据流，一个顶点表示一个算子。在分布式计算环境中，由于网络，单点机器故障等不确定因素，必然导致数据丢失。所以，分布式流计算引擎通常允许用户设置一个可靠度模式或者处理语义，来指定流计算引擎在处理流数据时可以提供的保障。一般有三种模式可以选择：at-most-once，at-least-once和exactly-once。

at-most-once是指流中的单个事件最多只被应用中的算子处理一次，意味着如果事件在应用的所有算子处理完成之前丢失了，流处理引擎不尝试进行重试或重新提交。

at-least-once则保障流中的事件被应用中的所有算子至少处理一次，也就是如果事件在所有算子成功处理完成之前丢失了，流处理引擎会从事件源中重放或重新提交该事件进行处理。但这样也有可能导致一个事件被多次重复处理。

exactly-once保障事件被应用中的算子仅仅处理一次。通常有两种机制来实现该语义：1，分布式快照/状态检查点；2，至少一次加消息重复；

分布式快照是受chandy-lamport算法启发，具体是应用中每个算子的状态被周期性的设置检查点，当遇到失败的时候，每个算子的状态都回滚到最近一次全局一致的检查点。回滚中，暂停所有的处理。源也被重置到最近一次检查点对应的位置。整个的流处理应用也被倒回到最近一次的一致状态。这样流处理就可以从这个状态重新开始。从这个实现过程，我们可以发现，最近一次检查点之后的事件是有可能被多次处理的。由于检查点可以异步执行，所以该方法对流处理引擎性能的影响较小，但对于较大的应用，失败概率较高，并且失败后，需要暂停应用并回滚所有算子的状态。该方法时非侵入性的，并且需要增加的资源较少。

另外一个实现exactly-once的方法是通过实现至少一次的事件传输，加上在单个算子上实现重复数据删除。流处理引擎通过这个方法来重放失败的事件，并在事件进入每个算子之前删除重复的事件。该机制要求每个算子维护一份事务日志，以便每个算子可以跟踪某个事件是否已经被处理过了。该机制可能需要较多的资源，尤其时存储。为了删除重复，流处理引擎需要跟踪每个算子处理过的事件，这可能导致大量的数据需要跟踪，另外每个算子处理去重也可能导致性能压力。然而，该机制受应用大小的影响较小。使用分布式快照，任何一个算子的失败都可能导致全局的暂停和状态回滚，而使用后面的机制，失败的影响可能只局限在单个算子内。当某个算子失败的时候，没有被完整处理的事件只用从上游资源中重新提交即可，对其他算子的性能影响较小。

从上面的分析可以发现，没有流处理引擎可以保障精确单次处理。因为不可能保障算子中用户自定义的逻辑对单个事件仅仅执行一次。算子中的计算逻辑仍旧需要保证幂等性。流计算引擎所宣称的精确单次处理，实际是保障提交到流处理引擎所管理的状态的持久化后端的更新是精确单次的。如flink文档中所说：

Given that Flink recovers from faults by rewinding and replaying the source data streams, when the ideal situation is described as exactly once this does not mean that every event will be processed exactly once. Instead, it means that every event will affect the state being managed by Flink exactly once.

## ref

- [flink fault tolerance](https://ci.apache.org/projects/flink/flink-docs-release-1.12/learn-flink/fault_tolerance.html)
- [exactly once is not exactly the same](https://www.splunk.com/en_us/blog/it/exactly-once-is-not-exactly-the-same.html)
- [chandy-lamport algorithm](https://en.wikipedia.org/wiki/Chandy-Lamport_algorithm)
- [fault tolerance in flink](https://developer.aliyun.com/article/782826?spm=5176.12901015.0.i12901015.48df525cdpeS0Y)
- [flink与外部系统交互](https://mp.weixin.qq.com/s/7EuWV-4s69CiE0rRJb2Raw)
