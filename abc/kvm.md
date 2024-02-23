# 在 Rocky Linux 9 / AlmaLinux 9 上安装 KVM

先决条件：

-   预装RockyLinux/AlmaLinux9
-   使用root用户或具有root权限的sudo用户
-   互联网连接

更多参考 https://zhuanlan.zhihu.com/p/632164296

```bash
# 1、验证是否启用了硬件虚拟化
# 该命令探测是否存在 VMX（虚拟机扩展(Virtual Machine Extension)），它是英特尔硬件虚拟化的 CPU 标志，或 SVM，它是 AMD 硬件虚拟化的标志。
cat /proc/cpuinfo | egrep "vmx|svm"

# 2、安装
dnf -y install qemu-kvm libvirt virt-install virt-top bridge-utils libguestfs guestfs-tools

# 3、安装完成后，运行以下命令查看是否已加载所需的KVM模块
lsmod | grep kvm

# 4、启动libvirtd并设置开启启动
systemctl enable --now libvirtd
```





# 使用virt-install安装虚拟机

## 安装Ubuntu

```bash
# 安装virt-install
dnf install virt-install

# 安装虚拟机
virt-install \
--name ubuntu2204 \
--os-variant ubuntu22.04 \
--boot cdrom,hd \
--vcpus 4,maxvcpus=6 \
--memory memory=4096,currentMemory=2048 \
--disk /var/lib/libvirt/images/ubuntu2204.img,size=32 \
--cdrom /var/lib/libvirt/images/ubuntu-22.04.3-live-server-amd64.iso \
--network default \
--graphics vnc,port=-1,listen=0.0.0.0,passwd=binbin \
--console pty,target_type=serial \
--noautoconsole

# 引用后端盘安装，ubuntu2204.img为已安装好ubuntu22.04系统的后端系统盘
virt-install \
--name ubuntu \
--boot hd \
--osinfo ubuntu22.04 \
--vcpus 4,maxvcpus=4 \
--cpu host-passthrough \
--memory memory=8192 \
--import \
--disk backing_store=/var/lib/libvirt/images/ubuntu2204.img,size=32 \
--network bridge=brvlan10 \
--graphics vnc,port=-1,listen=0.0.0.0,passwd=binbin \
--console pty,target_type=serial \
--noautoconsole
```



## 安装Win11

```bash
# Install Win11 23H2
virt-install \
--name win11 \
--os-variant win11 \
--boot cdrom,hd \
--vcpus 4,maxvcpus=6 \
--memory memory=4096,currentMemory=2048 \
--disk /var/lib/libvirt/images/Win11.img,size=64 \
--cdrom /var/lib/libvirt/images/Win11_23H2_Chinese_Simplified_x64.iso \
--network bridge=brvlan10 \
--graphics vnc,port=-1,listen=0.0.0.0,passwd=binbin \
--console pty,target_type=serial \
--noautoconsole

# 引用后端盘方式启用新虚拟机
virt-install \
--name Win11 \
--os-variant win11 \
--boot hd \
--vcpus 4,maxvcpus=6 \
--memory memory=4096 \
--disk backing_store=/var/lib/libvirt/images/Win11-Base.img,size=64 \
--network bridge=brvlan20,model=virtio \
--graphics vnc,port=-1,listen=0.0.0.0,passwd=binbin \
--console pty,target_type=serial \
--noautoconsole
```



## 安装ESXi（嵌套虚拟化实验）

```bash
# Install ESXi7
# 网卡类型只支持e1000e
# 磁盘总线类型只支持sata
virt-install \
--name esxi \
--boot cdrom,hd \
--osinfo linux2022 \
--vcpus 4,maxvcpus=6 \
--cpu host-passthrough \
--memory memory=32768,currentMemory=32768 \
--disk size=256,bus=sata \
--network bridge=brvlan10,model=e1000e \
--graphics vnc,port=-1,listen=0.0.0.0,passwd=binbin \
--console pty,target_type=serial \
--noautoconsole \
--cdrom /var/lib/libvirt/images/VMware-VMvisor-Installer-7.0U3l-21424296.x86_64.iso
```





# 虚拟机配置VNC连接

1、编辑虚拟机配置文件，在设备下添加以下内容，启动虚拟机后可通过宿主机IP和此处配置的port端口号连接到虚拟机：

```bash
<graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0' keymap='en-us' passwd='binbin'>
  <listen type='address' address='0.0.0.0'/>
</graphics>
```





# 虚拟机配置Console连接

1、编辑虚拟机配置文件，在设备下面添加以下内容：

```bash
<serial type='pty'>
    <target port='0'/>
</serial>
<console type='pty'>
    <target type='serial' port='0'/>
</console>
```



2、添加虚拟机内核启动参数（root用户执行），需要重启：

>   Centos/RockLinux

```bash
grubby --update-kernel=ALL --args="console=ttyS0,115200n8"
reboot
```

>   Ubuntu2204

```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub
sudo reboot
```

>   AlpineLinux(磁盘设备为/dev/vda)

```bash
# 卸载syslinux（可选）
apk del syslinux
# 安装grub软件包
apk add grub grub-bios
# 安装grub引导程序到磁盘
grub-install /dev/vda
# 开启系统允许从ttyS0登录
sed -i '/ttyS0/s/^#//' /etc/inittab
# 修改配置文件添加内核启动参数
echo 'GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0,115200n8"' >> /etc/default/grub
# 重新生成grub引导配置
grub-mkconfig -o /boot/grub/grub.cfg
# 重启系统
reboot
```



3、在宿主机中通过console连接虚拟机，在虚拟中通过按^] (Ctrl + ])退出虚拟机，回到宿主机命令行：

