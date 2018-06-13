# Executor HTTP API #
Mesos executor 可以通过以下两种方式构建：
1. 通过C++接口`ExecutorDriver`实现。`ExecutorDriver`实现了与Mesos Agent的通讯细节。Executor开发人员只需要通过`ExecutorDriver`为有效事件注册回调函数，并实现不同事件对应回调函数的自定义逻辑即可，这样的事件有很多，例如收到一个新的任务启动请求。因为`ExecutorDriver`接口是使用C++语言实现的，通过这种方式实现executor的基本要求是开发者必须使用C++或者可以绑定C++进行开发的语言。
2. 通过新的HTTP API实现。通过这种方式，executor开发者可以不使用C++语言或者原生客户端库进行开发。与原生客户端方式不同的是，executor与Mesos agent纯粹通过HTTP requests进行交互。虽然理论上可以直接使用executor HTTP API实现executor，但是大多数的executor开发者都会依据各自的开发语言选择一个对应的开发库来屏蔽一些HTTP API的实现细节；文档[《HTTP API client libraries》]()列举了这些库。

v1版本的Executor HTTP API在Mesos 0.28.0就有过介绍。到Mesos 1.0版本，HTTP API更稳定，并且推荐使用这种方式来开发新的Mesos executors。

## 概述 ##

Executor采用[/api/v1/executor]()作为与Agent交互的URI，Agent将监听在该URI上。需要注意的是在后面的文档中，我们将使用后缀/executor来指代该URI。Agent接收以JSON（Content-Type: application/json）或者以binary Protobuf (Content-Type: application/x-protobuf)编码消息体的HTTP POST请求，executor发送给Agent的第一个请求是SUBSCRIBE，并且Agent会返回一个流式应答（“200 OK” status code with Transfer-Encoding: chunked）。executor需要保持这个HTTP订阅长链接（除非发生网络错误、Agent进程重启、或其他软件bug等状况），并不断处理该链接上的response。（注意：不能使用HTTP 客户端库来实现这个，因为HTTP 客户端库只能解析链接关闭后的response）。为编码需要，请参考下面的**事件**章节。
所有后续（除SUBSCRIBE之外）发送到Agent的请求，都必须通过与之前提到的订阅链接不同的HTTP链接发送。Agent将向这些独立的HTTP POST请求返回“202 Accepted”状态码（或者对于失败的请求，将返回4XX或5XX状态码，详细内容将在后面的章节描述）。“202 Accepted”状态码表示请求已经接受或者在处理中，而不是表示请求已经处理完成。并且这些请求不一定会被Mesos处理（例如，agent在处理请求是发生了错误）。任何对这些请求的响应都会被注入前文提到的HTTP订阅长连接上。

## 调用 ##

下面这些请求会被Agent处理。这些请求的定义在[executor.proto]()这个文件中（对这个protobuff文件中的定义的修改取决于beta版的API是否完成开发），另外需要注意的是，当发送基于JSON编码的调用时，executor应该对发送内容进行BASE64编码，并使用UTF-8字符编码。

### SUBSCRIBE ###

SUBSCRIBE调用是executor和agent之间进行通信的第一步。这一步也是executor向“/executor”订阅事件流的步骤。

为了完成对agent的订阅，executor需要发送一个编码了`SUBSCRIBE`调用的HTTP POST请求。对这个请求的响应是交互的第一个事件——`SUBSCRIBED`事件（事件的具体细节参考 **事件** 章节），并且事件的响应会以[RecordIO]()的方式编码。

另外，如果executor在断链后重新链接到agent，那么仍然可以发送如下列表所示内容到agent：
- **Unacknowledged Status Updates**：executor需要保持一个待确认的状态变化列表，这些状态变化都没有得到Agent通过`ACKNOWLEDGE`事件确认。
- **Unacknowledged Tasks**：executor需要保持一个待确认的任务列表，任务被确认的标志是任务的任何一个状态变化得到了Agent的确认事件确认。

