# Docker安全
## 1 Docker安全性

Docker的核心支撑是容器技术，借助namespace和Cgroup，Docker可以实现资源的隔离与限制；借助Docker,用户可以创建多个容器，每个容器之间相互独立，从而达到轻量级的虚拟化。
现在讨论的Docker安全性话题颇多，主要集中在以下几方面：
*  容器的隔离性(…)
*  攻击防护性(比如说dos、ddos)
*  镜像安全等(比如说镜像篡改)
*  其它…

容器隔离性方面的安全主要围绕以下两个方面：
*  不会对host造成影响
*  不会对其它容器造成影响

##	2 容器的隔离性
### 2.1	Docker使用的命名空间
Docker使用5个命名空间来改变系统的进程：PID、Network、Mount、UTS、IPC；这些有给Docker一定的安全级别，但Docker还引用了很多没有namespace支撑的资源，由此无法达到KVM实施全面的安全保护目的
### 2.2	当前没有被没有被命名空间化的主要的内核子系统
*  SELinux
*  Cgroups 
*  /sys下的文件系统
*  /proc/sys， /proc/sysrq-trigger，/proc/irq， /proc/bus
*  /proc下kernel关于网络的参数

### 2.3	没有被没有被命名空间化的主要设备
*  /dev/mem
*  /dev/sd* 文件系统设备

## 3 安全加固
目前Docker的安全加固 一般围绕以下几个关键字展开：Selinux/apparmor/capability/seccomp，下文主要讲如何使用这些特性

### 3.1 Selinux 
如果想使用host上的selinux来限制docker启动的container，那运行docker daemon的时候一定要使能selinux;
```
 docker -d -D --selinux-enabled=true
 ```
 
下面的两个例子，一个是能成功启动container，一个不能成功启动Container，通过二者传入的不同参数和最终的不同结果的对比，可以看到selinux起到的作用。

运行container，role设置成guest_r，启动不成功
```
[root@localhost ~]# docker run -it --rm --security-opt label:user:guest_u\
--security-opt label:role:guest_r --security-opt label:type:init_t\
--security-opt label:level:s0 ubuntu /bin/bash
set process label write /proc/self/task/1/attr/exec: invalid argumentFATA[0001] Error response from daemon: Cannot start container 727c72632a691b1c8725226d6bfab19d7522086de3cc5764b6d221349c103f8b: set process label write /proc/self/task/1/attr/exec: invalid argument 
```
运行container，role设置成system_u，启动成功
 
```
[root@localhost ~]# 
[root@localhost ~]# docker run -it --rm --security-opt label:user:system_u\
--security-opt label:role:system_r --security-opt label:type:init_t\
--security-opt label:level:s0 ubuntu /bin/bash
root@97ba1db12a41:/#
```
### 3.2	Apparmor
在/etc/apparmor.d目录创建apparmor的配制文件，如docker：
```
#include <tunables/global>

profile docker flags=(attach_disconnected,mediate_deleted) {

  #include <abstractions/base>
  network,
  capability,
  file,
  umount,

  mount fstype=tmpfs,
  mount fstype=mqueue,
  mount fstype=fuse.*,
  mount fstype=binfmt_misc -> /proc/sys/fs/binfmt_misc/,
  mount fstype=efivarfs -> /sys/firmware/efi/efivars/,
  mount fstype=fusectl -> /sys/fs/fuse/connections/,
  mount fstype=securityfs -> /sys/kernel/security/,
  mount fstype=debugfs -> /sys/kernel/debug/,
  mount fstype=proc -> /proc/,
  mount fstype=sysfs -> /sys/,

  deny @{PROC}/sys/fs/** wklx,
  deny @{PROC}/sysrq-trigger rwklx,
  deny @{PROC}/mem rwklx,
  deny @{PROC}/kmem rwklx,
  deny @{PROC}/sys/kernel/[^s][^h][^m]* wklx,
  deny @{PROC}/sys/kernel/*/** wklx,

  deny mount options=(ro, remount) -> /,
  deny mount fstype=debugfs -> /var/lib/ureadahead/debugfs/,
  deny mount fstype=devpts,

  deny /sys/[^f]*/** wklx,
  deny /sys/f[^s]*/** wklx,
  deny /sys/fs/[^c]*/** wklx,
  deny /sys/fs/c[^g]*/** wklx,
  deny /sys/fs/cg[^r]*/** wklx,
  deny /sys/firmware/efi/efivars/** rwklx,
  deny /sys/kernel/security/** rwklx,
}
```
 
然后执行/etc/init.d/apparmor reload
创建container，并在container内ifconfig，可以看到网络是正常的，如：
```
root@ubuntu:/home/yangsk# docker run -ti --rm --security-opt apparmor:docker ubuntu /bin/bash
root@e55bf5f2a9a7:/# ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:05  
          inet addr:172.17.0.5  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:5/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:6 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:508 (508.0 B)  TX bytes:508 (508.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

root@e55bf5f2a9a7:/# 

```