```bash
[root@rockylinux ~]# virsh start centos7
Domain 'centos7' started

[root@rockylinux ~]# virsh list
 Id   Name      State
-------------------------
 1    centos7   running

[root@rockylinux ~]# virsh console centos7
Connected to domain 'centos7'
Escape character is ^] (Ctrl + ])

CentOS Linux 7 (Core)
Kernel 3.10.0-1160.el7.x86_64 on an x86_64

localhost login: root
Password:
Last login: Tue Oct 31 16:24:47 on ttyS0
[root@localhost ~]#
```



# 使用virt-top查看虚拟机信息

```bash
# 安装virt-top
dnf install virt-top

# 运行 virt-top --help查看命令帮助
virt-top
virt-top time  11:14:12 Host rockylinux x86_64 64/64CPU 2400MHz 128346MB
   ID S RDRQ WRRQ RXBY TXBY %CPU %MEM   TIME    NAME
    1 R    0    0    0    0  0.0  0.0   0:22.34 centos7
    5 R    0    0    0    0  0.0  0.0   0:31.01 ubuntu2204
   10 R    0    0    0    0  0.0  0.0   1:23.29 alpine
```



# 磁盘管理

创建磁盘

```bash
# 创建新的独立磁盘
qemu-img create -f qcow2 /var/lib/libvirt/image/test.img 10G

# 创建引用后端盘的磁盘
# /path/to/backend/image_base.img为后端盘路径
qemu-img create -f qcow2 -b /path/to/backend/image_base.img /var/lib/libvirt/image/test.img 10G
```



查看磁盘信息

```bash
qemu-img info /var/lib/libvirt/images/test.img
```



更改磁盘大小

```bash
qemu-img resize /var/lib/libvirt/images/test.img +100M
qemu-img resize --shrink /var/lib/libvirt/images/test.img -100M
```



删除磁盘

```bash
# 磁盘表现为系统中的一个普通文件，直接删除即可
rm -rf /var/lib/libvirt/images/test.img
```



虚拟机添加磁盘

```bash
# --live 实时生效
# --config 更新虚拟机xml配置文件
# --subdriver qcow2 指定磁盘镜像文件格式
virsh attach-disk win11 /var/lib/libvirt/images/test.img vdb --subdriver qcow2 --live --config
```



虚拟机分离磁盘

```bash
# --live 实时生效
# --config 更新虚拟机xml配置文件
virsh detach-disk win11 vdb --live --config
```





# 光盘管理

向虚拟机附加iso文件

```bash
#
virsh attach-disk win11 /var/lib/libvirt/images/virtio-win-0.1.229.iso sdb --type cdrom
```



更换虚拟机光盘文件

```bash
# 方式一
virsh attach-disk win11 /var/lib/libvirt/images/virtio-win-0.1.229.iso sdb --type cdrom

# 方式二
virsh change-media win11 sdb /var/lib/libvirt/images/Win11.iso
```



弹出虚拟机光盘文件

```bash
virsh change-media win11 sdb --eject
```





# 快照管理

生成快照

```bash
virsh snapshot-create-as ubuntu --name ubuntu-init --description "system ok"
```



查看虚拟机下快照

```bash
virsh snapshot-list ubuntu
 Name          Creation Time               State
----------------------------------------------------
 ubuntu-init   2023-10-31 14:20:23 +0800   shutoff
```



快照提交

```bash
# 将快照内容合并提交至后端盘
virsh 
```



# nmcli操作vlan接口

```bash
# 创建vlan接口
nmcli c add type vlan con-name ifvlan10 ifname ifvlan10 dev eno12399 id 10
nmcli c add type vlan con-name ifvlan20 ifname ifvlan20 dev eno12399 id 20
nmcli c add type vlan con-name ifvlan30 ifname ifvlan30 dev eno12399 id 30

# 给vlan接口配置IP
ip addr add 172.16.10.238/23 dev ifvlan10
ip addr add 172.16.20.238/23 dev ifvlan20
ip addr add 172.16.30.238/23 dev ifvlan20

# 启用接口
ip link set ifvlan10 up
ip link set ifvlan20 up
ip link set ifvlan30 up

# 停用接口
ip link set ifvlan10 down
ip link set ifvlan20 down
ip link set ifvlan30 down

# 删除vlan接口
nmcli c del ifvlan10
nmcli c del ifvlan20
nmcli c del ifvlan30

```



# brctl操作网桥

```bash
# 创建网桥
brctl addbr brvlan10
brctl addbr brvlan20
brctl addbr brvlan30

# 将接口加入网桥
brctl addif brvlan10 ifvlan10
brctl addif brvlan20 ifvlan20
brctl addif brvlan30 ifvlan30

# 启用网桥
ip link set brvlan10 up
ip link set brvlan20 up
ip link set brvlan30 up

# 停用网桥
ip link set brvlan10 down
ip link set brvlan20 down
ip link set brvlan30 down

```





# 启用嵌套虚拟化

查看是否启用了嵌套虚拟化

```bash
# Intel处理器
[root@rockylinux ~]# cat /sys/module/kvm_intel/parameters/nested
N

# N表示未启用，Y表示已启用
```



启用

```bash
# 编译配置文件，添加或注释已存在的配置项
vim /etc/modprobe.d/kvm.conf
# For Intel
options kvm_intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1

# 重新加载kvm_intel模块，删除kvm_intel模块前要确保所有虚拟机已关机，否则会有报错：“modprobe: FATAL: Module kvm_intel is in use”
modprobe -r kvm_intel
modprobe -a kvm_intel
```

