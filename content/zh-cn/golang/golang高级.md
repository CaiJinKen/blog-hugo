---
title: "golang高级001"
date: 2019-08-14T19:33:19+08:00
draft: false
type: posts
categories: ["golang"]
tags: ["golang"]
---

1. CSP模型，goroutine调度

   csp并发模型：csp并发模型不同于一般的多线程通过共享内存进行通信的方式，而是采用“通过通信进行共享，而不是通过共享进行通信”，用于多个独立并发的实体通过共享通信的channel进行通信的并发模型。csp中channel是第一类对象，它不关注发送消息的实体，而是关注发送消息时使用的channel。

   goroutine调度：

   M：内核线程

   P：cpu的一个核

   G：goroutine

2. 控制并发的方式

   * 通过channel进行并发控制
   * 通过sync包的WaitGroup进行并发控制
   * 通过context进行并发控制

3. GC

   * gc算法：三色标记法（标记-清除）

     流出如下：

     * 初始把所有对象标记为白色
     * 从全局变量和栈变量作为GC root，开始找到所有可达对象，标记为灰色，放入待处理队列
     * 遍历灰色对象队列，将其引用的对象标记为灰色放入队列，自身标记为黑色，并从队列中删除
     * 循环往复上一步，直到灰色队列为空
     * 剩余的白色对象即为垃圾对象，执行清理工作

   * 触发GC的条件：

     * 辅助GC：分配内存时，会判断当前heap内存分配量是否达到触发一轮GC的阀值（每轮GC完成后，阀值会被动态设置），如果超过阀值，则启动一轮GC
     * 显式调用runtime.GC()强制启动一轮GC
     * sysmon是运行时的守护进程，当超过 forcegcperiod (2分钟)没有运行GC会启动一轮GC
