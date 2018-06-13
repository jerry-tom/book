### 任务健康检查
&emsp;&emsp;有时应用程序崩溃，行为不端或变得无响应。为了发现和恢复这些情况，一些框架(例如：Marathon, Apache Aurora)实现了自己的逻辑，用于检查其任务的健康情况。这通常通过框架的scheduler发送“ping”请求来完成。比如,通过HTTP,到任务运行的主机，安排任务或executor对ping进行响应。虽然这个技术非常有用，但是通常实现的方式有几个缺点：

* 每个Mesos框架都使用自己的API和协议
* 框架开发者必须重新实现常用功能
* 如果任务和scheduler运行在不同的节点(通常是这种情况)，由scheduler发起的健康检查产生额外的网络流量；此外，任务和scheduler之间的网络故障可能使后者认为前者是不健康的，但可能不是这样
* 在框架scheduler中实现健康检查可能会性能瓶颈。如果框架管理大量的任务，为每个任务执行健康检查可能引起scheduler的性能问题。

&emsp;&emsp;为解决上述问题，Mesos1.2.0引入了Mesos-native健康检查设计，定义了通用api，用于命令，HTTP(S)和TCP健康检查，并为所有内置executors提供了参考实现。

&emsp;&emsp;请注意：与健康检查有关的一些功能在1.2.0之前的版本是可用的，但是它被认为是实验性的。

&emsp;&emsp;请注意：Mesos使用等效的waitpid()系统调用来监视每个基于进程的任务，包括Docker容器。这个技术允许检测和上报进程崩溃，但是对于进程仍然运行但是不响应的情况则不足。

&emsp;&emsp;本文档描述支持的健康检查类型，触及相关实施细节，并提及限制和注意事项。

#### Mesos本地健康检查(Mesos-native Health Checks)
&emsp;&emsp;与上面提到的最先进的"scheduler健康检查"模式相反，Mesos-native健康检查在agent节点上运行：Executor执行检查，而不是scheduler。这提高了可扩展性，但意味着从外部世界检测网络故障或任务可用性成为一个单独的问题。例如，如果任务在隔离的Agent上运行，它仍然会进行运行状况检查，如果这些运行状况检查失败--将被终止。不用说，由于网络隔离，所有这一切都将发生，而不需要通知框架scheduler。
&emsp;&emsp;利用任务状态更新将健康状况检查状态传递到Mesos master，并进一步传递给框架的scheduler，确保“至少一次”交付。bool类型的healthy用于传递健康状态，在某些情况下可能不足。这意味着健康检查失败的任务状态为RUNNING,healthy为false。目前，healthy字段只在TASK_RUNNING状态更新时设置。
&emsp;&emsp;当一个任务的状态转为非健康时，healthy字段设置为false的任务状态更新消息被发送给Mesos master，然后转发到scheduler。executor预计终止任务在连续多次检查非健康后，连续次数在HealthCheck protobuf协议的consecutive_failures字段中定义。
&emsp;&emsp;请注意：虽然scheduler当前无法取消由于健康检查失败而导致的kill任务，但是它可能会发出一个killTask命令本身。这可能有助于模拟一个“全局”策略来处理失败的健康检查任务(请参阅限制)
&emsp;&emsp;内置executor转发所有不健康的状态更新，以及任务变得健康的第一个健康更新，即当任务已经开始或在一个或多个不健康的更新发生之后。请注意，自定义的executor可能会使用不同的策略。
&emsp;&emsp;自定义executor可以使用健康检查库，用于健康检查的参考实现，所有内置executor依赖。

#### 健康检查剖析
&emsp;&emsp;HealthCheck的protobuf中描述了Mesos的健康检查。目前，只有任务可以进行健康检查，而不是任意进程或executor，即只有TaskInfo protobuf具有可选的HealthCheck字段。然而，值得注意的是，所有内置的执行程序将任务映射到进程。
&emsp;&emsp;executor有责任检查其所有任务的健康，因为只有executor知道如何解释TaskInfo。所有内置executor支持检查它们任务的健康
&emsp;&emsp;请注意：如果一个任务是由docker executor启动的Docker容器，它将被包装在docker run。对于所有其他任务，包括在mesos containerizer中启动的Docker容器，该命令将从任务的装入命名空间执行。