以下是一个`SUBSCRIBE`调用的例子：
```HTTP
SUBSCRIBE Request (JSON):

POST /api/v1/executor  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "type": "SUBSCRIBE",
  "executor_id": {
    "value": "387aa966-8fc5-4428-a794-5a868a60d3eb"
  },
  "framework_id": {
    "value": "49154f1b-8cf6-4421-bf13-8bd11dccd1f1"
  },
  "subscribe": {
    "unacknowledged_tasks": [
      {
        "name": "dummy-task",
        "task_id": {
          "value": "d40f3f3e-bbe3-44af-a230-4cb1eae72f67"
        },
        "agent_id": {
          "value": "f1c9cdc5-195e-41a7-a0d7-adaa9af07f81"
        },
        "command": {
          "value": "ls",
          "arguments": [
            "-l",
            "\/tmp"
          ]
        }
      }
    ],
    "unacknowledged_updates": [
      {
        "framework_id": {
          "value": "49154f1b-8cf6-4421-bf13-8bd11dccd1f1"
        },
        "status": {
          "source": "SOURCE_EXECUTOR",
          "task_id": {
            "value": "d40f3f3e-bbe3-44af-a230-4cb1eae72f67"
          },
        "state": "TASK_RUNNING",
        "uuid": "ZDQwZjNmM2UtYmJlMy00NGFmLWEyMzAtNGNiMWVhZTcyZjY3Cg=="
        }
      }
    ]
  }
}

SUBSCRIBE Response Event (JSON):
HTTP/1.1 200 OK

Content-Type: application/json
Transfer-Encoding: chunked

<event-length>
{
  "type": "SUBSCRIBED",
  "subscribed": {
    "executor_info": {
      "executor_id": {
        "value": "387aa966-8fc5-4428-a794-5a868a60d3eb"
      },
      "command": {
        "value": "\/path\/to\/executor"
      },
      "framework_id": {
        "value": "49154f1b-8cf6-4421-bf13-8bd11dccd1f1"
      }
    },
    "framework_info": {
      "user": "foo",
      "name": "my_framework"
    },
    "agent_id": {
      "value": "f1c9cdc5-195e-41a7-a0d7-adaa9af07f81"
    },
    "agent_info": {
      "host": "agenthost",
      "port": 5051
    }
  }
}
<more events>
```
注意：一旦一个executor启动了，Agent会等待一个由参数`--executor_registration_timeout`(Agent启动时设定)设定的期限，在该期限内，executor需要发送`SUBSCRIBE`调用订阅Agent。否则，Agent会强制摧毁运行executor的容器。

### UPDATE ###

executor通过该调用可靠地通知其所管理的任务的状态。为了能够让Mesos及时释放分配给任务的资源，在任务结束时立即将任务状态的改变（例如，`TASK_FINISHED`, `TASK_KILLED` 或者 `TASK_FAILED`）发送给Agent是至关重要的。

scheduler必须通过`ACKNOWLEDGE`（详细的语义解释请参考**事件**章节中的`ACKNOWLEDGE`事件）事件明确地回应该调用。executor必须保持一个待确认状态列表，以确保在executor与agent断链重连后，这个待确认状态列表通过`SUBSCRIBE`调用中的`unacknowledged_updates`字段发送给Agent。

以下是一个`UPDATE`调用的实例：
```HTTP
UPDATE Request (JSON):

POST /api/v1/executor  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "executor_id": {
    "value": "387aa966-8fc5-4428-a794-5a868a60d3eb"
  },
  "framework_id": {
    "value": "9aaa9d0d-e00d-444f-bfbd-23dd197939a0-0000"
  },
  "type": "UPDATE",
  "update": {
    "status": {
      "executor_id": {
        "value": "387aa966-8fc5-4428-a794-5a868a60d3eb"
      },
      "source": "SOURCE_EXECUTOR",
      "state": "TASK_RUNNING",
      "task_id": {
        "value": "66724cec-2609-4fa0-8d93-c5fb2099d0f8"
      },
      "uuid": "ZDQwZjNmM2UtYmJlMy00NGFmLWEyMzAtNGNiMWVhZTcyZjY3Cg=="
    }
  }
}

UPDATE Response:
HTTP/1.1 202 Accepted
```

