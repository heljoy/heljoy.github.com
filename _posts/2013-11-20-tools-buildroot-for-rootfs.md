---
layout: post
title: "tools: buildroot for rootfs"
description: "record making embedded linux system rootfs under buildroot"
category: linux
tags: [linux, rootfs, tools]
---
{% include heljoy/setup %}

<p class="paragraph">
在嵌入式系统开发前期往往需要大量的调试与测试工具，这些工具并不会引入到开发版或工程版中，主要为了减少固件体积以及避免调试或测试时对功能的影响，于是就产生了很多独立小巧调试工具集，类似PC上的DOC工具箱。busybox有嵌入式Linux环境调试的瑞士军力之称，特点是工具丰富，体积小，可生成为静态链接的执行文件植入到目标系统中，也可以打包生成根文件系统配合内核独立完成各种测试。本文主要介绍使用busybox工具自动构建根文件系统的buildroot软件包，其集成了包括交叉编译器、内核代码、根文件系统生成以及补丁管理等各种功能，可以说是嵌入式开发的IDE。
</p>

<!-- more -->

## Buildroot下载

<p class="paragraph">
Buildroot 是一种基于有序Makefile和patch等文件构建完整嵌入式Linux系统的工具，目前由Peter Korsgaard维护，基于GPLv2发布，项目主页：<a href="http://buildroot.uclibc.org/">buildroot</a>, 我们可以使用下面命令下载最新代码，如果没有额外的需求最好使用稳定分支。在写这篇文章时的稳定分支是2013.08.x。
</p>

{% highlight bash %}
# git clone git://git.buildroot.net/buildroot
# cd buildroot
# git checkout -b 2013.08 origin/2013.08.x
{% endhighlight %}

## Buildroot配置与编译

<p class="paragraph">
Buildroot有与linux相似的配置环境，可以使用menuconfig对各种选项配置，由于不同的平台构架编译相关的配置差异较大，配置前需要详细了解所使用的平台SOC相关信息，下面以手头上现有的ARM架构exynos5410开发板为例,说明平台相关的内容。
</p>

在Target Architecture选项里选择ARM(little endian)，我们使用的SOC属于小端，有关大小端的详细解释可以看[这里](http://en.wikipedia.org/wiki/Endianness)，如果你不清楚你使用的SOC，可以去供应商或ARM官网查询。

接下来Target Architecture Variant选项使用cortex-A15，exynos5410是samsung最新推出的八核ARM处理器，采用大小核架构，4个A7针对日常应用，4个A15可满足大型游戏等复杂应用，提供低功耗高性能方案。

Target ABI是有关浮点模式的支持，这里选择EABI，也与工具链相关，VFP也支持VFPv5-D16，指令集用ARM标准。

完成了与体系结构相关的配置接下来就轻松多了，在Build options项下，可以enable compiler cache提高编译速度，不过也可以不使用。不需要链接静态库，使用动态库放在制作的根文件系统里，各种小工具都编译成动态链接减小体积。是否在编译时添加调试信息以及GCC优化选项等最好不要打开如果没有使用到。

由于是在x86环境编译ARM目标代码，需要使用交叉工具链。为配置方便可以使用Sourcery CodeBench ARM 2013.05，在External toolchain下以选择。为减小体积可以去掉不需要的locale支持，这里选择C en_US就可以。

在System configuration里选择使用mdev管理设备，注意设置好调试串口和波特率。接下来还有kernel、bootloader等配置，我们有单独编译这两项，这里只用来生成根文件系统，有关busybox等工具配置可以使用默认。

注意Filesystem images需要根据我们的内核配置以及uboot支持来设置，在我使用的开发板uboot加载内核与ramdisk时需要image header，内核支持initial RAM filesystem，这里使用cpio打包root filesystem并使用gzip压缩。

## cpio根文件系统分析
<p class="paragraph">
如果不出意外，完成上面的配置后使用make命令就能生成一个xx.cpio.gz的映像文件，可配合内核启动到我们需要的调试模式。由于开发板上uboot需要image header才能正确处理文件，可以使用下面命令添加。
</p>

{% highlight bash %}
# mkimage -A arm -O linux -T ramdisk -C none -a 0x30800000 -n "buildroot" \ 
-d output/images/rootfs.cpio.gz buildroot-ramdisk.img
{% endhighlight %}

<p class="paragraph">
启动后使用root登录就可以使用我们则刚刚制作的系统了，如果缺少工具可以使用上面的配置进入packages等添加，另外也可以使用cpio等工具解压，再修改或添加自己的文件重新打包。
</p>

{% highlight bash %}
# mkdir init_rfs
# cd init_rfs
# gzip -dc < path_to_image/rootfs.cpio.gz | cpio --extract
# ls
bin  etc   init  lib32    media  opt   root  sbin  tmp  var
dev  home  lib   linuxrc  mnt    proc  run   sys   usr
{% endhighlight %}

<p class="paragraph">
initrd RAM就是使用cpio打包并gzip压缩的普通文件，上述解压后可以看到类似我们正在使用系统的目录结构。如果要添加自己静态编译的测试工具，可以放在bin或usr/bin目录，当然有源码也可以使用buildroot中的编译器动态链接后再放进去，再方便也可以利用buildroot框架自动编译，不需要重新拆包打包的，请参考官方手册。修改完成后再打包回去。
</p>

{% highlight bash %}
# find . | cpio -oc | gzip -c -9 >| path_to_image/rootfs_v1.cpio.gz
{% endhighlight %}

<p class="paragraph">
<span class="label label-important">注意:</span> 最后还需要添加image header才能被uboot支持。
</p>

## 相关链接

+ [The Buildroot user manual](http://buildroot.uclibc.org/downloads/manual/manual.html)
+ [Step-by-step Buildroot/Busybox Root File System](http://blog.chinaunix.net/uid-21977330-id-3711356.html)
