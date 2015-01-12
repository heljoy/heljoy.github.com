---
layout: post
title: "使用sk_buff结构的几点备份"
description: "need to konw about sk_buff"
category: modem
tags: [net, linux]
---
{% include heljoy/setup %}

<p class="paragraph">
sk_buff结构是linux内核buffer管理的一大神器，广泛分布在net代码中，是整个网络协议栈关键结构，重要程度丝毫不亚于list_head，在PXA1802与原XMM项目中也都是使用sk_buff在HSIC驱动以及上层io_dev驱动传输数据，组织结构，以下简单介绍sk_buff在项目中的使用以及注意点，都只涉及到buffer管理与构造等简单功能，仅当备份。
</p>

<!-- more -->

<p class="paragraph">
内核定义的sk_buff结构很长，在开发中大部分都没有使用，只使用到sk_buff list等管理buffer的一些工具和构造sk_buff内容的API，下面列出部分关心的结构字段。
</p>


{% highlight c %}
struct sk_buff {
	/* These two members must be first. */
	struct sk_buff		*next;
	struct sk_buff		*prev;

	ktime_t			tstamp;

	struct sock		*sk;
	struct net_device	*dev;

	char			cb[48] __aligned(8);
...
	unsigned int		len,
				data_len;
...
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head,
				*data;
	unsigned int		truesize;
	atomic_t		users;
}

struct sk_buff_head {
	/* These two members must be first. */
	struct sk_buff	*next;
	struct sk_buff	*prev;

	__u32		qlen;
	spinlock_t	lock;
};

{% endhighlight %}

<p class="paragraph">
驱动中主要使用了sk_buff结构的链表与数据索引功能，一般情况下sk_buff管理的都是一整块连续的内存块，len表示数据长度，data_len指分片数据的长度，tail与end分别指向数据结尾与可用buffer结尾，head与data分别指向可用buffer头与数据头，具体结构可用下图表示。
</p>
	
	head ------->  +----------+
	               | headroom |
	data ------->  +----------+
	               |          |
	               |   DATA   |
	               |          |
	tail ------->  +----------+
	               | tailroom |
	end -------->  +----------+
	
<p class="paragraph">
使用<strong>alloc_skb(len, GFP_XXX)</strong>分配到sk_buff结构一般headroom为空，head到end为指定的len长度，可使用下面API构造具体的传输结构数据，在MUX驱动中构造mux帧时需要使用到下面内容，传输HSIC urb数据和io_dev数据只需要用到skb的队列功能。
</p>

- <strong>skb_pull(h_len)</strong>去掉数据头的h_len字节，一般解析完结构性数据后再处理下一个数据项
- <strong>skb_push(h_len)</strong>在数据块头添加h_len字节内容，返回数据头地址，一般添加结构头
- <strong>skb_put(t_len)</strong>在数据结尾添加t_len字节内容，返回添加内容的起始地址
- <strong>skb_reserve(len)</strong>在数据块头预留len字节的空间，用于后面添加结构头

<p class="paragraph">
sk_buff结构为队列功能实现了单独的队列结构sk_buff_head结构，用于链接需要管理的所有sk_buff，针对队列操作也提供了下面的宏，简化处理。
</p>

- <strong>skb_queue_head_init</strong> 初始化队列
- <strong>skb_queue_head</strong>, <strong>skb_queue_tail</strong>添加sk_buff到队列
- <strong>skb_dequeue</strong>,<strong>skb_dequeue_tail</strong>删除sk_buff从队列
- <strong>skb_queue_purge</strong> 清空队列
- <strong>skb_queue_walk</strong> 循环队列，处理sk_buff时需要lock

<p class="paragraph">
一般的sk_buff操作有上面的API基本够用，可以省去自己重新构建数据结构管理buffer的工作，一些队列的操作也不需要关注竞争等，同时处理io_dev为net设备时可以直接使用sk_buff传到上层。
</p>

