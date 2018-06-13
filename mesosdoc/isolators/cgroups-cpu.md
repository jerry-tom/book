### 在Mesos Containerizer中支持cgroups 'cpu'和'cpuacct'子系统

cgroup/cpu隔离器允许操作员为Mesos集群内的容器提供CPU隔离和CPU使用情况。要开启cgroups/cpu隔离器，可以在mesos agent启动时，为—isolation标签指定cgroups/cpu。

#### 子系统

隔离器为了linux cgroups 开启cpu和cpuacct子系统，并且为每个通过mesos containerizer运行的容器指定cpu和cpuacct.

cgroups cpu子系统通过cgroups提供了两种限制cpu使用时间的机制:CFS shares和CFS 带宽控制。第一个(CFS shares)能够保证在系统负载很重时，一个cgroup最小数量的CPU“份额”。然而，当系统不忙时，它不能限制一个cgroup可得到的CPU时间片量。当你开启隔离器，这个机制一直有效。

第二个(CFS带宽控制)机制引入了一种方法来定义每个调度周期内cgroup使用CPU时间的上限。另外，输出带宽统计到cpu.stat文件中。CFS带宽机制可以通过mesos agent的—cgroups_enable_cfs标签开启。

cgroups cpuacct 子系统通过cgroups的任务提供cgroups任务使用情况的统计信息。当前，它只提供cpuacct.stat中的统计信息，以分别显示cgroup的任务在用户模式和内核模式下所花费的时间。

另外，隔离器基于cgroups.procs和tasks文件的信息提供容器中的进程和线程数量统计信息。

#### 使用CFS带宽限制时对应用中的影响

在调度期间，监视cpu使用情况，该时间段当前设置为100ms。一个任务在周期结束前，可以任意请求cpu时间。比如，如果一个任务分配了 1 CPUs，它可以使用2 CPU核50ms从而消费100ms CPU时间，并在剩余的50ms内进行节流。对延迟敏感的应用程序可能会因有效停止50ms而受到影响。

#### 统计

此隔离器会导出几个有价值的每个容器的CPU使用率度量，这些度量可以从mesos agent的监控/统计节点获取。

-  processes - cgroup中进程的数量
- threads - cgroup中线程的数量
- cpus_user_time_secs - 用户模式下cgroup任务花费的时间
- cpus_system_time_secs - 内核模式下cgroup任务花费的时间
- cpus_nr_periods - 已经过去的执行时间间隔的数量
- cpus_nr_throttled - cgroup被限制的次数
- cpus_throttled_time - cgroup中任务被限制的总时间

#### 配置

| 标志                                       | 说明                                       |
| ---------------------------------------- | ---------------------------------------- |
| —[no]-cgroups_cpu_enable_pids_and_tids_count | cgroup特性标志，用于开启统计容器中进程和线程。（默认：false）     |
| —[no]-cgroups_enable_cfs                 | cgroup特性标志，通过CFS带宽限制子功能开启cpu资源的硬限制。（默认：false） |
| —[no-]revocable_cpu_low_priority         | 运行带有可撤销cpu的容器，其优先级低于普通容器(不可撤销的cpu) 。（默认：true） |

 