### 在Mesos Containerizer中的Cgroups `blkio` 子系统支持

&emsp;&emsp; groups/blkio 隔离器为容器通过 [the *blkio* Linux cgroup subsystem](blkio.md) 提供block I/O性能隔离。要启用这个隔离器，请在启动mesos agent前，将groups/blkio附加到—isolation标签。

&emsp;&emsp; blkio子系统支持I/O统计信息采集，并允许操作者应用块设备的I/O控制策略。隔离器将mesos 容器的进程放入到单独的blkio cgroup层次结构中。当前，仅支持上报容器在块设备上的I/O统计信息到mesos agent。一个样本统计信息将会是这样：

``` json
[{
    "executor_id": "executor",
    "executor_name": "name",
    "framework_id": "framework",
    "source": "source",
    "statistics": {
        "blkio": {
            "cfq": [
                {
                    "io_merged": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_queued": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_service_bytes": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_service_time": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_serviced": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_wait_time": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ]
                }
            ],
            "cfq_recursive": [
                {
                    "io_merged": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_queued": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_service_bytes": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_service_time": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_serviced": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ],
                    "io_wait_time": [
                        {
                            "op": "TOTAL",
                            "value": 0
                        }
                    ]
                }
            ],
            "throttling": [
                {
                    "device": {
                        "major": 8,
                        "minor": 0
                    },
                    "io_service_bytes": [
                        {
                            "op": "READ",
                            "value": 0
                        },
                        {
                            "op": "WRITE",
                            "value": 4096
                        },
                        {
                            "op": "SYNC",
                            "value": 0
                        },
                        {
                            "op": "ASYNC",
                            "value": 4096
                        },
                        {
                            "op": "TOTAL",
                            "value": 4096
                        }
                    ],
                    "io_serviced": [
                        {
                            "op": "READ",
                            "value": 0
                        },
                        {
                            "op": "WRITE",
                            "value": 1
                        },
                        {
                            "op": "SYNC",
                            "value": 0
                        },
                        {
                            "op": "ASYNC",
                            "value": 1
                        },
                        {
                            "op": "TOTAL",
                            "value": 1
                        }
                    ]
                },
                {
                    "io_service_bytes": [
                        {
                            "op": "TOTAL",
                            "value": 4096
                        }
                    ],
                    "io_serviced": [
                        {
                            "op": "TOTAL",
                            "value": 1
                        }
                    ]
                }
            ]
        },
        "cpus_limit": 1.1,
        "mem_limit_bytes": 167772160,
        "timestamp": 1500335339.30187
    }
}]
```

&emsp;&emsp;更多blkio子系统的详细信息，请参阅Linux内核文档[Block I/O Controller](blkio.md) 