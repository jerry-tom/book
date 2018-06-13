## Container Network Interface(CNI) for Mesos Containers
&emsp;&emsp;本文档描述了网络／CNI隔离器，MesosContainerizer网络隔离器实现了容器网络接口(CNI)规范。网络/CNI隔离器允许使用MesosContainerizer启动的容器连接几种不同类型的IP networks。可以启动容器的网络技术从传统第三层/第二层网络(比如：VLAN, ipvlan, macvlan)，到为容器编排设计的新类别(比如: Calico, Weave和Flannel)。默认情况下，MesosContainerizer启用了网络/CNI隔离器。

### 目录
* 动机
* 使用
  * 配置CNI网络
  * 增加／删除／修改 CNI网络
  * 将容器安装到CNI网络
  * 访问容器网络命名空间
  * 传到网络labels和端口映射信息到CNI插件
* 网络类型
  * 桥接网络
  * CNI网络的端口映射插件
  * Calico网络
  * Cilium网络
  * Weave网络

### 动机
&emsp;&emsp;为每个容器设置单独的网络空间对于编排引擎如mesos是非常有吸引力的，因为它为容器提供网络隔离，并且允许用户在容器操作，就像在主机上操作一样。没有网络隔离用户必须处理网络资源的管理比如一个端点主机上的TCP/UDP端口，使应用程序的设计变得复杂化。
&emsp;&emsp;挑战在于实现编排引擎与底层网络的通信能力，以便配置到容器的IP连接。出现这个问题，是由于IPAM(ip地址管理系统)的选择和可用于启用ip连接的网络技术的多样性。为了解决这个问题，我们需要采用一个基于驱动程序的网络编排模型，其中MesosContainerizer可以将配置ip连接到商业智能卸载到容器，到特定网络驱动程序。
&emsp;&emsp;容器网络接口(CNI)是CoreOS提出的一种基于驱动程序模型的规范倡议。该规范定义了一个JSON模式，定义了CNI插件(网络驱动程序)的输入和输出。该规范还提供了容器运行时和CNI插件的明确分离。根据规范，容器运行时预期配置容器的namespace, 容器的唯一标识符(容器id)以及为插件定义了给定网络配置参数的JSON格式输入。插件的功能是创建一个veth对，并将veth对其中一个附加到容器的网络命名空间，veth对的另外一个连接到插件已知的网络。CNI规范允许多个网络同时存在，每个网络由规范名称表示，并与唯一的CNI配置相关联。已有的各种CNI网络插件，比如bridge, ipvlan, macvlan, Calico, Weave和Flannel.
&emsp;&emsp;因此，通过网络/cni隔离器在Mesos中引入对CNI的支持，为Mesos提供了巨大的灵活性，以便在各种网络技术上协调容器。

### 使用
&emsp;&emsp;网络/cni隔离器默认是开启的。然而，使用隔离器时，operator和框架需要某些操作。在本节中，我们指定operator在mesos上配置CNI网络所需的步骤及将容器附加到CNI网络的框架所需的步骤。

#### 配置CNI网络
&emsp;&emsp;为了配置network/cni隔离器，operator需在Agent启动时指定两个flags。如下所示：

``` text
sudo mesos-slave --master=<master IP> --ip=<Agent IP>
  --work_dir=/var/lib/mesos
  --network_cni_config_dir=<location of CNI configs>
  --network_cni_plugins_dir=<search path for CNI plugins>
```
&emsp;&emsp;请注意，网络/cni隔离器通过在启动时查看在--network_cni_config_dir的CNI配置来了解所有可用的网络。这意味着如果想在mesos agent启动后添加一个新的CNI网络，需要重新启动mesos agent。网络/cni隔离器已经设计有恢复能力，因此重启mesos agent(因此网络/cni隔离器)不会影响容器编排。

#### 增加/删除/修改 CNI网络
&emsp;&emsp;网络/cni隔离器通过读取--network_cni_config_dir指定的cni配置了解所有的CNI网络。因此，如果操作员想要添加CNI网络，则需要将相应的配置添加到--network_cni_config_dir
&emsp;&emsp;当网络/cni隔离器通过读取--network_cni_config_dir中的CNI配置文件来学习CNI网络时，它不会保留CNI配置的内存副本。网络/cni隔离器仅存储CNI网络到相应CNI配置文件的映射。每当网络/cni隔离器需要将容器附加到CNI网络时，它将从磁盘读取相应的配置，并使用指定JSON配置来调用相应的插件。虽然网络/cni隔离器不保存JSON配置的内存副本，但它检查用于启动容器的CNI配置。CNI配置的检查点保护与容器关联的资源，通过当容器被销毁时，正确删除它们，即使CNI配置被删除。
&emsp;&emsp;网络/cni隔离器总是从磁盘读取CNI配置的事实允许operator动态增加，修改和删除CNI配置，而不需要重启mesos agent。每当operator添加一个新的CNI配置，或修改一个存在的CNI配置时，当下一个容器使用这个指定的CNI网络启动时，mesos agent将加载这个新的配置。类似地，当operator删除CNI网络时，网络/CNI隔离器将"不知道"CNI网络(因为它将启动时引用此CNI网络)，以防framwork尝试在删除的CNI网络上启动容器。

