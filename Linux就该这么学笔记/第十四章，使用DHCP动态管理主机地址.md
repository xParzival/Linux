##### 1. 动态主机地址管理协议(DHCP,Dynamic Host Configuration Protocol)
* 动态主机配置协议（DHCP）是一种基于UDP协议且仅限于在局域网内部使用的网络协议，主要用于大型的局域网环境或者存在较多移动办公设备的局域网环境中，其主要用途是为局域网内部的设备或网络供应商自动分配IP地址等参数
* DHCP协议不仅可以为主机自动分配网络参数，还可以确保主机使用的IP地址是唯一的，更重要的是，还能为特定主机分配固定的IP地址
<div align=center>
    <img src="https://www.linuxprobe.com/wp-content/uploads/2015/07/DHCP%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86.png">
</div>

* DHCP相关概念
  1. 作用域：一个完整的IP地址段，DHCP协议根据作用域来管理网络的分布、分配IP地址及其他配置参数
  2. 超级作用域：用于管理处于同一个物理网络中的多个逻辑子网段。超级作用域中包含了可以统一管理的作用域列表
  3. 排除范围：把作用域中的某些IP地址排除，确保这些IP地址不会分配给DHCP客户端
  4. 地址池：在定义了DHCP的作用域并应用了排除范围后，剩余的用来动态分配给DHCP客户端的IP地址范围
  5. 租约：DHCP客户端能够使用动态分配的IP地址的时间
  6. 预约：保证网络中的特定设备总是获取到相同的IP地址

##### 2. 部署dhcpd服务程序
* dhcpd是Linux系统中用于提供DHCP协议的服务程序，配置文件目录etc/dhcp/dhcpd.conf
* 一个标准的配置文件应该包括全局配置参数、子网网段声明、地址配置选项以及地址配置参数。其中，全局配置参数用于定义dhcpd服务程序的整体运行参数；子网网段声明用于配置整个子网段的地址属性。dhcpd配置格式
  ```
  ddns-update-style interim;
  ignore client-updates; # 全局配置参数
  subnet 192.168.10.0 netmask 255.255.255.0 { # 子网网段声明
      ......;
      option routers 192.168.10.1;
      option subnet-mask 255.255.255.0; # 地址配置选项
      ......;
      default-lease-time 21600;
      max-lease-time 43200; # 地址配置参数
      ......;
  }
  ```
* 常用参数列表

  参数|作用
  -|-
  ddns-update-style 类型|定义DNS服务动态更新的类型，类型包括：none（不支持动态更新）、interim（互动更新模式）与ad-hoc（特殊更新模式）
  allow/ignore client-updates|允许/忽略客户端更新DNS记录
  default-lease-time 21600|默认超时时间
  max-lease-time 43200|最大超时时间
  option domain-name-servers 8.8.8.8|定义DNS服务器地址
  option domain-name "domain.org"|定义DNS域名
  range|定义用于分配的IP地址池
  option subnet-mask|定义客户端的子网掩码
  option routers|定义客户端的网关地址
  broadcast-address 广播地址|定义客户端的广播地址
  ntp-server IP地址|定义客户端的网络时间服务器（NTP）
  nis-servers IP地址|定义客户端的NIS域服务器的地址
  hardware 硬件类型 MAC地址|指定网卡接口的类型与MAC地址
  server-name 主机名|向DHCP客户端通知DHCP服务器的主机名
  fixed-address IP地址|将某个固定的IP地址分配给指定主机
  time-offset 偏移差|指定客户端与格林尼治时间的偏移差

##### 3. 自动管理IP地址
* DHCP服务器会自动把IP地址、子网掩码、网关、DNS地址等网络信息分配给有需要的客户端，而且当客户端的租约时间到期后还可以自动回收所分配的IP地址，以便交给新加入的客户端
* 实例：明天会有100名学员自带笔记本电脑来我司培训学习，请保证他们能够使用机房的本地DHCP服务器自动获取IP地址并正常上网
  
  参数名称|值
  -|-
  默认租约时间|21600s
  最大租约时间|43200s
  IP地址范围|192.168.10.50-192.168.10.150
  子网掩码|255.255.255.0
  网关地址|192.168.10.1
  DNS服务器地址|192.168.10.1
  搜索域|linuxprobe.com

* 虚拟机软件自带DHCP服务，为了避免与自己配置的dhcpd服务程序产生冲突，应该先将虚拟机软件自带的DHCP功能关闭
  ```
  vim /etc/dhcp/dhcpd.conf
  ddns-update-style none;
  ignore client-updates;
  subnet 192.168.10.0 netmask 255.255.255.0 {
  range 192.168.10.50 192.168.10.150;
  option subnet-mask 255.255.255.0;
  option routers 192.168.10.1;
  option domain-name "linuxprobe.com";
  option domain-name-servers 192.168.10.1;
  default-lease-time 21600;
  max-lease-time 43200;
  }
  ```

  参数|作用
  -|-
  ddns-update-style none;|设置DNS服务不自动进行动态更新
  ignore client-updates;|忽略客户端更新DNS记录
  subnet 192.168.10.0 netmask 255.255.255.0|作用域为192.168.10.0/24网段
  range 192.168.10.50 192.168.10.150;|IP地址池为192.168.10.50-150（约100个IP地址）
  option subnet-mask 255.255.255.0;|定义客户端默认的子网掩码
  option routers 192.168.10.1;|定义客户端的网关地址
  option domain-name "linuxprobe.com";|定义默认的搜索域
  option domain-name-servers 192.168.10.1;|定义客户端的DNS地址
  default-lease-time 21600;|定义默认租约时间（单位：秒）
  max-lease-time 43200;|定义最大预约时间（单位：秒）

* 配置完成后重启dhcpd服务程序并加入开机启动程序中

##### 4. 分配固定IP地址
* 在DHCP协议中有个术语是“预约”，它用来确保局域网中特定的设备总是获取到固定的IP地址。换句话说，就是dhcpd服务程序会把某个IP地址私藏下来，只将其用于相匹配的特定设备。要想把某个IP地址与某台主机进行绑定，就需要用到这台主机的MAC地址。MAC地址是网卡上面的一串独立的标识符，具备唯一性，因此不会存在冲突的情况
* 绑定IP地址配置格式，将如下配置添加到子网网段声明的配置中
  ```
  host 主机名称 {
      hardware ethernet 该主机mac地址;
      fixed-address 预指定的IP地址;
  }
  ```