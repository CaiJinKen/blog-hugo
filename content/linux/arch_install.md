---
title: "Arch_install"
date: 2019-07-13T22:24:48+08:00
draft: false 
tags: ["linux"]
---
#### 使用*nux 制作U盘启动盘
```bash
fdisk -l #查看当前系统中的磁盘设备，这里u盘设备是/dev/disk2

umount /dev/disk2 #卸载挂载点，备份u盘里面的重要文件，制作启动盘会执行格式化

sudo dd if=archlinux-2019.07.01-x86_64.iso of=/dev/disk2 bs=1m #这用时可能会比较久，稍等一下
```

#### 准备
制作好u盘启动盘之后，设置电脑从u盘启动，之后进入到arch live里默认以root用户登录，之后就可以进行准备操作了。
```bash
vim /etc/hostname #修改安装后的主机名

fdisk /dev/sda #分区，注意设备，我这里是sda

mkfs.ext4 /dev/sda1 #格式化分区

mount /dev/sda1 /mnt #挂载分区

vim /etc/pacman.d/mirrorlist #将中国区的源放到非注释的第一行，会提高软件包的下载速度，越靠前，优先级越高。

pacman -Syy #更新镜像源

dhcpcd #使用dhcp自动获取ip地址

ip addr show #应该能够看到分配到ip地址了，如果使用的wifi，并且加密方式是wpa，则执行以下命令，连接wifi

wpa_supplicant -i wpl3s0 -c <(wpa_passphrase wifi-ssid password) #wpl3s0 wifi-ssid password 分别是无线网卡名称，wifi名称，wifi密码
```

#### 安装
```bash
pacstrap /mnt base base-devel #开始安装到mnt，一路回车就好，稍等直到完成。

arch-chroot /mnt #改变root目录

vim /etc/locale.gen #删除en_US.UTF-8 和 zh_CN.UTF-8 前面的注释

locale-gen #生成语言区域的一些文件

ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime #设置时区为亚洲上海的时区(+08:00)
```

#### 安装引导程序GRUB
```bash
pacmanm -S os-prober grub #安装grub及依赖

grub-install --target=i386-pc /dev/sda #BIOS方式

grub-install /dev/sda #EFI方式

grub-mkconfig -o /boot/grub/grub.cfg #生成grub的配置文件
```

#### 退出chroot环境，从引导区启动
```bash
exit #退出chroot环境

umount -R /mnt #卸载挂载分区

reboot #重启
```

#### 安装桌面环境
```bash
pacman -Ss xf86-video #列出所有可用的驱动

pacman -S xf86-video-intel #Intel集成显卡驱动

pacman -S xorg #

pacman -S xfce4 xfce4-goodies #安装xfce

pacman -S plasma kde-applications kde-l10n-zh_cn #安装KDE

pacman -S sddm #安装图形化的桌面管理器

systemctl start sddm #启动桌面环境

systemctl enable sddm #设置开机启动sddm
```

###问题及建议
> 官方wiki很强大，遇到问题多看，多思考

> 安装完以后是一个非常干净资源占用很少的系统，我安装完不过也才用了1.9GB空间。wget ifconfig iw wpa_supplicant 等命令都没有，楼主只有Wi-Fi可以用，但刚刚安装完却不能连了。从u盘重新启动，然后解决方法如下：
```bash
mount /dev/sda1 /mnt #重新挂载

arch-chroot /mnt

pacman -S net-tools #安装网络工具包

pacman -S iw #安装iw

pacman -S wpa_supplicant #安装Wi-Fi wpa包
```
重新进入安装好的系统后，这些命令都有了。然后就可以连上Wi-Fi下载软件包了。
