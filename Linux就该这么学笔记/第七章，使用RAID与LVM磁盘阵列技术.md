##### 1. RAID(Redundant Arrays of Independent Disks独立冗余磁盘阵列)
* RAID磁盘阵列方案现在有十几种，最常用的有RAID0,RAID1,RAID5,RAID10等
  1. RAID0:把多块物理硬盘设备（至少两块）通过硬件或软件的方式串联在一起，组成一个大的卷组，并将数据依次写入到各个物理硬盘中。
  * 在最理想的状态下，硬盘设备的读写性能会提升数倍，能够有效地提升硬盘数据的吞吐速度
  * 若任意一块硬盘发生故障将导致整个系统的数据都受到破坏。不具备数据备份和错误修复能力
  <div align=center>
	  <img src="https://www.linuxprobe.com/wp-content/uploads/2016/06/RAID0.png">
  </div>

  2. RAID1:把两块以上的硬盘设备进行绑定，在写入数据时，是将数据同时写入到多块硬盘设备上（可以将其视为数据的镜像或备份）。当其中某一块硬盘发生故障后，一般会立即自动以热交换的方式来恢复数据的正常使用  
  * RAID 1技术虽然十分注重数据的安全性，但是因为是在多块硬盘设备中写入了相同的数据，因此硬盘设备的利用率得以下降
  * 由于需要把数据同时写入到两块以上的硬盘设备，这也在一定程度上增大了系统计算功能的负载
  <div align=center>
      <img src="https://www.linuxprobe.com/wp-content/uploads/2016/06/raid1.jpg">
  </div>

  3. RAID5:把硬盘设备的数据奇偶校验信息保存到其他硬盘设备中
  * RAID5磁盘阵列组中数据的奇偶校验信息并不是单独保存到某一块硬盘设备中，而是存储到除自身以外的其他每一块硬盘设备上，这样的好处是其中任何一设备损坏后不至于出现致命缺陷
  * RAID5技术实际上没有备份硬盘中的真实数据信息，而是当硬盘设备出现问题后通过奇偶校验信息来尝试重建损坏的数据。RAID5这样的技术特性“妥协”地兼顾了硬盘设备的读写速度、数据安全性与存储成本问题。
  <div align=center>
	  <img src="https://www.linuxprobe.com/wp-content/uploads/2015/02/raid5.gif">
  </div>

  4. RAID10:需要至少4块硬盘来组建，其中先分别两两制作成RAID1磁盘阵列，以保证数据的安全性；然后再对两个RAID1磁盘阵列实施RAID0技术，进一步提高硬盘设备的读写速度
  * 从理论上来讲，只要坏的不是同一组中的所有硬盘，那么最多可以损坏50%的硬盘设备而不丢失数据
  * 由于RAID 10技术继承了RAID 0的高读写速度和RAID 1的数据安全性，在不考虑成本的情况下RAID 10的性能都超过了RAID 5，因此当前成为广泛使用的一种存储技术
  <div align=center>
	  <img src="https://www.linuxprobe.com/wp-content/uploads/2016/06/raid-10-1024x508.png">
  </div>

* mdadm [模式] <RAID设备名称> [选项] [成员设备名称]：部署磁盘阵列
  
  参数|作用
  -|-
  -a|检测设备名称，添加磁盘。yes\|no是否自动为其创建设备文件
  -n|指定设备数量
  -l|指定RAID级别
  -C|创建阵列
  -v|显示过程
  -f|模拟设备破坏
  -r|移除物理设备
  -Q|查看摘要信息
  -D|查看详细信息
  -S|停止RAID磁盘阵列

  例子：部署四块硬盘的RAID10阵列,命名为/dev/md0
  > mdadm -Cv /dev/md0 -a yes -n 4 -l 10 /dev/sdb /dev/sdc /dev/sdd /dev/sde
* 损坏磁盘阵列及修复  
  * 移除故障的设备
    > mdadm /dev/md0 -f /dev/sdb
  * 添加新设备
    > umount /RAID
    > mdadm /dev/md0 -a /dev/sdb
* 磁盘阵列+备份盘：该技术的核心理念就是准备一块足够大的硬盘，这块硬盘平时处于闲置状态，一旦RAID磁盘阵列中有硬盘出现故障后则会马上自动顶替上去
  
