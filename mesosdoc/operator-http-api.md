## Operator HTTP API
&emsp;&emsp;Mesos 1.0.0添加了v1 Operator HTTP API的实验性支持。
### 概述
&emsp;&emsp;Mesos master和agent提供/api/v1端点作为operator相关操作的基本URL。
&emsp;&emsp;与scheduler和executor HTTP API类似，operator端点只接受HTTP POST请求。请求体应该采用JSON(Content-Type:application/json)或Protobuf(Content-Type:application/x-protobuf)编码。
&emsp;&emsp;对于Mesos可以同步并立即应答的请求，将发送状态码为200 OK的HTTP响应，可能包含以JSON或Protobuf编码的响应体。编码取决于请求中的Accept头(默认编码为JSON)。如果Accept-Encoding头设置为了"gzip"，响应内容将通过gzip压缩。
&emsp;&emsp;对于需要异步处理的请求(比如: RESERVE_RESOURCES)，将发送一个状态码为202 Accepted的HTTP响应。对于请求的结果，在(SUBSCRIBE)事件流中，RecordIO编码的HTTP响应被发送。目前，流式响应不支持gzip压缩。
### Master API
&emsp;&emsp;此API包含Mesos-master接受的所有call。这个信息的规范来源是master.proto(请注意:在测试API定稿前，protobuf的定义可能会改变)。这些call通常由人机操作员，工具或服务(例如Mesos WebUI)。当scheduler也可以使用这些调用时，预计scheduler将通过Scheduler HTTP API。
#### Calls To Master And Respones
&emsp;&emsp;以下是调用Mesos master的示例，结果来自API的同步响应。
#### GET_HEALTH
&emsp;&emsp;这个调用检索Mesos master的健康状况

``` text
GET_HEALTH HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_HEALTH"
}


GET_HEALTH HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_HEALTH",
  "get_health": {
    "healthy": true
  }
}

```

#### GET_FLAGS
&emsp;&emsp;这个调用检索Mesos-master整体的flag配置

``` text
GET_FLAGS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_FLAGS"
}


GET_FLAGS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_FLAGS",
  "get_flags": {
    "flags": [
      {
        "name": "acls",
        "value": ""
      },
      {
        "name": "agent_ping_timeout",
        "value": "15secs"
      },
      {
        "name": "agent_reregister_timeout",
        "value": "10mins"
      },
      {
        "name": "allocation_interval",
        "value": "1secs"
      },
      {
        "name": "allocator",
        "value": "HierarchicalDRF"
      },
      {
        "name": "authenticate_agents",
        "value": "true"
      },
      {
        "name": "authenticate_frameworks",
        "value": "true"
      },
      {
        "name": "authenticate_http_frameworks",
        "value": "true"
      },
      {
        "name": "authenticate_http_readonly",
        "value": "true"
      },
      {
        "name": "authenticate_http_readwrite",
        "value": "true"
      },
      {
        "name": "authenticators",
        "value": "crammd5"
      },
      {
        "name": "authorizers",
        "value": "local"
      },
      {
        "name": "credentials",
        "value": "/tmp/directory/credentials"
      },
      {
        "name": "framework_sorter",
        "value": "drf"
      },
      {
        "name": "help",
        "value": "false"
      },
      {
        "name": "hostname_lookup",
        "value": "true"
      },
      {
        "name": "http_authenticators",
        "value": "basic"
      },
      {
        "name": "http_framework_authenticators",
        "value": "basic"
      },
      {
        "name": "initialize_driver_logging",
        "value": "true"
      },
      {
        "name": "log_auto_initialize",
        "value": "true"
      },
      {
        "name": "logbufsecs",
        "value": "0"
      },
      {
        "name": "logging_level",
        "value": "INFO"
      },
      {
        "name": "max_agent_ping_timeouts",
        "value": "5"
      },
      {
        "name": "max_completed_frameworks",
        "value": "50"
      },
      {
        "name": "max_completed_tasks_per_framework",
        "value": "1000"
      },
      {
        "name": "quiet",
        "value": "false"
      },
      {
        "name": "recovery_agent_removal_limit",
        "value": "100%"
      },
      {
        "name": "registry",
        "value": "replicated_log"
      },
      {
        "name": "registry_fetch_timeout",
        "value": "1mins"
      },
      {
        "name": "registry_store_timeout",
        "value": "100secs"
      },
      {
        "name": "registry_strict",
        "value": "true"
      },
      {
        "name": "root_submissions",
        "value": "true"
      },
      {
        "name": "user_sorter",
        "value": "drf"
      },
      {
        "name": "version",
        "value": "false"
      },
      {
        "name": "webui_dir",
        "value": "/usr/local/share/mesos/webui"
      },
      {
        "name": "work_dir",
        "value": "/tmp/directory/master"
      },
      {
        "name": "zk_session_timeout",
        "value": "10secs"
      }
    ]
  }
}
```

#### GET_VERSION
&emsp;&emsp;这个调用检索Mesos master的版本信息

``` text
GET_VERSION HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_VERSION"
}


GET_VERSION HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_VERSION",
  "get_version": {
    "version_info": {
      "version": "1.0.0",
      "build_date": "2016-06-24 23:18:37",
      "build_time": 1466810317,
      "build_user": "root"
    }
  }
}
```

#### GET_METRICS
&emsp;&emsp;此调用给最终用户提供当前指标的快照。如果在这个调用中设置了timeout，则该超时将用于确定API做出响应的最长时间。如果超过了该时间，某些指标可能不会包含在响应中。

