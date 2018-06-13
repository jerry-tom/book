## Scheduler HTTP API
&emsp;&emsp;一个mesos的调度器可以通过下面两个不同的方式构建：
&emsp;&emsp;1、通过使用SchedulerDriver C++接口，SchedulerDriver处理与mesos master之间的相信通信。调度器开发者通过使用SchedulerDriver为重要事件注册回调来实现自定义调度逻辑，例如接收新资源或任务状态更新。因为SchedulerDriver接口是用c++编写的，这通常要求调度器开发者使用c++或者使用c++绑定语言(比如，当使用JVM-base语言时，JNI)
&emsp;&emsp;2、通过使用新的HTTP API.这允许在不使用C++或本地客户端库的情况下开发mesos调度器；相反，自定义调度器通过http请求与mesos master通信，如下所述。尽管理论上可以“直接”使用HTTP调度api(比如：使用通用的 http库)，但是大多数调度器开发人员应该使用他们自己熟悉的语言库开发详细的http api，具体见HTTP API 客户端库列表文档.
&emsp;&emsp;Scheduler HTTP API V1版本在mesos 0.24.0引入的。从mesos1.0，它被认为是稳定的，是开发新的mesos调度器推荐方式。

### 概述
&emsp;&emsp; 调度器通过/api/v1/scheduler主端点与mesos交互。在本文档其余部分，我们引用此端点及其后缀"/scheduler"。这个端口接受具有编码为JSON(Content-Type: application/json)或二进制Protobuf(Content-Type: application/x-protobuf)的HTTP POST请求。调度器发送给"/scheduler"端的第一个请求称为SUBSCRIBE，并导致流式响应("200 OK"状态码，传输编码：chucked)。
&emsp;&emsp;调度器应尽可能长时间保持订阅连接打开(除非发生网络，软件，硬件等错误),并且逐步处理响应。只能在连接关闭后才能解析响应的HTTP客户端库不能使用。有关使用的编码，请参阅下面的事件部分
&emsp;&emsp;对“/scheduler”端点的所有后续(非订阅)请求必须使用与用于订阅的连接不同的连接发送。mesos master通过"202 Accepted"状态码(或者，对于不成功的请求，使用4xx或5xx状态代码，详细信息在后面部分)响应这些HTTP POST请求。"202 Accepted"响应表示请求已经被接受处理，而不表示请求的处理已经完成。请求可能或可能没有被mesos执行(比如：mesos master在处理请求的过程中异常)。这些请求的任何异步响应将通过订阅的长连接传输回来。调度器可以通过多个不同的HTTP连接提交请求。

### 调用
&emsp;&emsp;下面这些调用被当前mesos master接受。此信息的规范来源是scheduler.proto。当发送JSON编码调用时，调度器应该使用Base64编码二进制，UTF-8编码字符串。所有非SUBSCRIBE调用都应该在http头中包含Mesos-Stream-Id,详细解释见SUBSCRIBE章节。SUBSCRIBE调用不应该在http头中包含Mesos-Stream-Id。

#### RecordIO响应格式
&emsp;&emsp;从SUBSCRIBE调用返回的响应(见下文)采用RecordIO格式编码，它基本上单个记录(json或序列化protobuf)以其字节单位的长度开始，后跟换行符，然后是数据：
&emsp;&emsp;RecordIO编码的流式响应的BNF语法是：

``` text
records         = *record

record          = record-size LF record-data

record-size     = 1*DIGIT
record-data     = record-size(OCTET)

recode-size应该解释为无符号64位整数(uint64)
```
&emsp;&emsp;举个例子，一个数据流可能看起来像这样：

``` text
128\n
{"type": "SUBSCRIBED","subscribed": {"framework_id": {"value":"12220-3440-12532-2345"},"heartbeat_interval_seconds":15.0}20\n
{"type":"HEARTBEAT"}675\n
...
```
&emsp;&emsp;如伪代码表示，这可以按照下面解析：

``` c++
  while (true) {
    do {
      lengthBytes = readline()
    } while (lengthBytes.length < 1)

    messageLength = parseInt(lengthBytes);
    messageBytes = read(messageLength);
    process(messageBytes);
  }
```
&emsp;&emsp;网络中介(比如：代理)可以自由地改变块边界；这不应该对应用程序(调度器)有任何影响。我们想要一种方法来一致地为JSON/Protobuf响应分割/编码两个事件，RecordIO格式允许我们这样做。

