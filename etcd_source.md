# 重点的源码目录加粗表示。client/clientv3如果不是想实现客户端的话可以不用看。
| 目录           | 说明                                            |
| -------------- | ----------------------------------------------- |
| __auth__       | 用户验证                                        |
| client         | V3之前的客户端                                  |
| clientv3       | V3版的客户端                                    |
| contrib        | 脚本例子，不属于etcd本身                        |
| Documentation  | Etcd文档                                        |
| embed          | 在应用内启动etcd的支持                          |
| etcdctl        | 控制台                                          |
| __etcdmain__   | 主入口                                          |
| __etcdserver__ | Etcd服务端                                      |
| functional     | 各组件的功能测试                                |
| hack           | 各种hack                                        |
| integration    | 集成测试                                        |
| __lease__      | 租约服务                                        |
| logos          |
| __mvcc__       | Etcd的多版本存储实现，提供KV支持的核心功能      |
| pkg            | Etcd依赖包                                      |
| proxy          | Grpc/http/tcp代理                               |
| __raft__       | Raft协议实现                                    |
| scripts        | 编译脚本                                        |
| security       | 一堆关于如何处理安全问题的文档…完全没意义的东西 |
| tests          | 大量使用Docker部署etcd的测试                    |
| tools          | Etcd的一些工具，如dumpdb，dumplogs，dumpmetrics |
| vendor         | 模块依赖，etcd开始转向go的模块依赖              |
| version        | 版本号                                          |
| __wal__        | write ahead log，实现事务的核心                 |


# 基本的逻辑关系如下

- main.go -> main() in etcdmain\main.go
  - main() calls 
    - checkSupportArch() in etcd.go
    - startEtcdOrProxyV2() in etcd.go
      - newConfig() in embed\config.go

| 目录         |   描述              |           | |
| ------------ | -------------- | --------- | ---- |
|main.go->\etcdmain\main.go main() |
| __etcdmain__ |       |
|              | main.go mian()||
|              | -> etcd.go checkSupportArch()||
|              | -> etcd.go startEtcdOrProxyV2() ||
|              | __etcdserver__ |           |   
|              |                | __raft__  |  
|              |                | __lease__ |   
|              |                | __mvcc__  |  

# 从入口到组件描述

- [etcdmain](etcd_src_main)

- [etcdserver](etcdserver_src)
    - [etcdserver\api](etcdserver_src_api)
      - [etcdserver\api\etcdhttp](etcdserver_src_api_etcdhttp)
- [auth](etcd_src_auth)