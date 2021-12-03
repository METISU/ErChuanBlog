# Mach-O简介

Mach-O是基于Mach内核的操作系统（例如iOS、macOS等）上的一种文件格式，包含可执行文件、目标代码或者Frameword等，可以先看下可执行文件的大致结构：

![Mach-O](https://user-images.githubusercontent.com/22512175/144608764-033425aa-f378-4cec-823a-229a56a29c16.png)

可以看到 Mach-O 主要由 3 部分组成:

* Mach Header: 主要描述了CPU架构、文件类型以及load command数量等
* Load Command: 表示文件如何加载进内存
* Data: 用于存放数据以及代码
  * Segments: Segments是具有特定权限的一组内存
  * Section: 更细维度的划分，每个Segments可能拥有0个或多个section

## Mach-O Header

首先进入 **mach-o/loader.h** ，