### 订阅(SUBSCRIBE)
&emsp;&emsp;这是调度器与mesos-master通信过程的第一步。这也被认为是订阅"/scheduler"事件流。
&emsp;&emsp;为了订阅mesos-master,调度器发送了一个包含所需的FramworkInfo的SUBSCRIBE消息的HTTP POST。注意，如果"subscribe.framework_info.id"没有指定，mesos-master认为这是一个新的调度器，并通过为其分配FramworkID来订阅它。HTTP响应是RecordIO格式的流；事件流从一个SUBSCRIBE事件开始(详细见Events章节)。该响应也包含Mesos-Stream-Id HTTP头部，mesos-master用来唯一标示订阅的调度器实例。这个Steam ID头部应该包含在所有通过订阅连接发送的mesos-master的非订阅调用中。Mesos-Stream-Id的值确保最多128个字节长。

``` text
SUBSCRIBE Request (JSON):

POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Accept: application/json
Connection: close

{
   "type"       : "SUBSCRIBE",
   "subscribe"  : {
      "framework_info"  : {
        "user" :  "foo",
        "name" :  "Example HTTP Framework"
      }
  }
}

SUBSCRIBE Response Event (JSON):
HTTP/1.1 200 OK

Content-Type: application/json
Transfer-Encoding: chunked
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

<event length>
{
 "type"         : "SUBSCRIBED",
 "subscribed"   : {
     "framework_id"               : {"value":"12220-3440-12532-2345"},
     "heartbeat_interval_seconds" : 15
  }
}
<more events>
```
&emsp;&emsp;或者，如果设置了"subscribe.framework_info.id"，mesos-master认为这是来自一个已订阅过的调度器在断连(比如，由于mesos-master/调度器故障或网络中断)重连后的请求，以SUBSCRIBED事件回应。有关详细信息，见下面“断连重连”章节。
&emsp;&emsp;注意：在旧版的API中，(再)注册的回调还包括MasterInfo,其中包含有关当前驱动连接的mesos-master信息。新版的API,因为调度器明确订阅leader mesos-master(详细见下面“Master Detection”章节)，它不再相关。
&emsp;&emsp;如果订阅不论什么原因失败(比如：无效请求)，则回复一个带错误信息的作为body一部分的HTTP 4xx响应，并且关闭连接。
&emsp;&emsp;调度器只有在通过发送SUBSCRIBE请求并收到SUBSCRIBE响应打开与mesos-master的持久连接后，才能向"/scheduler"发出其他的HTTP请求。在没有订阅的情况下进行调用，将产生“403禁止”而不是“202接受”响应。如果http请求格式错误(比如：HTTP头格式错误)，调度器可能收到“400 错误请求”响应。
&emsp;&emsp;请注意：SUBSCRIBE调用的HTTP头部不应该包含"Mesos-Stream-Id";mesos-master总是会为每个订阅提供一个新的唯一的"Mesos-Stream-Id"。

### TEARDOWN
&emsp;&emsp;当调度器想要拆毁自己时，由调度器发送。当Mesos收到这个请求，它将关闭所有executors(从而关闭任务)。接着mesos移除这个framwork，并且关闭从这个调度器到mesos-master所有打开的连接

``` text
TEARDOWN Request (JSON):
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "TEARDOWN"
}

TEARDOWN Response:
HTTP/1.1 202 Accepted
```

### ACCEPT（接受）
&emsp;&emsp;当调度器接受mesos-master发来的offer时，由调度器发送。ACCEPT请求包含调度器想要在这些offer上执行的操作类型(比如：launch task, lauch task group,reserve resources,create volumes)。请注意，在调度器回复(接受或拒绝)offer之前，该offer的资源将被视为分配给了这个调度器。此外，任何没有在ACCEPT调用中使用的offer的资源应该拒绝，以便提供其他调度器。换句话说，就是同一个OfferID不能在多个ACCEPT call中使用。当我们向Mesos中增加新的特性时，这些语义可能会改变(例如:持久性，保留，offer优化，resizeTask等)。

