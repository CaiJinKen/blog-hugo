---
title: "Namespace"
date: 2019-08-14T19:33:19+08:00
draft: false
type: posts
categories: ["linux"]
tags: ["linux","docker"]
---

### 简介

linux namespace 是linux内核提供的一种环境（资源）隔离的方式，主要提供了UTS、IPC、network、User、mount、PID的隔离

| 类别    | 系统调用参数  | 内核版本     | 作用                 |
| ------- | ------------- | ------------ | -------------------- |
| UTS     | CLONE_NEWUTS  | Linux 2.6.19 | 隔离主机名称，域名称 |
| IPC     | CLONE_NEWIPC  | Linux 2.6.10 | 隔离进程间通信       |
| Network | CLONE_NEWNET  | Linux 2.6.29 | 隔离网络             |
| User    | CLONE_NEWUSER | Linux 3.8    | 隔离用户             |
| Mount   | CLONE_NEWNS   | Linux 2.4.19 | 隔离挂载点           |
| PID     | CLONE_NEWPID  | linux 2.6.24 | 隔离进程ID           |

#### UTC

未隔离之前，我们先看看主机名，打开两个终端，终端1执行`hostname`，查看当前主机名，第二个终端中执行`sudo hostname xxx`再在终端1中执行`hostname`，可以发现主机名称已经被修改了。

为了能够更好的感受下namespace，自己动手写个程序是最好不过的方式了，下面是我写的一个demo，文件名是namespace.go

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	cmd := exec.Command(os.Args[1])
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	fmt.Println("pid:", syscall.Getpid())

	if err := cmd.Run(); err != nil {
		panic(err)
	}

}
```

编译，运行，如下：

```bash
jin@jingege:~/go-test$ go build namespace.go 
jin@jingege:~/go-test$ sudo ./namespace bash
[sudo] password for jin: 
pid: 2526
root@jingege:~/go-test# hostname jingeg2
root@jingege:~/go-test# hostname
jingeg2
```

重新打开一个终端，查看主机名称：

```bash
jin@jingege:~/go-test$ hostname
jingege
```

现在主机名称还是原来的jingege，并没有被改变，所以，UTS主机名称隔离是成功了的，关键就在于用namespace-uts执行bash的时候，我们设置了Cloneflags 为CLONE_NEWUTS，所以，所执行的bash是在一个新的uts命名空间中。

细心的童鞋可能注意到了，用户变成了root，提示符也变成了#，这是因为我们是用sudo执行的命令，uts并不能隔离用户。

#### IPC

在上面的栗子中，主机名称被隔离了，那么IPC呢？我们接着在上面两个终端中分别执行`ipcmk -Q`和`ipcs -q` 创建/查看ipc队列。

```bash
root@jingege:~/go-test# ipcmk -Q
Message queue id: 0
```

```bash
jin@jingege:~/go-test$ ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x307d5895 0          root       644        0            0           
```

看得出来在一个终端中创建的IPC queue在另外一个终端中可以被看到，说明此时IPC并没有被隔离。

我们修改一下上面的程序，把Cloneflags改为

```go
cmd.SysProcAttr = &syscall.SysProcAttr{                  
      Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC,            
}
```

退出被隔离UTS的终端，重新编译、运行

```bash
root@jingege:/mnt/mac_share/jin-practice/go-test# exit
jin@jingege:/mnt/mac_share/jin-practice/go-test$ go build namespace-uts.go 
jin@jingege:/mnt/mac_share/jin-practice/go-test$ sudo ./namespace-uts bash
[sudo] password for jin: 
pid: 2611
root@jingege:/mnt/mac_share/jin-practice/go-test# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    