##### 健康检查 - Command
&emsp;&emsp;CommandInfo protobuf描述了Command健康检查；某些字段被忽略：CommandInfo.user和CommandInfo.uris。Command健康检查指定用于验证任务运行状况的任意命令。Executor启动命令并检查其退出码状态：0表示成功，其他状态表示失败。
&emsp;&emsp;请注意：如果一个任务是由docker executor启动的Docker容器，它将被包装在docker run中。对于所有其他任务，包括在mesos containerizer中启动的Docker容器，该命令将从任务的装入命令空间执行。
&emsp;&emsp;要指定command健康检查，请将type设置为HealthCheck::COMMAND并且填充CommandInfo，例如：

``` text
HealthCheck healthCheck;
healthCheck.set_type(HealthCheck::COMMAND);
healthCheck.mutable_command()->set_value("ls /checkfile > /dev/null");

task.mutable_health_check()->CopyFrom(healthCheck);
```

##### 健康检查 - HTTP(S)
&emsp;&emsp;具有scheme,port,path和statuses字段的HealthCheck.HTTPCheckInfo protobuf描述了HTTP(S)健康检查。使用curl命令发送一个GET请求到scheme://<host>:port/path。请注意，<host>当前不可配置，并自动解析为127.0.0.1(请参阅limitations)。scheme字段仅支持"http"和"https"值。port字段必须指定任务正在侦听的实际端口，而不是映射的端口。
&emsp;&emsp;内置的executor将200盒399之间的状态处理为成功；自定义的executor可以采用不同的策略，例如：利用statuses字段
&emsp;&emsp;请注意，设置HealthCheck.HTTPCheckInfo.status，对于内置的executors是没有效果的。
&emsp;&emsp;如果需要，executor在启动curl命令之前输入任务的网络命名空间。
&emsp;&emsp;要指定HTTP健康检查，请将type设置为HealthCheck::HTTP并且填充HTTPCheckInfo，例如:

``` text
HealthCheck healthCheck;
healthCheck.set_type(HealthCheck::HTTP);
healthCheck.mutable_http()->set_port(8080);
healthCheck.mutable_http()->set_scheme("http");
healthCheck.mutable_http()->set_path("/health");

task.mutable_health_check()->CopyFrom(healthCheck);
```

##### 健康检查 - TCP
&emsp;&emsp;HealthCheck.TCPCheckInfo protobuf描述了TCP健康检查，它具有单个port字段，它必须指定任务正在监听的实际端口，而非映射端口。使用Mesos的mesos-tcp-connect命令探测这个任务，该命令尝试建立一个到<host>:port的TCP链接。请注意，<host>当前不可配置，并自动解析为127.0.0.1（请参阅limitations）.
&emsp;&emsp;如果连接能够建立，就可以认为健康检查是成功的。
&emsp;&emsp;如果需要，executor在启动mesos-tcp-connect命令之前输入任务的网络命名空间。
&emsp;&emsp;要指定TCP健康检查，请将type设置为HealthCheck::TCP并且填充TCPCheckInfo，例如:

``` text
HealthCheck healthCheck;
healthCheck.set_type(HealthCheck::TCP);
healthCheck.mutable_tcp()->set_port(8080);

task.mutable_health_check()->CopyFrom(healthCheck);
```

##### 常用选项
&emsp;&emsp;HealthCheck protobuf包含常用选项，用于调节executor如何解读健康检查：

&emsp;&emsp; * delay_seconds 等待的时间，直到开始任务健康检查
&emsp;&emsp; * interval_seconds 健康检查的间隔时间
&emsp;&emsp; * timeout_seconds 等待健康检查完成的时间，超过这个时间，健康检查将被中止并认为检查失败。
&emsp;&emsp; * consecutive_failures 连续健康检查失败的次数，超过这个次数executor杀死任务
&emsp;&emsp; * grace_period_seconds 任务启动后，忽略健康检查失败的时间。一旦第一次健康检查成功，宽限期不再适用。请注意，宽限期包含delay_seconds,比如，设置grace_period_seconds < delay_seconds不起作用。
&emsp;&emsp;请注意：由于每次执行健康检查时都会启动一个帮助命令(请参阅限制)，将timeout_seconds设置为一个较小的值，例如<5s，可能导致间歇性故障。
&emsp;&emsp;举个例子，下面的代码指定了一个Docker容器的任务，一个简单的监听端口8080的HTTP服务，任务启动开始，每秒钟执行一次HTTP健康检查，并在前15秒允许连续故障，响应时间不到1秒

