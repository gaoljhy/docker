目前，Docker已经支持非常多的，CentOS、RHEL、Ubuntu、Oracle Linux、OS X、Microsoft Windows等。


#### 安装Docker的先决条件

* 运行64位CPU架构的计算机（ x86_64 和 amd64 ）
* 运行linux 3.8或更高版本内核。一些老版本的2.6.x或其后的内核也能够运行Docker，但是运行结果会有很大的不同
* 内核必须支持一种合适的存储驱动（storage driver）例如

     * Device manager
     * AUFS
     * vfs
     * btrfs
     
     默认存储驱动通常是Device Mapper
            
* 内核必须支持并开启cgroup和命名空间（namespace）功能

 

#### 检查前提条件

1.内核 

确认是否安装了3.8或更高的内核版本

```
[root@vlnx251105 ~]# uname -a
Linux vlnx251105.zyg.com 3.10.0-514.26.2.el7.x86_64 #1 SMP Tue Jul 4 15:04:05 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
```
  


2.检查 Device Mapper

这里使用 Divece Mapper作为Docker的存储驱动，为Docker提供存储能力。CentOS 6或更高版本宿主机中，应该已经安装Device Mapper


```
[root@vlnx251105 ~]# ls -l /sys/class/misc/device-mapper/

[root@vlnx251105 ~]# grep device-mapper /proc/devices 
253 device-mapper
```
  

如果没有检测到 Device Mapper，可以自行安装，并加载dm_mod内核模块


```
yum install device-mapper
modprobe dm_mod
```


#### 安装Docker

```
[root@vlnx251105 ~]# yum install docker
```
  


#### 启动Docker守护进程

```
[root@vlnx251105 ~]# systemctl enable docker

[root@vlnx251105 ~]# systemctl start docker

[root@vlnx251105 ~]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
```



