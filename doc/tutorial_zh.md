## Redis3.0 设计架构

<!-- vim-markdown-toc GFM -->

* [1 概述](#1-概述)
    * [1.1 背景](#11-背景)
    * [1.2 设计目标](#12-设计目标)
    * [1.3 名词解释](#13-名词解释)
    * [1.4 系统组件](#14-系统组件)
    * [1.5 整体架构](#15-整体架构)
        * [1.5.1 Controller](#151-controller)
            * [功能](#功能)
            * [工作原理](#工作原理)
        * [1.5.2 Proxy](#152-proxy)
            * [拓扑发现](#拓扑发现)
            * [后端 failover](#后端-failover)
            * [请求过程](#请求过程)
        * [1.5.3 RedisCluster](#153-rediscluster)
            * [多 IDC 支持](#多-idc-支持)
            * [同源增量同步](#同源增量同步)
* [2 OP 操作](#2-op-操作)
* [3 其他](#3-其他)

<!-- vim-markdown-toc -->

## 1 概述
### 1.1 背景

社区版本 RedisCluster 已经加入基于 Gossip 的集群化方案，各节点通过协议交换配置信息最终达到状态自维护的集群模式。但是由于 Ksarch 服务的产品线有跨地域需求，而跨地域的网络延迟和网络异常时常发生，进而导致基于 Gossip 的 Cluster 内部消息传递和状态判断就变得不可信，跨地域网络故障或网络划分会导致集群状态不可控。

### 1.2 设计目标

基于社区 RedisCluster 实现跨地域情况下的集群管理，可以实现自动 failover，实现平台化运维，状态全部收敛于 RedisCluster 内部，proxy 和 controller 组件全部是无状态，并实现在主从切换时同源增量同步，减小全量同步带来的网络带宽消耗和服务可用性下降。

### 1.3 名词解释

> * Redis3.0：ksarch-redis3.0
> * RedisCluster：社区版本 Redis3.0
> * Region：地域，bj,nj,hz,gz 等
> * Master Region：主地域，即 redis master 所在地域
> * MachineRoom：物理机房 cq01,nj02,nj03...
> * LogicMachineRoom：逻辑机房 jx,tc,nj,nj03,gz...
> * RegionLeader：地域的 controller leader
> * ClusterLeader：整个集群的 controller leader

![architecture](pic/rediscluster1.png)


### 1.4 系统组件

> * `RedisCluster`：社区发布 Redis3.0.3，增加多 IDC 支持，支持同源增量同步，支持关闭自动 Failover，支持读写权限配置。
> * `Twemproxy`：基于 Twitter 的 twemproxy 开发，增加多地域支持，添加 server 拓扑自动更新，增加 ACK 和 MOVED 协议用于短时间内 twemproxy 的集群拓扑和 rediscluster 集群拓扑不一致时的请求转发。
> * `Controller`：从 RedisCluster 获取集群状态，然后对集群状态进行判断，对本地域 PFAIL 节点进行判活，然后将本地域节点的状态发送给 ClusterLeader，最终由 ClusterLeader 进行向集群广播节点 FAIL 的消息。

### 1.5 整体架构

单集群多地域架构如图所示。
![arch](pic/rediscluster2.png)

#### 1.5.1 Controller
Controller 工作在地域级别，往往一个地域有多个逻辑机房或物理机房，一个地域可以部署多个 controller，但是同一地域内只有一个实际工作。当实际工作的 controller 挂掉后，集群会进行重新选择新主。

##### 功能
集群控制入口：

> * 节点读写权限控制
> * 节点主从切换
> * 主故障自动选主
> * 从故障自动封禁
> * 数据迁移控制
> * 数据 Rebalance

集群信息：
> * 信息采集
> * 信息处理
> * cli 工具和 Dashboard 提供集群信息和接口

##### 工作原理

Controller 在启动时会根据配置中指定的 redis seed 中的 server 中随机选择一个来获取集群拓扑，这组 seed 在生产环境中会配置成本机房内的 redis server 列表。Controller 会定时的进行集群拓扑的获取，随机选择一个 seed 进行初始创建本地域集群视图，并检查其他 seed 视角下的集群拓扑是否一致，如果不一致说明集群状态正在变化，需要进行重试。

原生 RedisCluster 集群中超过半数的节点一致认为不可用节点为 PFAIL 时会将该不可用节点标记为 FAIL，如果该节点是 Master，则会向整个集群进行广播节点不可用；如果 Slave 发现是自己的 Master 挂掉，则会发起投票选举新主。

修改后的 RedisCluster 集群中，当出现有的节点不可用后，内部通过 Gossip 协议会进行节点状态检查，其他节点会标记该不可用的节点状态为 PFAIL，该状态为本地视角。并且不会再进行后续的 PFAIL 到 FAIL 状态的转换，该部分操作可以配置为由 controller 自动操作或者需要手动触发完成，RedisCluster 集群只用来生成集群拓扑的本地视角。


#### 1.5.2 Proxy

基于 Twemproxy 开发，增加集群拓扑发现功能，启动时 proxy 会从 seed 列表中随机选择一个 server 来获取集群状态，然后通过 lua 脚本解析集群信息并创建拓扑结构和 slot 信息。

##### 拓扑发现
启动后 proxy 会间隔从 RedisCluster 通过 cluster nodes extra 命令来来获取集群信息，这组信息中包含了集群的主从关系和 slot 的分布，创建请求的路由信息。其中 replicaset 为一主多从结构，其中多个 slave 是根据逻辑机房分组。

##### 后端 failover

> * twemproxy 自带 failover：

配置文件 server_failure_limit 配置了后端连续失败几次后在一小段时间内剔除该节点。

> * 集群拓扑更新：对于请求的每个 key，会进行 hash 取模后对应于 16384 个 slot 中的某一个，该 slot 对应的后端 server replicaset，该 replicaset 是一主多从结构，对于写请求只能请求到 Master，对于读请求会根据定义的逻辑机房访问优先顺序从 replicaset 中选择一个 server。当从 RedisCluster 获取的节点的读写权限为 `-` 状态（总共有四种读写权限：`rw,r-,-w,-`)，则下次创建拓扑时会自动剔除该节点。

##### 请求过程
由于 proxy 是间隔从集群内部获取集群拓扑，所以可能存在某时间点 proxy 获取的集群状态和真正的状态不一致，所以 proxy 实现了 RedisCluster 内部的 ASK 和 MOVED 命令，当 proxy 路由信息和集群内部不一致时可以通过请求转发来完成请求。

> * `key->slot`: hash(key) mod 16384
> * `slot->server`: 写请求路由到 master，读请求按逻辑机房优先顺序选择 server


#### 1.5.3 RedisCluster
##### 多 IDC 支持

社区原生 RedisCluster 针对单地域设计，而我们的场景大多数都是一主多从的跨地域部署

> * Write 写主地域，然后同步给各地域的 Slave
> * Read 就近访问。

##### 同源增量同步
原生社区版本 Redis 增量同步能力有限，要求主从关系不变，并且连接中断时间有限情况下才可以增量同步。同源情况下切换主，slave 也需要进行全量同步。实现是为每个 slave 都留一个 backlog buffer，正常同步过来的写请求都会写 backlog buffer 和记录与 Master 断开时间点的 LastReplOffset 和 LastMasterRunid，当需要进行同步时先进行是否可以满足增量同步的条件：
> * 新 Master 的 LasterMasterRunid 与 Slave 请求 PSYNC 时发来的 MasterRunid 相同
> * 新 Master 的 LastReplOffset 大于等于 Slave 请求的 Offset
> * LastMasterRunid 存在时间小于 10 秒

## 2 OP 操作

[OP 操作手册](./op)

## 3 其他

[WIKI](https://github.com/meetbill/cc/wiki)
