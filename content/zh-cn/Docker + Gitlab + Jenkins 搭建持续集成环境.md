---
title: "Docker + Gitlab + Jenkins 搭建持续集成环境"
date: 2020-05-02T20:20:42+08:00
draft: false
type: 'posts'
categories: ["CI/CD"]
tags: ['others']
---

#### Docker + Gitlab + Jenkins 搭建持续集成环境

##### 安装docker

```bash
sudo apt update

sudo apt install apt-transport-https \
     ca-certificates \
     curl \
     gnupg-agent \
     software-properties-common
     
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository \
     "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
   
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

##### 安装Gitlab

```bash
sudo docker pull gitlab/gitlab-ce:nightly
```

由于众所周知的原因，国内拉取某些镜像慢的一批，所以我们需要把docker镜像源换一下，

```bash
sudo vim /etc/docker/daemon.json
```

我这里替换为网易和百度的源，当然也可以换成任意你想要的一个或多个

```bash
{
  "registry-mirrors": [
    "https://hub-mirror.c.163.com",
    "https://mirror.baidubce.com"
  ]
}
```

之后，重启docker

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

镜像拉取好了以后，启动容器

```bash
# 新建gitlab挂载卷目录
sudo mkdir /opt/gitlab

# 启动容器
sudo docker run -p 443:443 -p 22:22 -p 80:80 --name gitlab -v /opt/gitlab/config:/etc/gitlab -v /opt/gitlab/logs:/var/log/gitlab -v /opt/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce:nightly
```

容器启动好了以后，访问你http://<你的ip>:80，这里，我的ip为192.168.4.128，所以为 http://192.168.4.128:80，设置URL，这个就是以后访问gitlab的URL，保存好以后，会让设置初始密码的界面，设置完成以后，会跳转到登陆页面，管理员用户名为root。

之后，就可以管理用户、组、项目等，再次略过不提。我建起来不是提供公用服务的，所以，用户不能让用户自己注册账号，所以需要取消允许用户注册的选项。需要添加用户的时候手动添加。

Setting->General->Sign-up restrictions ->sign-up enabled

##### 安装Jenkins

```bash
sudo docker pull jenkinsci/blueocean:1.23.2

# 新建Jenkins挂载卷目录
sudo mkdir /opt/jenkins

# 启动容器
sudo docker run -d --name jenkins -u root -p 8080:8080 -p 50000:50000 -v /opt/jenkins/:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkinsci/blueocean:1.23.2
```

容器启动好了以后，访问你http://<你的ip>:8080，这里，我的ip为192.168.4.128，所以为 http://192.168.4.128:8080，看到需要密码，我们去log里面找

```bash
sudo docker logs jenkins
```

从里面就可以找到这样的密码

```
*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

90de0fcc19bf4b5ca40b1042ccdff466

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

或者

```bash
sudo docker exec -it jenkins bash
cat /var/jenkins_home/secrets/initialAdminPassword
```

使用这个密码登陆进去以后，就到了插件安装页面，我选择的是安装推荐的那些插件，你可以选择自定义安装。

等待一会儿，插件安装玩了会让你填写Jenkins Admin用户的账户信息，填写完保存以后就用这个账号作为管理员账号了。

#### 遇到的问题及解决方法：

1、gitlab smtp不生效：

```bash
external_url 'http://192.168.4.128:80'

gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.163.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "邮箱地址"
gitlab_rails['smtp_password'] = "你的密码"
gitlab_rails['smtp_domain'] = "163.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
```

配置好了以后，进入容器，重新加载配置

```bash
sudo docker exec -it gitlab bash
gitlab-ctl reconfigure
```

手动添加用户以后，发现用户没有收到邮件，测试一下邮件发送服务是否正常

```bash
sudo docker exec -it gitlab bash
gitlab-rails console

>Notify.test_email('测试邮件地址', 'title', 'content').deliver_now
```

发现报了一下错误：

```bash
sh: 1: /usr/sbin/sendmail: not found
```

之后重新找到配置邮件的地方，重新配置并重新加载配置，重新执行发送邮件，发现错误信息为密码错误，经过查阅资料，发现第三方登录163邮箱，用的不是密码，而是一个授权码，登陆邮箱，获取授权码，将配置中的密码换成授权码

重新加载配置并测试以后，发现错误信息如下

```bash
550 User has no permission
```

原来是邮箱设置中，还需要设置客户端授权码需要开启，开启之后，发现邮件发送成功，并且邮箱成功收到了邮件。

打开邮件，发现链接地址为 http://43178sfe，是容器的ID，并不是一个合法有效的URL，更新配置如下，重新加载测试，发现正常了

```bash
external_url 'http://192.168.4.128:80'
```

2、Jenkins go 安装失败

默认安装go不同版本时，从golang.org下载，由于众所周知的原因，会失败。

这时候，可以选择手动安装，完了以后配置GOPATH GOROOT就行

我的话，不想自己搞，不同版本弄起来麻烦，我多试了几次之后，神奇的安装好了