``` text
GET_METRICS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_METRICS",
  "get_metrics": {
    "timeout": {
      "nanoseconds": 5000000000
    }
  }
}


GET_METRICS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_METRICS",
  "get_metrics": {
    "metrics": [
      {
        "name": "allocator/event_queue_dispatches",
        "value": 1.0
      },
      {
        "name": "master/slaves_active",
        "value": 0.0
      },
      {
        "name": "allocator/mesos/resources/cpus/total",
        "value": 0.0
      },
      {
        "name": "master/messages_revive_offers",
        "value": 0.0
      },
      {
        "name": "allocator/mesos/allocation_runs",
        "value": 0.0
      },
      {
        "name": "master/mem_used",
        "value": 0.0
      },
      {
        "name": "master/valid_executor_to_framework_messages",
        "value": 0.0
      },
      {
        "name": "allocator/mesos/resources/mem/total",
        "value": 0.0
      },
      {
        "name": "log/recovered",
        "value": 1.0
      },
      {
        "name": "registrar/registry_size_bytes",
        "value": 123.0
      },
      {
        "name": "master/slaves_inactive",
        "value": 0.0
      },
      {
        "name": "master/messages_unregister_slave",
        "value": 0.0
      },
      {
        "name": "master/gpus_total",
        "value": 0.0
      },
      {
        "name": "master/disk_revocable_total",
        "value": 0.0
      },
      {
        "name": "master/gpus_percent",
        "value": 0.0
      },
      {
        "name": "master/mem_revocable_used",
        "value": 0.0
      },
      {
        "name": "master/slave_shutdowns_completed",
        "value": 0.0
      },
      {
        "name": "master/invalid_status_updates",
        "value": 0.0
      },
      {
        "name": "master/slave_removals",
        "value": 0.0
      },
      {
        "name": "master/messages_status_update",
        "value": 0.0
      },
      {
        "name": "master/messages_framework_to_executor",
        "value": 0.0
      },
      {
        "name": "master/cpus_revocable_percent",
        "value": 0.0
      },
      {
        "name": "master/recovery_slave_removals",
        "value": 0.0
      },
      {
        "name": "master/event_queue_dispatches",
        "value": 0.0
      },
      {
        "name": "master/messages_update_slave",
        "value": 0.0
      },
      {
        "name": "allocator/mesos/resources/mem/offered_or_allocated",
        "value": 0.0
      },
      {
        "name": "master/messages_register_framework",
        "value": 0.0
      },
      {
        "name": "master/cpus_percent",
        "value": 0.0
      },
      {
        "name": "master/slave_reregistrations",
        "value": 0.0
      },
      {
        "name": "master/cpus_revocable_total",
        "value": 0.0
      },
      {
        "name": "master/gpus_revocable_total",
        "value": 0.0
      },
      {
        "name": "master/valid_status_updates",
        "value": 0.0
      },
      {
        "name": "system/load_15min",
        "value": 1.25
      },
      {
        "name": "master/event_queue_http_requests",
        "value": 0.0
      },
      {
        "name": "master/messages_decline_offers",
        "value": 0.0
      },
      {
        "name": "master/tasks_staging",
        "value": 0.0
      },
      {
        "name": "master/messages_register_slave",
        "value": 0.0
      },
      {
        "name": "allocator/mesos/resources/disk/offered_or_allocated",
        "value": 0.0
      },
      {
        "name": "system/mem_free_bytes",
        "value": 2320146432.0
      },
      {
        "name": "system/cpus_total",
        "value": 4.0
      },
      {
        "name": "master/mem_percent",
        "value": 0.0
      },
      {
        "name": "master/event_queue_messages",
        "value": 0.0
      },
      {
        "name": "master/messages_reregister_slave",
        "value": 0.0
      },
      {
        "name": "master/gpus_used",
        "value": 0.0
      },
      {
        "name": "registrar/state_fetch_ms",
        "value": 16.787968
      },
      {
        "name": "master/messages_launch_tasks",
        "value": 0.0
      },
      {
        "name": "master/gpus_revocable_percent",
        "value": 0.0
      },
      {
        "name": "master/disk_percent",
        "value": 0.0
      },
      {
        "name": "system/load_1min",
        "value": 1.74
      },
      {
        "name": "registrar/queued_operations",
        "value": 0.0
      },
      {
        "name": "master/slaves_disconnected",
        "value": 0.0
      },
      {
        "name": "master/invalid_status_update_acknowledgements",
        "value": 0.0
      },
      {
        "name": "system/load_5min",
        "value": 1.65
      },
      {
        "name": "master/tasks_failed",
        "value": 0.0
      },
      {
        "name": "master/slave_registrations",
        "value": 0.0
      },
      {
        "name": "master/frameworks_connected",
        "value": 0.0
      },
      {
        "name": "allocator/mesos/event_queue_dispatches",
        "value": 0.0
      },
      {
        "name": "master/messages_executor_to_framework",
        "value": 0.0
      },
      {
        "name": "system/mem_total_bytes",
        "value": 8057147392.0
      },
      {
        "name": "master/cpus_revocable_used",
        "value": 0.0
      },
      {
        "name": "master/tasks_killing",
        "value": 0.0
      },
      {
        "name": "allocator/mesos/resources/cpus/offered_or_allocated",
        "value": 0.0
      },
      {
        "name": "master/messages_exited_executor",
        "value": 0.0
      },
      {
        "name": "master/valid_status_update_acknowledgements",
        "value": 0.0
      },
      {
        "name": "master/disk_used",
        "value": 0.0
      },
      {
        "name": "master/gpus_revocable_used",
        "value": 0.0
      },
      {
        "name": "master/disk_revocable_percent",
        "value": 0.0
      },
      {
        "name": "master/mem_revocable_percent",
        "value": 0.0
      },
      {
        "name": "master/invalid_executor_to_framework_messages",
        "value": 0.0
      },
      {
        "name": "master/slave_shutdowns_scheduled",
        "value": 0.0
      },
      {
        "name": "master/slave_removals/reason_registered",
        "value": 0.0
      },
      {
        "name": "master/messages_suppress_offers",
        "value": 0.0
      },
      {
        "name": "master/uptime_secs",
        "value": 0.038900992
      },
      {
        "name": "allocator/mesos/resources/disk/total",
        "value": 0.0
      },
      {
        "name": "master/slave_removals/reason_unregistered",
        "value": 0.0
      },
      {
        "name": "master/disk_total",
        "value": 0.0
      },
      {
        "name": "master/messages_resource_request",
        "value": 0.0
      },
      {
        "name": "master/cpus_total",
        "value": 0.0
      },
      {
        "name": "master/valid_framework_to_executor_messages",
        "value": 0.0
      },
      {
        "name": "master/cpus_used",
        "value": 0.0
      },
      {
        "name": "master/slave_removals/reason_unhealthy",
        "value": 0.0
      },
      {
        "name": "master/messages_kill_task",
        "value": 0.0
      },
      {
        "name": "master/slave_shutdowns_canceled",
        "value": 0.0
      },
      {
        "name": "master/messages_deactivate_framework",
        "value": 0.0
      },
      {
        "name": "master/messages_unregister_framework",
        "value": 0.0
      },
      {
        "name": "master/mem_revocable_total",
        "value": 0.0
      },
      {
        "name": "master/messages_reregister_framework",
        "value": 0.0
      },
      {
        "name": "master/dropped_messages",
        "value": 0.0
      },
      {
        "name": "master/invalid_framework_to_executor_messages",
        "value": 0.0
      },
      {
        "name": "master/tasks_error",
        "value": 0.0
      },
      {
        "name": "master/tasks_lost",
        "value": 0.0
      },
      {
        "name": "master/messages_reconcile_tasks",
        "value": 0.0
      },
      {
        "name": "master/tasks_killed",
        "value": 0.0
      },
      {
        "name": "master/tasks_finished",
        "value": 0.0
      },
      {
        "name": "master/frameworks_inactive",
        "value": 0.0
      },
      {
        "name": "master/tasks_running",
        "value": 0.0
      },
      {
        "name": "master/tasks_starting",
        "value": 0.0
      },
      {
        "name": "registrar/state_store_ms",
        "value": 5.55392
      },
      {
        "name": "master/mem_total",
        "value": 0.0
      },
      {
        "name": "master/outstanding_offers",
        "value": 0.0
      },
      {
        "name": "master/frameworks_active",
        "value": 0.0
      },
      {
        "name": "master/messages_authenticate",
        "value": 0.0
      },
      {
        "name": "master/disk_revocable_used",
        "value": 0.0
      },
      {
        "name": "master/frameworks_disconnected",
        "value": 0.0
      },
      {
        "name": "master/slaves_connected",
        "value": 0.0
      },
      {
        "name": "master/messages_status_update_acknowledgement",
        "value": 0.0
      },
      {
        "name": "master/elected",
        "value": 1.0
      }
    ]
  }
}
```

#### GET_LOGGING_LEVEL
&emsp;&emsp;此调用检索Mesos master的日志级别

``` text
GET_LOGGING_LEVEL HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_LOGGING_LEVEL"
}


GET_LOGGING_LEVEL HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_LOGGING_LEVEL",
  "get_logging_level": {
    "level": 0
  }
}
```

#### SET_LOGGING_LEVEL
&emsp;&emsp;设置Mesos master的指定持续时间的日志详细级别。Mesos日志记录使用glog。此库仅使用详细的日志记录，这意味着除非设置了详细的日志级别(默认为0，libprocess使用级别1，2和3)，否则不会输出任何内容。

``` text
SET_LOGGING_LEVEL HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "SET_LOGGING_LEVEL",
  "set_logging_level": {
    "duration": {
      "nanoseconds": 60000000000
    },
    "level": 1
  }
}


SET_LOGGING_LEVEL HTTP Response:

HTTP/1.1 202 Accepted
```

#### LIST_FILE
&emsp;&emsp;该调用检索Mesos-master目录文件列表。

``` text
LIST_FILES HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "LIST_FILES",
  "list_files": {
    "path": "one/"
  }
}


LIST_FILES HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "LIST_FILES",
  "list_files": {
    "file_infos": [
      {
        "gid": "root",
        "mode": 16877,
        "mtime": {
          "nanoseconds": 1470820172000000000
        },
        "nlink": 2,
        "path": "one/2",
        "size": 4096,
        "uid": "root"
      },
      {
        "gid": "root",
        "mode": 16877,
        "mtime": {
          "nanoseconds": 1470820172000000000
        },
        "nlink": 2,
        "path": "one/3",
        "size": 4096,
        "uid": "root"
      },
      {
        "gid": "root",
        "mode": 33188,
        "mtime": {
          "nanoseconds": 1470820172000000000
        },
        "nlink": 1,
        "path": "one/two",
        "size": 3,
        "uid": "root"
      }
    ]
  }
}
```

#### READ_FILE
&emsp;&emsp;从文件中读取数据。该调用需要在Mesos-master读取文件的路径，开始读取的位置和最大读取字节长度。

``` text
READ_FILE HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "READ_FILE",
  "read_file": {
    "length": 6,
    "offset": 1,
    "path": "myname"
  }
}


READ_FILE HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "READ_FILE",
  "read_file": {
    "data": "b2R5",
    "size": 4
  }
}
```

#### GSE_STATE
&emsp;&emsp;此调用检索整个集群状态

