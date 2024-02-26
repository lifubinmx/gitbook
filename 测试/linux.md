# linux

## 在SSH远程会话中使用本地变量



## 终端打字机效果

> 在终端模拟打字机效果输出（逐字符显示）

```bash
# 安装pv
brew install pv

# 执行命令查看动态输出效果
echo 'Hello World' | pv -qL 10
cat /etc/hosts | pv -qL 10
```

## Iscsi Initiator使用

> 系统：AlmaLinux9/RockyLinux9/RHEL9

安装软件

```bash
# 安装软件包，device-mapper-multipath为可选包，服务器到存储有多条路径情况下使用。
dnf install iscsi-initiator-utils device-mapper-multipath
```

设置InitiatorName

```bash
# 生成InitiatorName配置文件
cat > /etc/iscsi/initiatorname.iscsi <<EOF
InitiatorName=iqn.1994-05.com.redhat:almalinux2091
EOF
```

配置iscsid认证

> 此为可选项，当iscsi存储服务器设置了CHAP认证时才需要配置

```bash
# 编辑/etc/iscsi/iscsid.conf，修改以下内容
node.session.auth.authmethod = CHAP
node.session.auth.username = user
node.session.auth.password = password
```

启用iscsid服务

```bash
systemctl enable --now iscsid
```

配置MulitpathIO（可选）

```bash
mpathconf --enable --with_multipathd y

cat >> /etc/multipath.conf <<EOF
devices {
        device {
                vendor                 "LIO-ORG"
                hardware_handler       "1 alua"
                path_grouping_policy   "failover"
                path_selector          "queue-length 0"
                failback               60
                path_checker           tur
                prio                   alua
                prio_args              exclusive_pref_bit
                fast_io_fail_tmo       25
                no_path_retry          queue
        }
}
EOF

systemctl reload multipathd
```

发现目标

```bash
[root@AlmaLinux ~]# iscsiadm -m discovery -t st -p 172.16.10.252
172.16.10.252:3260,1 iqn.2000-01.com.synology:NAS02.default-target.51f4b1891d9
```

连接目标

```bash
[root@AlmaLinux ~]# iscsiadm -m node -T iqn.2000-01.com.synology:NAS02.default-target.51f4b1891d9 -l
Logging in to [iface: default, target: iqn.2000-01.com.synology:NAS02.default-target.51f4b1891d9, portal: 172.16.10.252,3260]
Login to [iface: default, target: iqn.2000-01.com.synology:NAS02.default-target.51f4b1891d9, portal: 172.16.10.252,3260] successful.
[root@AlmaLinux ~]# lsblk
NAME               MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                  8:0    0   32G  0 disk
├─sda1               8:1    0  200M  0 part /boot
└─sda2               8:2    0 31.8G  0 part
  └─almalinux-root 253:0    0 31.8G  0 lvm  /
sdb                  8:16   0    1G  0 disk
```

断开目标

```bash
[root@AlmaLinux ~]# iscsiadm -m node -T iqn.2000-01.com.synology:NAS02.default-target.51f4b1891d9 -u
Logging out of session [sid: 1, target: iqn.2000-01.com.synology:NAS02.default-target.51f4b1891d9, portal: 172.16.10.252,3260]
Logout of [sid: 1, target: iqn.2000-01.com.synology:NAS02.default-target.51f4b1891d9, portal: 172.16.10.252,3260] successful.
```

## nmcli

参考 https://docs.rockylinux.org/zh/gemstones/RL9\_network\_manager/

### 配置网络接口静态信息

brvlan10 网络接口名称

autoconnect 开启随系统启动

ipv4.address 静态IP

ipv4.gateway 网关

ipv4.dns DNS

```bash
nmcli connect modify brvlan10 \
ipv4.method manual \
autoconnect yes \
ipv4.address 172.16.10.1/24 \
ipv4.gateway 172.16.10.254 \
ipv4.dns 10.11.65.202
```

### 配置网络接口使用DHCP

enp0s1 网络接口名称

```bash
nmcli c m enp0s1 ipv4.method dhcp
```

### 配置网络接口禁用IP地址

```basah
nmcli c m enp0s1 ipv4.method disabled
```

### 创建网桥

```bash
# 创建网桥
nmcli c add type bridge con-name br0 ifname br0

# 将端口加入网桥
nmcli c add type bridge-slave con-name eno12399 ifname eno12399 master br0
```

### Bond

```bash
# 创建bond接口
nmcli c add type bond con-name bond6 ifname bond6 mode 6

# 将接口加入bond
nmcli c add type bond-slave ifname eno12419 master bond6
nmcli c add type bond-slave ifname eno12429 master bond6
```

### Tema

