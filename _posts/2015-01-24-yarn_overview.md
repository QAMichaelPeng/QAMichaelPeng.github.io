---
title:  翻译|YARN overview
layout: post
---
MapReduce在hadoop-0.23经过了彻底改造，现在变成了MapReduce 2.0(MRv2)，或者YARN。

YARN的基础理念是将JobTracker两个基本功能，资源管理与作业调度监控拆分到两个独立的守护进程中。一个全局的ResourceManager(RM)负责资源分配，每个应用有一个ApplicationMaster(AM)。一个应用可以是传统意思上的一个MapReduce作业，或者是一个有向无环的作业集合。

ResourceManager和每个节点上的NodeManager(NM)构成了数据计算框架。ResourceManager是在系统里所有的应用中分配资源的唯一权威。

每个应用的ApplicaitonMaster,实际上是框架相关的库。主要承担与ResourceManager协调资源和在NodeManager上执行和监控任务。

![Yarn Architecture](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/yarn_architecture.gif)

ResouceManager有两个主要组件：Scheduler和ApplicationsManager.

Scheduler负责在一定的限制条件，如容量，队列等之下为不同的应用分配资源。Scheduler是纯的scheduler，不负责监控或追踪应用状态。同样，它也不保证因为应用失败或硬件失败而重启应用。

Scheduler通过响应应用的资源请求来完成资源调度。这些都基于把memory, CPU, disk, network等抽象为一个资源容器(container)。在第一版中，只用到了内存资源。

Scheduler有一个可插拔的策略插件负责将集群资源在不同的队列和应用中分配。当前MapReduce的scheduler,如CapacityScheduler和FairScheduler都是这种插件的实例。

CapacityScheduler支持层次化队列(hierarchical)来使集群中的资源共享更加可预测。

ApplicationsManager负责处理作业提交，获得第一个容器以执行应用相关的ApplicationMaster并提供当ApplicationMaster失败时重启的服务。

NodeManager是每台机器上的框架代理，负责管理容器，监控容器的资源使用（cpu,内存，硬盘，网络)并向ResourceManager/Scheduler报告。

每应用ApplicationMaster负责从scheduler处获得容器所需的资源，跟踪容器状态和监控进度。

MRV2维护了和之前稳定版本(hadoop-1.x)的API兼容性。这意味着所有的Map-Reduce作业可以不需修改，在MRv2下重编译后即可运行。

本文翻译自 [Apache Hadoop NextGen MapReduce (YARN)](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html),遵循[Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).