``` text
GET_STATE HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_STATE"
}


GET_STATE HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_STATE",
  "get_state": {
    "get_agents": {
      "agents": [
        {
          "active": true,
          "agent_info": {
            "hostname": "myhost",
            "id": {
              "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-S0"
            },
            "port": 34626,
            "resources": [
              {
                "name": "cpus",
                "role": "*",
                "scalar": {
                  "value": 2.0
                },
                "type": "SCALAR"
              },
              {
                "name": "mem",
                "role": "*",
                "scalar": {
                  "value": 1024.0
                },
                "type": "SCALAR"
              },
              {
                "name": "disk",
                "role": "*",
                "scalar": {
                  "value": 1024.0
                },
                "type": "SCALAR"
              },
              {
                "name": "ports",
                "ranges": {
                  "range": [
                    {
                      "begin": 31000,
                      "end": 32000
                    }
                  ]
                },
                "role": "*",
                "type": "RANGES"
              }
            ]
          },
          "pid": "slave(3)@127.0.1.1:34626",
          "registered_time": {
            "nanoseconds": 1470820172046531840
          },
          "total_resources": [
            {
              "name": "cpus",
              "role": "*",
              "scalar": {
                "value": 2.0
              },
              "type": "SCALAR"
            },
            {
              "name": "mem",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "disk",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "ports",
              "ranges": {
                "range": [
                  {
                    "begin": 31000,
                    "end": 32000
                  }
                ]
              },
              "role": "*",
              "type": "RANGES"
            }
          ],
          "version": "1.1.0"
        }
      ]
    },
    "get_executors": {
      "executors": [
        {
          "agent_id": {
            "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-S0"
          },
          "executor_info": {
            "command": {
              "shell": true,
              "value": ""
            },
            "executor_id": {
              "value": "default"
            },
            "framework_id": {
              "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-0000"
            }
          }
        }
      ]
    },
    "get_frameworks": {
      "frameworks": [
        {
          "active": true,
          "connected": true,
          "framework_info": {
            "checkpoint": false,
            "failover_timeout": 0.0,
            "hostname": "abcdev",
            "id": {
              "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-0000"
            },
            "name": "default",
            "principal": "my-principal",
            "role": "*",
            "user": "root"
          },
          "registered_time": {
            "nanoseconds": 1470820172039300864
          },
          "reregistered_time": {
            "nanoseconds": 1470820172039300864
          }
        }
      ]
    },
    "get_tasks": {
      "completed_tasks": [
        {
          "agent_id": {
            "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-S0"
          },
          "executor_id": {
            "value": "default"
          },
          "framework_id": {
            "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-0000"
          },
          "name": "test-task",
          "resources": [
            {
              "name": "cpus",
              "role": "*",
              "scalar": {
                "value": 2.0
              },
              "type": "SCALAR"
            },
            {
              "name": "mem",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "disk",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "ports",
              "ranges": {
                "range": [
                  {
                    "begin": 31000,
                    "end": 32000
                  }
                ]
              },
              "role": "*",
              "type": "RANGES"
            }
          ],
          "state": "TASK_FINISHED",
          "status_update_state": "TASK_FINISHED",
          "status_update_uuid": "IWjmPnfgQCWxGVlNNwctcg==",
          "statuses": [
            {
              "agent_id": {
                "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-S0"
              },
              "container_status": {
                "network_infos": [
                  {
                    "ip_addresses": [
                      {
                        "ip_address": "127.0.1.1"
                      }
                    ]
                  }
                ]
              },
              "executor_id": {
                "value": "default"
              },
              "source": "SOURCE_EXECUTOR",
              "state": "TASK_RUNNING",
              "task_id": {
                "value": "eb5cb680-a998-4605-8811-e79db8734c02"
              },
              "timestamp": 1470820172.07315,
              "uuid": "hTaLQ0b5Q1OZuab7QclTKQ=="
            },
            {
              "agent_id": {
                "value": "628984d0-4213-4140-bcb0-99d7ef46b1df-S0"
              },
              "container_status": {
                "network_infos": [
                  {
                    "ip_addresses": [
                      {
                        "ip_address": "127.0.1.1"
                      }
                    ]
                  }
                ]
              },
              "executor_id": {
                "value": "default"
              },
              "source": "SOURCE_EXECUTOR",
              "state": "TASK_FINISHED",
              "task_id": {
                "value": "eb5cb680-a998-4605-8811-e79db8734c02"
              },
              "timestamp": 1470820172.09382,
              "uuid": "IWjmPnfgQCWxGVlNNwctcg=="
            }
          ],
          "task_id": {
            "value": "eb5cb680-a998-4605-8811-e79db8734c02"
          }
        }
      ]
    }
  }
}
```

#### GET_AGENTS
&emsp;&emsp;此调用检索Mesos-master已知的所有agent信息

``` text
GET_AGENTS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_AGENTS"
}


GET_AGENTS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{{
  "type": "GET_AGENTS",
  "get_agents": {
    "agents": [
      {
        "active": true,
        "agent_info": {
          "hostname": "host",
          "id": {
            "value": "3669ea49-c3c4-4b13-adee-05b8f9cb2562-S0"
          },
          "port": 34626,
          "resources": [
            {
              "name": "cpus",
              "role": "*",
              "scalar": {
                "value": 2.0
              },
              "type": "SCALAR"
            },
            {
              "name": "mem",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "disk",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "ports",
              "ranges": {
                "range": [
                  {
                    "begin": 31000,
                    "end": 32000
                  }
                ]
              },
              "role": "*",
              "type": "RANGES"
            }
          ]
        },
        "pid": "slave(1)@127.0.1.1:34626",
        "registered_time": {
          "nanoseconds": 1470820171393027072
        },
        "total_resources": [
          {
            "name": "cpus",
            "role": "*",
            "scalar": {
              "value": 2.0
            },
            "type": "SCALAR"
          },
          {
            "name": "mem",
            "role": "*",
            "scalar": {
              "value": 1024.0
            },
            "type": "SCALAR"
          },
          {
            "name": "disk",
            "role": "*",
            "scalar": {
              "value": 1024.0
            },
            "type": "SCALAR"
          },
          {
            "name": "ports",
            "ranges": {
              "range": [
                {
                  "begin": 31000,
                  "end": 32000
                }
              ]
            },
            "role": "*",
            "type": "RANGES"
          }
        ],
        "version": "1.1.0"
      }
    ]
  }
}
```

#### GET_FRAMEWORKS
&emsp;&emsp;此调用检索Mesos-master已知的所有framwork的信息。

``` text
GET_FRAMEWORKS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_FRAMEWORKS"
}


GET_FRAMEWORKS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_FRAMEWORKS",
  "get_frameworks": {
    "frameworks": [
      {
        "active": true,
        "connected": true,
        "framework_info": {
          "checkpoint": false,
          "failover_timeout": 0.0,
          "hostname": "myhost",
          "id": {
            "value": "361be53a-4d1b-42c1-bec3-e3979eff90bd-0000"
          },
          "name": "default",
          "principal": "my-principal",
          "role": "*",
          "user": "root"
        },
        "registered_time": {
          "nanoseconds": 1470820171578306816
        },
        "reregistered_time": {
          "nanoseconds": 1470820171578306816
        }
      }
    ]
  }
}
```

#### GET_EXECUTORS
&emsp;&emsp;查询Mesos-master已知的所有executors

``` text
GET_EXECUTORS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_EXECUTORS"
}


GET_EXECUTORS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_EXECUTORS",
  "get_executors": {
    "executors": [
      {
        "agent_id": {
          "value": "f2ddc41d-6284-405e-8642-34953093140f-S0"
        },
        "executor_info": {
          "command": {
            "shell": true,
            "value": "exit 1"
          },
          "executor_id": {
            "value": "default"
          },
          "framework_id": {
            "value": "f2ddc41d-6284-405e-8642-34953093140f-0000"
          }
        }
      }
    ]
  }
}
```

#### GET_TASKS
&emsp;&emsp;查询Mesos-master已知的所有任务