### MESSAGE ###

executor通过该调用发送任意二进制数据给scheduler。需要注意的是Mesos既不解析发送的内容，也不保证消息能够正确投递到scheduler。该调用中的`data`字段必须是基于BASE64的原始字节编码。

以下是一个`MESSAGE`调用的例子：
```HTTP
MESSAGE Request (JSON):

POST /api/v1/executor  HTTP/1.1

Host: agenthost:5051
Content-Type: application/json
Accept: application/json

{
  "executor_id": {
    "value": "387aa966-8fc5-4428-a794-5a868a60d3eb"
  },
  "framework_id": {
    "value": "9aaa9d0d-e00d-444f-bfbd-23dd197939a0-0000"
  },
  "type": "MESSAGE",
  "data": "t+Wonz5fRFKMzCnEptlv5A=="
}

MESSAGE Response:
HTTP/1.1 202 Accepted
```

## 事件 ##

executor需要维持一个HTTP **长连接** 到Agent的`/executor`地址，该长连接即使在收到`SUBSCRIBED`事件后也仍然保持。维持这种长连接，需要在HTTP头中设定 “Connection: keep-alive” 和 “Transfer-Encoding: chunked”，并且不能设定“Content-Length”字段。后续所有由Mesos产生的与该executor相关的事件都将发送到这个长连接上。Agent会以 [RecordIO](). 的格式编码这些事件，也就是事件按字节计算的长度的字符串表示，跟上以JSON或者binary Protobuf（可能是被压缩的）格式编码的消息体构成的字符串。需要注意的是：长度字段的值不可能为'0'，并且长度字段的大小等于一个无符号整数的大小（也就是64位）。另外鉴于基于HTTP的分块编码通常在应用层（executor）不可见，所以基于RecordIO格式的编码消息需要在executor侧完成解码。事件消息的编码类型由HTTP Post请求头中的Accept字段决定（例如：“Accept: application/json”）。

下面是Agent当前会发送的事件，这些事件的定义在[executor.proto]()文件中。需要明确一点：当事件的编码类型是JSON时，Agent会以BASE64编码事件原始字节成UTF-8字符集的字符串。

### SUBSCRIBED ###

`SUBSCRIBED`是在executor发送`SUBSCRIBE`请求后，agent在HTTP长链接上发送的第一个事件。参考前文**调用**章节的描述。

### LAUNCH ###

当Agent需要给executor分配一个新任务时会发送该事件。executor需要回应一个`UPDATE`消息给Agent，用来明确任务初始化成功还是失败。

executor必须维持一个待确认的任务列表（参考**调用**中的`SUBSCRIBE`章节）。如果因为某些原因，executor与Agent断链了，这些待确认的任务必须作为`SUBSCRIBE`调用中的`tasks`字段的内容发送给Agent。

以下是一个`LAUNCH`事件的示例：
```HTTP
LAUNCH Event (JSON)

<event-length>
{
  "type": "LAUNCH",
  "launch": {
    "framework_info": {
      "id": {
        "value": "49154f1b-8cf6-4421-bf13-8bd11dccd1f1"
      },
      "user": "foo",
      "name": "my_framework"
    },
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

### LAUNCH_GROUP ###

这是在Mesos 1.1.0 中增加的**实验性**事件。

当Agent需要给executor分配一个任务组的时候发送该事件。executor需要对任务组中的每一个任务发送一个`UPDATE`事件作为回应给Agent，以确认每个任务的初始化成功或失败情况。

executor同样需要维持一个待确认任务列表（参考`LAUNCH`事件）。

如下是一个`LAUNCH_GROUP`事件的示例：
```HTTP
LAUNCH_GROUP Event (JSON)

