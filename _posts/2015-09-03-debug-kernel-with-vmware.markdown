---
layout: post
title: "[转]使用Vmware调试Linux内核"
comments: true
tags: [debug, kernel]
date: 2015-09-03T23:08:56+08:00
---

转自 [http://f0ol.github.io/2015/09/03/debug-kernel-with-vmware.html](http://f0ol.github.io/2015/09/03/debug-kernel-with-vmware.html)

之前调试内核使用的是kgdb，需要自己编译内核加入kgdb的支持，
这样就没办法调试现有的发行版自带的内核。qemu支持gdbstub，
但是qemu对虚拟机的管理不是特别方便，并且在Mac OS下无法开
启kvm，运行速度慢。bochs和VirtualBox都内建有调试器，bochs
是纯模拟器，运行速度慢，VirtualBox内建的调试器功能不是特
别完善。Vmware支持gdbstub，调试内核时比前面的模拟器/虚拟机
方便许多。

## 虚拟机配置

修改需要调试的虚拟机的.vmx配置文件，加入下面两行：

```Python
debugStub.listen.guest64 = "TRUE"
debugStub.listen.guest32 = "TRUE"
```

启动虚拟机。

## 使用gdb附加

在主机上打开gdb，加载带符号的内核，然后输入：

```Python
target remote :8864
```

32位cpu对应的是8832端口。假如有多个虚拟机开着debugStub，
端口号依次增加。

连接成功后就可以调试了。

## 源码级调试

Ubuntu带符号的内核在 http://ddebs.ubuntu.com/pool/main/l/linux/ 下载。
对应的源码在 http://mirrors.aliyun.com/ubuntu/pool/main/l/linux/ 中搜索
`linux-source`，下载对应版本的deb包。下载的deb包使用`dpkg -X`解包。

下载之后使用gdb加载带符号的内核，`set substitute-path`修改源码路径，
就可以在源码级别对内核进行调试了。

## 其他

Vmware的debugStub功能没有正式的官方文档，下面记录一下自己
使用时候的一些经验。

* 使用`monitor r`命令可以看idt、gdt、msr等寄存器。

* 不仅能调试Linux内核，还能调试bootloader、Windows内核，
调试Windows内核时可以写程序将Windows的符号转换成gdb能
识别的格式，以方便调试。

* 不清楚Vmware使用的是硬件断点还是软件断点，有时候下多于
4个断点的时候会出现不能单步的问题。

* Vmware还有catchpoint这样比较高级的功能，可以在一个地方做个镜像，
之后可以恢复到对应的状态。
