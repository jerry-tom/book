# Mesos 官方技术文档 #

> 本书的目的是同步翻译mesos document，方便开发人员及时掌握最新的动态信息，本书由私人团队维护，纯粹出于公益目的。如有任何第三方利用该翻译文档用作商业目的，本团队持有诉诸法律解决的权利。

## Mesos基本原理 ##

- [Mesos体系架构]()介绍了一些基本的概念
- [介绍Mesos的视频和幻灯片讲义]()

## 运行Mesos ##

- [Getting Started]() 入门指南介绍编译和安装Mesos的基本方法
- [Agent Recovery]() Agent故障恢复描述Agent无缝升级的方法以及在Agent出现故障crash的时候如何确保executor的安全
- [Authentication]() 鉴权
- [Authorization]()认证
- [Configuration]() 命令行参数的配置
- [Container Image]() 描述mesos容器如何支持镜像
- [Containerizer]() 容器技术概述以及使用场景
    - [Containerizer Internals]() Mesos容器实现细节
    - [Docker Containerizer]() 如何以任务的方式或者executor的方式启动docker镜像
    - [Mesos Containerizer]() Mesos默认的容器方式，支持linux和POSIX系统
        - [CNI support]() 容器网络接口
        - [Docker Volume Support]() Docker卷积挂载
- [Framework Rate Limiting]()
- [Task Health Checking]()
- [High Availability]()
- [HTTP Endpoints]()
- [Logging]()
- [Maintenance]()
- [Monitoring]()
- [Operational Guide]()
- [Roles]()
- [SSL]()
- [Tools]()
- [Upgrades]() 
- [Weights]()
- [Windows Support]() 

## 高级特性 ##

- [Attributes and Resources]() 
- [Fetcher Cache]() 
- [Multiple Disks]() 
- [Networking]() 
    - [Container Network Interface (CNI)](cni.md) 
    - [Port Mapping Isolator]() 
- [Nvidia GPU Support]() 
- [Oversubscription]() 
- [Persistent Volume ]() 
- [Quota ]() 
- [Replicated Log]() 
- [Reservation]() 
- [Shared Resources]() 

## API ##

- [API Client Libraries]()  列举了封装了http api接口的客户端库
- [Doxygen]()  C++的API文档
- [Executor HTTP API](executor-http-api.md)  描述executor和mesos Agent通讯使用的http API
- [Operator HTTP API]()  描述操作员和mesos master/Agent通讯用的http API
- [Scheduler HTTP API](scheduler-http-api.md)  描述scheduler和mesos master通讯使用的http API
- [Versioning]()  HTTP API及其发布版本

