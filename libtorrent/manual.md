## libtorrent API 文档
### 概览
libtorrent接口包括几个类。主要的类是session，它包含所有torrents的主循环。  
基本用法如下：
- 构造一个session
- 从设置文件中加载session状态（见 load_state()）
- 启动扩展（见 add_extension()）
- 启动DHT,LSD,UPnP,NAT-PMP等（见 start_dht(), start_lsd(), start_upnp(), start_natpmp()）
- 解析torrent文件，并将它们加到session中（见 torrent_info, async_add_torrent(), add_torrent()）
- 主循环（见 session）
    - 轮询警告（见 wait_for_alert(), pop_alerts()）
    - 处理torrent更新（见 state_update_alert）
    - 处理其他警告（见 alert）
    - 查询session以获取信息（见 session::status()）
    - 从session中添加或删除torrent（见 remove_torrent()）
- 保存所有torrent_handle的恢复数据（可选项，见 save_resume_data()）
- 保存session状态（见 save_state()）
- 析构session对象  

本手册介绍了每个类和功能，你可能还想看看本教程。  
有关如何创建torrent文件的详细描述，请参见 create_torrent

### 前置声明
不建议从libtorrent名称空间中声明类型，因为在将来的版本中它可能会被放弃。而是在libtorrent中包含libtorrent/fwd.hpp用于所有公共类型的前向声明。

### 故障排除
开发人员面临的一个常见问题是torrents停止运行而没有解释。这是关于在哪个条件下，libtorrent将停止你的torrents的描述，如何找到他以及如何处理。  
确保保持对torrents的暂停状态，错误状态和上传模式的跟踪。默认情况下，torrents是自动管理的，这意味着libtorrent会暂停，恢复，抓取并自动退出上传模式。  
每当torrent遇到指明错误时，它将被停止，并且torrent_status::error将描述导致该错误的错误。如果对torrents进行自动管理，则会根据每个种子下载者数量定期对其进行剪贴并暂停或恢复。这将有效地播种最需要种子的种子。  
如果torrent遇到磁盘写错误，它将进入上传模式。这意味着它将不会下载任务内容，只会上传。假定写错误是由于磁盘已满或写许可权错误引起的。如果torrent是自动管理的，它将定期退出上传模式，并尝试再次将内容写入磁盘。这意味着如果问题解决，torrent将从某些磁盘错误中恢复。如果没有自动管理torrent，则必须调用set_upload_mode()才能重新打开下载。
有关如何解决性能问题的详细指南, 请参阅[troubleshooting](troubleshooting.md)

### 网络原语
libtorrent名称空间中有一些typedef，它们从boost::asio名称空间中提取网络类型，这些是：
``` text
using address = boost::asio::ip::address;
using address_v4 = boost::asio::ip::address_v4;
using address_v6 = boost::asio::ip::address_v6;
using boost::asio::ip::tcp;
using boost::asio::ip::udp;
```
这些类型在<libtorrent/socket.hpp>头文件中声明。
using语句将使你可以轻松访问：
``` text
tcp::endpoint
udp::endpoint
```
这些都是在libtorrent中使用的endpoint类型. endpoint是具有关联端口的IP地址。  
有关这些类型的文档，请参阅[asio documentation](boost_asio.md)

### 例外情况
libtorrent中的许多函数都有两个版本，一个版本会再错误时抛出异常，另一个版本会使用error_code参考，并在错误信息中填充错误代码。