```text
ACCEPT Request (JSON):
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"   : {"value" : "12220-3440-12532-2345"},
  "type"           : "ACCEPT",
  "accept"         : {
    "offer_ids"    : [
                      {"value" : "12220-3440-12532-O12"}
                     ],
     "operations"  : [
                      {
                       "type"         : "LAUNCH",
                       "launch"       : {
                         "task_infos" : [
                                         {
                                          "name"        : "My Task",
                                          "task_id"     : {"value" : "12220-3440-12532-my-task"},
                                          "agent_id"    : {"value" : "12220-3440-12532-S1233"},
                                          "executor"    : {
                                            "command"     : {
                                              "shell"     : true,
                                              "value"     : "sleep 1000"
                                            },
                                            "executor_id" : {"value" : "12214-23523-my-executor"}
                                          },
                                          "resources"   : [
                                                           {
                                    "name"  : "cpus",
                                    "role"  : "*",
                                    "type"  : "SCALAR",
                                    "scalar": {"value": 1.0}
                                       },
                                                           {
                                    "name"  : "mem",
                                    "role"  : "*",
                                    "type"  : "SCALAR",
                                    "scalar": {"value": 128.0}
                                       }
                                                          ]
                                         }
                                        ]
                       }
                      }
                     ],
     "filters"     : {"refuse_seconds" : 5.0}
  }
}

ACCEPT Response:
HTTP/1.1 202 Accepted
```

### DECLINE（拒绝）
&emsp;&emsp;由调度器发送明确拒绝收到的offer(s)。请注意，这与发送没有操作的ACCEPT调用一样。

``` text
DECLINE Request (JSON):
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "DECLINE",
  "decline"         : {
    "offer_ids" : [
                   {"value" : "12220-3440-12532-O12"},
                   {"value" : "12220-3440-12532-O13"}
                  ],
    "filters"   : {"refuse_seconds" : 5.0}
  }
}

DECLINE Response:
HTTP/1.1 202 Accepted
```

### REVIVE（复活）
&emsp;&emsp;由调度器发送以删除它以前通过ACCETP或DECLINE调用设置的任何/所有过滤器

``` text
REVIVE Request (JSON):
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "REVIVE"
}

REVIVE Response:
HTTP/1.1 202 Accepted
```

### KILL(杀死)
&emsp;&emsp;由调度器发送以杀死指定的任务。如果有调度器有自定义的executor，kill将转发给该executor；由executor杀死任务，并发送TASK_KILLED(或TASK_FAILED)更新。如果当mesos-master或mesos-agent收到kill请求时，任务还未派发给executor，将产生TASK_KILLED，并且任务不再发给executor。请注意，如果该任务属于一个任务组，杀死一个任务导致任务组中的所有任务都将被终止。一旦Mesos收到一个任务的终止状态，将释放该任务的资源。如果任务对于mesos-master是未知的，将产生TASK_LOST。

``` text
KILL Request (JSON):
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "KILL",
  "kill"            : {
    "task_id"   :  {"value" : "12220-3440-12532-my-task"},
    "agent_id"  :  {"value" : "12220-3440-12532-S1233"}
  }
}

KILL Response:
HTTP/1.1 202 Accepted
```

### SHUTDOWN(关闭)
&emsp;&emsp;由调度器发送以关闭自定义的executor(请注意：这是一个新的接口，在旧版的API不存在)。当一个executor收到shutdown事件，它将终止所有其运行的任务(发送TASK_KILLED状态更新)，并终止自己。如果executor没有在某个超时时间(通过mesos-agent的flag --executor_shutdown_grace_period配置)，mesos-agent将强制销毁容器(executor和它的任务)，并将活动的task转为TASK_LOST。

``` text
SHUTDOWN Request (JSON):
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "SHUTDOWN",
  "shutdown"        : {
    "executor_id"   :  {"value" : "123450-2340-1232-my-executor"},
    "agent_id"      :  {"value" : "12220-3440-12532-S1233"}
  }
}

SHUTDOWN Response:
HTTP/1.1 202 Accepted
```

### ACKNOWLEDGE(应答)
&emsp;&emsp;由调度器发送以应答状态更新。请注意，在新版的api中，调度器负责显式地确认接受到status.uuid设置的状态更新。这些状态将重试，直到得到调度器的应答。调度器不必应答没有设置的status.uuid的状态更新，因为他们也不会重试。uuid字段包含以Base64编码的原始字节。

``` text
ACKNOWLEDGE Request (JSON):
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "ACKNOWLEDGE",
  "acknowledge"     : {
    "agent_id"  :  {"value" : "12220-3440-12532-S1233"},
    "task_id"   :  {"value" : "12220-3440-12532-my-task"},
    "uuid"      :  "jhadf73jhakdlfha723adf"
  }
}

ACKNOWLEDGE Response:
HTTP/1.1 202 Accepted
```

