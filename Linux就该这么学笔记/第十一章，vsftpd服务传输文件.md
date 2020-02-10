##### 1. 文件传输协议
* FTP是一种在互联网中进行文件传输的协议，基于客户端/服务器模式，默认使用20、21号端口，其中端口20（数据端口）用于进行数据传输，端口21（命令端口）用于接受客户端发出的相关FTP命令与参数。FTP服务器普遍部署于内网中，具有容易搭建、方便管理的特点。而且有些FTP客户端工具还可以支持文件的多点下载以及断点续传技术，因此FTP服务得到了广大用户的青睐
![FTP协议传输拓扑](https://www.linuxprobe.com/wp-content/uploads/2015/07/FTP%E8%BF%9E%E6%8E%A5%E8%BF%87%E7%A8%8B.png)
* FTP服务器是按照FTP协议在互联网上提供文件存储和访问服务的主机，FTP客户端则是向服务器发送连接请求，以建立数据传输链路的主机。FTP协议有下面两种工作模式
  * 主动模式：FTP服务器主动向客户端发起连接请求
  * 被动模式：FTP服务器等待客户端发起连接请求（FTP的默认工作模式）
* vsftpd（very secure ftp daemon，非常安全的FTP守护进程）是一款运行在Linux操作系统上的FTP服务程序，不仅完全开源而且免费，此外，还具有很高的安全性、传输速度，以及支持虚拟用户验证等其他FTP服务程序不具备的特点
* 安装vsftpd服务程序```yum isntall vsftpd```
* vsftpd服务程序主配置文件常用参数及其作用
  
  参数|作用
  -|-
  listen=[YES\|NO]|是否以独立运行的方式监听服务
  listen_address=IP地址|设置要监听的IP地址
  listen_port=21|设置FTP服务的监听端口
  download_enable=[YES\|NO]|是否允许下载文件
  userlist_enable=[YES\|NO]<br>userlist_deny=[YES\|NO]|设置用户列表为“允许”还是“禁止”操作
  max_clients=0|最大客户端连接数，0为不限制
  max_per_ip=0|同一IP地址的最大连接数，0为不限制
  anonymous_enable=[YES\|NO]|是否允许匿名用户访问
  anon_upload_enable=[YES\|NO]|是否允许匿名用户上传文件
  anon_umask=022|匿名用户上传文件的umask值
  anon_root=/var/ftp|匿名用户的FTP根目录
  anon_mkdir_write_enable=[YES\|NO]|是否允许匿名用户创建目录
  anon_other_write_enable=[YES\|NO]|是否开放匿名用户的其他写入权限（包括重命名、删除等操作权限）
  anon_max_rate=0|匿名用户的最大传输速率（字节/秒），0为不限制
  local_enable=[YES\|NO]|是否允许本地用户登录FTP
  local_umask=022|本地用户上传文件的umask值
  local_root=/var/ftp|	本地用户的FTP根目录
  chroot_local_user=[YES\|NO]|是否将用户权限禁锢在FTP目录，以确保安全
  local_max_rate=0|本地用户最大传输速率（字节/秒），0为不限制

##### 2. vsftpd服务程序
* vsftpd允许用户以三种认证模式登录到FTP服务器上
  * 匿名开放模式：是一种最不安全的认证模式，任何人都可以无需密码验证而直接登录到FTP服务器
  * 本地用户模式：是通过Linux系统本地的账户密码信息进行认证的模式，相较于匿名开放模式更安全，而且配置起来也很简单。但是如果被黑客破解了账户的信息，就可以畅通无阻地登录FTP服务器，从而完全控制整台服务器
  * 虚拟用户模式：是这三种模式中最安全的一种认证模式，它需要为FTP服务单独建立用户数据库文件，虚拟出用来进行口令验证的账户信息，而这些账户信息在服务器系统中实际上是不存在的，仅供FTP服务程序进行认证使用
* 使用FTP协议传输文件之前必须开启SELinux域中对FTP服务的允许策略
* 匿名开放模式
  * vsftpd服务程序默认开启了匿名开放模式，我们需要做的就是开放匿名用户的上传、下载文件的权限，以及让匿名用户创建、删除、更名文件的权限
  * 向匿名用户开放权限的参数以及作用

    参数|作用
    -|-
    anonymous_enable=YES|允许匿名访问模式
    anon_umask=022|匿名用户上传文件的umask值
    anon_upload_enable=YES|允许匿名用户上传文件
    anon_mkdir_write_enable=YES|允许匿名用户创建目录
    anon_other_write_enable=YES|允许匿名用户修改目录名称或删除目录
* 本地用户模式
 * vsftpd服务程序所在的目录中默认存放着两个名为“用户名单”的文件(ftpusers和user_list)，里面写有某位用户的名字，就不再允许这位用户登录到FTP服务器上

  参数|作用
  -|-
  anonymous_enable=NO|禁止匿名访问模式
  local_enable=YES|允许本地用户模式
  write_enable=YES|设置可写权限
  local_umask=022|本地用户模式创建文件的umask值
  userlist_deny=YES|启用“禁止用户名单”，名单文件为ftpusers和user_list
  userlist_enable=YES|开启用户作用名单文件功能
* 虚拟用户模式
  1. 在/etc/vsftpd目录创建用于进行FTP认证的用户数据库文件，其中奇数行为账户名，偶数行为密码
     ```
     vim /etc/vsftpd/vuser.list # 建立用户名单文件
     db_load -T -t hash -f /etc/vsftpd/vuser.list vuser.db # 明文信息 既不安全，也不符合让vsftpd服务程序直接加载的格式，因此需要使用db_load 命令用哈希（hash）算法将原始的明文信息文件转换成数据库文件
     chmod 600 vuser.db # 降低数据库文件的权限（避免其他人看到数据库文件 的内容）
     rm -f vuser.list # 把原始的明文信息文件删除
     ```
  2. 创建vsftpd服务程序用于存储文件的根目录以及虚拟用户映射的系统本地用户。FTP服务用于存储文件的根目录指的是当虚拟用户登录后所访问的默认位置
     * 由于Linux系统中的每一个文件都有所有者、所属组属性，例如使用虚拟账户“张三”新建了一个文件，但是系统中找不到账户“张三”，就会导致这个文件的权限出现错误。为此，需要再创建一个可以映射到虚拟用户的系统本地用户。简单来说，就是让虚拟用户默认登录到与之有映射关系的这个系统本地用户的家目录中，虚拟用户创建的文件的属性也都归属于这个系统本地用户，从而避免Linux系统无法处理虚拟用户所创建文件的属性权限
     * 为了方便管理FTP服务器上的数据，可以把这个系统本地用户的家目录设置为/var目录（该目录用来存放经常发生改变的数据）。并且为了安全起见，我们将这个系统本地用户设置为不允许登录FTP服务器，这不会影响虚拟用户登录，而且还可以避免黑客通过这个系统本地用户进行登录
     ```
     useradd -d /var/ftproot -s /sbin/nologin virtual
     chmod -Rf 755 /var/ftproot
     ```
  3. 建立用于支持虚拟用户的PAM文件
     * PAM（可插拔认证模块）是一种认证机制，通过一些动态链接库和统一的API把系统提供的服务与认证方式分开，使得系统管理员可以根据需求灵活调整服务程序的不同认证方式
     ![PAM结构](https://www.linuxprobe.com/wp-content/uploads/2015/07/PAM%E8%AE%A4%E8%AF%81%E6%9C%BA%E5%88%B6%E7%9A%84%E4%BD%93%E7%B3%BB%E5%9B%BE.jpg)
     * 新建一个用于虚拟用户认证的PAM文件vsftpd.vu，其中PAM文件内的“db=”参数为使用db_load命令生成的账户密码数据库文件的路径，但不用写数据库文件的后缀
       ```
       vim /etc/pam.d/vsftpd.vu
       auth required pam_userdb.so db=/etc/vsftpdvuser
       account required pam_userdb.so db=/etc/vsftpdvuser
       ```
  4. 在vsftpd服务程序的主配置文件中通过pam_service_name参数将PAM认证文件的名称修改为vsftpd.vu，PAM作为应用程序层与鉴别模块层的连接纽带，可以让应用程序根据需求灵活地在自身插入所需的鉴别功能模块。当应用程序需要PAM认证时，则需要在应用程序中定义负责认证的PAM配置文件，实现所需的认证功能。
     * 例如，在vsftpd服务程序的主配置文件中默认就带有参数pam_service_name=vsftpd，表示登录FTP服务器时是根据/etc/pam.d/vsftpd文件进行安全认证的。现在我们要做的就是把vsftpd主配置文件中原有的PAM认证文件vsftpd修改为新建的vsftpd.vu文件即可。该操作中用到的参数以及作用如下表
  
       参数|作用
       -|-
       anonymous_enable=NO|禁止匿名开放模式
       local_enable=YES|允许本地用户模式
       guest_enable=YES|开启虚拟用户模式
       guest_username=virtual|指定虚拟用户账户
       pam_service_name=vsftpd.vu|指定PAM文件
       allow_writeable_chroot=YES|允许对禁锢的FTP根目录执行写入操作，而且不拒绝用户的登录请求

  5. 为虚拟用户设置不同的权限。虽然账户zhangsan和lisi都是用于vsftpd服务程序认证的虚拟账户，但是我们依然想对这两人进行区别对待
     * 这可以通过vsftpd服务程序来实现。只需新建一个目录，在里面分别创建两个以zhangsan和lisi命名的文件，其中在名为zhangsan和lisi的文件中写入允许的相关权限（使用匿名用户的参数）
       ```
       mkdir /etc/vsftpd/vusers_dir/
       cd /etc/vsftpd/vusers_dir/
       touch lisi
       vim zhangsan
       anon_upload_enable=YES
       anon_mkdir_write_enable=YES
       anon_other_write_enable=YES
       ```
     * 然后再次修改vsftpd主配置文件，通过添加user_config_dir参数来定义这两个虚拟用户不同权限的配置文件所存放的路径。为了让修改后的参数立即生效，需要重启vsftpd服务程序并将该服务添加到开机启动项中
       ```
       vim /etc/vsftpd/vsftpd.conf
       user_config_dir=/etc/vsftpd/vusers_dir
       ```

##### 3. 简单文件传输协议
* 简单文件传输协议（Trivial File Transfer Protocol，TFTP）是一种基于UDP协议在客户端和服务器之间进行简单文件传输的协议。顾名思义，它提供不复杂、开销不大的文件传输服务（可将其当作FTP协议的简化版本）
* TFTP的命令功能不如FTP服务强大，甚至不能遍历目录，在安全性方面也弱于FTP服务。而且，由于TFTP在传输文件时采用的是UDP协议，占用的端口号为69，因此文件的传输过程也不像FTP协议那样可靠。但是，因为TFTP不需要客户端的权限认证，也就减少了无谓的系统和网络带宽消耗，因此在传输琐碎（trivial）不大的文件时，效率更高
* 配置步骤
  1. 在服务器上安装服务器端TFTP软件包```yum install tftp-server```
  2. 在客户端安装TFTP客户端```yum install tftp```
  3. 在RHEL7系统中，TFTP服务是使用xinetd服务程序来管理的。在安装TFTP软件包后，还需要在xinetd服务程序中将其开启，把默认的禁用（disable）参数修改为no
  4. 重启xinetd服务并将它添加到系统的开机启动项中，以确保TFTP服务在系统重启后依然处于运行状态。考虑到有些系统的防火墙默认没有允许UDP协议的69端口，因此需要手动将该端口号加入到防火墙的允许策略中
  5. TFTP的根目录为/var/lib/tftpboot。使用tftp命令访问其中的文件。在使用tftp命令访问文件时，可能会用如下的参数
   
     命令|作用
     -|-
     ?|帮助信息
     put|上传文件
     get|下载文件
     verbose|显示详细的处理信息
     status|显示当前的状态信息
     binary|使用二进制进行传输
     ascii|使用ASCII码进行传输
     timeout|设置重传的超时时间
     quit|退出