``` text
GET_TASKS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_TASKS"
}


GET_TASKS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_TASKS",
  "get_tasks": {
    "tasks": [
      {
        "agent_id": {
          "value": "d4bd102f-e25f-46dc-bb5d-8b10bca133d8-S0"
        },
        "executor_id": {
          "value": "default"
        },
        "framework_id": {
          "value": "d4bd102f-e25f-46dc-bb5d-8b10bca133d8-0000"
        },
        "name": "test",
        "resources": [
          {
            "name": "cpus",
            "role": "*",
            "scalar": {
              "value": 2.0
            },
            "type": "SCALAR"
          },
          {
            "name": "mem",
            "role": "*",
            "scalar": {
              "value": 1024.0
            },
            "type": "SCALAR"
          },
          {
            "name": "disk",
            "role": "*",
            "scalar": {
              "value": 1024.0
            },
            "type": "SCALAR"
          },
          {
            "name": "ports",
            "ranges": {
              "range": [
                {
                  "begin": 31000,
                  "end": 32000
                }
              ]
            },
            "role": "*",
            "type": "RANGES"
          }
        ],
        "state": "TASK_RUNNING",
        "status_update_state": "TASK_RUNNING",
        "status_update_uuid": "ycLTRBo8TjKFTrh4vsBERg==",
        "statuses": [
          {
            "agent_id": {
              "value": "d4bd102f-e25f-46dc-bb5d-8b10bca133d8-S0"
            },
            "container_status": {
              "network_infos": [
                {
                  "ip_addresses": [
                    {
                      "ip_address": "127.0.1.1"
                    }
                  ]
                }
              ]
            },
            "executor_id": {
              "value": "default"
            },
            "source": "SOURCE_EXECUTOR",
            "state": "TASK_RUNNING",
            "task_id": {
              "value": "1"
            },
            "timestamp": 1470820172.32565,
            "uuid": "ycLTRBo8TjKFTrh4vsBERg=="
          }
        ],
        "task_id": {
          "value": "1"
        }
      }
    ]
  }
}
```

#### GET_ROLES
&emsp;&emsp;查询roles信息

``` text
GET_ROLES HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_ROLES"
}


GET_ROLES HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_ROLES",
  "get_roles": {
    "roles": [
      {
        "name": "*",
        "weight": 1.0
      },
      {
        "frameworks": [
          {
            "value": "74bddcbc-4a02-4d64-b291-aed52032055f-0000"
          }
        ],
        "name": "role1",
        "resources": [
          {
            "name": "cpus",
            "role": "role1",
            "scalar": {
              "value": 0.5
            },
            "type": "SCALAR"
          },
          {
            "name": "mem",
            "role": "role1",
            "scalar": {
              "value": 512.0
            },
            "type": "SCALAR"
          },
          {
            "name": "ports",
            "ranges": {
              "range": [
                {
                  "begin": 31000,
                  "end": 31001
                }
              ]
            },
            "role": "role1",
            "type": "RANGES"
          },
          {
            "name": "disk",
            "role": "role1",
            "scalar": {
              "value": 1024.0
            },
            "type": "SCALAR"
          }
        ],
        "weight": 2.5
      }
    ]
  }
}
```

#### GET_WEIGHTS
&emsp;&emsp;此调用检索role权重信息

``` text
GET_WEIGHTS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_WEIGHTS"
}


GET_WEIGHTS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_WEIGHTS",
  "get_weights": {
    "weight_infos": [
      {
        "role": "role",
        "weight": 2.0
      }
    ]
  }
}
```
#### UPDATE_WEIGHTS
&emsp;&emsp;此调用更新指定role的权重。

``` text
UPDATE_WEIGHTS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "UPDATE_WEIGHTS",
  "update_weights": {
    "weight_infos": [
      {
        "role": "role",
        "weight": 4.0
      }
    ]
  }
}


UPDATE_WEIGHTS HTTP Response:

HTTP/1.1 202 Accepted
```

#### GET_MASTER
&emsp;&emsp;此调用检索Mesos-master信息

``` text
GET_MASTER HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_MASTER"
}


GET_MASTER HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_MASTER",
  "get_master": {
    "master_info": {
      "address": {
        "hostname": "myhost",
        "ip": "127.0.1.1",
        "port": 34626
      },
      "hostname": "myhost",
      "id": "310ffdac-0b73-408d-acf0-2adcd21cb4b7",
      "ip": 16842879,
      "pid": "master@127.0.1.1:34626",
      "port": 34626,
      "version": "1.1.0"
    }
  }
}
```

#### RESERVE_RESOURCES
&emsp;&emsp;此调用在特定Agent上动态预留资源。此调用需要agent_id和资源的详细信息。如下：
``` text
RESERVE_RESOURCES HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "RESERVE_RESOURCES",
  "reserve_resources": {
    "agent_id": {
      "value": "1557de7d-547c-48db-b5d3-6bef9c9640ef-S0"
    },
    "resources": [
      {
        "type": "SCALAR",
        "name": "cpus",
        "reservation": {
          "principal": "my-principal"
        },
        "role": "role",
        "scalar": {
          "value": 1.0
        }
      },
      {
        "type": "SCALAR",
        "name": "mem",
        "reservation": {
          "principal": "my-principal"
        },
        "role": "role",
        "scalar": {
          "value": 512.0
        }
      }
    ]
  }
}

RESERVE_RESOURCES HTTP Response:

HTTP/1.1 202 Accepted
```

#### UNRESERVE_RESOURCES
&emsp;&emsp;此调用在特定的Agent上不保留动态资源。此调用需要agent_id和资源的详细信息，如下：

``` text
UNRESERVE_RESOURCES HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "UNRESERVE_RESOURCES",
  "unreserve_resources": {
    "agent_id": {
      "value": "1557de7d-547c-48db-b5d3-6bef9c9640ef-S0"
    },
    "resources": [
      {
        "type": "SCALAR",
        "name": "cpus",
        "reservation": {
          "principal": "my-principal"
        },
        "role": "role",
        "scalar": {
          "value": 1.0
        }
      },
      {
        "type": "SCALAR",
        "name": "mem",
        "reservation": {
          "principal": "my-principal"
        },
        "role": "role",
        "scalar": {
          "value": 512.0
        }
      }
    ]
  }
}


UNRESERVE_RESOURCES HTTP Response:

HTTP/1.1 202 Accepted
```

#### CREATE_VOLUMES
&emsp;&emsp;此调用在预留资源上创建持久卷。该请求被异步转发给预留资源所在的Agent。异步消息可能没有被交付或在Agent上创建卷可能失败。此调用需要Agent_id或卷的详细信息，如下：

``` text
CREATE_VOLUMES HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "CREATE_VOLUMES",
  "create_volumes": {
    "agent_id": {
      "value": "919141a8-b434-4946-86b9-e1b65c8171f6-S0"
    },
    "volumes": [
      {
        "type": "SCALAR",
        "disk": {
          "persistence": {
            "id": "id1",
            "principal": "my-principal"
          },
          "volume": {
            "container_path": "path1",
            "mode": "RW"
          }
        },
        "name": "disk",
        "role": "role1",
        "scalar": {
          "value": 64.0
        }
      }
    ]
  }
}


CREATE_VOLUMES HTTP Response:

HTTP/1.1 202 Accepted
```

#### DESTROY_VOLUMES
&emsp;&emsp;该调用销毁持久卷。该请求被异步转发到预留资源所在的Mesos Agent

``` text
DESTROY_VOLUMES HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "DESTROY_VOLUMES",
  "destroy_volumes": {
    "agent_id": {
      "value": "919141a8-b434-4946-86b9-e1b65c8171f6-S0"
    },
    "volumes": [
      {
        "disk": {
          "persistence": {
            "id": "id1",
            "principal": "my-principal"
          },
          "volume": {
            "container_path": "path1",
            "mode": "RW"
          }
        },
        "name": "disk",
        "role": "role1",
        "scalar": {
          "value": 64.0
        },
        "type": "SCALAR"
      }
    ]
  }
}


DESTROY_VOLUMES HTTP Response:

HTTP/1.1 202 Accepted
```

#### GET_MAINTENANCE_STATUS
&emsp;&emsp;此调用检索集群的维护状态

``` text
GET_MAINTENANCE_STATUS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_MAINTENANCE_STATUS"
}


GET_MAINTENANCE_STATUS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_MAINTENANCE_STATUS",
  "get_maintenance_status": {
    "status": {
      "draining_machines": [
        {
          "id": {
            "ip": "0.0.0.2"
          }
        },
        {
          "id": {
            "hostname": "myhost"
          }
        }
      ]
    }
  }
}
```

#### GET_MAINTENANCE_SCHEDULE
&emsp;&emsp;此调用检索集群的维护计划

``` text
GET_MAINTENANCE_SCHEDULE HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_MAINTENANCE_SCHEDULE"
}


GET_MAINTENANCE_SCHEDULE HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_MAINTENANCE_SCHEDULE",
  "get_maintenance_schedule": {
    "schedule": {
      "windows": [
        {
          "machine_ids": [
            {
              "hostname": "myhost"
            },
            {
              "ip": "0.0.0.2"
            }
          ],
          "unavailability": {
            "start": {
              "nanoseconds": 1470849373150643200
            }
          }
        }
      ]
    }
  }
}
```

#### UPDATE_MAINTENANCE_SCHEDULE
&emsp;&emsp;此调用更新集群的维护计划

