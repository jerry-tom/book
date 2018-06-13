### 块IO控制

#### 概览

&emsp;&emsp;cgroup子系统"blkio"实现块IO控制器。似乎有需要各种IO控制策略(如：比例带宽BW，最大带宽BW)无论是在叶节点还是在存储层次结构的中间节点。计划是对blkio控制器使用相同的基于cgroup的管理界面，并根据用户选项在后台切换IO策略。

&emsp;&emsp;目前实现了两个IO控制策略。第一个是成比例的加权时间划分磁盘策略。在CFQ中实现了该策略。于是，当使用CFQ时，此策略仅在叶节点上生效。第二个是限制策略，可以用来在设备上指定较高的IO速率限制。这个策略在通用块层中实现，可以使用在叶子节点以及更高级别的逻辑设备，比如设备映射器。

#### HOWTO

##### 带宽的比重划分

&emsp;&emsp; 你可以做个简单测试，在两个不同的cgroups中运行两个dd线程。你可以这样做：

&emsp;&emsp; - Enable Block IO controller

​		CONFIG_BLK_CGROUP=y

&emsp;&emsp;- Enable group scheduling in CFQ

​		CONFIG_CFQ_GROUP_IOSCHED=y

&emsp;&emsp;- 编译并引导到内核，并挂载IO控制器(blkio)；见cgroup.txt，为什么需要cgroup？

``` text
mount -t tmpfs cgroup_root /sys/fs/cgroup
mkdir /sys/fs/cgroup/blkio
mount -t cgroup -o blkio none /sys/fs/cgroup/blkio
```

&emsp;&emsp;- 创建两个组

``` text
mkdir -p /sys/fs/cgroup/blkio/test1/  /sys/fs/cgroup/blkio/test2
```

&emsp;&emsp; - 设置组test1和test2权重

``` text
echo 1000 > /sys/fs/cgroup/blkio/test1/blkio.weight
echo 500 > /sys/fs/cgroup/blkio/test2/blkio.weight
```

&emsp;&emsp;- 在相同的磁盘创建两个一样大小的文件file1, file2(比如说每个512MB)，并在不同的cgroup启动两个dd线程读取这些文件。

``` text

	sync
	echo 3 > /proc/sys/vm/drop_caches

	dd if=/mnt/sdb/zerofile1 of=/dev/null &
	echo $! > /sys/fs/cgroup/blkio/test1/tasks
	cat /sys/fs/cgroup/blkio/test1/tasks

	dd if=/mnt/sdb/zerofile2 of=/dev/null &
	echo $! > /sys/fs/cgroup/blkio/test2/tasks
	cat /sys/fs/cgroup/blkio/test2/tasks
```

&emsp;&emsp; - 在宏观层面，第一个dd应该第一个结束。为了获取更多精准数据，请保持查看(通过脚本帮助)，在blkio.disk_time和test1,test2组的blkio.disk_sectors文件。这将告诉多少磁盘时间(以毫秒为单位)，每组获取多少扇发送到磁盘。我们提供磁盘时间的公平性，所以理想情况下，cgroup的io.disk_time应该与权重成比例。

##### 限制/上限策略

&emsp;&emsp;- 启用块IO控制器

​	CONFIG_BLK_CGROUP=y

&emsp;&emsp;- 在块层中启用限流

​	CONFIG_BLK_DEV_THROTTLING=y

&emsp;&emsp;- 挂载blkio控制器（见cgroup.txt，为什么需要cgroups）

​	mount -t cgroup -o blkio none /sys/fs/cgroup/blkio

&emsp;&emsp;- 在特定设备上为root组指定带宽速率。格式是："<major>:<minor> <bytes_per_second>"

​	echo "8:16 1048576" > /sys/fs/cgroup/blkio/blkio.throttle.read_bps_device

​	上面配置将会对root组在具有major/minor数字8:16的设备上设置1MB/s的限制。

&emsp;&emsp;- 运行dd来读取一个文件，检查速率是否被限制在1MB/s

``` tex
 # dd iflag=direct if=/mnt/common/zerofile of=/dev/null bs=4K count=1024
        1024+0 records in
        1024+0 records out
        4194304 bytes (4.2 MB) copied, 4.0001 s, 1.0 MB/s
```

​	写限制可以使用blkio.throttle.write_bps_device文件

#####  分级cgroups

&emsp;&emsp;CFQ和限流实现了层次结构的支持；然而，限流层次结构的支持启用，仅且仅当"sane_behavior"在cgroup端开启，目前这是一个开发选项，而不是公开提供。

&emsp;&emsp; 如果有人创建了如下层次结构。

``` text
			root
			/  \
		     test1 test2
			|
			test3
```

&emsp;&emsp;默认的CFQ和带"sane_behavior"的限流将正确处理层次结构。有关CFQ层次结构支持的详细信息，参见Documentation/block/cfq-iosched.txt。有关限流，所有限制应用到整个子树，而所有的统计数据都是直接有cgroup中任务生产IO的本地数据。

&emsp;&emsp; 从cgroup端启用不带"sane_behavior"的节流将几乎对待所有groups在相同组，就看起来和下面一样。

``` text
					pivot
			     /  /   \  \
			root  test1 test2  test3
```