### RECONCILE()
&emsp;&emsp;由调度器发送以查询非终端任务的状态。这将导致mesos-master为列表中的每个任务发回UPDATE事件。mesos不再清楚的任务将返回TASK_LOST状态。如果任务列表为空，mesos-master将发回这个framwork所有已知的任务的UPDATE事件。

``` text
RECONCILE Request (JSON):
POST /api/v1/scheduler   HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "RECONCILE",
  "reconcile"       : {
    "tasks"     : [
                   { "task_id"  : {"value" : "312325"},
                     "agent_id" : {"value" : "123535"}
                   }
                  ]
  }
}

RECONCILE Response:
HTTP/1.1 202 Accepted
```

### MESSAGE(消息)
&emsp;&emsp;由调度器发送以发送任意二进制数据到executor。Mesos既不解释这个数据，也不对这个消息传递给executor做任何保证。数据由Base64编码二进制数据。

``` text
MESSAGE Request (JSON):
POST /api/v1/scheduler   HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "MESSAGE",
  "message"         : {
    "agent_id"       : {"value" : "12220-3440-12532-S1233"},
    "executor_id"    : {"value" : "my-framework-executor"},
    "data"           : "adaf838jahd748jnaldf"
  }
}

MESSAGE Response:
HTTP/1.1 202 Accepted
```

### REQUEST(请求)
&emsp;&emsp;由调度器发送以从mesos-master或分配器请求资源。内置的分层分配器简单地忽略此请求，但其他分配器(模块)可以以可定制的方式解释此请求。

``` text
Request (JSON):
POST /api/v1/scheduler   HTTP/1.1

Host: masterhost:5050
Content-Type: application/json
Mesos-Stream-Id: 130ae4e3-6b13-4ef4-baa9-9f2e85c3e9af

{
  "framework_id"    : {"value" : "12220-3440-12532-2345"},
  "type"            : "REQUEST",
  "requests"        : [
      {
         "agent_id"       : {"value" : "12220-3440-12532-S1233"},
         "resources"      : {}
      }
  ]
}

REQUEST Response:
HTTP/1.1 202 Accepted
```

### Events(事件)
&emsp;&emsp;调度器应该保持与"/scheduler"端点的长连接(即使在收到SUBSCRIBE HTTP响应事件后)。这是通过没有"Content-Length"报头设置的“Connection: keep-alive”和“Transfer-Encoding: chunked”的HTTP报头指示。由mesos生成的此framwork相关的所有后续事件通过这个连接流式传输。Mesos-master使用RecordIO格式编码每个事件，即以字节为单位用字符串表示事件的长度，之后是以JSON或二进制Protobuf(可能是压缩的)编码的事件。事件的长度是一个无符号的64位整数(编码为文本值)，永远不会为"0"。另外，请注意，RecordIO编码应该由调度器解码，而底层HTTP分块编码通常在应用层(调度器)不可见。事件使用的内容编码类型将由POST请求的Accept头(例如，Accept:application/json)确定。
&emsp;&emsp;下面这些事件当前由mesos-master发送。此信息的规范来源是在scheduler.proto。请注意，当发送JSON编码的事件时，mesos-master用Base64编码原始字节，用UTF-8编码字符串。

#### SUBSCRIBE(订阅)
&emsp;&emsp;这是mesos-master发送的第一个事件，当调度器通过长连接发送一个SUBSCRIBE请求时。有关格式，请参阅Call的“SUBSCRIBE”章节。

#### OFFERS()
&emsp;&emsp;每当有新的资源可以提供给Framwork时，由mesos-master发送。每个offer对应agent上一组资源。直到调度器接受或拒绝一个offer，这些资源都被认为分配给这个调度器，除非该offer被撤销，比如，由于agent失联或--offer_timeout

``` text
OFFERS Event (JSON)

<event-length>
{
  "type"    : "OFFERS",
  "offers"  : [
    {
      "offer_id"     : {"value": "12214-23523-O235235"},
      "framework_id" : {"value": "12124-235325-32425"},
      "agent_id"     : {"value": "12325-23523-S23523"},
      "hostname"     : "agent.host",
      "resources"    : [
                        {
                         "name"   : "cpus",
                         "type"   : "SCALAR",
                         "scalar" : {"value" : 2},
                         "role"   : "*"
                        }
                       ],
      "attributes"   : [
                        {
                         "name"   : "os",
                         "type"   : "TEXT",
                         "text"   : {"value" : "ubuntu16.04"}
                        }
                       ],
      "executor_ids" : [
                        {"value" : "12214-23523-my-executor"}
                       ]
    }
  ]
}
```

