# etcd从入门到放弃
- [etcd简介](etcd-intro)
  
    名字etcd来源于distributed etc（Linux的文件夹/etc，存储各种配置信息的地方）

    1. etcd受zookeeper启发，由CoreOS于2013年6月开源，使用go语言实现， 最初被设计为解决集群管理中OS升级时的分布式并发控制、配置文件的存储和分发，是一个能提供一致性的KV存储系统。
    2. 基于etcd的特性，etcd还可以用来做服务发现、集群调度等等一些列服务
    3. kubernets使用了etcd，并且云计算公司大量的采用了etcd集群和kubernets集群。
    4. 发展（变化）迅速，随时升级，云计算公司积极参与代码的改进。
    5. 最后，利用类似zookeeper和etcd这种系统，可以方便的把已有的系统扩展为分布式系统。
   
- [etcd与zookeeper的不同](etcd-diff-zookeeper)
  - 集群内部的协议
    - zookeeper使用的是类似paxos协议，paxos协议的缺点是难以理解
    - etcd使用的是raft协议。raft协议的发明是为了解决paxos不说人话的问题。
  - 接口API
    - zookeeper有专门的客户端
    - etcd使用http(s)+json通讯，跨平台和语言
- [etcd上手](etcd-demo)
  
  上手比较简单，只有两个文件，一个是服务器etcd，一个是控制台etcdctl。照着doc即可以创建一个3个节点的集群。
  直接用go的客户端会比较好，其它的客户端不是维护者提供，存在一定的API不支持问题。

- [etcd应用](etcd-usage)
  
  基本与zk一样。
  
  - 服务发现
    - 服务应用x通过etcd客户端，使用租约模式向etcd写入一个key，值为服务应用x的信息，如ip、端口等等，设置一个TTL，定时更新TTL
    - 需要调用服务应用x的应用y，通过etcd找到这个key，获取对应的ip和端口等等信息，再建立连接
  - 消息订阅与发布
    - 应用配置的更新，启动时把所有配置载入，然后设置一个watcher，动态更新订阅者
    - 分布式系统节点的机器状态可以放在etcd中，用于通知客户端该机器目前空闲
    - 分布式系统的中间状态，如跑完某项任务后日志路径产生或者中间状态产生，可以通过watcher告诉订阅方
  - 分布式锁/队列
    - 几个的服务应用首先向同一个路径写入，如a向/testlock/写入<locka,某些值x>，b向/testlock/写入<lockb,某些值y>，c向/testlock写入<lock,某些值z>，接着a、b、c检查/testlock目录下的最小revision的项是不是自己写入的，如果是则获取锁成功，否则设置watcher等待最小revision的项被删除，再根据revision判断自己是不是获得了锁。例如，a、b、c写入后，locka的revision是7，lockb的revision是21，lockc的revision是3，那么c最先获取锁，a接着获取锁，b最后获取锁
  - 集群监控与Leader选举
    - 通过租约模式设置key的TTL值，超时没有心跳就表示该节点不可用。监控应用可以设置watcher订阅这些key，key消失后发邮件通知或者自动重启服务

