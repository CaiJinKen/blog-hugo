---
title: "axios的使用"
date: 2020-04-18T21:24:19+08:00
draft: false
type: posts
categories: ["前端"]
tags: ["JavaScript","axis"]
---

### axios 的使用

[axios](https://github.com/axios/axios) 是前端最常见的网络库。主要调用方式有两种形式：axios(config) 和 axios(url, config)，下面简要介绍axios的使用。



#### 1、请求方法

常用的http请求方法有以下5个



| 方法   | 作用                           |
| ------ | ------------------------------ |
| GET    | 请求资源                       |
| POST   | 新增资源                       |
| PUT    | 更新资源（需要资源的所有信息） |
| PATCH  | 更新资源的部分信息             |
| DELETE | 删除资源                       |



例如有user对象：`{"id":1, "name":"axios-user", "age":20}`

使用axios发起请求可以用axios(config)的形式，如下所示：

```javascript
axios({
  method:"get",//请求方法
  url:"/users",//请求的URL
  params:{id:1},//get、delete方法的请求参数，会放到url之后
  data:{}//post、put、patch方法的请求参数，会放到body中
}).then((resp)=>{
  //请求成功的处理
  console.log(resp.data)//控制台打印返回的body数据
}).catch((err)=>{
  //请求失败的处理
  console.log(err)
}).then(()={
  //不管成功与否，都会执行
  console.log("此处一定会执行")
})
```

对于其他的方式，换一下config对象种的method即可



#### 2、别名

如果有很多的请求，每个我们都需要写一堆像上面1中的axios的配置信息，一个是写起来比较麻烦，另外一个，代码看起来也比较乱，axios也提供了对应请求方法的别名函数，用axios(url, config)的形式，如下所示：

```javascript
//get 
axios("/users",{params:{id:1}}).then((resp)=>{})
//等同于如下方式
axios.get("/users",{params:{id:1}}).then((resp)=>{})

//post
axios.post("/user",{id:2,name:"axios-user2",age:22})

//put 只更新部分信息也需要将整个资源对象提交
axios.put("/user",{id:2,name:"axios-xxx",age:22})

//patch 只需提交唯一标识和更改的数据项即可
axios.patch("/users",{id:2,age:18})

//delete
axios.delete("/users",{params:{id:2}})
```



#### 3、并发请求

axios 也支持同时发起多个请求，并且可以获取到每个请求的结果，使用axios.all() 和 axios.spread()即可，all()方法的参数是函数数组，spread()方法接受的是回调函数，函数中的参数即是all()中的请求结果，顺序和all()保持一致。

例子如下：

```javascript
axios.all(
	[
    axios.get("/users"),//请求1
    axios.get("/students"),//请求2
    axios.post("/teachers",{id:5,name:"teacher wang",age:35})//请求3
  ]
).then(
  axios.spread((resp1,resp2,resp3)=>{
    //resp1,resp2,resp3分别是请求1、请求2、请求3的请求结果，顺序和all中保持一致
    console.log(resp1,resp2,resp3)
  })
})
```



#### 4、axios 实例

在上面的方式中，我们使用的是axios提供默认的实例，对于应用中只有一个服务的时候，可以用默认的实例，但是如果需要请求多个服务（不同ip或者不同域名）的时候，默认的实例就无可奈何了，比如上面接口"/users"、"/students"、"/teachers"是3个不同的服务提供的，这个时候我们可以用axios提供的create()方法自己建实例，如下：

```javascript
//用户实例
const userInstance = axios.create({baseURL:"http://api.users.jin.com"});

//学生实例
const studentInstace = axios.create({baseURL:"http://api.student.jin.com"})

//教师实例
const teacherInstace = axios.create({baseURL:"http://api.teacher.jin.com"})

//然后使用不同的实例去访问不同的资源
userInstace.get("/users")
teacherInstace.post("/teachers",{id:5,name:"xxx",age:40})
```



#### 5、常用参数配置

在以上的例子中，我们已经用到了axios的配置，比如method、params、baseURL，这节主要介绍一下axios常用的一些配置项及其作用

| 配置项       | 作用                                    |
| ------------ | --------------------------------------- |
| method       | 发起请求的方法                          |
| baseURL      | 请求地址+端口（默认80）                 |
| url          | 请求path                                |
| headers      | 请求头                                  |
| params       | 请求参数                                |
| data         | 请求body                                |
| timeout      | 超时时间，单位毫秒，默认0，代表永不超时 |
| maxRedirects | 最大重定向的次数                        |

配置项可以在新建实例或者发起请求的时候使用，不同的地方在于，新建实例的时候使用对所有使用该实例的请求都有效，发起请求的时候使用只对该请求有效：

```javascript
//创建实例时使用
const teacherInstance = axios.create({
  baseURL:"http://api.teacher.jin.com",
  url:"/teachers",
  headers:{"token":"xxxxxx"},
  timeout:1000 //1000ms
})

//使用实例
teacherInstance.get("/",{params:{id:1}})
//等同于 GET http://api.teacher.jin.com/taachers?id=1
//超时时间为1000ms，同时请求头带token信息

//单个请求中覆盖实例的配置
teacherInstance.get("/",{params:{id:1},timeout:2000})
//超时时间为2000，同时请求头带token
```



#### 6、拦截器

对于axios请求，可以在发起请求之前，或者处理响应之前做某些操作，比如发起请求前带上认证信息，获取响应失败的时候统一处理等，这些操作可以放到拦截器里面去统一来做。

例如：

```javascript
//创建axios实例
let instance = axios.create({})

//添加请求拦截器
instance.interceptors.request.use(config=>{
  config.headers.token="token"
  return config//一定记得把配置项返回回去
},err=>{
  return Promis.reject(err)
})

//添加响应拦截器
instance.interceptors.response.use(resp=>{
  console.log(resp.data)
  return resp.data
},err=>{
  return Promis.reject(err)
})
```