```bash
# 创建team口
nmcli c add type team con-name team0 ifname team0 config '{"runner": {"name": "loadbalance"}}'

# 将端口加入team口
nmcli c add type team-slave con-name eno12399 ifname eno12399 master team0
nmcli c add type team-slave con-name eno12409 ifname eno12409 master team0
nmcli c add type team-slave con-name eno12419 ifname eno12419 master team0
nmcli c add type team-slave con-name eno12429 ifname eno12429 master team0

# 查看team口状态
teamdctl team0 state
```

## machine-id

machine-id是一个由systemd生成和管理的32位十六进制字符串，用于在系统中唯一标识计算机。

当系统启动时，systemd会生成一个机器ID并存储在/etc/machine-id文件中。如果该文件不存在或内容无效，系统会生成新的机器ID并写入该文件中。

要修改系统的machine-id，需要清除旧的机器ID并生成新的机器ID。具体步骤如下：

清除旧的机器ID，可以使用以下命令：

```bash
rm /etc/machine-id
```

生成新的机器ID。在清除旧的机器ID后，可以使用以下命令生成新的机器ID：

```bash
systemd-machine-id-setup
```

执行该命令后，系统会生成一个新的机器ID并自动写入/etc/machine-id文件中。

默认ubuntu系统使用的netplan工具配置网络通过DHCP获取IP地址时会使用系统的machine-id做DCHP客户端标识符发送给DHCP服务器，如果需要修改成使用mac地址做为客户端标识符获取IP地址，需要在配置文件中增加 `dhcp-identifier: mac`来明确指定。

## 磁盘扩容

磁盘空间扩容用于虚拟化环境中对虚拟机磁盘的容量调整，需要注意的是只能对与空闲空间相邻的分区进行扩容，基本上是磁盘的最后一个分区

不能直接对逻辑分区进行扩容，如果要对逻辑分区进行扩容，需要首先对逻辑分区所在的扩展分区进行扩容，之后再对相应的逻辑分区进行扩容

对于使用LVM的环境，可以使用此方法先对PV容量进行扩容，即会增加PV所在VG的空闲容量，之后再使用lvextend和resize2fs对相应LV进行扩容

### CentOS

```bash
# 安装工具包
yum install e2fsprogs cloud-utils-growpart

# 目前磁盘容量及分区大小
[root@AlmaLinux ~]# fdisk -l /dev/sda
Disk /dev/sda: 38 GiB, 40802189312 bytes, 79691776 sectors
Disk model: QEMU HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xce54c2a1

Device     Boot    Start      End  Sectors  Size Id Type
/dev/sda1  *        2048   411647   409600  200M 83 Linux
/dev/sda2         411648 67108863 66697216 31.8G 8e Linux LVM
/dev/sda3       67108864 79691742 12582879    6G  5 Extended
/dev/sda5       67110912 79691742 12580831    6G 83 Linux

# 增加虚拟机磁盘容量2G后
[root@AlmaLinux ~]# fdisk -l /dev/sda
Disk /dev/sda: 40 GiB, 42949672960 bytes, 83886080 sectors
Disk model: QEMU HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xce54c2a1

Device     Boot    Start      End  Sectors  Size Id Type
/dev/sda1  *        2048   411647   409600  200M 83 Linux
/dev/sda2         411648 67108863 66697216 31.8G 8e Linux LVM
/dev/sda3       67108864 79691742 12582879    6G  5 Extended
/dev/sda5       67110912 79691742 12580831    6G 83 Linux

# 首先对扩展分区进行扩容
[root@AlmaLinux ~]# growpart /dev/sda 3 # 此处是指/dev/sda的第3个分区
CHANGED: partition=3 start=67108864 old: size=12582879 end=79691742 new: size=16777183 end=83886046

# 对扩展分区中的逻辑分区进行扩容
[root@AlmaLinux ~]# growpart /dev/sda 5 # 此处是指/dev/sda的第5个分区
CHANGED: partition=5 start=67110912 old: size=12580831 end=79691742 new: size=16775135 end=83886046

# resize分区大小
[root@AlmaLinux ~]# resize2fs /dev/sda5 # 此处是指/dev/sda5设备
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/sda5 is mounted on /mnt; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/sda5 is now 2096891 (4k) blocks long.

# 扩容后磁盘及分区大小
[root@AlmaLinux ~]# fdisk -l /dev/sda
Disk /dev/sda: 40 GiB, 42949672960 bytes, 83886080 sectors
Disk model: QEMU HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xce54c2a1

Device     Boot    Start      End  Sectors  Size Id Type
/dev/sda1  *        2048   411647   409600  200M 83 Linux
/dev/sda2         411648 67108863 66697216 31.8G 8e Linux LVM
/dev/sda3       67108864 83886046 16777183    8G  5 Extended
/dev/sda5       67110912 83886046 16775135    8G 83 Linux
```

### Ubuntu

安装工具包的命令为 `apt install cloud-guest-utils e2fsprogs`，其余操作与CentOS系统相同。

### Alpine

安装工具包的命令为 `apk add e2fsprogs-extra cloud-utils-growpart`，其余操作与CentOS系统相同。
