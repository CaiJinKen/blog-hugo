---
title: "Cgroup_subsystem"
date: 2019-08-22T21:22:50+08:00
draft: false
type: posts
categories: ["linux"]
tags: ['docker','linux']
---

上一篇[cgroup](https://caijinken.github.io/linux/cgroup/)主要讲了cgroup和常用的subsystem，算是理论部分，都说实践出真理，这篇文章就是实践部分了，介绍如何使用。

### subsystem

1. cpuset

   * 限制
     * cpuset.cpus: 设置任务可以使用的CPU核心的编号，格式为：from_number-to_number, number，用`,`隔开
     * cpuset.mems:  设置cpu可使用的memory node，格式同上
   * 统计
     * cpuset.effective_cpus: 在使用的cpu核心
     * cpuset.effective_mem: 在使用的memory node
     * cpuset.memory_pressure：cupset中的分页

2. cpu

   * 限制
     * cpu.cfs_quota_us: 设定周期内最多可使用的时间
     * cpu.cfs_period_us: 设置使用cpu的周期时间，需要和cfs_quota_us配合使用
   * 统计
     * cpu.stat: 显示各种统计信息
     * cpuacct.stat: 统计用户态和内核态使用的cpu时长
     * cpuacct.usage_*:  统计每个任务使用的cpu时长

3. blkio

   * 限制
     * blkio.throttle.read_bps_device: 设置读取设备的速率上限，格式为：device_num:node_number byte_per_second
     * blkio.throttle.read_iops_device: 设置每秒读取设备的次数上限
     * blkio.throttle.write_bps_device: 设置写设备的速率上限
     * blkio.throttle.write_iops_device: 设置每秒写设备的次数
   * 统计
     * blkio.reset_stats: 重置统计信息
     * blkio.throttle.io_service_bytes: 读设备/写设备的字节数
     * blkio.throttle.*_recursive: 各种统计数据的递归版本。这些文件显示与非递归对应文件相同的信息，但包括来自所有后代cgroup的统计信息
     * blkio.throttle.io_serviced: io次数

4. memory

   * 限制

     * memory.limit_in_bytes: 设置内存使用限制，单位有k、m、g，-1时无限制
     * memory.max_usage_in_bytes: 设置最大内存使用量
     * memory.move_charge_at_immigrate: 设置移动的花费
     * memory.soft_limit_in_bytes: 软限制，需要比limit_in_bytes小
     * memory.oom_control：改参数填 0 或 1， `0`表示开启，当 cgroup 中的进程使用资源超过界限时立即杀死进程，`1`表示不启用。默认情况下，包含 memory 子系统的 cgroup 都启用

     * memory.kmem.limit_in_bytes: 设置内核使用的内存使用限制
     * memory.kmem.max_usage_in_bytes: 设置内核使用的最大内存限制
     * memory.kmem.tcp.limit_in_bytes: 设置tcp buf内存的硬限制
     * memory.kmem.tcp.max_usage_in_bytes: 设置tcp buf使用的最大内存

   * 统计

     * memory.stat: 显示各种统计数据
     * memory.usage_in_bytes: 显示内存的当前使用情况
     * memory.numa_stat: 显示每个numa节点的内存使用量
     * memory.kmem.usage_in_bytes: 显示当前内核内存分配

5. devices

   * 限制
     * devices.allow: 允许使用的设备，语法：device_type device_number:node_number access_type。device type有三种：b（块设备）、c（字符设备）、a（全部）；access type也有三种：r（读）、w（写）、m（创建）
     * devices.deny: 禁止使用的设备，语法同上。
   * 统计
     * devices.list: 这个 cgroup 中的task 设定访问控制的设备

### 示例

1. 限制任务使用的cpu核心

   ```bash
   root@k53:~# mkdir /sys/fs/cgroup/cpuset/cgroup-test
   #可以使用符号-指定范围，比如1-5,8就是1,2,3,4,5,8这几个核心，因为在我虚拟机中只有1核，如果输入的范围有错误的话，会有“write error: Numerical result out of range”错误提示
   root@k53:~# echo '1'>/sys/fs/cgroup/cpuset/cgroup-test/cpuset.cpus
   # 将进程pid xxx加入任务中，则xxx最多就只能使用1核心，每次只能加一个
   root@k53:~# echo xxx>>/sys/fs/cgroup/cpuset/cgroup-test/tasks
   ```

2. 限制任务使用的cpu周期

   ```bash
   root@k53:~# mkdir /sys/fs/cgroup/cpu/cgroup-test
   #数值范围是1000 - 1000,000，即0.1% - 100%
   root@k53:~# echo '20000'>/sys/fs/cgroup/cpu/cgroup-test/cpu.cfs_quota_us 
   # 将进程pid xxx加入任务中，则xxx最多只能使用20%的cpu周期
   root@k53:~# echo xxx>/sys/fs/cgroup/cpu/cgroup-test/tasks
   ```

3. 限制任务使用的块设备速率

   ```bash
   root@k53:~# mkdir /sys/fs/cgroup/blkio/cgroup-test
   # 设置写磁盘的速率最高为2000b/s
   root@k53:~# echo '8:0 2000'>/sys/fs/cgroup/blkio/cgroup-test/blkio.throttle.write_bps_device 
   root@k53:~# echo xxx>/sys/fs/cgroup/blkio/cgroup-test/tasks
   ```

   查看主设备号和次设备号的方法：

   ```bash
   # 查看设备信息
   jin@k53:~$ cat /proc/devices 
   Character devices:
     1 mem
     4 /dev/vc/0
     4 tty
     .........
   Block devices:
     7 loop
     8 sd
     9 md
    11 sr
    65 sd
     .........
   
   #查看设备主次编号，第五列和第六列分别为设备的主编号和次编号
   jin@k53:~$ ls -l /dev
   total 0
   brw-rw---- 1 root    disk      8,   1 Aug 23 05:14 sda1
   brw-rw---- 1 root    disk      8,   2 Aug 23 05:14 sda2
   crw-r--r-- 1 root    root     10, 235 Aug 23 05:14 autofs
   drwxr-xr-x 2 root    root         280 Aug 23 05:14 block
   drwxr-xr-x 2 root    root          80 Aug 23 05:14 bsg
   crw-rw---- 1 root    disk     10, 234 Aug 23 05:14 btrfs-control
   drwxr-xr-x 3 root    root          60 Aug 23 05:14 bus
   lrwxrwxrwx 1 root    root           3 Aug 23 05:14 cdrom -> sr0
   drwxr-xr-x 2 root    root        3680 Aug 23 05:14 char
   crw------- 1 root    root      5,   1 Aug 23 05:14 console
   lrwxrwxrwx 1 root    root          11 Aug 23 05:14 core -> /proc/kcore
   crw------- 1 root    root     10,  58 Aug 23 05:14 cpu_dma_latency
   crw------- 1 root    root     10, 203 Aug 23 05:14 cuse
   drwxr-xr-x 7 root    root         140 Aug 23 05:14 disk
   drwxr-xr-x 3 root    root         100 Aug 23 05:14 dri
   ```

4. 限制任务使用的内存

   ```bash
   root@k53:~# mkdir /sys/fs/cgroup/memory/cgroup-test
   # 设置最多只能使用100m内存，单位可以为k，m，g
   root@k53:~# echo 100m>/sys/fs/cgroup/memory/cgroup-test/memory.limit_in_bytes
   root@k53:~# 
   root@k53:~# echo xxx>/sys/fs/cgroup/memory/cgroup-test/tasks
   ```

5. 设置任务设备黑/白名单

   ```bash
   root@k53:~# mkdir /sys/fs/cgroup/devices/cgroup-test
   # 禁止任务写入sda2(8:2)数据
   oot@k53:~# echo b 8:2 w>/sys/fs/cgroup/devices/cgroup-test/devices.deny 
   root@k53:~# 
   root@k53:~# echo xxx>/sys/fs/cgroup/devices/cgroup-test/tasks
   ```

参考资料：

[cgroup简介](https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v1/index.html)