#### RESCIND(废除)
&emsp;&emsp;当一个特别的offer不再有效(比如：这个offer对应的agent被移除)时，由mesos-master发送，因此offer需要被废除。由scheduler产生的关于该offer的任何调用(ACCEPT/DECLINE)将是无效的。

``` text
RESCIND Event (JSON)

<event-length>
{
  "type"    : "RESCIND",
  "rescind" : {
    "offer_id"  : { "value" : "12214-23523-O235235"}
  }
}
```

#### UPDATE(更新)
&emsp;&emsp;每当executor,agent或者master产生任务状态更新时，由mesos-master发送。executor应使用状态更新来可靠地传达其管理的任务的状态。至关重要的是，一旦任务终结executor应立刻发送终结状态更新(比如：TASK_FINISHED, TASK_KILLED, TASK_FAILED)，以便mesos释放为该任务分配的资源。scheduler也有责任明确地确认接收到可靠重试的状态更新。请参阅上面的Calls章节部分中ACKNOWLEAGE语义。请注意，uuid和数据是以Base64编码的原始字节

``` text
UPDATE Event (JSON)

<event-length>
{
  "type"    : "UPDATE",
  "update"  : {
    "status"    : {
        "task_id"   : { "value" : "12344-my-task"},
        "state"     : "TASK_RUNNING",
        "source"    : "SOURCE_EXECUTOR",
        "uuid"      : "adfadfadbhgvjayd23r2uahj",
        "bytes"     : "uhdjfhuagdj63d7hadkf"
      }
  }
}
```

#### MESSAGE(消息)
&emsp;&emsp;由executor生成的自定义消息通过mesos-master转发给scheduler。此消息Mesos不解释，仅仅转发(没有可靠性保证)给scheduler。如果消息因任何原因丢失，由executor重发。“data”字段包含编码为Base64的原始字节。

``` text
MESSAGE Event (JSON)

<event-length>
{
  "type"    : "MESSAGE",
  "message" : {
    "agent_id"      : { "value" : "12214-23523-S235235"},
    "executor_id"   : { "value" : "12214-23523-my-executor"},
    "data"          : "adfadf3t2wa3353dfadf"
  }
}
```

#### FAILURE(事故)
&emsp;&emsp;当一个agent被从集群移除(比如：健康检查失败)或一个executor终止时，由mesos-master发送。此事件与该agent上任何活动任务的终止UPDATE事件以及属于该agent的任何未处理offer的RESCIND事件是重合的。请注意，在FAILURE, UPDATE和RESCIND事件之间没有保证顺序。

``` text
FAILURE Event (JSON)

<event-length>
{
  "type"    : "FAILURE",
  "failure" : {
    "agent_id"      : { "value" : "12214-23523-S235235"},
    "executor_id"   : { "value" : "12214-23523-my-executor"},
    "status"        : 1
  }
}
```

#### ERROR(错误)
&emsp;emsp;当一个异步错误事件产生(比如，framwork使用未授权的角色订阅)时，由mesos-master发送。建议：framwork在接收到错误时，中止；并在必要时重试订阅。

``` text
ERROR Event (JSON)

<event-length>
{
  "type"    : "ERROR",
  "message" : "Framework is not authorized"
}
``` 

#### HEARTBEAT(心跳)
&emsp;&emsp;该事件由mesos-master定期发送以通知scheduler连接是存活的。这还有助于确保网络中间件由于缺少数据流而不关闭订阅长连接。请参阅接下来的章节，关于scheduler如何能使用该事件来处理网络分区。

``` text
HEARTBEAT Event (JSON)

<event-length>
{
  "type"    : "HEARTBEAT"
}
```

