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
开发人员面临的一个常见问题是torrents停止运行而没有解释。