``` text
UPDATE_MAINTENANCE_SCHEDULE HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "UPDATE_MAINTENANCE_SCHEDULE",
  "update_maintenance_schedule": {
    "schedule": {
      "windows": [
        {
          "machine_ids": [
            {
              "hostname": "myhost"
            },
            {
              "ip": "0.0.0.2"
            }
          ],
          "unavailability": {
            "start": {
              "nanoseconds": 1470820233192017920
            }
          }
        }
      ]
    }
  }
}


UPDATE_MAINTENANCE_SCHEDULE HTTP Response:

HTTP/1.1 202 Accepted
```

#### START_MAINTENANCE
&emsp;&emsp;此调用启动集群的维护，这将使一组机器失效

``` text
START_MAINTENANCE HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "START_MAINTENANCE",
  "start_maintenance": {
    "machines": [
      {
        "hostname": "myhost",
        "ip": "0.0.0.3"
      }
    ]
  }
}


START_MAINTENANCE HTTP Response:

HTTP/1.1 202 Accepted
```

#### STOP_MAINTENANCE
&emsp;&emsp;停止集群的维护，这将使一组机器备份

``` text
STOP_MAINTENANCE HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "STOP_MAINTENANCE",
  "stop_maintenance": {
    "machines": [
      {
        "hostname": "myhost",
        "ip": "0.0.0.3"
      }
    ]
  }
}


STOP_MAINTENANCE HTTP Response:

HTTP/1.1 202 Accepted
```

#### GET_QUOTA
&emsp;&emsp;此调用检索集群配置的配额

``` text
GET_QUOTA HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "GET_QUOTA"
}


GET_QUOTA HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_QUOTA",
  "get_quota": {
    "status": {
      "infos": [
        {
          "guarantee": [
            {
              "name": "cpus",
              "role": "*",
              "scalar": {
                "value": 1.0
              },
              "type": "SCALAR"
            },
            {
              "name": "mem",
              "role": "*",
              "scalar": {
                "value": 512.0
              },
              "type": "SCALAR"
            }
          ],
          "principal": "my-principal",
          "role": "role1"
        }
      ]
    }
  }
}
```

#### SET_QUOTA
&emsp;&emsp;此调用设置由特定角色使用的资源的配额

``` text
SET_QUOTA HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "SET_QUOTA",
  "set_quota": {
    "quota_request": {
      "force": true,
      "guarantee": [
        {
          "name": "cpus",
          "role": "*",
          "scalar": {
            "value": 1.0
          },
          "type": "SCALAR"
        },
        {
          "name": "mem",
          "role": "*",
          "scalar": {
            "value": 512.0
          },
          "type": "SCALAR"
        }
      ],
      "role": "role1"
    }
  }
}


SET_QUOTA HTTP Response:

HTTP/1.1 202 Accepted
```

#### REMOVE_QUOTA
&emsp;&emsp;此调用移除特定角色的配额

``` text
REMOVE_QUOTA HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "REMOVE_QUOTA",
  "remove_quota": {
    "role": "role1"
  }
}


REMOVE_QUOTA HTTP Response:

HTTP/1.1 202 Accepted
```

#### Evnets
&emsp;&emsp;当前，仅仅当向mesos master api发送SUBSCRIBE呼叫时,调用此接口导致流式相应。

``` text
SUBSCRIBE Request (JSON):

POST /api/v1  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json

{
  "type": "SUBSCRIBE"
}

SUBSCRIBE Response Event (JSON):
HTTP/1.1 200 OK

Content-Type: application/json
Transfer-Encoding: chunked

<event-length>
{
  "type": "SUBSCRIBED",

  "subscribed" : {
    "get_state" : {...}
  }
}
<more events>
```
&emsp;&emsp;即使获取SUBSCRIBED HTTP响应事件后，客户端仍需保持与事件节点的持久连接。通过"Connection: keep-alive"和"Transfer-Encoding: chunked"标题，同时没有"Content-Length".所有mesos产生的订阅事件在这条连接上流式传输。Mesos master使用RecordIO格式编码每个事件，比如：字符串表示事件的长度，以字节为单位，后跟JSON或二进制Protobuf编码事件。
&emsp;&emsp;以下事件当前由mesos master发出。这个信息的规范来源是master.proto。请注意，当发送JSON编码事件时，master将以Base64编码原始字节，以UTF-8编码字符串.

##### SUBSCRIBED
&emsp;&emsp;这是master发送的第一个事件，当客户端通过长连接发送subscribed请求时。这包括集群状态的快照。有关详情，请参阅上面的SUBSCRIBE。对集群状态的后续更改可能会导致更多事件(目前仅支持TASK_ADDED和TASK_UPDATED)

##### TASK_ADDED
&emsp;&emsp;每当任务被添加到mesos master时发送当一个新的任务被mesos master处理时，或当agent故障转移到mesos master时，可能会发生这种情况。

``` text
TASK_ADDED Event (JSON)

<event-length>
{
  "type": "TASK_ADDED",

  "task_added": {
    "task": {
      "name": "dummy-task",
      "task_id": {
        "value": "d40f3f3e-bbe3-44af-a230-4cb1eae72f67"
      },
      "agent_id": {
        "value": "f1c9cdc5-195e-41a7-a0d7-adaa9af07f81"
      },
      "command": {
        "value": "sleep",
        "arguments": [
          "100"
        ]
      }
    }
  }
}
```

##### TASK_UPDATED
&emsp;&emsp;当任务在mesos master中的状态发生变化时发送。当mesos master收到一个状态更新或产生一个状态时，可能发生这种情况。即使状态更新被agent重发，但是并不是mesos master收到的所有状态更新都会导致发送事件。

``` text
TASK_UPDATED Event (JSON)

<event-length>
{
  "type": "TASK_UPDATED",

  "task_updated": {
    "task_id": {
        "value": "42154f1b-adcd-4421-bf13-8bd11adfafaf"
    },

    "framework_id": {
        "value": "49154f1b-8cf6-4421-bf13-8bd11dccd1f1"
    },

    "agent_id": {
        "value": "2915adf-8aff-4421-bf13-afdafaf1f1"
    },

    "executor_id": {
        "value": "adfaf-adff-2421-bf13-adf23tafa21"
    },

    "state" : "TASK_RUNNING"
  }
}
```

##### FRAMWORK_ADDED
&emsp;&emsp;每当mesos master知道framwork时发送。当一个新的framwork注册到master时，可能会发生这种情况。

``` json
FRAMEWORK_ADDED Event (JSON)

<event-length>
{
  "type": "FRAMEWORK_ADDED",

  "framework_added": {
    "framework": {
      "active": true,
      "allocated_resources": [],
      "connected": true,
      "framework_info": {
        "capabilities": [
          {
            "type": "RESERVATION_REFINEMENT"
          }
        ],
        "checkpoint": true,
        "failover_timeout": 0,
        "id": {
          "value": "a9ba2984-99c4-4183-8cd1-f7313426e21c-0147"
        },
        "name": "inverse-offer-example-framework",
        "role": "*",
        "user": "root"
      },
      "inverse_offers": [],
      "recovered": false,
      "registered_time": {
        "nanoseconds": 1501191957829317120
      },
      "reregistered_time": {
        "nanoseconds": 1501191957829317120
      }
    }
  }
}
```

##### FRAMEWORK_UPDATE
&emsp;&emsp;每当framwork在断开链接(网络错误)或mesos master故障转移重新向mesos master注册时发送。

``` json
FRAMEWORK_UPDATED Event (JSON)

<event-length>
{
  "type": "FRAMEWORK_UPDATED",

  "framework_updated": {
    "framework": {
      "active": true,
      "allocated_resources": [],
      "connected": true,
      "framework_info": {
        "capabilities": [
          {
            "type": "RESERVATION_REFINEMENT"
          }
        ],
        "checkpoint": true,
        "failover_timeout": 0,
        "id": {
          "value": "a9ba2984-99c4-4183-8cd1-f7313426e21c-0147"
        },
        "name": "inverse-offer-example-framework",
        "role": "*",
        "user": "root"
      },
      "inverse_offers": [],
      "recovered": false,
      "registered_time": {
        "nanoseconds": 1501191957829317120
      },
      "reregistered_time": {
        "nanoseconds": 1501191957829317120
      }
    }
  }
}
```

##### FRAMWORK_REMOVED
&emsp;&emsp;每当framwork被移除时发送。当framwork被操作者明确下线或在故障迁移超时时间内，没有重新注册到mesos master时，可能发生这种情况。