在上面的基础上，在apparmor的配制文件中去掉“network,”一行
```
#include <tunables/global>

profile docker flags=(attach_disconnected,mediate_deleted) {

  #include <abstractions/base>

  #network,
  capability,
  file,
  umount,

  mount fstype=tmpfs,
  mount fstype=mqueue,
  mount fstype=fuse.*,
  mount fstype=binfmt_misc -> /proc/sys/fs/binfmt_misc/,
  mount fstype=efivarfs -> /sys/firmware/efi/efivars/,
  mount fstype=fusectl -> /sys/fs/fuse/connections/,
  mount fstype=securityfs -> /sys/kernel/security/,
  mount fstype=debugfs -> /sys/kernel/debug/,
  mount fstype=proc -> /proc/,
  mount fstype=sysfs -> /sys/,

  deny @{PROC}/sys/fs/** wklx,
  deny @{PROC}/sysrq-trigger rwklx,
  deny @{PROC}/mem rwklx,
  deny @{PROC}/kmem rwklx,
  deny @{PROC}/sys/kernel/[^s][^h][^m]* wklx,
  deny @{PROC}/sys/kernel/*/** wklx,

  deny mount options=(ro, remount) -> /,
  deny mount fstype=debugfs -> /var/lib/ureadahead/debugfs/,
  deny mount fstype=devpts,

  deny /sys/[^f]*/** wklx,
  deny /sys/f[^s]*/** wklx,
  deny /sys/fs/[^c]*/** wklx,
  deny /sys/fs/c[^g]*/** wklx,
  deny /sys/fs/cg[^r]*/** wklx,
  deny /sys/firmware/efi/efivars/** rwklx,
  deny /sys/kernel/security/** rwklx,
}
```
然后执行/etc/init.d/apparmor reload
创建container，并在container内ifconfig，可以看到网络是不正常的，如：
```
root@ubuntu:/home/yangsk# docker run -ti --rm --security-opt apparmor:docker ubuntu /bin/bash
root@1b4628bd5806:/# ifconfig
warning: no inet socket available: No such file or directory
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:06  
          inet6 addr: fe80::42:acff:fe11:6/64 Scope:Link
          UP BROADCAST RUNNING  MTU:1500  Metric:1
          RX packets:6 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:508 (508.0 B)  TX bytes:508 (508.0 B)

lo        Link encap:Local Loopback  
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

root@1b4628bd5806:/# 
```
 
### 3.3	Capability
这个相对来说比较简单，如：
```
root@ubuntu:/home/yangsk# docker run -ti --rm --cap-add=all ubuntu /bin/bash
root@38e3eb8de268:/# ifconfig eth0 172.17.0.2
root@38e3eb8de268:/# exit
exit
root@ubuntu:/home/yangsk# docker run -ti --rm --cap-add=all --cap-drop=NET_ADMIN ubuntu /bin/bash 
root@779eff647701:/# ifconfig eth0 172.17.0.2
SIOCSIFADDR: Operation not permitted
SIOCSIFFLAGS: Operation not permitted
root@779eff647701:/# 
```

### 3.4	seccomp 
对于系统操作来说，不是很重要的系统调用程序，最好被禁用，以防止被破坏的容器被滥用或误用。目前docker的lxc driver支持这个特性，native drive已经加入了这个特性，并已经合入docker，但docker中尚没有接口来直接使用这个特性。关于seccomp合入libcontainer的过程，详见：
* https://github.com/docker/libcontainer/pull/384
* https://github.com/docker/libcontainer/pull/529
* https://github.com/docker/libcontainer/pull/613
* https://github.com/docker/libcontainer/pull/633

#### 3.4.1	docker 使用lxc driver ，使用seccomp
1.首先需要安装lxc (ubuntu上直接apt-get  install  lxc)
2.生成lxc seccomp配制文件; docker/contrib目录下提供相关的脚本：
./mkseccomp.pl < mkseccomp.sample > seccomp.conf
3.启动daemon
docker -d -D -e lxc
4．启动容器
docker run -ti --rm  --lxc-conf lxc.seccomp=seccomp.conf  ubuntu /bin/bash
#### 3.4.2	docker 使用native driver ，使用seccomp
目前docker中尚没有接口，可以使用直接使用seccomp
#### 3.5	Full Virtualisation
将Docker运行在一个完全虚拟化的方案中，比如说KVM, 这将阻止一个内核漏洞在Docker镜像中被利用导致容器扩为主系统。

## 4	未来发展
### 4.1	Docker支持user ns (两个安全终级目标之一)
容器中的root并不是真正的root
将解决80%现有的技术隐患，极大增强可用性和安全性
libcontainer已经支持user ns，但Docker中尚未实现user ns的功能.
目前社区尚未实现
### 4.2	非root运行Docker daemon(两个安全终级目标之一)
进一步增强安全性
降低使用门槛，很多减法操作不需要了
目前社区尚未实现
### 4.3	docker热升级
目前社区尚未实现