#### 将容器安装到CNI网络
&emsp;&emsp;Framwork可以通过在NetworkInfo protobuf中设置name字段来指定要将其容器附加到的CNI网络。name字段作为MESOS-4758的一部分在Networkinfo protobuf中有介绍。此外，通过在每个protobuf中指定具有不同name的NetworkInfo protobuf的多个实例, MesosContainer将容器附加到指定的所有不同的CNI网络。
&emsp;&emsp;容器的默认行为时加入到主机网络。例如，如果framwork在NetworkInfo protobuf中没有指定name字段，则网络/cni隔离器将是该容器的无操作，并且不会将新的网络命名空间与容器关联。这将有效地使容器使用主机的网络空间，并将其附加到主机网络。

``` text
**注意**：当指定多个 `NetworkInfo` protobuf时，可以将容器附加到不同的CNI网络。如果某个`NetworkInfo` protobuf没有指定`name`字段，`network/cni`隔离器只是"跳过"这个protobuf,将容器附加到除了`host network`之外的所有指定的CNI网络。附上一个容器到主机网络以及其他CNI网络，你需要将容器附加到CNI网络(例如bridge/macvlan),然后依次连接到主机网络。
```

#### 传递网络labels和端口映射信息到CNI插件
&emsp;&emsp;当调用CNI插件时(比如，通过ADD命令),隔离器会根据CNI规范制定网络配置JSON中的args字段，将一些Mesos元数据传递给插件。当前，隔离器仅将相关网络的NetworkInfo信息传递给插件。这只是NetworkInfo protobuf的JSON表示形式。例如:

``` json
{
  "name" : "mynet",
  "type" : "bridge",
  "args" : {
    "org.apache.mesos" : {
      "network_info" : {
        "name" : "mynet",
        "labels" : {
          "labels" : [
            { "key" : "app", "value" : "myapp" },
            { "key" : "env", "value" : "prod" }
          ]
        },
        "port_mappings" : [
          { "host_port" : 8080, "container_port" : 80 },
          { "host_port" : 8081, "container_port" : 443 }
        ]
      }
    }
  }
}
```

&emsp;&emsp;需要注意的是NetworkInfo中的labels或port_mappings由启动容器的framworks来设置，隔离器将这些信息传递给CNI插件。根据规范，CNI插件的特权是使用这些元数据信息，因为他们认为在将容器附加/分离到CNI网络时是合适的。比如，CNI插件可以使用labels来实施特定与域的策略，或者使用port_mappings来实现NAT规则

#### 访问容器网络名称空间
&emsp;&emsp;network/cni在需要附加容器到CNI网络时，它将网络namespace分配给容器。网络namespace在主机文件系统上进行检查点设置，可用于调试到网络名称空间的网络连接。对于给定的network/cni隔离器检查点，它的namespace在：

``` text
/var/run/mesos/isolators/network/cni/<container ID>/ns
```

&emsp;&emsp;网络namespace可以与iproute2软件包中的ip命令一起使用，方法是创建一个到网络namespace的符号链接。假设容器id是5baff64c-d028-47ba-864e-a5ee679fc069，你可以创建如下软连接:

``` text
ln -s /var/run/mesos/isolators/network/cni/5baff64c-d028-47ba-8ff64c64e-a5ee679fc069/ns /var/run/netns/5baff64c
```

&emsp;&emsp;现在我们可以使用网络namespace标识符5baff64c来使用iproute2包在新的网络namespace中运行命令:

``` text
ip netns exec 5baff64c ip link
```

&emsp;&emsp;同样你也可以查看容器的路由表通过运行：

``` text
ip netns exec 5baff64c ip route show
```
&emsp;&emsp;注意：一旦 MESOS-5278完成，在容器网络namespace中执行命令将被简化，并且我们将不再依赖iproute2包来调试Mesos容器网络。

### Networking Recipes
&emsp;&emsp;本章节介绍在不同CNI网络启动容器的示例。对于每个示例，假设CNI配置存在于/var/libmesos/cni/config中，插件当前在/var/lib/mesos/cni/plugins。因此Agents需要通过下面命令启动:

