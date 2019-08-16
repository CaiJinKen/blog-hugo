---
title: "Namespace_Improve"
date: 2019-08-16T21:03:23+08:00
draft: false
tags: ["linux","docker"]
---

在上一篇[namespace](https://caijinken.github.io/linux/namespace/)的文章中，我们了解并通过程序写了一个简单版的容器，体验了Linux namespace的功能，但是还有两个美中不足的地方，一个是需要使用root权限才能够运行我们的程序，另外一个我们在容器中使用ps看到pid为1的进程是`/proc/self/exe child sh`，而不是sh。所以这篇文章就讲讲如何解决这两个问题。

#### root权限

在Linux namespace中，只有user namespace 是可以被普通用户创建的，其他的namespace 都需要有root权限才可以。但这样就会有问题：

1、并不是每个用户都有sudo权限；

2、进程的安全性。

其实第一点还好解决，关键是第二点，运行的进程是以root权限运行的，这就有可能导致安全问题，比如，容器内的进程受到攻击，控制权到了攻击者的手中，此时的进程是用root权限运行的，所以攻击者可以用该进程执行任意权限的操作。

也有人可能会想，我们不是已经把所有的namespace都clone出来了么，rootfs也改变了，即使被攻击了，影响的范围也就是container中。说的是没错，不过，如果我们把主机的目录当作卷挂载到了container中，那么攻击者所能影响的范围就扩大到了主机中。所以，我们就把这个影响减少到最少---当前用户的权限。看看代码：

```go
	uid := os.Getuid()
	gid := os.Getgid()

	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUSER | syscall.CLONE_NEWUTS | syscall.CLONE_NEWNS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWNET | syscall.CLONE_NEWPID,
		UidMappings: []syscall.SysProcIDMap{{0, uid, 1}},
		GidMappings: []syscall.SysProcIDMap{{0, gid, 1}},
		Credential: &syscall.Credential{
			Uid: 0,
			Gid: 0,
		},
	}
```

第一行和第二行得到当前登陆用户的uid/gid (user id/ group id)；

第六行，我们做了一个uid的映射，syscall.SysProcIDMap第一个参数是容器内的用户的id，0是root，第二个参数是需要映射的用户的权限，第三个参数是映射范围，一般用1就可以，表示一一对应；第七行也是一样，只不过是gid的映射；

第8行，设置容器内用户的uid和gid，都为0，说明是用户是root，用户所在的组是root组

重新编译运行：

```bash
jin@k53:~/go-test$ go build namespace.go 
jin@k53:~/go-test$ ./namespace run sh
p pid: 4342
c p pid: 0
c pid: 1
/ # 
/ # 
```

可以看到现在不用root权限就可以运行了。docker 也不需要root权限就能运行呢，向docker又前进了一小步。



#### PID 1

我们使用ps命令可以看到，pid为1的进程并不是我们想要的sh，而是/proc/self/exec

```bash
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /proc/self/exe child sh
    4 root      0:00 sh
    6 root      0:00 ps
```

那么我们怎么把sh变成pid为1的进程呢？？？把/proc/self/exe干掉，它被kill掉以后，pid最小的就变成sh了，看起是这样的，不过可别忘了，linux中pid为1的进程一般都是init进程，比如systemd之类，如果init进程被kill了，那么其他的进程也会被干掉了，系统就gg了，所以这样是行不通的。我们可以试一下，可以看到并没什么用。

```bash
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /proc/self/exe child sh
    4 root      0:00 sh
    5 root      0:00 ps
/ # kill -9 1
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /proc/self/exe child sh
    4 root      0:00 sh
    6 root      0:00 ps
/ # 
```

linux中有个函数execve，可以使用执行的进程替换父进程。可以使用man execve看一下使用方法。

```bash
jin@k53:~$ man execve

EXECVE(2)                                                                                Linux Programmer's Manual                                                                               EXECVE(2)

NAME
       execve - execute program

SYNOPSIS
       #include <unistd.h>

       int execve(const char *filename, char *const argv[],
                  char *const envp[]);

DESCRIPTION
       execve()  executes the program pointed to by filename.  This causes the program that is currently being run by the calling process to be replaced with a new program, with newly initialized stack,
       heap, and (initialized and uninitialized) data segments.
```

我们把child中的exec.Command 换成syscall.Exec()

```go
func child() {
	must(syscall.Chroot("/home/jin/rootfs/"))
	must(os.Chdir("/"))
	os.Setenv("HOME", "/")
	must(syscall.Mount("proc", "proc", "proc", 0, ""))
	must(syscall.Sethostname([]byte("container")))

	path, err := exec.LookPath(os.Args[2])
	must(err)

	must(syscall.Exec(path, append([]string{path}, os.Args[3:]...), os.Environ())) 
	must(syscall.Unmount("proc", 0))
}
```

syscall.Exec()和exec.Command()有个很大的不同是，exec()的可执行文件的路径必须是绝对路径，Command()

则没有这个要求，它会调用exec.LookPath()去自动获得path。

最后编译运行

```bash
jin@k53:~/go-test$ ./namespace run sh
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    4 root      0:00 ps
/ # 

```

可以看到，我们自己写的container已经和docker运行bash之后是一样的了。耶耶耶！✌️