``` json
FRAMEWORK_REMOVED Event (JSON)

<event-length>
{
  "type": "FRAMEWORK_REMOVED",

  "framework_removed": {
    "framework_info": {
      "capabilities": [
        {
          "type": "RESERVATION_REFINEMENT"
        }
      ],
      "checkpoint": true,
      "failover_timeout": 0,
      "id": {
        "value": "a9ba2984-99c4-4183-8cd1-f7313426e21c-0147"
      },
      "name": "inverse-offer-example-framework",
      "role": "*",
      "user": "root"
    }
  }
}
```

##### AGENT_ADDED
&emsp;&emsp;每当mesos master知道agent时发送。当Agent首次注册或在mesos master故障切换后重新注册时，可能会发生这种情况。

``` text
AGENT_ADDED Event (JSON)

<event-length>
{
  "type": "AGENT_ADDED",

  "agent_added": {
    "agent": {
      "active": true,
      "agent_info": {
        "hostname": "172.31.2.24",
        "id": {
          "value": "c3946a13-75b4-4d3c-9d0e-fc10038dca85-S3"
        },
        "port": 5051,
        "resources": [],
      },
      "allocated_resources": [],
      "capabilities": [
        {
          "type": "MULTI_ROLE"
        },
        {
          "type": "HIERARCHICAL_ROLE"
        },
        {
          "type": "RESERVATION_REFINEMENT"
        }
      ],
      "offered_resources": [],
      "pid": "slave(1)@172.31.2.24:5051",
      "registered_time": {
        "nanoseconds": 1500993262264135000
      },
      "reregistered_time": {
        "nanoseconds": 1500993263019321000
      },
      "total_resources": [],
      "version": "1.4.0"
    }
  }
}
```

##### AGENT_REMOVED
&emsp;&emsp;每当agent移除时发送。当agent计划进行维护时，可能会发生这种情况。(注意：可能会有这种情况：Agent可能会在被移除后又变为活动状态，即如果mesos master已知道已知的"dead"的agents。见 MESOS-5965)

 ``` json
 AGENT_REMOVED Event (JSON)

<event-length>
{
  "type": "AGENT_REMOVED",

  "agent_removed": {
    "agent_id": {
      "value": "c3946a13-75b4-4d3c-9d0e-fc10038dca85-S3"
    }
  }
}
 ```
#### Agent API
&emsp;&emsp;此API包含所有被agent接受的调用。这个信息的规范来源是agent.proto(注：在beta API完成之前，protobuf定义可能会发生更改)。这些调用通常由人机操作员，工具或服务(例如Mesos WebUI)产生。虽然executors也可以进行这些调用，但预计这些调用者将使用Executor HTTP API。

##### Calls To Agent And Responses
&emsp;&emsp;以下是来自API的同步响应的Agent的调用示例。

##### GET_HEALTH
&emsp;&emsp;请求和响应类似于GET_HEALTH调用mesos master 

##### GET_FLAGS
&emsp;&emsp;这个调用检索Agent的flag配置

``` text
GET_FLAGS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "GET_FLAGS"
}


GET_FLAGS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_FLAGS",
  "get_flags": {
    "flags": [
      {
        "name": "acls",
        "value": ""
      },
      {
        "name": "appc_simple_discovery_uri_prefix",
        "value": "http://"
      },
      {
        "name": "appc_store_dir",
        "value": "/tmp/mesos/store/appc"
      },
      {
        "name": "authenticate_http_readonly",
        "value": "true"
      },
      {
        "name": "authenticate_http_readwrite",
        "value": "true"
      },
      {
        "name": "authenticatee",
        "value": "crammd5"
      },
      {
        "name": "authentication_backoff_factor",
        "value": "1secs"
      },
      {
        "name": "authorizer",
        "value": "local"
      },
      {
        "name": "cgroups_cpu_enable_pids_and_tids_count",
        "value": "false"
      },
      {
        "name": "cgroups_enable_cfs",
        "value": "false"
      },
      {
        "name": "cgroups_hierarchy",
        "value": "/sys/fs/cgroup"
      },
      {
        "name": "cgroups_limit_swap",
        "value": "false"
      },
      {
        "name": "cgroups_root",
        "value": "mesos"
      },
      {
        "name": "container_disk_watch_interval",
        "value": "15secs"
      },
      {
        "name": "containerizers",
        "value": "mesos"
      },
      {
        "name": "credential",
        "value": "/tmp/directory/credential"
      },
      {
        "name": "default_role",
        "value": "*"
      },
      {
        "name": "disk_watch_interval",
        "value": "1mins"
      },
      {
        "name": "docker",
        "value": "docker"
      },
      {
        "name": "docker_kill_orphans",
        "value": "true"
      },
      {
        "name": "docker_registry",
        "value": "https://registry-1.docker.io"
      },
      {
        "name": "docker_remove_delay",
        "value": "6hrs"
      },
      {
        "name": "docker_socket",
        "value": "/var/run/docker.sock"
      },
      {
        "name": "docker_stop_timeout",
        "value": "0ns"
      },
      {
        "name": "docker_store_dir",
        "value": "/tmp/mesos/store/docker"
      },
      {
        "name": "docker_volume_checkpoint_dir",
        "value": "/var/run/mesos/isolators/docker/volume"
      },
      {
        "name": "enforce_container_disk_quota",
        "value": "false"
      },
      {
        "name": "executor_registration_timeout",
        "value": "1mins"
      },
      {
        "name": "executor_shutdown_grace_period",
        "value": "5secs"
      },
      {
        "name": "fetcher_cache_dir",
        "value": "/tmp/directory/fetch"
      },
      {
        "name": "fetcher_cache_size",
        "value": "2GB"
      },
      {
        "name": "frameworks_home",
        "value": ""
      },
      {
        "name": "gc_delay",
        "value": "1weeks"
      },
      {
        "name": "gc_disk_headroom",
        "value": "0.1"
      },
      {
        "name": "hadoop_home",
        "value": ""
      },
      {
        "name": "help",
        "value": "false"
      },
      {
        "name": "hostname_lookup",
        "value": "true"
      },
      {
        "name": "http_authenticators",
        "value": "basic"
      },
      {
        "name": "http_command_executor",
        "value": "false"
      },
      {
        "name": "http_credentials",
        "value": "/tmp/directory/http_credentials"
      },
      {
        "name": "image_provisioner_backend",
        "value": "copy"
      },
      {
        "name": "initialize_driver_logging",
        "value": "true"
      },
      {
        "name": "isolation",
        "value": "posix/cpu,posix/mem"
      },
      {
        "name": "launcher_dir",
        "value": "/my-directory"
      },
      {
        "name": "logbufsecs",
        "value": "0"
      },
      {
        "name": "logging_level",
        "value": "INFO"
      },
      {
        "name": "oversubscribed_resources_interval",
        "value": "15secs"
      },
      {
        "name": "perf_duration",
        "value": "10secs"
      },
      {
        "name": "perf_interval",
        "value": "1mins"
      },
      {
        "name": "qos_correction_interval_min",
        "value": "0ns"
      },
      {
        "name": "quiet",
        "value": "false"
      },
      {
        "name": "recover",
        "value": "reconnect"
      },
      {
        "name": "recovery_timeout",
        "value": "15mins"
      },
      {
        "name": "registration_backoff_factor",
        "value": "10ms"
      },
      {
        "name": "resources",
        "value": "cpus:2;gpus:0;mem:1024;disk:1024;ports:[31000-32000]"
      },
      {
        "name": "revocable_cpu_low_priority",
        "value": "true"
      },
      {
        "name": "sandbox_directory",
        "value": "/mnt/mesos/sandbox"
      },
      {
        "name": "strict",
        "value": "true"
      },
      {
        "name": "switch_user",
        "value": "true"
      },
      {
        "name": "systemd_enable_support",
        "value": "true"
      },
      {
        "name": "systemd_runtime_directory",
        "value": "/run/systemd/system"
      },
      {
        "name": "version",
        "value": "false"
      },
      {
        "name": "work_dir",
        "value": "/tmp/directory"
      }
    ]
  }
}
```

##### GET_VERSION
&emsp;&emsp;请求和响应类似GET_VERSION调用mesos master

##### GET_METRICS
&emsp;&emsp;请求和响应类似GET_METRICS调用mesos master

##### GET_LOGGING_LEVEL
&emsp;&emsp;请求和响应类似GET_LOGGING_LEVEL调用mesos master

##### SET_LOGGING_LEVEL
&emsp;&emsp;请求和响应类似SET_LOGGING_LEVEL调用mesos master

##### LIST_FILES
&emsp;&emsp;请求和响应类似LIST_FILES调用mesos master