### Disconnections(断开连接)
&emsp;&emsp;如果到"／scheduler"的订阅长连接(通过SUBSCRIBE请求打开)断开，Mesos-master认为这个scheduler断开连接。连接可能因为这些原因断开，比如：scheduler重启，scheduler故障切换，网络错误等。请注意，Mesos-master不跟踪到"/scheduler"的非订阅连接，因为它不是持久连接。
&emsp;&emsp;如果Mesos-master认为订阅连接断开，它将标记这个scheduler为"disconnected"，并且启动一个故障切换超时(故障切换超时是FramworkInfo的一部分)。它还会删除其队列中的任何待定事件。另外，Mesos-master用“403 Forbidden”拒绝后续到"/scheduler"的非订阅HTTP请求，直到scheduler重新订阅"/scheduler"为止。如果scheduler没有在故障切换的超时时间内重新订阅，Mesos-master认为这个scheduler永久停止，并关闭所有该scheduler的executors，从而杀死该scheduler的所有任务。因此，所有schedulers产品都被建议使用一个比较大的故障切换时间(比如，4周)。
&emsp;&emsp;请注意：要想在故障切换超时到期前，强制关闭framwork(比如：在framwork开发和调试期间)，framwork可用发送“TEARDOWN”调用(Scheduler API的一部分)，或者操作员通过使用mesos-master的"/teardown"端点(Operator API的一部分)。
&emsp;&emsp;如果scheduler意识到其对"/scheduler"的订阅连接断开或mesos-master已改变(比如：通过zookeeper)，scheduler应该重新订阅(使用退避策略)。这是通过在新的长连接上发送"SUBSCRIBE"请求(使用framwork ID)到(可能是新的)mesos-master上的"/scheduler"端点来完成的。除非收到了SUBSCRIBED事件，否则不应该发送非订阅HTTP请求到“/scheduler”；这些请求将导致“403 Forbidden”。
&emsp;&emsp;如果mesos-master没有意识到订阅连接断开，但是scheduler意识到它，scheduler可能打开了一个新的到"/scheduler"的长连接通过“SUBSCRIBE”。在这种情况下，mesos-master关闭现有的订阅连接，并允许在新的连接上订阅。这里不变的是mesos-master仅允许给定的framwork ID的一个订阅长连接。
&emsp;&emsp;mesos-master使用Mesos-Stream-Id头部来区分不同的scheduler实例。在具有多个实例的高可用性scheduler的情况下，这可以防止在某些故障情况下不必要的行为。每个唯一的Mesos-Stream-Id仅仅在每个单独的订阅连接生命周期内有效。每个SUBSCRIBE的请求回复中包含一个Mesos-Stream-Id，这个ID必须包含在通过这个订阅连接发送的所有后续非订阅调用中。每当建立新的订阅连接时，将生成一个新的Mesos-Stream-Id，并应在该连接的生命周期中使用。

#### 网络分区(Network partitions)
&emsp;&emsp;在网络分区的情况下，scheduler和mesos-master之间的订阅连接可能不一定中断。为了检测这种情况，mesos master定期(比如：15s)发送HEARTBEAT事件(类是Twitter的Streaming API)。如果scheduler在时间窗口内没有收到一束(比如，5)这些heartbeats，它应该立刻断开连接并重新订阅。强烈建议scheduler使用指数退避策略(比如，最多15秒)，以避免重新连接时对mesos-master的压力。Scheduler可以使用类似超时(例如，75秒)来接收对任何HTTP请求的响应。

### Master发现(Master detection)
&emsp;&emsp;Mesos-master有一个高可用模式，使用多个Mesos masters；一个活动的主master(称为leader或leading master)和几个备用masters，以防主master异常。Masters选择leader，通过zookeeper协作选举。更多详细信息，请参阅高可用章节。
&emsp;&emsp;Scheduler需要向leading master发出HTTP请求。如果请求发给非leading的master，将收到一个"HTTP 307 临时重定向"，其中头部的"Location"指向leading master。
&emsp;&emsp;当scheduler命中非leading master时，通过重定向的订阅工作流的例子如下：

``` text
Scheduler -> Master
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost1:5050
Content-Type: application/json
Accept: application/json
Connection: keep-alive

{
  "framework_info"	: {
    "user" :  "foo",
    "name" :  "Example HTTP Framework"
  },
  "type"			: "SUBSCRIBE"
}

Master -> Scheduler
HTTP/1.1 307 Temporary Redirect
Location: masterhost2:5050


Scheduler -> Master
POST /api/v1/scheduler  HTTP/1.1

Host: masterhost2:5050
Content-Type: application/json
Accept: application/json
Connection: keep-alive

{
  "framework_info"	: {
    "user" :  "foo",
    "name" :  "Example HTTP Framework"
  },
  "type"			: "SUBSCRIBE"
}
```

&emsp;&emsp;如果scheduler知道mesos集群所有master的主机名列表，可以通过这个机制找到leading的master。或者，scheduler可以使用一个库，通过zookeeper(或etcd)URL检测leading master。对于C++库，master基于zookeeper的发现，请参阅src/scheduler/scheduler.cpp