##### 2. LVM(Logical Volume Manager逻辑卷管理器)
* LVM可以允许用户对硬盘资源进行动态调整  
  逻辑卷管理器是Linux系统用于对硬盘分区进行管理的一种机制，理论性较强，其创建初衷是为了解决硬盘设备在创建分区后不易修改分区大小的缺陷
* LVM技术是在硬盘分区和文件系统之间添加了一个逻辑层，它提供了一个抽象的卷组，可以把多块硬盘进行卷组合并。这样一来，用户不必关心物理硬盘设备的底层架构和布局，就可以实现对硬盘分区的动态调整
<div align=center>
    <img src="https://www.linuxprobe.com/wp-content/uploads/2015/02/%E9%80%BB%E8%BE%91%E5%8D%B7.png">
</div>

* 逻辑卷管理器结构：
  * PV:物理卷(Physical Volume)处于LVM中的最底层，可以将其理解为物理硬盘、硬盘分区或者RAID磁盘阵列
  * VG:卷组(Volume Group)建立在物理卷之上，一个卷组可以包含多个物理卷，而且在卷组创建之后也可以继续向其中添加新的物理卷
  * LV:逻辑卷(Logical Volume)是用卷组中空闲的资源建立的，并且逻辑卷在建立后可以动态地扩展或缩小空间。逻辑卷的大小是每个基本物理区域(Physical Extent)的倍数
  * PE:物理区域(Physical Extent)是物理卷中可用于分配的最小存储单元，物理区域大小在建立卷组时指定，一旦确定不能更改，同一卷组所有物理卷的物理区域大小需一致，新的PV加入到VG后，PE的大小自动更改为VG中定义的PE大小
* 部署逻辑卷

  功能/命令|物理卷管理|卷组管理|逻辑卷管理
  -|-|-|-
  扫描|pvscan|vgscan|lvscan
  建立|pvcreate|vgcreate|lvscan
  显示|pvdisplay|vgdisplay|lvdisplay
  删除|pvremove|vgremove|lvremove
  扩展|-|vgextend|lvextend
  缩小|-|vgreduce|lvreduce

* resize2fs [参数] 文件系统设备 指定容量：用来增大或者收缩未加载的“ext2/ext3/ext4”文件系统的大小
* 部署步骤：
  1. 让新添加的硬盘设备支持LVM技术，用pvcreate创建物理卷
  2. 用vgcreate把硬盘设备加入到指定名称的卷组中，然后vgdiaplay查看卷组状态
  3. lvcreate创建出一个指定大小的逻辑卷设备  
     * -L：以容量为单位创建
     * -l：以基本单元PE的个数为单位创建，默认每个PE大小为4MB
  4. 生成好的逻辑卷进行格式化并挂载使用
    Linux系统会把LVM中的逻辑卷设备存放在/dev设备目录中（实际上是做了一个符号链接），同时会以卷组的名称来建立一个目录，其中保存了逻辑卷的设备映射文件（即/dev/卷组名称/逻辑卷名称）
  5. 查看挂载状态并写入/etc/fstab配置文件使其永久生效
* 扩容逻辑卷：扩容前先卸载设备和挂载点
  1. 使用lvextend扩容指定逻辑卷容量
  2. fsck命令检查硬盘完整性，并用resize2fs给文件系统扩容
  3. 重新挂载硬盘设备并查看挂载状态
* 缩小逻辑卷：缩容前先卸载设备和挂载点。缩容有丢失数据风险
  1. fsck命令检查文件系统完整性
  2. 先用resize2fs调整文件系统到指定容量，再用lvreduce缩减指定逻辑卷容量
  3. 重新挂载硬盘设备并查看挂载状态
* 删除逻辑卷：依次删除逻辑卷、卷组、物理卷设备
  1. 取消逻辑卷与目录的挂载关联，删除配置文件中永久生效的设备参数
  2. lvremove删除逻辑卷设备，需要输入y来确认操作
  3. vgremove删除卷组，此处只写卷组名称即可，可不写设备的绝对路径
  4. pvremove删除物理卷设备