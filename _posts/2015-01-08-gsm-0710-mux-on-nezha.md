---
layout: post
title: "GSM 07.10 MUX 协议驱动解析"
description: "3GPP GSM 07.10 MUX on nezha HSIC channel 0"
category: modem
tags: [modem, pxa1802, linux]
---
{% include heljoy/setup %}

<p class="paragraph">
PXA1802使用HSIC接口与AP连接，HSIC枚举实现7个虚拟通道ttyUSB0~6，为方便扩展，Marvell在HSIC虚拟通道0上实现了<a href="">3GPP GSM-07.10 MUX协议</a>，MUX协议提供多路数据流复用底层单一通道，多路数据流互不干扰的多路复用方法。MUX通道主要用于AT命令，通话音量控制，NVM操作等小数据量的通讯，网络PS业务和CP调试LOG都直接使用HSIC虚拟通道。本文主要介绍MUX协议帧结构、数据收发、通道控制等信息。
</p>

<!-- more -->

<p class="paragraph">
MUX协议允许多个并发的活动运行在一个底层TE(Terminal Equipment)和MS(Mobile Station)的串行接口上，下图是GSM－07.10文档上描述MUX协议栈的层次结构，显示了多层协议与功能层次结构，multiplexer层提供4个独立的数据流支持，发送数据时并不需要添加额外的数据帧，数据流通道使用DLC(Data Link Connection)表示，每个DLC有独立的编号，在PXA1802驱动中convergence层没有使用，physical层直接使用HSIC通道实现。
</p>


	                            TE                                 MS
	                  +--------------------+             +--------------------+ 
	  TE Process(4)   |    |     |    |    |             |    |     |    |    |    MS Process(4)
	                  +----+-----+----+----+             +----+-----+----+----+
	Convergence Layer |    |     |    |    |             |    |     |    |    |  Convergence Layer
	                  +--------------------+             +--------------------+
	Multiplexer Layer |                    |             |                    |  Multiplerxer Layer
	                  +--------------------+             +--------------------+
	  Physical Layer  |                    |             |                    |    Physical Layer
	                  +--------------------+             +--------------------+
	                            | |                                 | |
	                            | |                                 | |
	                            | +---------------------------------+ |
	                            +-------------------------------------+
	        

<p class="paragraph">
MUX协议内容的细节可以参考本文结尾的链接，以下只介绍在PXA1802中MUX协议驱动模块实现中需要关注的重点，这些包括会话建立与结束，数据传输与接收以及部分控制帧，标准协议的其他细节在驱动实现时只作了解，另外，驱动也没有完全实现标准协议的所有内容。
</p>

## MUX帧结构

<p class="paragraph">
GSM-07.10支持3种传输模式，项目中只实现最简单的Basic模式，其余高级模式都带有数据保护或错误恢复等功能，是对Basic模式的加强，后面也只在Basic模式上介绍各种操作的实现。协议一共定义了6种类型帧，帧结构都相同，都包含一个开始标识字节和结尾标识字节，一个地址字节，一个控制字节，一到两个长度字节，内容空间和一个校验字节，具体帧结构定义如下。
</p>

	  Flag Addr Ctrl  Len   Info     FCS Flag
	 +----+----+----+-----+----------+---+---+
	 |  1 | 1  | 1  | 1~2 | User def | 1 | 1 |
	 +----+----+----+-----+----------+---+---+

- Flag是帧开始与结尾的标记，占一个字节，是一个常量值，在Basic模式里为0x9F。一次可传输多个帧，上一帧的结尾标记可作为下一帧的开始标记。
- Addr占一个字节，3~8bit表示MUX通道编号，1bit是扩展位，固定为1，2bit标记当前是命令帧还是响应帧。
- Ctrl占一个字节，表示数据包的类型，其中5bit是P/F位，具体含义参考文档5.2.1.3 Setion。
- Len占一到两个字节，1bit是扩展位，如果内容字段长度小于0x7F，Len占一个字节，扩展位为1，内容长度占Len的2~8bit
- FCS 是校验字节，是Flag后面，Info前面内容的校验信息，如果是UI类型的帧，还包含Info部分，校验算法文档有给出


<p class="paragraph">
所有在TE和MS间传输的帧都是上面描述的结构，部分控制帧没有Info字段，用Ctrl表示帧类型，Basic模式协议一共支持下面6种类型帧，AT命令等数据内容通常使用UIH类型帧传输。
</p>

- Set Asynchronous Balanced Mode (SABM)--建立逻辑通道;
- command Disconnect (DISC) command-取消逻辑通道;
- Unnumbered Acknowledgement (UA) response-响应建立通道;
- Disconnected Mode (DM) response-响应取消逻辑通道;
- Unnumbered information with header check (UIH);
- Unnumbered Information (UI).
 

## DLC的创建与释放流程

