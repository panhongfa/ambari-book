# 创建Yum缓存代理服务

通常情况下，我们提供的自定义yum本地源压缩包中包含了大数据平台部署、运行和维护过程中的所有软件rpm包。除非是有特殊需要，比如开发人员试验某些新软件包的功能，或是运维人员安装自己熟悉的某些工具软件等，才需要从外部导入rpm包。因此，绝大多数场景下这个步骤都是可选的，运维人员要根据自己的实际需求来决定是否执行该步骤。

在需要导入外部rpm包的情况下，由于生产环境通常都是与外网进行隔离的，就算是可以通过其他手段获取到外网权限，但是通过修改集群中某些主机的网络配置和repo文件很显然是非常低效和十分危险的，极易造成集群主机之间的配置不一致，甚至数据泄露或丢失。

理想的解决方案应该是：

* yum安装软件时优先从集群内的本地源中查找，若存在则直接下载并安装。
* 若没有找到合适的rpm包，则将http请求转发到一台具有外网访问权限的proxy主机。
* proxy解析并执行该http请求进而从外部的标准源中获取rpm包，并缓存在本地。
* 将rpm包转发给原始请求的那台主机，yum完成软件安装。

其结构图如下：

Centos7环境下目前没有开源的工具能够满足以上的解决方案，不过Ubuntu14环境下有一个apt-cacher-ng的工具能满足要求，并且其最新版本增加了对yum所使用的rpm及repodata的支持。接下来描述的就是其实际安装和配置过程。

## proxy主机初始化

利用Usb的系统安装盘在proxy主机上安装Ubuntu 14.04.5 LTS操作系统，安装完成后首先配置主机名。

```
hostnamectl set-hostname proxy.bigdata.wh.com  #设置主机名
```

然后利用ifconfig查询Ubuntu系统识别的网卡信息，执行结果如下。

```
root@proxy:/etc/network# ifconfig 
eth0      Link encap:Ethernet  HWaddr 08:00:27:54:e0:3b         #以太网口eth1
          inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe54:e03b/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:22809 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4259 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:31386840 (31.3 MB)  TX bytes:344498 (344.4 KB)

eth1      Link encap:Ethernet  HWaddr 08:00:27:a0:59:b6          #以太网口eth1
          inet addr:192.168.36.111  Bcast:192.168.37.255  Mask:255.255.254.0
          inet6 addr: fe80::a00:27ff:fea0:59b6/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:76679 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3088 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:8766387 (8.7 MB)  TX bytes:401980 (401.9 KB)

lo        Link encap:Local Loopback                               #单机环回网卡
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:34 errors:0 dropped:0 overruns:0 frame:0
          TX packets:34 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:4547 (4.5 KB)  TX bytes:4547 (4.5 KB)
```

Ubuntu 14系统的网络主配置文件为/etc/network/interfaces，通常情况下主配置文件只配置loopback本地环回，其他的以每个网口对应于一个eth\*.cfg配置文件的方式存放在/etc/network/interfaces.d目录下，并且通过主配置文件来全部加载。

```
root@proxy:/etc/network# cat interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# Source interfaces
# Please check /etc/network/interfaces.d before changing this file
# as interfaces may have been defined in /etc/network/interfaces.d
# NOTE: the primary ethernet device is defined in
# /etc/network/interfaces.d/eth0
# See LP: #1262951
source /etc/network/interfaces.d/*.cfg            #其他的网口配置以eth0.cfg、eth1.cfg的方式存放在该目录，并在此处加载

#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
auto eth1                                         #Vagrant工具自动配置的网口，它直接在主配置文件中附加了
iface eth1 inet dhcp
    post-up route del default dev $IFACE || true
#VAGRANT-END
```

然后，根据proxy主机的物理网口连接情况和机房内所属网段的路由设置，选择使用哪个网口以及何种网络配置方式。

* 若路由器支持DHCP方式连接，则按照如下方式配置所选的以太网口。

```
root@proxy:/etc/network# cat interfaces.d/eth1.cfg 
# The primary network interface
auto eth1
iface eth1 inet dhcp
```

* 若路由器只允许以指定的静态IP方式连接，则按照如下方式来配置所选的以太网口。

```
root@proxy:/etc/network# cat interfaces.d/eth1.cfg 
# The primary network interface
auto eth1
iface eth1 inet static
address 192.168.36.100
gateway 192.168.37.254
netmask 255.255.254.0
```

完成配置后，需要利用ifdown/ifup命令来重启该以太网口。