- [etcd架构设计](etcd-arch)
  
  etcd的架构比较简单，raft协议实现 + 多版本并发控制 + boltdb存储 + grpc通讯， 然后是一个etcdserver将这几个模块合并在一起，通过http(s)向外界暴露API。
  
  1. etcd由一个主节点和N个冗余节点组成。非主节点要么是member，要么是learner，区别在于只有member可以竞选leader。

  2. 集群的状态由raft协议维护。冗余节点会定时向主节点发送心跳信息，若冗余节点收主节点的心跳信息超时，则认为主节点已经不可用，member们可以发起leader选举。同样的，如果主节点没收到节点的心跳信息，则认为该节点已经不可用。节点可以动态增加。集群的节点一般是奇数，3，5，7，9...以满足可用性需求。

  3. 写操作只能由主节点完成：主节点在写入自身的WAL后，会通知其它节点写入，然后告知客户端写入完成。因此，从冗余节点读数据的话，存在数据不一致的风险。存储引擎使用的是boltdb。
    所有的增删都需要通过WAL，以保持一致性。索引会写入到文件，用内存映射读取。同一个key会保留多个版本，好处是你爱用哪个版本用哪个，并发的时候内部处理比较轻松。坏处是磁盘IO会是个问题。

  4. 节点间的通信由grpc完成，一般的log用zap输出。

  5. etcd为Prometheus提供了勉强够用的信息采集接口，虽然留有信息采集接口扩展，但硬编码了Prometheus的采集。
   
  6. 集群的recover比较简单，直接从快照恢复。

  ![Image of arch](https://static001.infoq.cn/resource/image/31/b2/31ac4dbb96e2336a0a415250c2fce5b2.png)
- [etcd的raft实现](etcd-raft)
- [etcd的潜在问题](etcd-potenial-issues)

    以下几个问题来源于etcd自身的设计和工程管理。

  - 弱一致性
  
    这基本是采用了最终一致性作为解决办法的分布式系统必然会遇到的问题。这种最终一致性解决方案为了追求高可用，会异步的在执行写入的节点未全面完成写入之前，就告诉客户端写入已经完成。
    一旦出现网络波动，客户端的数据丢失是必然的，区别只是多还是少。

    解决办法：没有。这是设计上的取舍。
    
  - GC问题

    etcd没有预分配内存，动态申请内存在内存不足的情况下会因为GC产生可用性波动，这也是采用托管语言实现分布式系统必然会遇到的问题。无论是java/.net/go，都会有同样的问题。
    解决的办法是预申请内存，将数据空间预先留足。

  - 测试不足

    基本组件的测试案例不足以覆盖基本的情况。无论是基本的组件，还是兼容性测试，都缺乏足够的测试案例。有足够的回归测试，社区碰到的线上问题大概率不会发生。
    
    例如
    - [万级K8s集群背后etcd稳定性及性能优化实践](https://www.cnblogs.com/tencent-cloud-native/p/13614986.html)
    - [三年之久的 etcd 3 数据不一致 bug 分析](http://dockone.io/article/10077)

    解决办法有但是不可能实现——老老实实的设计测试方案和写测试案例。脏活累活还便宜竞争对手的活，谁干谁傻。

    由于etcd更新非常的频繁，如果etcd使用的范围被放大，那这个问题只会越来越严重。

  - 写放大问题
  
    etcd的MVCC一方面为同一个key提供多个版本的值，但同时不手动压缩的话，那么这些版本的key会一直存在。如果不注意使用，放在了一个写频繁的应用场景，那么在数据量大的时候容易出现性能问题。
    解决办法是重写事务处理的代码，实现最起码的的隔离（Read Committed)和合适的锁粒度。
  
  - 调试信息不足
  
    全靠谁踩坑谁补。和测试不足一样，脏活累活。

- [etcd源码概览](etcd-source-reading-in-genral)

    重点的源码目录加粗表示。client/clientv3如果不是想实现客户端的话可以不用看。

    | 目录          | 说明                                            |
    | ------------- | ----------------------------------------------- |
    | __auth__          | 用户验证                                        |
    | client        | V3之前的客户端                                  |
    | clientv3      | V3版的客户端                                    |
    | contrib       | 脚本例子，不属于etcd本身                        |
    | Documentation | Etcd文档                                        |
    | embed         | 在应用内启动etcd的支持                          |
    | etcdctl       | 控制台                                          |
    | __etcdmain__      | 主入口                                          |
    | __etcdserver__    | Etcd服务端                                      |
    | functional    | 各组件的功能测试                                |
    | hack          | 各种hack                                        |
    | integration   | 集成测试                                        |
    | __lease__         | 租约服务                                        |
    | logos         |
    | __mvcc__          | Etcd的多版本存储实现，提供KV支持的核心功能      |
    | pkg           | Etcd依赖包                                      |
    | proxy         | Grpc/http/tcp代理                               |
    | __raft__          | Raft协议实现                                    |
    | scripts       | 编译脚本                                        |
    | security      | 一堆关于如何处理安全问题的文档…完全没意义的东西 |
    | tests         | 大量使用Docker部署etcd的测试                    |
    | tools         | Etcd的一些工具，如dumpdb，dumplogs，dumpmetrics |
    | vendor        | 模块依赖，etcd开始转向go的模块依赖              |
    | version       | 版本号                                          |
    | __wal__           | write ahead log，实现事务的核心                 |