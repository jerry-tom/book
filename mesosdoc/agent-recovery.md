## Agent恢复
&emsp;&emsp;如果主机上mesos-agent进程退出(可能由于mesos的bug，或操作者更新时杀掉进程)，由mesos-agent管理的任何executors/tasks将继续运行。当mesos-agent重新启动时，操作员可以控制如何处理这些旧的executors/tasks:

``` text
1、默认，所有旧的mesos-agent管理的executors/tasks进程被终止
2、如果framwork注册到mesos master时启动量checkpointing，任何属于这个framwork的executors可以重新连接到新的mesos-agent进程，并继续不间断运行
```
&emsp;&emsp;因此，启动framwork的checkpointing功能使任务能够容忍mesos-agent升级或意外的崩溃，而不会出现任何当机时间。
&emsp;&emsp;Agent恢复通过agent checkpoint信息(比如：Task Info， Executor Info, Status Updates)关于管理到本地磁盘的task和executors.如果framwork启动了checkpointing,任何随后重启的agent将恢复checkpointed信息，并重新连接任何正在运行的executors.
&emsp;&emsp;注意，如果agent运行的操作系统重启，所有运行在改主机上的executors和tasks被终止，并且当主机恢复时，不会自动重新运行。
&emsp;&emsp;但是，agent可以在主机重新启动之后恢复其agent id.假若agent的恢复运行到agent信息不匹配，这可能由于与重新启动相关的资源更改而发生，那么agent将回到恢复作为一个新的agent(存在这种行为)。在其他情况下，比如checkpointed资源(比如: persistent volumes)和agent的资源不相融，恢复将会失败(存在这种行为)。
### Framwork配置
&emsp;&emsp;framwork可以通过在向mesos master注册时，在其FramworkInfo中设置checkpoint标志来控制是否可以恢复其executors.启动此特性会导致运行该framwork启动tasks的每个agent的I/O开销增加。默认情况下，framwork不checkpoint它们的状态
### Agent配置
&emsp;&emsp;三个配置项控制mesos agent恢复行为:
* strict: 是否在严格模式下执行agent恢复[默认: true]
  * 如果strict=true,所有恢复错误都被认为是致命的。
  * 如果strict=false,恢复期间的任何错误(例如，checkpoint数据的损坏)被忽略，并恢复尽可能多的状态。
* recover: 是否恢复状态更新，并重新连接旧的executors[默认: reconnect]
  * 如果recover=reconnect,重连任何旧的存活的executors，只要executors的框架启动了checkpointing点。
  * 如果recover=cleanup,终止任何旧的存活的executors并退出。当执行不兼容的agent或executor升级时使用此选项。

``` text
注意：如果不存在checkpointing信息，则不执行恢复，并且agent作为一个新的agent注册到mesos master
```

* recovery_timeout: 为Agent恢复分配的时间[默认：15分钟]
  * 如果Agent需要比recovery_timeout更长的时间来恢复, 则任何等待重新连接agent的executors将自动终止。

``` text
注意：如果没有一个framwork启用了检查点，则当Agent die并且不会恢复时，agent中运行的executors和任务就会die。
```

&emsp;&emsp;重新启动的Agent应该在超时内重新注册到mesos master(默认为75秒：请参阅--max_agent_ping_timeouts和--agnet_ping_timeout配置标志)。如果Agent花费比此超时更长的时间来重新注册，则mesos master将关闭agent，从而将关闭任何存活的executors/tasks。因此，强烈建议自动执行重启agent进程(例如，使用诸如monit或systemd的进程管理)

### systemd和进程生命周期的已知问题
&emsp;&emsp;有一个已知问题，当使用systemd来启动mesos-agent时。该问题的描述可以在MESOS-3425中找到，并且所有相关工作都可以在MESOS-3007中进行跟踪。该问题在Mesos 0.25.0中修复了，在mesos容器中当启用cgroups隔离时。posix隔离和docker containerizer的进一步固定可用于0.25.1,0.26.1,0.27和0.28.0。
