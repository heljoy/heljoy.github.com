---
layout: post
title: "在虚拟盘上调整文件系统尺寸"
description: "enlarge your extn file system on virtualbox disk"
category: lfs
tags: [lfs, tips, fs]
---
{% include heljoy/setup %}

<p class="paragraph">
在做LFS时单独建了一个2GB的SCSI虚拟盘，第二次编译GCC时提示磁盘空间不足，再回头看文档发现最少要求2.8GB的磁盘空间。前面也编译了一些东西，不能就这样删除了再创建一个大点的磁盘，于是结合网上的资料自己动手扩容。这篇文章的做法同样也适用实际系统，仅当学习过程的一点记录。
</p>

<!-- more -->

## enlarge your virtualbox disk

<p class="paragraph">
首先需要更大的磁盘空间，使用virtualbox提供的工具完成。
</p>

{% highlight bash %}
# VBoxManage modifyhd exp.vdi 8096               #8GB for exp.vdi disk
{% endhighlight %}

## enlarge your partion and filesystem

<p class="paragraph">
启动系统，使用fdisk可以发现对应的磁盘空间变大了，但原来的文件系统还是只占有了2GB空间。我并没有使用LVM，需要先调整分区大小再调整文件系统大小。
</p>

{% highlight bash %}
# fdisk -l /dev/sdb
Disk /dev/sdb: 8589 MB, 8589934592 bytes
22 heads, 16 sectors/track, 47662 cylinders, total 16777216 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xd6b8fa28

   Device Boot      Start         End      Blocks   Id  System
   /dev/sdb1            2048     4194303     2096128   83  Linux
{% endhighlight %}

<p class="paragraph">
由于之前使用fdisk将整个磁盘只分了一个区，分区自然就在最前面了。这次同样使用fdisk先删除分区，再重新将整个磁盘建一个分区。注意新分区的起始位置要与原分区一致。
</p>

{% highlight bash %}
# fdisk /dev/sdb
<delete partion, and create another one with some Start>
{% endhighlight %}

<p class="paragraph">
扩容分区后就需要扩容分区上的文件系统了，现在的发行版中应该都包含有这样的工具。
</p>

{% highlight bash %}
# e2fsck -f /dev/sdb1
e2fsck 1.42 (29-Nov-2011)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
/dev/sdb1: 91624/131072 files (0.1% non-contiguous), 380784/524032 blocks

# resize2fs /dev/sdb1
resize2fs 1.42 (29-Nov-2011)
Resizing the filesystem on /dev/sdb1 to 2096896 (4k) blocks.
The filesystem on /dev/sdb1 is now 2096896 blocks long.
{% endhighlight %}

## some tips

<p class="paragraph">
上述扩容的空间需要在原分区之后，适用性就有些限制，现在更常用的做法是使用LVM管理磁盘卷。
</p>