root@jingege:/mnt/mac_share/jin-practice/go-test# 
```

看到没？此时查看IPC queue时，并没有看到ID为0的那个queue。此时，IPC隔离成功。耶✌️

#### network

此时在不同终端再查看下网络

```bash
root@jingege:~/go-test# ifconfig 
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.15  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::a00:27ff:feed:97bf  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:ed:97:bf  txqueuelen 1000  (Ethernet)
        RX packets 15508  bytes 1096500 (1.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 11863  bytes 7336446 (7.3 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 8792  bytes 29614791 (29.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8792  bytes 29614791 (29.6 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

就会发现，网络信息时一样的，并没有被隔离，接下来我们再修改一下Cloneflags:

```go
cmd.SysProcAttr = &syscall.SysProcAttr{                  
      Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWNET,
}
```

退出bash，重新编译，运行，并查看网络信息：

```bash
root@jingege:～/go-test# exit
exit
jin@jingege:～/go-test$ go build namespace.go 
jin@jingege:～/go-test$ sudo ./namespace bash
pid: 2653
root@jingege:～/go-test# ifconfig 
root@jingege:～/go-test# 
```

看到此时网络信息也没有了，说明网络隔离成功。

#### Mount, User

查看挂载信息

```bash
root@jingege:~/go-test# mount -l
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
udev on /dev type devtmpfs (rw,nosuid,relatime,size=468108k,nr_inodes=117027,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,noexec,relatime,size=100668k,mode=755)
/dev/sda2 on / type ext4 (rw,relatime)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
```

可以看到挂载信息也没有隔离，修改Cloneflags重新编译运行，可以看到还是有些变化的，只是不太明显，次处先略过。

### PID

修改我们的程序为：

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	switch os.Args[1] {
	case "run":
		run()
	case "child":
		child()
	default:
		fmt.Println("invalid param.")
	}
}

func run() {
	fmt.Println("run args:", os.Args)
	cmd := exec.Command("/proc/self/exe", append([]string{"child"}, os.Args[2:]...)...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stdout

	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWNET | syscall.CLONE_NEWUSER | syscall.CLONE_NEWPID,
	}

	fmt.Println("run pid:", syscall.Getpid())
	if err := cmd.Run(); err != nil {
		panic(err)
	}
}

func child() {
	fmt.Println("child args:", os.Args)
	cmd := exec.Command(os.Args[2])
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	fmt.Println("child pid:", syscall.Getpid())
	if err := cmd.Run(); err != nil {
		panic(err)
	}
}
```

重新编译运行

```bash
jin@k53:~/go-test$ go build namespace.go 
jin@k53:~/go-test$ sudo ./namespace run bash
run args: [./namespace run bash]
run pid: 1643
child args: [/proc/self/exe child bash]
child pid: 1
bash: /home/jin/.bashrc: Permission denied
nobody@k53:~/go-test$ 
nobody@k53:~/jin-practice/go-test$ 
nobody@k53:~/jin-practice/go-test$ 
```

看到打印的child pid 为1了，而且设置了newsier 现在用户变成了nobody了！

执行top或者查看/proc/

```bash
root@jingege:~/go-test# ls /proc/
1     18    248   4    639   asound       kcore         scsi
10    19    25    402  641   buddyinfo    key-users     self
1044  2     26    409  655   bus          keys          slabinfo
1045  20    2651  425  656   cgroups      kmsg          softirqs
11    203   2672  426  661   cmdline      kpagecgroup   stat
12    2034  27    427  677   consoles     kpagecount    swaps
128   205   2713  428  679   cpuinfo      kpageflags    sys
129   21    2719  429  688   crypto       loadavg       sysrq-trigger
13    2103  28    440  692   devices      locks         sysvipc
130   2104  29    455  696   diskstats    mdstat        thread-self
131   2125  3     574  700   dma          meminfo       timer_list
132   2186  30    575  730   driver       misc          tty

```

可以看到有很多的数字，每个数字都对应一个进程，数字就是进程ID（pid），说明此时PID并没有被隔离，还是一堆堆的pid。

为什么呢？因为很多工具其实读取的都是/proc/下的信息，此时读取的也是主机的/proc目录，所以当然还是能看到之前的pid了，那怎么来验证呢？很自然的，我们想到linux的chroot命令，用来限制程序可访问的资源范围，那我们重新改下程序运行的root，然后重新挂载一个/proc不就可以了么？child部分的代码：

```go
func child() {
	fmt.Println("child args:", os.Args)
	cmd := exec.Command(os.Args[2])
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := syscall.Chroot("/home/jin/go-test/rootfs/"); err != nil {
		panic(err)
	}
	if err := os.Chdir("/"); err != nil {
		panic(err)
	}
	if err := syscall.Mount("proc", "proc", "proc", 0, ""); err != nil {
		panic(err)
	}
	fmt.Println("child pid:", syscall.Getpid())
	if err := cmd.Run(); err != nil {
		panic(err)
	}
}
```

重新编译，运行：

```bash
jin@k53:~/go-test$ go build namespace.go
jin@k53:~/go-test$ sudo ./namespace run /bin/sh
run args: [./namespace run /bin/sh]
run pid: 1492
child args: [/proc/self/exe child /bin/sh]
child pid: 1
/ # 
/ # 
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /proc/self/exe child /bin/sh
    4 root      0:00 /bin/sh
    5 root      0:00 ps
/ # ls /proc/
1                  cgroups            dma                iomem              kmsg               meminfo            partitions         softirqs           timer_list         zoneinfo
4                  cmdline            driver             ioports            kpagecgroup        misc               pressure           stat               tty
6                  consoles           execdomains        irq                kpagecount         modules            sched_debug        swaps              uptime
acpi               cpuinfo            fb                 kallsyms           kpageflags         mounts             schedstat          sys                version
asound             crypto             filesystems        kcore              loadavg            mtrr               scsi               sysrq-trigger      version_signature
buddyinfo          devices            fs                 key-users          locks              net                self               sysvipc            vmallocinfo
bus                diskstats          interrupts         keys               mdstat             pagetypeinfo       slabinfo           thread-self        vmstat
/ # 
/ # mount
proc on /proc type proc (rw,relatime)
/ # 
```

可以看到/proc/下已经是全新的了，不再有之前主机中的信息了，而且通过mount命令查看挂载信息，也已经是隔离开了的。至此，一个资源隔离开的“容器”环境已经初见雏形了。

### 总结

通过上面我们的程序可以看到，通过namespace，现在程序所使用的资源已经被隔离开了，而namespace就是对资源进行隔离的技术。接下来，思考下如何对程序所使用的资源进行限制呢?