```
vagrant@proxy:~$ sudo ifdown eth1
Internet Systems Consortium DHCP Client 4.2.4
Listening on LPF/eth1/08:00:27:a0:59:b6
Sending on   LPF/eth1/08:00:27:a0:59:b6
Sending on   Socket/fallback
DHCPRELEASE on eth1 to 192.168.30.254 port 67 (xid=0x157b26fb)

vagrant@proxy:~$ sudo ifup eth1
Internet Systems Consortium DHCP Client 4.2.4
Listening on LPF/eth1/08:00:27:a0:59:b6
Sending on   LPF/eth1/08:00:27:a0:59:b6
Sending on   Socket/fallback
DHCPDISCOVER on eth1 to 255.255.255.255 port 67 interval 3 (xid=0x89fef906)
DHCPREQUEST of 192.168.36.111 on eth1 to 255.255.255.255 port 67 (xid=0x6f9fe89)
DHCPOFFER of 192.168.36.111 from 192.168.37.254
DHCPACK of 192.168.36.111 from 192.168.37.254
bound to 192.168.36.111 -- renewal in 17083 seconds.
SIOCDELRT: No such process
```

最后，手动修改/etc/resolv.conf文件设置ISP提供的DNS服务器地址。

```
root@proxy:~# cat /etc/resolv.conf 
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 10.0.2.3
nameserver 192.168.30.1
```

## apt-cacher-ng安装及配置

Ubuntu 14操作系统默认源修改成国内源，以加快访问和下载速度。

```
cat << eof > /etc/apt/sources.list
deb http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ trusty-backports main restricted universe multiverse
eof
```

利用apt-get命令安装apt-cacher-ng和服务管理程序sysv-rc-conf。

```
apt-get install -y apt-cacher-ng sysv-rc-conf
```

启动apt-cacher-ng服务并配置跟随系统自启动。

```
service apt-cacher-ng start
sysv-rc-conf apt-cacher-ng on
```

设置防火墙ufw，开放ssh所使用的22端口和apt-cacher-ng所使用的3142端口。

```
ufw default deny
ufw enable
ufw allow ssh
ufw allow 3142
```

通过Web浏览器访问apt-cacher-ng主页。

![](/assets/apt-cacher-web.png)

apt-cacher-ng的主配置文件路径为/etc/apt-cacher-ng/acng.conf，编辑acng.conf确保以下配置项有效。

```
CacheDir: /var/cache/apt-cacher-ng     #存储已下载完毕的rpm包的缓存目录
LogDir: /var/log/apt-cacher-ng         #默认的日志文件存放路径
SupportDir: /usr/lib/apt-cacher-ng     #存放辅助文件及脚本的默认端口
Port:3142                              #默认的http访问端口
ReportPage: acng-report.html           #在默认的web生成统计报告
VerboseLog: 1                          #记录更详细的Log信息
```

另外，在acng.conf中有多列以'Remap-‘作为前缀的配置项，这每一行都表示一个资源重定向规则，其语法表示为：`Remap-RepositoryName: MergingURLs ; TargetURLs ; OptionalFlags`。不过需要注意的是`MergingURLs`默认的根路径是`$SupportDir`，而`TargetURLs`默认的根路径是/etc/apt-cacher-ng。

```
# Repository remapping. See manual for details.
# In this example, some backends files might be generated during package
# installation using information collected on the system.
# Examples:
Remap-debrep: file:deb_mirror*.gz /debian ; file:backends_debian # Debian Archives
Remap-uburep: file:ubuntu_mirrors /ubuntu ; file:backends_ubuntu # Ubuntu Archives
Remap-debvol: file:debvol_mirror*.gz /debian-volatile ; file:backends_debvol # Debian Volatile Archives
Remap-cygwin: file:cygwin_mirrors /cygwin # ; file:backends_cygwin # incomplete, please create this file or specify preferred mirrors here
Remap-sfnet:  file:sfnet_mirrors # ; file:backends_sfnet # incomplete, please create this file or specify preferred mirrors here
Remap-alxrep: file:archlx_mirrors /archlinux # ; file:backend_archlx # Arch Linux
Remap-fedora: file:fedora_mirrors # Fedora Linux
Remap-epel:   file:epel_mirrors # Fedora EPEL
Remap-slrep:  file:sl_mirrors # Scientific Linux
Remap-gentoo: file:gentoo_mirrors.gz /gentoo ; file:backends_gentoo # Gentoo Archives
```

于是我们来添加一个基于Centos7的rpm资源包的重定向规则。

```
Remap-centos: file:centos_mirrors /centos ; file:backends_centos # Centos Rpm
```

然后再创建文件centos\_mirrors和backends\_centos，表示所有来自于centos\_mirrors定义地址的http请求全部被重定向到backends\_centos定义的某个地址。

* centos\_mirrors文件



* backends\_centos文件

```
cat << eof > /etc/apt-cacher-ng/backends_centos
http://mirrors.163.com/centos/
http://mirrors.aliyun.com/centos/
http://mirrors.cn99.com/centos/
eof
```

## 集群主机代理配置

## 查看proxy缓存