##### READ_FILE
&emsp;&emsp;请求和响应类似READ_FILE调用mesos master

##### GET_STATE
&emsp;&emsp;这个请求检索Agent的全部状态。即，有关在集群中运行的任务，框架和Task的信息

``` text
GET_STATE HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "GET_STATE"
}


GET_STATE HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_STATE",
  "get_state": {
    "get_executors": {
      "executors": [
        {
          "executor_info": {
            "command": {
              "arguments": [
                "mesos-executor",
                "--launcher_dir=/my-directory"
              ],
              "shell": false,
              "value": "my-directory"
            },
            "executor_id": {
              "value": "1"
            },
            "framework_id": {
              "value": "8903b84e-112f-4b5f-aad3-7366f6ae7ecc-0000"
            },
            "name": "Command Executor (Task: 1) (Command: sh -c 'sleep 1000')",
            "resources": [
              {
                "name": "cpus",
                "role": "*",
                "scalar": {
                  "value": 0.1
                },
                "type": "SCALAR"
              },
              {
                "name": "mem",
                "role": "*",
                "scalar": {
                  "value": 32.0
                },
                "type": "SCALAR"
              }
            ],
            "source": "1"
          }
        }
      ]
    },
    "get_frameworks": {
      "frameworks": [
        {
          "framework_info": {
            "checkpoint": false,
            "failover_timeout": 0.0,
            "hostname": "myhost",
            "id": {
              "value": "8903b84e-112f-4b5f-aad3-7366f6ae7ecc-0000"
            },
            "name": "default",
            "principal": "my-principal",
            "role": "*",
            "user": "root"
          }
        }
      ]
    },
    "get_tasks": {
      "launched_tasks": [
        {
          "agent_id": {
            "value": "8903b84e-112f-4b5f-aad3-7366f6ae7ecc-S0"
          },
          "framework_id": {
            "value": "8903b84e-112f-4b5f-aad3-7366f6ae7ecc-0000"
          },
          "name": "",
          "resources": [
            {
              "name": "cpus",
              "role": "*",
              "scalar": {
                "value": 2.0
              },
              "type": "SCALAR"
            },
            {
              "name": "mem",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "disk",
              "role": "*",
              "scalar": {
                "value": 1024.0
              },
              "type": "SCALAR"
            },
            {
              "name": "ports",
              "ranges": {
                "range": [
                  {
                    "begin": 31000,
                    "end": 32000
                  }
                ]
              },
              "role": "*",
              "type": "RANGES"
            }
          ],
          "state": "TASK_RUNNING",
          "status_update_state": "TASK_RUNNING",
          "status_update_uuid": "2qlPayEJRJGPeaWlahI+WA==",
          "statuses": [
            {
              "agent_id": {
                "value": "8903b84e-112f-4b5f-aad3-7366f6ae7ecc-S0"
              },
              "container_status": {
                "executor_pid": 19846,
                "network_infos": [
                  {
                    "ip_addresses": [
                      {
                        "ip_address": "127.0.1.1"
                      }
                    ]
                  }
                ]
              },
              "executor_id": {
                "value": "1"
              },
              "source": "SOURCE_EXECUTOR",
              "state": "TASK_RUNNING",
              "task_id": {
                "value": "1"
              },
              "timestamp": 1470898839.48066,
              "uuid": "2qlPayEJRJGPeaWlahI+WA=="
            }
          ],
          "task_id": {
            "value": "1"
          }
        }
      ]
    }
  }
}
```

##### GET_CONTAINERS
&emsp;&emsp;这个调用检索运行在mesos agent上容器的信息。它包含ContainerStatus和ResourceStatistics以及容器的一些元数据

``` text
GET_CONTAINERS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "GET_CONTAINERS"
}


GET_CONTAINERS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_CONTAINERS",
  "get_containers": {
    "containers": [
      {
        "container_id": {
          "value": "f0f97041-1860-4b4a-b279-91fec4e0ebd8"
        },
        "container_status": {
          "network_infos": [
            {
              "ip_addresses": [
                {
                  "ip_address": "192.168.1.20"
                }
              ]
            }
          ]
        },
        "executor_id": {
          "value": "default"
        },
        "executor_name": "",
        "framework_id": {
          "value": "cbe3c0f1-5655-4110-bc01-ae658a9dbab9-0000"
        },
        "resource_statistics": {
          "mem_limit_bytes": 2048,
          "timestamp": 0.0
        }
      }
    ]
  }
}
```

##### GET_FRAMEWORKS
&emsp;&emsp;此调用检索所有agent已知的framework的信息

``` text
GET_FRAMEWORKS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "GET_FRAMEWORKS"
}


GET_FRAMEWORKS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_FRAMEWORKS",
  "get_frameworks": {
    "frameworks": [
      {
        "framework_info": {
          "checkpoint": false,
          "failover_timeout": 0.0,
          "hostname": "myhost",
          "id": {
            "value": "17e8c0d4-5ee2-4937-bc1c-06c39eddb004-0000"
          },
          "name": "default",
          "principal": "my-principal",
          "role": "*",
          "user": "root"
        }
      }
    ]
  }
}
```

##### GET_EXECUTORS
&emsp;&emsp;此调用检索所有agent已知的executors的信息

``` text
GET_EXECUTORS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "GET_EXECUTORS"
}


GET_EXECUTORS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_EXECUTORS",
  "get_executors": {
    "executors": [
      {
        "executor_info": {
          "command": {
            "arguments": [
              "mesos-executor",
              "--launcher_dir=/my-directory"
            ],
            "shell": false,
            "value": "/my-directory"
          },
          "executor_id": {
            "value": "1"
          },
          "framework_id": {
            "value": "5ffcfa79-00c4-4d93-94a3-2f3844126fd9-0000"
          },
          "name": "Command Executor (Task: 1) (Command: sh -c 'sleep 1000')",
          "resources": [
            {
              "name": "cpus",
              "role": "*",
              "scalar": {
                "value": 0.1
              },
              "type": "SCALAR"
            },
            {
              "name": "mem",
              "role": "*",
              "scalar": {
                "value": 32.0
              },
              "type": "SCALAR"
            }
          ],
          "source": "1"
        }
      }
    ]
  }
}
```

##### GET_TASKS
&emsp;&emsp;此调用检索所有agent已知的任务信息

``` text
GET_TASKS HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "GET_TASKS"
}


GET_TASKS HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "GET_TASKS",
  "get_tasks": {
    "launched_tasks": [
      {
        "agent_id": {
          "value": "70770d61-d666-4547-a808-787f63b00cf2-S0"
        },
        "framework_id": {
          "value": "70770d61-d666-4547-a808-787f63b00cf2-0000"
        },
        "name": "",
        "resources": [
          {
            "name": "cpus",
            "role": "*",
            "scalar": {
              "value": 2.0
            },
            "type": "SCALAR"
          },
          {
            "name": "mem",
            "role": "*",
            "scalar": {
              "value": 1024.0
            },
            "type": "SCALAR"
          },
          {
            "name": "disk",
            "role": "*",
            "scalar": {
              "value": 1024.0
            },
            "type": "SCALAR"
          },
          {
            "name": "ports",
            "ranges": {
              "range": [
                {
                  "begin": 31000,
                  "end": 32000
                }
              ]
            },
            "role": "*",
            "type": "RANGES"
          }
        ],
        "state": "TASK_RUNNING",
        "status_update_state": "TASK_RUNNING",
        "status_update_uuid": "0RC72iyRTQefoUL0ClcL0g==",
        "statuses": [
          {
            "agent_id": {
              "value": "70770d61-d666-4547-a808-787f63b00cf2-S0"
            },
            "container_status": {
              "executor_pid": 27140,
              "network_infos": [
                {
                  "ip_addresses": [
                    {
                      "ip_address": "127.0.1.1"
                    }
                  ]
                }
              ]
            },
            "executor_id": {
              "value": "1"
            },
            "source": "SOURCE_EXECUTOR",
            "state": "TASK_RUNNING",
            "task_id": {
              "value": "1"
            },
            "timestamp": 1470900791.21577,
            "uuid": "0RC72iyRTQefoUL0ClcL0g=="
          }
        ],
        "task_id": {
          "value": "1"
        }
      }
    ]
  }
}
```

##### LAUNCH_NESTED_CONTAINER
&emsp;&emsp;此调用启动嵌套的容器。任何授权实体(包括executor本身，其task或操作员)都可以使用此API启动嵌套容器。