<event-length>
{
  "type": "LAUNCH_GROUP",
  "launch_group": {
    "task_group" : {
      "tasks" : [
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
      ]
    }
  }
}
```

### KILL ### 

scheduler通过`KILL`事件终止一个指定任务的执行。当executor终止或杀死一个任务的时候，需要给Agent回应一个任务结束的状态变化(例如：`TASK_FINISHED`, `TASK_KILLED` 或者 `TASK_FAILED`)。当Agent收到这个任务结束状态的时候，该任务的资源将被Agent标记为已释放。

以下是一个`KILL`事件的示例：
```HTTP
LAUNCH Event (JSON)

<event-length>
{
  "type" : "KILL",
  "kill" : {
    "task_id" : {"value" : "d40f3f3e-bbe3-44af-a230-4cb1eae72f67"}
  }
}
```

### ACKNOWLEDGED ###

作为一种消息可靠传递的机制，`ACKNOWLEDGED`事件用作Agent向executor确认某个状态变化消息已经收到了。已经被确认的状态改变不能再重新发送。

以下是一个`ACKNOWLEDGED`事件的示例：
```HTTP
ACKNOWLEDGED Event (JSON)

<event-length>
{
  "type" : "ACKNOWLEDGED",
  "acknowledged" : {
    "task_id" : {"value" : "d40f3f3e-bbe3-44af-a230-4cb1eae72f67"},
    "uuid" : "ZDQwZjNmM2UtYmJlMy00NGFmLWEyMzAtNGNiMWVhZTcyZjY3Cg=="
  }
}
```

### MESSAGE ###

scheduler生成自定义消息，并尽一切可能发送给executor。这个消息就像是Mesos负责投递的，然后投递的过程并不确保安全。如果消息因为任何原因丢失了，scheduler需要自己重试。注意消息中的`data`字段是基于BASE64的原始字节编码。

以下是一个`MESSAGE`事件的示例：
```HTTP
MESSAGE Event (JSON)

<event-length>
{
  "type" : "MESSAGE",
  "message" : {
    "data" : "c2FtcGxlIGRhdGE="
  }
}
```

### SHUTDOWN ###

Agent需要关闭一个executor时发送该事件。当一个executor收到`SHUTDOWN`事件时，executor需要杀死旗下的所有任务，发送`TASK_KILLED`状态变化事件，并优雅地退出。如果executor在`MESOS_EXECUTOR_SHUTDOWN_GRACE_PERIOD`（executor启动时agent设置的一个环节变量）指定的期限内没有退出，Agent将强制摧毁executor运行的容器，并为该executor遗留下的所有活着的任务发送`TASK_LOST`状态变化消息。

以下是一个`SHUTDOWN`事件的示例：
```HTTP
SHUTDOWN Event (JSON)

<event-length>
{
  "type" : "SHUTDOWN"
}
```

### ERROR ###

当一个异步错误发生时，agent发送该事件。建议executor在收到一个`ERROR`事件并重试订阅后退出。

以下是一个`ERROR`事件的示例：
```HTTP
ERROR Event (JSON)

