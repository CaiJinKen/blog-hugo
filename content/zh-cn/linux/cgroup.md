---
title: "Cgroup"
date: 2019-08-19T20:18:42+08:00
draft: false
type: posts
categories: ["linux"]
tags: ['linux','docker']
---

### 简介

Croups(control groups)是Linux内核提供的一种可以限制、记录、隔离进程(进程组)所使用的的物理资源（如：CPU、memory、IO、device等）的机制。是轻量级虚拟化技术（docker、lxc）的基础之一。另外的基础包括[namespace](https://caijinken.github.io/linux/namespace/)、[ufs(union file system)](https://caijinken.github.io/linux/namespace/)等。

### 概念

* 基本概念
  - 任务（task）。在cgroup中，任务就是系统的一个进程。
  - 控制组（control group）。控制组就是一组按照某种标准划分的进程，比如按内存大小，cpu的使用率，或者io的速率限制划分成组。一个进程可以加入到某个控制组，也可以从一个组迁移到另外一个组。一个组中的进程可以使用cgroup以控制组为单位分配的资源，同时受到组分配到的资源的限制。
  - 层级（hierarchy）。控制组可以组织成为层级的形式，组成一棵控制组树。控制组树上的子节点是父节点控制组的孩子，继承父控制组的资源限制。
  - 子系统（subsystem）。一个子系统就是一个资源控制器，子系统必须附加（attach）到一个层级上才能使用。
* 关系和限制
  - 一个子系统最多只能附加到一个层级。
  - 一个层级可以有多个子系统。
  - 一个任务可以是多个cgroup的成员，但是这些cgroup必须是不同的层级。
  - 任务创建子任务时，子任务自动成为其父任务所在的cgroup成员。可以把子任务移动到不同的cgroup中，但最开始时，它总是继承父任务的cgroup。

### 系统支持的子系统

* 查看系统支持的cgroup

  ```bash
  # 1、安装cgroup工具，如果已经安装，请忽略
  sudo apt install cgroup-tools
  # 2、查看系统的子系统
  $lssubsys
  cpuset
  cpu,cpuacct
  blkio
  memory
  devices
  freezer
  net_cls,net_prio
  perf_event
  hugetlb
  pids
  rdma
  jin@k53:~$
  
  jin@k53:~$ uname -a
  Linux k53 5.0.0-25-generic #26-Ubuntu SMP Thu Aug 1 12:04:58 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
  jin@k53:~$ lsb_release -a
  No LSB modules are available.
  Distributor ID:  Ubuntu
  Description:  Ubuntu 19.04
  Release:  19.04
  Codename:  disco
  jin@k53:~$ 
  ```

  可以看到，我现在系统使用的内核版本是5.0.0-25，所支持的子系统也很多，下面简单介绍下cgrop子系统的作用

* 作用

  | 子系统     | 作用                                                         |
  | ---------- | ------------------------------------------------------------ |
  | cpuset     | 为cgroup中的任务分配指定的cpu核心（多核下）                  |
  | cpu        | 提供调度程序为任务使用cpu的限制                              |
  | cpuacct    | 为cgroup中的任务所使用的cup生成使用报告（统计信息）          |
  | blkio      | 为cgroup中的任务所使用的块设备设定读/写限制                  |
  | memory     | 为cgroup中的任务所使用的内存进行限制，并自动生成内存使用报告 |
  | devices    | 设定cgroup中任务允许/拒绝访问的设备                          |
  | freezer    | 挂起/恢复cgroup中的任务                                      |
  | net_cls    | 可以给 packet 打上 classid 的标签，用于过滤分类。分类完了可以使用tc（traffic controller）限制任务的网络速率 |
  | net_prio   | 可以为各个 cgroup 中的任务动态配置每个网络接口的流量优先级   |
  | perf_event | perf复用了trace的所有插桩点，并且加入了采样法(硬件PMU)。PMU是一种非常重要的数据采集方法，因为它大部分是硬件的，所以可以做到一些软件做不到的事情，获取到一些底层硬件的信息。主要根据事件进行性能分析 |
  | hugetlb    | 用于支持大页面，使得任务可以根据需要灵活地选择虚存页面大小，用于减少缺爷中断，提高性能 |
  | pids       | 设定任务能创建的进程数量                                     |
  | rdma       | RDMA让计算机可以直接存取其他计算机的内存，而不需要经过处理器的处理 |

### 示例

```bash
# 创建目录
jin@k53:~$ mkdir cgroup-test
# 将目录挂载，类型为cgroup
jin@k53:~$ sudo mount -t cgroup -o none,name=cgrpup-test cgroup-test ./cgroup-test/
# 查看层级
jin@k53:~$ cd cgroup-test && ls
cgroup.clone_children  cgroup.sane_behavior  release_agent
cgroup.procs           notify_on_release     tasks
jin@k53:~/cgroup-test$ 
```

上面的步骤我们创建了一个cgroups的层级，并进行挂载，可以看到自动生成了几个文件，cgroup.clone_children用来指定子节点是否继承父节点的配置，tasks设定使用该控制组的任务，release_agent用来指定任务退出时的操作。上面我们说过，子层级是会继承父层级的配置的，我们来试一下

```bash
jin@k53:~/cgroup-test$ mkdir cgroup01
jin@k53:~/cgroup-test$ mkdir cgroup02
```

之后，我们查看cgroup01和cgroup02里面的文件，和cgroup-test是一样的，并且，组cgroup-test层级设定的限制依然是对cgroup01或者cgroup02中的任务起作用的。

通过命令`mount -t cgroup` 我们可以看到，系统已经默认给我们创建好了一个顶层层级，并且设置好了不同的子系统，比如memory、cpu、blkio等，我们直接使用，或者以创建子层级的方式使用即可，推荐使用子层级的方式。

### 实战

我们通过上面的知识，知道cgroup的作用了，现在可以来实际使用感受一下。我就以memory这个子系统作为示例，我们申请200m内存，看看使用了memory做限制的效果。

* 没有使用cgroup做限制时

  ```bash
  jin@k53:~$ sudo apt install stress
  jin@k53:~$ stress --vm-bytes 200m --vm-keep -m 1 &
  jin@k53:~$ top
  top - 09:06:23 up  4:30,  2 users,  load average: 0.15, 0.03, 0.01
  Tasks: 103 total,   2 running, 101 sleeping,   0 stopped,   0 zombie
  %Cpu(s): 99.7 us,  0.3 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  MiB Mem :    983.1 total,     88.4 free,    437.6 used,    457.1 buff/cache
  MiB Swap:   1964.0 total,   1957.5 free,      6.5 used.    394.4 avail Mem 
  
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  
   3033 jin       20   0  208616 204920    268 R  99.7  20.4   0:11.22 stress   
      1 root      20   0   99856  10284   7660 S   0.0   1.0   0:01.39 systemd  
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.00 kthreadd
  ```

  可以看到，我的内存是983.1M，stress占用了20.4%，也就是200.5524M

* 使用cgroup做限制

  ```bash
  # 创建一个层级
  jin@k53:~$ sudo mkdir /sys/fs/cgroup/memory/memory-test
  # 设置内存限制为100m
  jin@k53:~$ sudo echo "100m">/sys/fs/cgroup/memory/memory-test/memory.limit_in_bytes
  # 创建一个测试文件
  jin@k53:~$ touch test.go
  ```

  编辑文件，文件内容如下：

  ```go
  func main() {
    cmd := exec.Command("stress", "--vm-bytes", "200m", "--vm-keep", "-m", " 1")
    cmd.Stdin = os.Stdin
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Run(); err != nil {
       panic(err)
    }
  }
  ```

  编译运行该文件，可以看到内存使用变成了10.1%，也就是99.2931M

  ```bash
  jin@k53:~$ go build test.go
  jin@k53:~$ sudo ./test &
  jin@k53:~$ top
  Tasks: 109 total,   4 running, 105 sleeping,   0 stopped,   0 zombie
  %Cpu(s):  2.2 us, 21.3 sy,  0.0 ni,  0.0 id, 59.9 wa,  0.0 hi, 16.5 si,  0.0 st
  MiB Mem :    983.1 total,    272.0 free,    338.4 used,    372.6 buff/cache
  MiB Swap:   1964.0 total,   1849.5 free,    114.5 used.    494.4 avail Mem 
  
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND  
   3083 root      20   0  208616 101796    268 R  32.6  10.1   0:08.99 stress   
      1 root      20   0   99856  10180   7640 S   0.0   1.0   0:01.41 systemd
  ```

  bingo！✌️，资源限制成功了。。。

### 总结

通过上面的讲解和栗子，我们可以看到linux是如何通过cgroup对进程使用的资源进行限制的，但是，对于整个cgroup的子系统的了解还不是特别深，下面这篇[cgroup-subsystem](https://caijinken.github.io/linux/cgroup_subsystem/)就对如何使用子系统做了介绍。