``` text
sudo mesos-slave --master=<master IP> --ip=<Agent IP>
--work_dir=/var/lib/mesos
--network_cni_config_dir=/var/lib/mesos/cni/config
--network_cni_plugins_dir=/var/lib/mesos/cni/plugins
--isolation=filesystem/linux,docker/runtime
--image_providers=docker
```

&emsp;&emsp; 除了CNI配置参数外，我们还启动了Agent，能够在MesosContainerizer启动容器镜像。我们在Mesoscontainerizer中通过开启filesystem/linux和docker/runtime隔离器并将镜像提供者设置为docker来启动此功能。

&emsp;&emsp; 介绍在特定CNI网络上启动容器的框架示例，mesos-execute CLI框架已修改为带—network的flag，这将允许此示例框架在特定的CNI网络上启动容器。你可以在Mesos安装目录<mesosinstallation>/bin/mesos-execute中找到mesos-execute框架。

#### A bridge network

&emsp;&emsp; 网桥插件附加容器到一个linux网桥。Linux网桥可以配置为连接到VLAN和VxLAN，允许容器插入到现有的二层网络。下面我们举一个例子，CNI配置指示MesosContainerizer调用一个网桥插件将一个容器连接到一个Linux网桥。该配置还指示网桥插件通过调用一个host-local IPAM为容器分配ip地址。