<event-length>
{
  "type" : "ERROR",
  "error" : {
    "message" : "Unrecoverable error"
  }
}
```

## Executor 环境变量 ##

下面这些环境变量都是Agent在executor启动时设置的，并且executor在启动后可以使用的：
- `MESOS_FRAMEWORK_ID`: scheduler的`FrameworkID`是`SUBSCRIBE`调用必须的部分。
- `MESOS_EXECUTOR_ID`: executor的`ExecutorID`也是`SUBSCRIBE`调用需要的。
- `MESOS_DIRECTORY`: executor在本机文件系统中的工作目录的路径（该环境变量已经弃用）。
- `MESOS_SANDBOX`: 对于使用镜像的Mesos容器或者docker容器，该环境变量表示容器中沙盒的映射路径（由agent的启动参数`sandbox_directory`确定）。对于没有指定镜像的普通命令任务，它表示沙盒在文件系统中的路径，含义同`MESOS_DIRECTORY`，不同的是`MESOS_DIRECTORY`只表示沙盒在本机文件系统中的路径。
- `MESOS_AGENT_ENDPOINT`: 表示Agent的地址，例如ip:port，executor使用该环境变量来与Agent通讯。
- `MESOS_CHECKPOINT`: 如果该值设置为true，表示framework打开了校验机制。
- `MESOS_EXECUTOR_SHUTDOWN_GRACE_PERIOD`: agent等待executor关闭的总时间（例如：60secs，3mins等等），如果超过这个时间，Agent会发送一个`SHUTDOWN`事件。

如果`MESOS_CHECKPOINT`环境变量被设置，也就是说framework开启了校验功能，下列这些环境变量也会被设置，并且executor在做断链重试的时候会用到他们。

- `MESOS_RECOVERY_TIMEOUT`: executor与Agent断链时，executor会在该环境变量设定的时间范围内重试链接，如果超时executor会主动退出。该环境变量的值可以通过Agent的启动参数`--recovery_timeout`指定。
- `MESOS_SUBSCRIPTION_BACKOFF_MAX`: 表示当executor与Agent断链时，executor两次重试链接之间的最大间隔时间（例如：250ms,1mins等等）。该环境变量的值可以通过Agent的启动参数`--executor_reregistration_timeout`指定。

此外需要注意的是，executor也继承了Agent的所有环境变量。

## 断链 ##

如果executor到Agent /executor的HTTP长连接（通过`SUBSCRIBE`调用建立的）断开，则executor和Agent之间处于断链状态。Agent进程故障等都可能是造成断链的原因。

当Agent检测到断链时，重试行为取决于framework的校验机制是否开启：
- 如果framework的校验机制没有开启，则Agent会认为executor不会重试订阅操作，并且会优雅退出。
- 如果framework的校验机制开启，则Agent会认为executor会在`MESOS_RECOVERY_TIMEOUT`指定的期限内以合适的[补偿策略]()重试订阅操作。如果到超时都没有和Agent建立成功HTTP订阅长连接，则executor会优雅退出。

## Agent故障恢复 ##

当Agent启动之后，Agent就有[故障恢复能力]()。这个能力能让Agent修复状态变化并与之前的executor重新建立连接。当前Agent支持如下的恢复机制，并可以通过启动参数`--recover`设定：
- **reconnect**(默认机制)：这种模式允许Agent重连所有之前的executor，前提是这些executor对应的framework开启了校验机制。一旦恢复行为开始，则直到所有断链的executor都重新链接上，并且所有挂起的executor都被摧毁了，Agent才会标记恢复行为结束。因此，这强制要求每个executor在间隔周期（`MESOS_SUBSCRIPTION_BACKOFF_MAX`）内至少要重试链接一次；否则Agent会将executor当做挂起的或无反应的，并关闭这个executor。
- **cleanup**: 这种模式下，Agent会杀死所有存活的executor，然后退出。Operators在给Agent或executor进行不兼容升级的时候通常会执行这种机制。当Agent收到一个开启了校验机制的framework的executor发送的`SUBSCRIBE`请求后，Agent会在executor重连的同时发送一个`SHUTDOWN`事件给它。对于挂起状态的executor，Agent会等待由`--executor_shutdown_grace_period`设定的时间（在Agent启动时可以配置），然后强制杀死executor所在的容器。

## 补偿策略 ##

当executor与Agent断链时，他们通常会按照一定的策略来完成重新订阅过程，例如：线性补偿策略。断链通常发生在Agent进程停止的时候（例如：重启或升级）。executor每次重试链接的时间间隔必须小于`MESOS_SUBSCRIPTION_BACKOFF_MAX`环境变量设定的值。