##### 1. 配置网络服务
* 配置网络参数  
  在RHEL 5、RHEL 6中，网卡配置文件的前缀为eth，第1块网卡为eth0，第2块网卡为eth1；以此类推。而在RHEL 7中，网卡配置文件的前缀则以ifcfg开始，加上网卡名称共同组成了网卡配置文件的名字，例如ifcfg-eno16777736
  1. Vim编辑器配置网络参数
     1. 切换到/etc/sysconfig/network-scripts目录中（存放着网卡的配置文件）
     2. 使用Vim编辑器修改网卡文件ifcfg-enoxxxxxxxx，逐项写入下面的配置参数并保存退出
        > 设备类型：TYPE=Ethernet  
        > 地址分配模式：BOOTPROTO=static  
        > 网卡名称：DEVICE=enoxxxxxxxx  
        > 是否启动：ONBOOT=yes  
        > IP地址：IPADDR=192.168.10.10  
        > 子网掩码：NETMASK=255.255.255.0  
        > 网关地址：GATEWAY=192.168.10.1  
        > DNS地址：DNS1=192.168.10.1  

     3. 重启网络服务并测试网络是否联通
        ```systemctl restart network```
  2. nmtui命令配置网络参数:图形化界面，按照对话框设置即可
* 创建网络会话  
  * RHEL和CentOS系统默认使用NetworkManager来提供网络服务，这是一种动态管理网络配置的守护进程，能够让网络设备保持连接状态  
  * 可以使用nmcli命令来管理Network Manager服务
    ```
    nmcli connection show
    nmcli con show ipfig-enoxxxxxxxx
    ```
  * RHEL7系统支持网络会话功能，允许用户在多个配置文件中快速切换。如果我们在公司网络中使用笔记本电脑时需要手动指定网络的IP地址，而回到家中则是使用DHCP自动分配IP地址。这就需要麻烦地频繁修改IP地址，但是使用了网络会话功能后一切就简单多了—只需在不同的使用环境中激活相应的网络会话，就可以实现网络配置信息的自动切换  
    1. 创建网络会话
        ```
        nmcli connection add con-name 网络会话名 ifname 本机网卡名 autoconnect no type ethernet ip4 192.168.10.10/24 gw4 192.168.10.1
        nmcli connection add con-name 网络会话名 type ethernet ifname 本机网卡名
        ```
    2. 激活网络会话
        ```nmcli connection up 指定网络会话名```
* 网卡绑定技术
  * 一般来讲，生产环境必须提供7×24小时的网络传输服务。借助于网卡绑定技术，不仅可以提高网络传输速度，更重要的是，还可以确保在其中一块网卡出现故障时，依然可以正常提供网络服务。假设我们对两块网卡实施了绑定技术，这样在正常工作中它们会共同传输数据，使得网络传输的速度变得更快；而且即使有一块网卡突然出现了故障，另外一块网卡便会立即自动顶替上去，保证数据传输不会中断
  * 网卡绑定方法
    1. 添加一块新网卡设备，与已有网卡为相同模式
    2. 配置两张网卡设备的配置文件。需要注意的是，这些原本独立的网卡设备此时需要被配置成为一块“从属”网卡，服务于“主”网卡，不应该再有自己的IP地址等信息
        > 设备类型：TYPE=Ethernet  
        > 地址分配模式：BOOTPROTO=static  
        > 网卡名称：DEVICE=enoxxxxxxxx  
        > 是否启动：ONBOOT=yes  
        > 是否允许非root用户控制该设备：USERCTL=no  
        > 网卡绑定之后的名称：MASTER=与主网卡名一致  
        > 是否为从属网卡：SLAVE=yes

    3. 配置绑定后的主网卡
        > 设备类型：TYPE=Ethernet  
        > 地址分配模式：BOOTPROTO=static  
        > 网卡名称：DEVICE=指定名称  
        > 是否启动：ONBOOT=yes  
        > 是否允许非root用户控制该设备：USERCTL=no  
        > IP地址：IPADDR=192.168.10.10  
        > 子网掩码：NETMASK=255.255.255.0  
        > 网关地址：GATEWAY=192.168.10.1  
        > DNS地址：DNS1=192.168.10.1  
        > 是否启用network mamager使网卡配置实时生效，不需要重启：NM_CONTROLLED=no

    4. 让Linux内核支持网卡绑定驱动
      * 绑定模式有七种(0-6),常用以下三种之一
        1. mode0(平衡负载模式)：平时两块网卡均工作，且自动备援，但需要在与服务器本地网卡相连的交换机设备上进行端口聚合来支持绑定技术
        2. mode1(自动备援模式)：平时只有一块网卡工作，在它故障后自动替换为另外的网卡
        3. mode6(平衡负载模式)：平时两块网卡均工作，且自动备援，无须交换机设备提供辅助支持
         ```
         vim /etc/modprobe.d/bond.conf
         alias bond0 bonding
         options bond0 miion=100 mode=6
         ```  
         miion表示监视网络链接的频度(毫秒), 我们设置的是100毫秒
    5. 重启网络服务后网卡绑定操作即可成功。正常情况下只有bond0网卡设备才会有IP地址等信息