``` text
TaskInfo task = createTask(...);

// Use Netcat to emulate an HTTP server.
const string command =
    "nc -lk -p 8080 -e echo -e \"HTTP/1.1 200 OK\r\nContent-Length: 0\r\n\"";
task.mutable_command()->set_value(command)

Image image;
image.set_type(Image::DOCKER);
image.mutable_docker()->set_name("alpine");

ContainerInfo* container = task.mutable_container();
container->set_type(ContainerInfo::MESOS);
container->mutable_mesos()->mutable_image()->CopyFrom(image);

// Set `grace_period_seconds` here because it takes
// some time to launch Netcat to serve requests.
HealthCheck healthCheck;
healthCheck.set_type(HealthCheck::HTTP);
healthCheck.mutable_http()->set_port(8080);
healthCheck.set_delay_seconds(0);
healthCheck.set_interval_seconds(1);
healthCheck.set_timeout_seconds(1);
healthCheck.set_grace_period_seconds(15);

task.mutable_health_check()->CopyFrom(healthCheck);
```

#### Under the Hood
&emsp;&emsp;所有内置的executor都依赖于健康检查库，该库位于"src/checks"。executor为每个任务创建一个健康检查实例，并且通过额外的参数将定义的健康检查一起传递。在返回中，healthcheck库通知executor任务健康状态的变化。
&emsp;&emsp;该库依赖用于HTTP(S)检查的curl和用于TCP检查的mesos-tcp-connect（后者是mesos绑定的简单命令）。
&emsp;&emsp;该库最不平凡的事情之一是在linux agent上进入响应任务的命名空间(mnt,net)。要执行健康检查的命令，健康检查程序必须位于与被检查进程相同的命名空间中；这可以通过调用docker run，以便在docker containerizer情况下运行健康检查命令，或者通过在mesos containerizer的情况下显式调用setns(),设置mnt命名空间(参见containerization in Mesos)。要执行HTTP(S)或TCP健康检查，最可靠的解决方案是共享与被检查进程相同的网络命名空间；在docker containerizer情况下，setns()设置网络命名空间，需要显示调用。而mesos containerizer保证executor和它的任务在同一个网络命名空间。 
&emsp;&emsp;请注意：自定义的executors可能用也可能没有用这个库。请检查相应的framwork文件
&emsp;&emsp;不管executor如何，用于任务健康检查的所有资源都被视为任务的资源分配。因此如果指定了Mesos-native健康检查，则为任务定义添加一些额外的资源（比如：0.05cpu和32MB内存）是个好主意。

#### 当前限制
&emsp;&emsp;* 当一个任务变成不健康的，则在HealthCheck.consecutive_failures失败后，其被视为被杀死。这个决定是由executor在本地进行的，scheduler没有办法进行干预和反应。解决办法是将HealthCheck.consecutive_failures设置为一些较大的值，以便scheduler可以做出反应。一个可能的解决方案是引入一项处理不健康任务的"global"策略（见MESOS-6171）
&emsp;&emsp;* HTTP(S)和TCP健康检查使用127.0.0.1作为目标ip。因此，如果任务想要支持HTTP(S)或TCP健康检查，除了它们需要监听的接口外，还需要监听环形回路(见MESOS-6517)
&emsp;&emsp;* HTTP(S)健康检查依赖curl；如果curl不可用，健康检查被认为失败。
&emsp;&emsp;* TCP健康检查在windows上不支持(见MESOS-6117)
&emsp;&emsp;* 一个任务只允许一个健康检查(见MESOS-5962)
&emsp;&emsp;* 每次运行健康检查时，都会启动一个helper command。这会引入一些运行时间开销(见MESOS-6766)
&emsp;&emsp;* 没有健康检查的任务可能与具有健康检查但仍处于宽限期的任务无法区分。应该引入一个额外的状态(见MESOS-6417)
&emsp;&emsp;* 任务的健康状态不能从外部分配，例如由operator通过一个endpoint指定


