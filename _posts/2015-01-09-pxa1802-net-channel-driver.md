---
layout: post
title: "PXA1802网络接口驱动解析"
description: "pxa1802 net channel driver"
category: modem
tags: [modem, net, pxa1802, linux]
---
{% include heljoy/setup %}

<p class="paragraph">
PXA1802与AP连接的HSIC接口通道3用于提供PS网络服务，创建3个net_device分别在4G/3G/2G时传输网络数据到CP，3个net_device底层都对应到HSIC通道3，由于CP同一时间只能驻留在一种网络，3个net_device并不是多路复用，每次只会打开一个，完成数据的上传和下载，理论上可以只提供一个net_device供上层使用，Marvell在网络数据包前添加有特定的包头，用于标记数据包和长包传输。
</p>

<!-- more -->

## 网络数据包头结构

{% highlight c %}
struct ccinethdr{
	__be16 iplen;
	__u8 reserved;
	__u8 offset_len;
	__u32 cid;
} __attribute__((packed));
{% endhighlight %}

<p class="paragraph">
每个发送到CP的网络数据包都会加一个上面的包头，主要填充iplen数据包长和cid发送节点编号(这个编号不清楚CP会不会使用到)，offset_len字段一般为0，AP与CP都会设置为0，包的结尾8Byte对齐，不够补零。多个数据包可以拼成一个长包一次传输，每个数据包都包含包头，包间没有额外的标记，比MUX包的结构简单很多。
</p>

## 网络数据包的发送

<p class="paragraph">
应用传输到net_device的数据包会简单添加一个包头再放到一个链表上，由发送线程统一发到HSIC驱动，完成到CP的传输。发送线程针对链表上的数据包可以拼接成一个长包(长包的最大长度限制为16000Btye)再提交到HSIC驱动，HSIC驱动长短包的传输时间差别不大，因此提高单次传输的数据量可以提升网络的响应速度，另外，针对应用传输的数据包与ACK包还可以重新排序，可避免大量的数据包阻塞ACK包的回传，优化网络性能，但与网络环境有关，没有量化测试数据。
</p>

{% highlight c %}
if ((ip_header->version == 4 && skb->len <= 96) ||
		(ip_header->version == 6 && skb->len <= 128))
	skbpriv(skb)->ackp = true;

netdev->trans_start = jiffies;
headroom = setup_ccinet_hdr(&hdr, iod, skb->len);
tailroom = SIPC_PADLEN(skb->len);
/* need expand size for head and pading */
if ( tailroom + headroom > skb_tailroom(skb) + skb_headroom(skb)) {
	mif_debug("%s: skb_copy_expand needed\n", iod->name);
	skb_new = skb_copy_expand(skb, headroom, tailroom, GFP_ATOMIC);
	if (!skb_new) {
		mif_info("%s: ERR! skb_copy_expand fail\n", iod->name);
		return NETDEV_TX_BUSY;
	}
	/* caller will free old skb with NETDEV_TX_BUSY response */
	dev_kfree_skb_any(skb);
} else
	skb_new = skb;

/* make space for packget hdr */
if (headroom > skb_headroom(skb_new)) {
	skb->data = memmove(skb->head + sizeof(*hdr), skb->data, len);
	skb_set_tail_pointer(skb, len);
}

memcpy(skb_push(skb_new, headroom), &hdr, headroom);
memset(skb_put(skb, tailpad), 0, tailroom);

// add queue for deliver
skb_queue_tail(&ld->sk_tx_q, skb);

{% endhighlight %}

<p class="paragraph">
具体的发送线程比较简单，主要拼接多个skb成一个skb，另外针对ack的优化并没有实验表明提升了网络效率，需要更多的实验调整ack与数据包的比例等，可以不实现。
</p>

## 网络数据包的接收

<p class="paragraph">
CP侧发送过来的数据包也是按照AP发包的规则组成的，提取包头后取出包头上指定长度的数据组成skb直接发给应用，数据长度超过包长的再取下一包，HSIC驱动保证AP或CP取到的数据都是以包头开始的整包数据或几包数据。
</p>

{% highlight c %}
while( len > 0)
{
	hdr = (struct ccinethdr	*)pkhead;
	iplen = be16_to_cpu(hdr->iplen);
	ippacket = pkhead+sizeof(*hdr)+hdr->offset_len;
	if( hdr->cid >= MAX_CID_NUM )
		printk("cid is error :%d, the whole packet len is %d\n", hdr->cid, size);

	data_rx(ippacket, iplen, hdr->cid);
	padding_size = rx_padding_size(sizeof(*hdr) + iplen + hdr->offset_len);
	pkhead += sizeof(*hdr) + hdr->offset_len + iplen + padding_size;
	len -= sizeof(*hdr) + hdr->offset_len + iplen + padding_size;
	n ++;
	if(len < 0)
		printk("some packet is lost!\n");
}
{% endhighlight %}

## 网络吞吐量优化

<p class="paragraph">
在手机入网测试时对网络的吞吐量有要求，目前LTE网络的理论值是下行150Mbps上行50Mbps，但实测HSIC通道的吞吐量最大约为180Mbps，对网络测试有一定的限制，并且不同的总线频率最大的速率也有差别，为尽可能减少对网络速率测试的影响，同时兼顾功耗要求，监控到速率提高时也要提升CPU总线频率。
</p>

<p class="paragraph">
<span class="label label-important">PS:</span> 在EXYNOS系列SOC上都需要手动针对通道速率动态提升系统时钟频率，以满足更高速率的需求，后期可以考虑单独实现成系统服务，提供自动监控并调整频率。
</p>