``` text
LAUNCH_NESTED_CONTAINER HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "LAUNCH_NESTED_CONTAINER",
  "launch_nested_container": {
    "container_id": {
      "parent": {
        "parent": {
          "value": "27d44d12-ce9e-455f-9282-f580d8b56cad"
        },
        "value": "f5015d94-8093-477d-9551-9452acfad495"
      },
      "value": "3192b9d1-db71-4699-ae25-e28dfbf42de1"
    },
    "command": {
      "environment": {
        "variables": [
          {
            "name": "ENV_VAR_KEY",
            "type": "VALUE",
            "value": "env_var_value"
          }
        ]
      },
      "shell": true,
      "value": "exit 0"
    }
  }
}

LAUNCH_NESTED_CONTAINER HTTP Response (JSON):

HTTP/1.1 200 OK
```

##### WAIT_NESTED_CONTAINER
&emsp;&emsp;此调用等待嵌套容器终止或退出。任何授权实体(包括executor本身，其task或操作员)都可以使用此API等待嵌套容器。

``` text
WAIT_NESTED_CONTAINER HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "WAIT_NESTED_CONTAINER",
  "wait_nested_container": {
    "container_id": {
      "parent": {
        "value": "6643b4be-583a-4dc3-bf23-a1ffb26dd452"
      },
      "value": "3192b9d1-db71-4699-ae25-e28dfbf42de1"
    }
  }
}

WAIT_NESTED_CONTAINER HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/json

{
  "type": "WAIT_NESTED_CONTAINER",
  "wait_nested_container": {
    "exit_status": 0
  }
}
```

##### KILL_NESTED_CONTAINER
&emsp;&emsp;此调用启动嵌套容器的销毁。任何授权实体(包括executor自身，其任务及操作者)都可以使用此API来kill嵌套容器。

``` text
KILL_NESTED_CONTAINER HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "KILL_NESTED_CONTAINER",
  "kill_nested_container": {
    "container_id": {
      "parent": {
        "value": "62d15977-acd4-4167-ae08-2e3738dc3ad6"
      },
      "value": "3192b9d1-db71-4699-ae25-e28dfbf42de1"
    }
  }
}

KILL_NESTED_CONTAINER HTTP Response (JSON):

HTTP/1.1 200 OK
```

##### LAUNCH_NESTED_CONTAINER_SESSION
&emsp;&emsp;此调用启动嵌套容器，其生命周期与建立此连接的HTTP调用的生命周期相关。只要连接处于活动状态，嵌套容器的STDOUT和STDERR就会一直发送回客户端。

``` text
LAUNCH_NESTED_CONTAINER_SESSION HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/recordio
Message-Accept: application/json

{
  "type": "LAUNCH_NESTED_CONTAINER_SESSION",
  "launch_nested_container_session": {
    "container_id": {
      "parent": {
        "parent": {
          "value": "bde04877-cb26-4277-976e-3ecf0c02e76b"
        },
        "value": "134bae93-cf5c-4938-87bf-f779bfcd0092"
      },
      "value": "e193a755-8528-4673-a05b-2cc2a01a8b94"
    },
    "command": {
      "environment": {
        "variables": [
          {
            "name": "ENV_VAR_KEY",
            "type": "VALUE",
            "value": "env_var_value"
          }
        ]
      },
      "shell": true,
      "value": "while [ true ]; do echo $(date); sleep 1; done"
    }
  }
}

LAUNCH_NESTED_CONTAINER_SESSION HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/recordio
Message-Content-Type: application/json

90
{
  "type":"DATA",
  "data": {
    "type":"STDOUT",
    "data": "TW9uIEZlYiAyNyAwNzozOTozOCBVVEMgMjAxNwo="
  }
}90
{
  "type":"DATA",
  "data": {
    "type":"STDOUT",
    "data": "TW9uIEZlYiAyNyAwNzozOTozOSBVVEMgMjAxNwo="
  }
}90
{
  "type":"DATA",
  "data": {
    "type":"STDERR",
    "data": "TW9uIEZlYiAyNyAwNzozOTo0MCBVVEMgMjAxNwo="
  }
}
...
```

##### ATTACH_CONTAINER_INPUT
&emsp;&emsp;此调用附加到容器的主进程STDIN，并将其输入。此调用只能针对已使用关联的IOSwitchboard启动的容器进行(i.e. 嵌套容器通过一个LAUNCH_NESTED_CONTAINER_SESSION调用启动，或者在其ContainerInfo中使用TTYInfo启动的普通容器)。一次只能为一个给定的容器激活一个ATTACH_CONTAINER_INPUT调用。随后的附加尝试将失败。
&emsp;&emsp;通过ATTACH_CONTAINER_INPUT流发送的第一个消息必须是CONTAINER_ID类型，并且包含要附加到的容器的ContainerID.后续消息必须是PROCESS_IO类型，但它们可能包含DATA或CONTROL的子类型。数据消息的类型必须为STDIN，并且包含实际数据流到要附加到的容器的STDIN。当前，唯一有效的CONTROL消息发送心跳以保持连接的存活。我们日后可能会添加更多CONTROL消息。

``` text
ATTACH_CONTAINER_INPUT HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/recordio
Message-Content-Type: application/json
Accept: application/json

214
{
  "type": "ATTACH_CONTAINER_INPUT",
  "attach_container_input": {
    "type": "CONTAINER_ID",
    "container_id": {
      "value": "da737efb-a9d4-4622-84ef-f55eb07b861a"
    }
  }
}163
{
  "type": "ATTACH_CONTAINER_INPUT",
  "attach_container_input": {
    "type": "PROCESS_IO",
    "process_io": {
      "type": "DATA",
      "data": {
        "type": "STDIN",
        "data": "dGVzdAo="
      }
    }
  }
}210
{
  "type":
  "ATTACH_CONTAINER_INPUT",
  "attach_container_input": {
    "type": "PROCESS_IO",
    "process_io": {
      "type": "CONTROL",
      "control": {
        "type": "HEARTBEAT",
        "heartbeat": {
          "interval": {
            "nanoseconds": 30000000000
          }
        }
      }
    }
  }
}
...

ATTACH_CONTAINER_INPUT HTTP Response (JSON):

HTTP/1.1 200 OK
```

##### ATTACH_CONTAINER_OUTPUT
&emsp;&emsp;此调用附加到容器的主进程的STDOUT和STDERR,并将其输出流传送回客户端。此调用只能针对已经使用关联的IOSwitchboard启动的容器进行(比如：通过LAUNCH_NESTED_CONTAINER_SESSION调用启动的嵌套容器或在其ContainerInfo字段中使用TTYInfo启动的普通容器)，给定容器一次可以启用多个ATTACH_CONTAINER_OUTPUT调用。

``` json
ATTACH_CONTAINER_OUTPUT HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/recordio
Message-Accept: application/json

{
  "type": "ATTACH_CONTAINER_OUTPUT",
  "attach_container_output": {
    "container_id": {
      "value": "e193a755-8528-4673-a05b-2cc2a01a8b94"
    }
  }
}

ATTACH_CONTAINER_OUTPUT HTTP Response (JSON):

HTTP/1.1 200 OK

Content-Type: application/recordio
Message-Content-Type: application/json

90
{
  "type":"DATA",
  "data": {
    "type":"STDOUT",
    "data": "TW9uIEZlYiAyNyAwNzozOTozOCBVVEMgMjAxNwo="
  }
}90
{
  "type":"DATA",
  "data": {
    "type":"STDOUT",
    "data": "TW9uIEZlYiAyNyAwNzozOTozOSBVVEMgMjAxNwo="
  }
}90
{
  "type":"DATA",
  "data": {
    "type":"STDERR",
    "data": "TW9uIEZlYiAyNyAwNzozOTo0MCBVVEMgMjAxNwo="
  }
}
...
```

##### REMOVE_NESTED_CONTAINER
&emsp;&emsp;此调用触发删除嵌套容器及其工件(例如，沙箱和运行时文件夹)。此调用只能针对已经终止的容器，并且其父容器未被销毁。任何授权实体(包括执行者本身，其任务或运营商)都可以使用此API调用

``` json
REMOVE_NESTED_CONTAINER HTTP Request (JSON):

POST /api/v1  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "REMOVE_NESTED_CONTAINER",
  "remove_nested_container": {
    "container_id": {
      "parent": {
        "value": "6643b4be-583a-4dc3-bf23-a1ffb26dd452"
      },
      "value": "3192b9d1-db71-4699-ae25-e28dfbf42de1"
    }
  }
}

REMOVE_NESTED_CONTAINER HTTP Response (JSON):

HTTP/1.1 200 OK
```

 
 