##### 2. 远程控制服务
* 配置sshd服务  
  * SSH(Secure Shell)是一种能够以安全的方式提供远程登录的协议，也是目前远程管理Linux系统的首选方式。在此之前，一般使用FTP或Telnet来进行远程登录。但是因为它们以明文的形式在网络中传输账户密码和数据信息，因此很不安全  
  * 想要使用SSH协议来远程管理Linux系统，则需要部署配置sshd服务程序。sshd是基于SSH协议开发的一款远程管理服务程序，不仅使用起来方便快捷，而且能够提供两种安全验证的方法：
    1. 基于口令的验证—用账户和密码来验证登录
    2. 基于密钥的验证—需要在本地生成密钥对，然后把密钥对中的公钥上传至服务器，并与服务器中的公钥进行比较；该方式相较来说更安全
  * sshd服务的配置信息保存在/etc/ssh/sshd_config文件中
  
    参数|作用
    -|-
    Port 22|默认的sshd服务端口
    ListenAddress 0.0.0.0|设定sshd服务器监听的IP地址
    Protocol 2|SSH协议的版本号
    HostKey /etc/ssh/ssh_host_key|SSH协议版本为1时，DES私钥存放的位置
    HostKey /etc/ssh/ssh_host_rsa_key|SSH协议版本为2时，RSA私钥存放的位置
    HostKey /etc/ssh/ssh_host_dsa_key|SSH协议版本为2时，DSA私钥存放的位置
    PermitRootLogin yes|设定是否允许root管理员直接登录
    StrictModes yes|当远程用户私钥改变时直接拒绝连接
    MaxAuthTries 6|最大密码尝试次数
    MaxSessions 10|最大终端数
    PasswordAuthentication yes|是否允许密码验证
    PermitEmptyPasswords no|是否允许空密码登录（很不安全）

  * ssh [参数] 主机IP地址：远程登录指定IP的主机
* 安全密钥验证
  * 加密是对信息进行编码和解码的技术，它通过一定的算法（密钥）将原本可以直接阅读的明文信息转换成密文形式。密钥即是密文的钥匙，有私钥和公钥之分。在传输数据时，如果担心被他人监听或截获，就可以在传输前先使用公钥对数据加密处理，然后再行传送。这样，只有掌握私钥的用户才能解密这段数据，除此之外的其他人即便截获了数据，一般也很难将其破译为明文信息
  * 如果正确配置了密钥验证方式，那么sshd服务程序将更加安全。具体的配置其步骤如下：
    1. 在客户端主机中生成“密钥对”
       ```
       ssh-keygen
       Generating public/private rsa key pair.
       Enter file in which to save the key (/root/.ssh/id_rsa):按回车键或设置密钥的存储路径
       Created directory '/root/.ssh'.
       Enter passphrase (empty for no passphrase):直接按回车键或设置密钥的密码
       Enter same passphrase again:再次按回车键或设置密钥的密码
       ```
    2. 把客户端主机中生成的公钥文件传送至远程主机
       ```
       ssh-copy-id 远程主机的IP地址
       ```
    3. 对服务器进行设置使其只允许密钥验证  
       在/etc/ssh/sshd_config文件中修改```PasswordAuthentication no```
  * 各配置文件解释：
    1. 客户机上
       * $HOME/.ssh/id_rsa：使用ssh-keygen命令后生成的私钥
       * $HOME.ssh/id_rsa.pub：使用ssh-keygen命令后生成的公钥
       * $HOME.ssh/known_hosts：本客户机登录过的远程主机
    2. 远程主机上
       * $HOME/.ssh/authorized_keys：可登录本远程主机的客户机公钥
* 远程传输命令
  * scp [参数] 本地文件 远程账户@远程IP地址:远程目录：传送文件到远程主机

    参数|作用
    -|-
    -v|显示详细连接进度
    -P|指定远程主机的sshd端口号
    -r|用于传输文件夹
    -6|使用IPv6协议

  * scp [参数] 远程账户@远程IP地址:远程文件 本地目录：把远程主机文件传送到本地

##### 3. 不间断会话服务
* 一般情况下，当与远程主机的会话被关闭时，在远程主机上运行的命令也随之被中断
* screen是一款能够实现多窗口远程控制的开源服务程序，简单来说就是为了解决网络异常中断或为了同时控制多个远程终端窗口而设计的程序。用户还可以使用screen服务程序同时在多个远程会话中自由切换，能够做到实现如下功能
  1. 会话恢复：即便网络中断，也可让会话随时恢复，确保用户不会失去对远程会话的控制
  2. 多窗口：每个会话都是独立运行的，拥有各自独立的输入输出终端窗口，终端窗口内显示过的信息也将被分开隔离保存，以便下次使用时依然能看到之前的操作记录
  3. 会话共享：当多个用户同时登录到远程服务器时，便可以使用会话共享功能让用户之间的输入输出信息共享
* 使用Yum软件仓库安装screen
  1. 以rhel7的光盘镜像作为本地Yum源来配置，挂载光盘镜像到一个挂载点
  2. 创建Yum仓库配置文件
     ```
     vim /etc/yum.repos.d/rhel7.repo
     [rhel7]
     name=rhel7
     baseurl=file:///挂载点目录
     enabled=1
     gpgcheck=0
     ```
  3. 用Yum仓库安装screen服务程序
     ```yum install screen```
* 管理远程会话
  
  参数|作用
  -|-
  -S|创建会话窗口
  -d|指定会话离线
  -r|恢复指定会话
  -x|一次性恢复所有会话
  -ls|显示当前已有的所有会话
  -wipe|删除目前无法使用的会话

  * exit退出已经进入的会话
  * 在日常的生产环境中，可以直接使用screen命令执行要运行的命令，这样在命令中的一切操作也都会被记录下来，当命令执行结束后screen会话也会自动结束，例如```screen vim memo.txt```
  * 关闭会话窗口，这样的操作在传统的远程控制中一定会导致正在运行的命令也突然终止，但在screen不间断会话服务中则不会这样。我们只需查看一下刚刚离线的会话名称，然后尝试恢复回来就可以继续工作
  * 会话共享功能
    * A用户连接到远程主机，创建一个会话窗口
    * B用户链接到远程主机，使用命令```screen -x```获取会话，即可看到A相同的内容
