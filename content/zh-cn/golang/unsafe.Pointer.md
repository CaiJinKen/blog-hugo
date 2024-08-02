---
title: "unsafe.Pointer"
date: 2020-05-02T20:20:42+08:00
draft: false
type: posts
categories: ["golang"]
tags: ['golang']
---

unsafe.Pointer相当于指向任意类型的指针(*指针)，但它有四种特殊的操作不同于一般的指针：

1. 任意类型的指针(*)可以转换为Pointer
2. Pointer指针也可以转换为任意类型的指针
3. uintptr可以转换成Pointer
4. Pointer可以也可以转换为uintptr

即:

 Pointer <--->(*)

uintptr<----> Pointer

Pointer允许程序忽略类型检测从而读写任意的内存，所以在使用unsafe包的时候要格外小心。

比如将float64转化为int64，正常情况下是需要遵守go的类型检测，进行类型强转之后才能正常运行：

```go
var (
  sourceValue float64
  targetValue int46
)
sourceValue = 300.00
targetValue = int64(sourceValue)
// targetValue = sourceValue //err: type not match
```

使用示例：

* 将\*T1转换为\*T2

  ```go
  func Float64bits(f float64) uint64{
    return (*uint64)(unsafe.Pointer(&f))
  }
  
  //unsafe.Pointer(&f)即第一条规则：将任意类型指针转换为Pointer
  //(*uint64)(unsafe.Pointer)即第二条规则：将Pointer转换为任意类型
  ```

* 将Pointer转换为uintptr(取地址值)

  ```go
  func Pointer2uintptr(p unsafe.Pointer) uintprt{
    return (uintptr)(p)
  }
  //(uintprt)(unsafe.Pointer)即第三条规则：将Pointer转换为uintptr
  ```

  `Note:`一般情况下不要这么用，因为uintptr本质上是整数类型，当Pointer中的指针值变了以后，uintptr的值并不随着原始指针的值变化，这里仅仅只是出于演示的目的这么写，所以别用任何变量保存Pointer转换为uintptr之后的值

* 将指针和偏移做运算以后返回

  ```go
  p = unsafe.Pointer(uintptr(p)+offset)
  //offset 是一个整型的值，可以是计算后的，比如
  //对于结构体：
  f = unsafe.Pointer(uintptr(unsafe.Pointer(&s))+unsafe.Offsetof(s.f))
  //上面等同于
  f = unsafe.Pointer(&s.f)
  
  //对于slice
  e := unsafe.Pointer(uintptr(unsafe.Pointer(&s[i]))+i*unsafe.Sizeof(s[0]))
  //上面等同于
  e := unsafe.Pointer(&s[i])
  ```

* 将reflect.Value.Pointer 或者reflect.Value.UnsafeAddr转换为Pointer

  ```go
  p := (*int)(unsafe.Pointer(reflect.ValueOf(new(int)).Pointer()))
  ```

* 通过Pointer将string和slice互换

  先看下string和slice的底层结构：

  ```go
  type StringHeader struct {
    Data uintptr
    Len  int
  }
  
  type SliceHeader struct {
    Data uintptr
    Len  int
    Cap  int
  }
  ```

  如果slice保存的是字符串的字节数组的话，在底层表现上它和string唯一的区别就是多了Cap（容量）字段。既然结构差不多，其实可以通过Pointer进行互转：

  ```go
  var s string
  var bts []byte
  
  //string to slice
  sh:=(*reflect.StringHeader)(unsafe.Pointer(&s))
  bh = (*reflect.SliceHeader)(unsafe.Pointer(&bts))
  bh.Data,bh.Len,bh.Cap = sh.Data,sh.Len,sh.Len
  bts = *(*[]byte)(bh)
  //等同于
  bh := (*reflect.SliceHeader)(unsafe.Pointer(&s))
  bh.Len = len(s)
  bts = *(*[]byte)(bh)
  
  //slice to string
  bh:=(*reflect.SliceHeader)(unsafe.Pointer(&bh))
  sh:=(*reflect.StringHeader)(unsafe.Pointer(&s))
  sh.Data,sh.Len = bh.Data,bh.Len
  //等同于
  s = *(*string)(unsafe.Pointer(&bts))
  ```

从上面的几个例子中，我们就可以看的出来，unsafe.Pointer的作用就是在`必要`的地方作为类型转换的桥梁。