&emsp;&emsp; 首先，根据[CNI repository](https://github.com/containernetworking/cni)中的说明编译CNI插件，然后拷贝网桥二进制文件到每个mesos agent的插件目录中。

&emsp;&emsp; 接下来，创建配置文件并将其复制到每个mesos agent的CNI配置目录中。

``` json
{
"name": "cni-test",
"type": "bridge",
"bridge": "mesos-cni0",
"isGateway": true,
"ipMasq": true,
"ipam": {
    "type": "host-local",
    "subnet": "192.168.0.0/16",
    "routes": [
    { "dst":
      "0.0.0.0/0" }
    ]
  }
}
```

&emsp;&emsp; CNI配置告知网桥插件将容器附加到一个名为mesos-cni0的网桥。如果网桥不存在，网桥插件将创建一个。

&emsp;&emsp; 注意ipam字典中route部分很重要。对于mesos，作为容器启动的exector需要向Agent注册,以便成功启动任务。因此，Agent ip必须从容器ip可达，反之亦然。在这个特定的例子中，我们为容器指定了一个默认的route，允许容器到达网关可以路由的任何网络，对于这个CNI配置来说，这个网络本身就是网桥。

&emsp;&emsp; CNI配置另外一个有趣的属性是ipMasq选项。将其设置为true，将在主机的网络空间新增一个iptable规则，将对来自容器的所有流量进行SNAT，并流出Agent。这将允许容器访问外部，即使容器位于不能从Agent外部路由的地址空间中。

&emsp;&emsp; 下面我们给出一个运行ubuntu容器的例子，并连接它到mesos-cni0网桥。你可以运行Ubuntu容器通过使用如下的mesos-execute框架:

``` text
sudo mesos-execute --command=/bin/bash
  --docker_image=ubuntu:latest --master=<master IP>:5050 --name=ubuntu
  --networks=cni-test --no-shell
```

&emsp;&emsp; 上面的命令将从docker hub拉取Ubuntu镜像，并通过使用MesosContainerizer启动它，然后将其附加到mesos-cni0网桥。 

&emsp;&emsp; 你可以通过创建一个到网络命名空间的软链并并运行 ip 命令来检验Ubuntu容器的网络设置，像"访问容器网络名称空间"一节中所述。

&emsp;&emsp; 假设我们在/var/run/netns/5baff64c中为网络命名空间创建了一个引用。在容器网络空间中IP地址和路由表的输出如下：

``` text
$ sudo ip netns exec 5baff64c ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 8a:2c:f9:41:0a:54 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::882c:f9ff:fe41:a54/64 scope link
       valid_lft forever preferred_lft forever

$ sudo ip netns exec 5baff64c ip route show
default via 192.168.0.1 dev eth0
192.168.0.0/16 dev eth0  proto kernel  scope link  src 192.168.0.2
```

#### CNI网络的端口映射器插件

&emsp;&emsp; 对于专用，隔离的网络，例如桥接网络，其容器的IP地址不能从主机外部路由。提供具有DNAT能力的容器变得势在必行，以便运行在容器上的服务可以暴露在运行容器的主机之外。

&emsp;&emsp; 不幸的是，在containernetworking/cni库中没有提供端口映射功能的CNI插件可用。因此，我们开发了一个位于Mesos 代码库中的端口映射插件，叫做mesos-cni-port-mapper。mesos-cni-port-mapper被设计为与需要DNAT功能的任何其他CNI插件一起使用。其中最明显的是桥接CNI插件。

&emsp;&emsp; 我们通过一个CNI配置示例来解释mesos-cni-port-mapper插件的操作语义，该配置允许mesos-cni-port-mapper向网桥插件提供DNAT功能。

``` json
{
  "name" : "port-mapper-test",
  "type" : "mesos-cni-port-mapper",
  "excludeDevices" : ["mesos-cni0"],
  "chain": "MESOS-TEST-PORT-MAPPER",
  "delegate": {
      "type": "bridge",
      "bridge": "mesos-cni0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.0.0/16",
        "routes": [
        { "dst":
          "0.0.0.0/0" }
        ]
      }
  }
}
```

&emsp;&emsp; 对于上面的CNI配置，除了mesos-cni-port-mapper插件接受的参数外，在插件CNI配置中要重点关注的是"delegate"字段。"delegate"字段允许mesos-cni-port-mapper包装任何其他CNI插件的CNI配置，并允许插件向任何CNI网络提供DNAT功能。在这种特殊情况下，mesos-cni-port-mapper为运行在mesos-cni0网桥上的容器提供DNAT功能。下面列出了mesos-cni-port-mapper接受的参数：

- name：CNI网络的名称
- type：端口映射CNI插件的名称
- chain：iptables DNAT规则将在NAT表中添加的链。这允许操作者在给定的CNI网络下分组DNAT规则，从而更好的管理iptables规则。
- excludeDevices：这些是不应用DNAT规则的入口设备列表。
- delegate：这是一个json字典，它保存了CNI插件的CNI JSON配置，port-mapper插件需要调用。

&emsp;&emsp; mesos-cni-port-mapper在很大程度上依赖于iptables来为CNI网络提供DNAT功能。为了port-mapper插件可以正常工作，我们对iptables最低限度的需要，如下列表所示的：

- iptables 1.4.20或更高：这是因为我们需要使用-w选项，以便允许自动写iptables。
- 需要iptables的xt_comments模块：我们使用注释模块标记iptables规则属于一个容器。这些标示被用作关键字，同时在特定容器被删除时删除iptables规则。

&emsp;&emsp; 最后，port-mapper插件的CNI配置告诉插件如何以及在哪里安装iptables规则，以及哪个CNI插件"委托"容器的附加/分离，port-mapper信息本身是通过查看有Mesos传递给port-mapper插件的CNI配置的args字段中设置的NetworkInfo来学习的。更多详情请参阅"传递网络标签和端口映射信息给CNI插件"章节。

#### Calico网络

&emsp;&emsp; Calico提供第三方的CNI插件，可以与Mesos CNI一起工作。

&emsp;&emsp; Calico采用纯粹的Layer-3方法进行联网，为每个mesos任务分配一个唯一的，可以路由的IP地址。任务路由通过运行在每个mesos-agent上的BGP vRouter分配，它利用现有的Linux转发引擎，而不需要隧道，NAT或overlays。Calico支持丰富而灵活的网络策略，它使用每个计算节点的预定ACLs来执行，已提供租户隔离，安全组和外部可达约束。

&emsp;&emsp; 设置和使用Calico-CNI更多信息，请见 [Calico's guide on adding Calico-CNI to Mesos](https://github.com/projectcalico/calico-containers/blob/master/docs/mesos/ManualInstallCalicoCNI.md).

#### Cilium网络

&emsp;&emsp; Cilium提供了一个工作在mesos的CNI插件。

&emsp;&emsp; Cilium为Linux容器框架带来了HTTP-aware网络安全过滤。使用称为BPF的新Linux内核技术，Cilium提供了一种简单有效的方法来定义和实施网络层，HTTP层的安全策略。

&emsp;&emsp; 有关在Mesos中使用Cilium的更多信息，请参阅 [Getting Started Using Mesos Guide](http://docs.cilium.io/try-mesos).

#### Weave网络

&emsp;&emsp; Weave提供了一个CNI实现，可以与mesos一起使用。

&emsp;&emsp; Weave通过分配一个ip-per-container并在每个节点上提供一个快速的DNS来提供无忧的配置。Weave速度很快，通过自动选择主机之间最快的路径。完全支持组播地址和路由。它内置了NAT穿越和加密功能，即使在网络分区过程中也能继续工作。最后，即使有多跳，多云部署也很容易设置和维护。

&emsp;&emsp; 设置和使用Weave CNI更多信息，请参阅[Weave's CNI documentation](https://www.weave.works/docs/net/latest/cni-plugin/).