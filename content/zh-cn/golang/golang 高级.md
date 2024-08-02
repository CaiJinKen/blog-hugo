---
title: "golang高级002"
date: 2019-08-14T19:33:19+08:00
draft: false
type: posts
categories: ["golang"]
tags: ["golang"]
---

* map查找、类型断言、chan都会产生两个值，后一个值为bool类型

  map为返回是否有对应的key，断言为是为判断的类型，chan为是否成功获取值（调用close关闭chan时，也会获取到零值，不能判断是发送的就是零值还是close关闭的，此时ok就有用了，为false）

* 如果一个类型本身是一个指针的话，是不允许出现在接收器中的

  ```go
  type P *int
  func (P)String() string{return ""}// compile error
  ```
