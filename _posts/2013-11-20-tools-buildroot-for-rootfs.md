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

## Buildroot配置与编译

## cpio根文件系统分析

## 添加自定义工具
