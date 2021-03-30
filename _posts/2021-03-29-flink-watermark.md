---
title: 漫话flink watermarks及windows 
description: 漫话flink watermarks及windows
categories:
  - big-data
tags:
  - flink watermarks windows
---

流计算中，常会有一些聚合类的需求场景，如：统计过去一个小时内有多少用户购买了某个产品。flink通过引入windows来解决此类问题。windows本质是把原始的流数据切分成多个buckets，计算在单一的bucket中进行。对数据切分后，会引入新的问题，如数据的乱序和延迟等，flink通过watermarks/allowlateness/sideoutput来解决诸如此类的问题。总的来说，windows把流数据分块处理，watermarks则确定什么点不再等待更早的数据并触发window进行计算，allowlateness则允许将窗口的关闭时间再延迟多一段时间，最后使用sideoutput将window彻底关闭后无法处理的数据导出到其他地方处理。

## time in flink

flink支持event time、ingestion time、processing time，也就是事件时间、提取时间、处理时间。event time是事件在现实世界中发生的时间，通常由事件中时间戳描述；ingestion time是事件进入flink流处理系统的时间，也就是flink读取数据源的时间；processing time是数据流入到某个具体的算子时系统的具体时间，也就是flink处理该事件时的当前系统时间。通过env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);设置基本时间特性，定义数据流源得行为方式，及像KeyedStream.timeWindow(Time.seconds(30))这样的窗口操作应该使用哪种时间。

## windows

windows将流拆分成一个个大小有限的buckets，可在每个bucket中进行计算。当属于此window的第一个元素到达，就会创建一个window，当时间超过其结束时间加上用户指定的允许延时，window将被删除。

window有如下组件，window assigner：分配器，用来决定某个事件被分配到某个或某些window中。triger：触发器，决定一个窗口何时被计算或删除，触发策略可能类似于：窗口中事件数量超过一定量，watermark超过窗口结束时间等。evictor：剔除器，可在触发器触发之后，应用函数之前或之后从窗口中删除元素。window还拥有函数，如：processwindowfunction，reducefunction，aggregatefunction或foldfunction。函数包含应用于窗口内容的计算，而触发器则指定应用函数的条件。

窗口类型有：滚动窗口（tumbling window，无重叠）；滑动窗口（sliding window，有重叠）；会话窗口（session window，后动间隙）。滚动窗口分配器将每个事件分配固定窗口大小的窗口。滚动窗口大小固定且不重叠。滑动窗口分配器将事件分配给固定窗口大小的窗口，滑动窗口有两个参数，一个指定窗口的大小，另外一个指定窗口启动的频率。因此，如果滑动频率小于窗口大小，滑动窗口可重叠，事件被分配到多个窗口。会话窗口分配器配置会话间隙，定义所需的不活动时间长度，当时间段到期，当前会话关闭，后续事件被分配到新的会话窗口。

## watermarks

watermark本质上是一个时间戳，是基于已经收到的事件而估算是否还有事件未到达，它反应的是事件发生的时间，而不是处理时间。所以watermark应该被翻译成”水位线“而不是”水印“。

watermark是一种告诉flink一个事件延迟多少的方式，它定义了什么时候不用再等待更早的数据，watermarks在不断变化，它实际上是作为数据流的一部分随数据流流动，当opeartor收到watermarks时，它明白早于该时间的消息已经完全抵达，即假设不会再有时间时间早于水位线时间的事件到达了。只有水位线超过窗口对应的结束时间，窗口才会进行计算和关闭。

watermark设定的方法有：标点水位线（punctuated watermark）和周期水位线（periodic watermark）。标点水位线通过数据流中某些特殊标记事件来触发新水位线的生成，该方式下，窗口的触发和时间无关。周期水位线则周期性的产生watermark，如一定的时间间隔或一定数量的时间等。watermark提升的时间间隔由用户设定，在两次watermark间隔内会有一定的事件流入，用户可根据流入记录，计算新的watermark。

虽说watermark表明早于它的事件不应该再出现，但由于网络延迟、消息积积压等原因，接收到watermark之前的事件是不可避免的，这就是迟到事件。迟到事件出现时，window已经关闭并产出了计算结果，处理方法有：1，重新激活已经关闭的window并重新计算以修正结果；2，将迟到事件收集起来令行处理；3，将迟到事件视为错误事件并丢弃。flink默认丢弃迟到事件，另外两种使用allowlateness和sideoutput来处理。sideoutput可将迟到事件单独放到一个数据流分支，会作为window计算结果的副产品，以便用户获取并进行特殊处理。allowlateness允许用户设定一个最大延迟时间，flink会在窗口关闭后一直保持窗口状态直到超过允许的最大延迟时间，这期间迟到的事件不会被丢弃，而是默认触发窗口重新计算。由于保存window状态需要额外内存，并且如果window计算使用了processwindowfunction，还可能使得每个迟到事件触发一次窗口的全量计算，代价较大，所以迟到时间不宜太长，迟到事件不应过多，否则考虑降低watermark提高的速度或调整算法。

## 相关实现类

![watermark](/assets/images/classes/watermark.drawio.svg)

## ref

- [event time](https://ci.apache.org/projects/flink/flink-docs-release-1.12/dev/event_time.html)
- [watermark机制](https://cloud.tencent.com/developer/article/1693282)
