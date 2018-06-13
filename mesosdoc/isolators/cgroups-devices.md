### Mesos Containerizer支持cgroups 'devices'子系统

cgroups/devices隔离器允许操作者为mesos containerizer运行的容器提供device隔离。它使用cgroups device whitelist controller去跟踪，强制打开和mknod限制设备文件。要开启cgroups/devices隔离，需要在启动mesos agent时，添加cgroups/devices到—isolation标签。

#### 默认白名单设备

如果你开启了这个隔离器，默认的，下列设备会列入每个容器的白名单。

每个白名单目录都有4个字段。类型是 a (所有), c (字符),或b (块)。'all'意思应用到所有类型，所有主要和次要数字。主要和次要要么是整形，要么是*。访问是r(读),w(写)和m(mknod)的一个组合。

- c * : * m : 使用mknod创建新字符设备
- b * : * m : 使用monod创建新的块设备
- c 5:1 rwm: 读/写 /dev/console
- c 4:0  rwm: 读/写 /dev/tty0
- c 4:1 rwm: 读/写 /dev/tty1
- c 136:* rwm: 读/写 /dev/pts/*
- c 5:2 rwm: 读/写 /dev/ptmx
- c 10:200 rwm: 读/写 /dev/net/tun
- c 1:3 rwm: 读/写 /dev/null
- c 1:5 rwm: 读/写 /dev/zero
- c 1:7 rwm: 读/写 /dev/full
- c 5:0 rwm: 读/写 /dev/tty
- c 1:9 rwm: 读/写 /dev/urandom
- c 1:8 rwm: 读/写 /dev/random

注意，cgroups设备白名单控制是基于设备编号的。这与填充/dev正交，通常由udev或devtmpfs完成。

无论设备是否被列入白名单，CAP_MKNOD能力总是被请求执行mknod(2)

#### 额外的白名单设备

操作者可以配置mesos agent增加额外的白名单设备，通过在mesos agent上使用—allowed_devices标签。该标签采用一个JSON对象(或包含该JSON对象的文件路径)。比如：

``` json
{
  "allowed_devices": [
    {
      "device": {
        "path": "/path/to/device"
      },
      "access": {
        "read": true,
        "write": false,
        "mknod": false
      }
    }
  ]
}
```