<p class="paragraph">
应用打开设备准备发送数据前，MUX驱动会阻塞在打开进程中，直到打开成功。MUX驱动在收到打开消息时，会向打开通道发送SABM帧，如果DLC0此时没有被打开，还会先在DLC0上执行一次打开操作，待DLC0打开成功后才允许打开别的DLC通道。一般来说，CP侧收到SABM帧后会响应UA帧表示建立通道，AP收到UA帧再返回打开成功状态。关闭DLC时发送DISC帧，等待响应DM帧时表示DLC关闭成功。理论上打开与关闭是对等的，AP与CP侧都可以主动发起，但项目实现中只有AP可以发起打开与关闭，并且HSIC通道0固定运行MUX协议，不能被关掉。以下给出理论上DLC打开与关闭流程。
</p>


	   Userspace          AP  Mux driver            CP Mux driver
	
	   fd_open(i) --------> mux_open(i)
	                                   -----SABM(0)----->
	                                   <-----UA(0)-------
	                                   -----SABM(i)----->
	                                   <-----UA(i)-------
	             <--Opened--
	        .
	  write("AT") -------> mux_write()
	                                   ----UIH("AT")---->
	                                   <----UIH("OK")----
	  "OK" = read()<-------mux_push()
	        .
	   fd_close(i) ------> mux_close(i)
	                                   -----DISC(i)----->
	                                   <-----DM(i)-------


<p class="paragraph">
另外，MUX驱动中还支持一个MSC(Modem Status Command), PN(Port Negotiation)等，主要是AP与CP侧MUX驱动同步一些状态，参考协议文档了解具体信息。
</p>

## DLC数据发送与接收流程

<p class="paragraph">
除DLC0外，应用可以使用每个DLC发送数据到CP侧对应的DLC，CP也会将处理结果返回到该DLC，多个应用可以使用不同编号的DLC与CP通讯，由MUX协议保证通道的独立性。实际上层应用开发时具体每个DLC的用途需要参考CP的说明文档。协议有描述DLC通道的优先级，驱动实现时所有发到DLC通道的数据都会填充成MUX帧，按优先级序号链接到优先级链表上，发送线程从链表上按顺序取出MUX帧，发送到HSIC驱动。优先级相同的DLC帧轮流发送。具体实现流程用部分代码表示。
</p>

{% highlight c %}
while (count) {
	size = min_t(uint, count, dlc->mtu);
	skb = alloc_skb(size + MUX_SKB_RESERVE, GFP_KERNEL);
	if (!skb)
		break;
	// 为帧头部预留空间
	skb_reserve(skb, SKB_HEAD_RESERVE);
	// 填充发送内容数据
	memcpy(skb_put(skb, size), buf + sent, size);

	// 填充Len字段，长度大于0x7F时占两字节
	if (size > 127) {
		hdr = (void *) skb_push(skb, 4);
		put_unaligned(cpu_to_le16(__len16(size)), (__le16 *) &hdr->len);
	} else {
		hdr = (void *) skb_push(skb, 3);
		hdr->len = __len8(size);
	}
	hdr->flag = 0x9F;
	hdr->addr = dlc->addr;
	hdr->ctrl = __ctrl(RFCOMM_UIH, 0);

	// fill FCS
	crc = skb_put(skb, 1);
	*crc = __fcs((void *) hdr);

	// fill end tag
	flag = skb_put(skb, 1);
	*flag = 0x9F;

	// tx mux frame queue for deliver
	skb_queue_tail(&dlc->mux_tx_q, skb);

	// next frame
	sent += size;
	count -= size;
	}
{% endhighlight %}

<p class="paragraph">
应用发送到DLC的数据都会分成小于mtu字节的帧，填充帧头与校验后挂到发送链表，发送线程根据链表顺序取对应DLC上的帧发送到HSIC驱动，由底层完成实际传输。根据MUX协议，在发送线程中可合并同一DLC上的多个帧一次传输到CP，这部分目前还没有实现，发送线程部分代码如下，HSIC部分的传输涉及到USB协议，会在单独的文章里面讲述。
</p>

{% highlight c %}
	/* tx all skb by priority list, one skb for each by priority */
mux_for_each:
	cur_priority = 0; /* set highest priority first */

	list_for_each_entry_safe(dlc, dlc_n, &mux_data->tx_mux_head, mux_list) {
		if (skb_queue_empty(&dlc->mux_tx_q))
			continue;
		
		/* something about prioity queue
		 * Prority 0 channel will be sended first(MUX control)
		 * higher prority channel will deliver all skb than low 
		 * the same prority channel will deliver one skb by turns
		 */
		if (cur_priority == 0)
			cur_priority = dlc->priority;
		else if (cur_priority < dlc->priority)
			goto mux_for_each;

		skb = skb_dequeue(&dlc->mux_tx_q);

		ret = ld->send(ld, skb);
		if (ret < 0) {
			mif_err("ld->send fail (%s, err %d)\n", dlc->name, ret);
			dev_kfree_skb_any(skb);
		}
	}

{% endhighlight %}

## Links

+  [3GPP关于GSM TS 07.10协议的文档] (http://www.3gpp.org/ftp/Specs/archive/07_series/07.10/)
+  [关于07.10协议简单介绍] (http://blog.csdn.net/codejoker/article/details/6205898)
+  Implementation of the GSM 07.10 Linux Device Driver,Tuukka Karvonen,21.04.2004

