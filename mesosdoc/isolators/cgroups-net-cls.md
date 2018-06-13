### 在Mesos Containerizer中支持cgroups 'net_cls'子系统

cgroups/net_cls隔离器允许操作者为一个mesos集群中的容器提供网络性能隔离和网络分段。要启用cgroups/net_cls隔离器，请在启动mesos agent时，附加cgroups/net_cls到—isolation标签。

