---
layout: post
title: "modem连接通道HSIC驱动分析"
description: "cdc modem hsic driver"
category: modem
tags: [modem, usb, linux]
---
{% include heljoy/setup %}

<p class="paragraph">
由于3G/4G时代蜂窝网络速率的提升，原来modem与AP连接使用的普通UART接口已经不能满足速率需求，
在EXYNOS系列AP上主要使用HSIC(High Speed Inter Chip)接口外接modem，HSIC与USB2.0接口软件协议栈相同，
底层物理层直接使用TTL逻辑电平表示数据，是PCB板间芯片高速互连的专用接口。
关于HSIC具体信息可以参考<a herf="http://blog.csdn.net/arnoldlu/article/details/19169371">《USB HSIC IP》</a>。
</p>

<!-- more -->

<p class="paragraph">
modem一般使用USB CDC-ACM协议，协议主要是规定了USB转UART接口的一些细则，包括流控、波特率等等，
modem的USB接口主要使用协议的数据传输功能，其他参数都可以使用默认值，
波特率并不会影响传输速率，底层速率由USB HOST的性能与URB包的大小决定，流控功能也是软件上阻塞包的接收与发送，
以下也主要讨论HSIC驱动中数据的传输过程。
</p>

<p class="paragraph">
modem通常只有一个CDC-ACM接口(interface)，USB协议规定一个接口最多可以支持16个端点(endpoint)，
其中端点0只用于传输控制信息，是双向端点，其余端点可以传输普通数据，但都只能单向的传输。
一个IO(UART or NET)接口需要一个输出和一个输入端点，PXA1802除端点0外使能了7个输出和7个输入端点，
最多在HSIC接口驱动层就直接7个IO设备访问，XMM6260只使用了其余4个输出和4个输入端点。
</p>

<p class="paragraph">
与普通USB驱动相同，HSIC驱动也是使用usb_register接口注册驱动结构，
USB CORE识别到有对应pid/vid接口的设备连接后就会绑定在驱动结构，关于USB CORE的细节可以参考其他章节。
</p>

{% highlight c %}
static struct usb_driver if_hsic_driver = {
	.name =		"nezha_modem",
	.probe =	if_hsic_probe,
	.disconnect =	if_hsic_disconnect,
	.id_table =	if_hsic_ids,
	.suspend =	if_hsic_suspend,
	.resume =	if_hsic_resume,
	.reset_resume =	if_hsic_reset_resume,
	.supports_autosuspend = 1,
};
{% endhighlight %}

<p class="paragraph">
modem设备在上电后会在HSIC总线上发connect信号，USB HOST HUB检测到有设备连接后会发起枚举，
识别设备的基本信息并创建USB设备节点，USB设备驱动再根据设备的配置信息，接口信息等绑定到USB接口驱动，
就会进到上面<strong>if_hsic_probe</strong>，具体流程在USB CORE有说明。
绑定到接口需要根据端点创建IO通道，通常情况时这里的IO通道会被更上一层usespace IO层复用，并不直接提供IO接口。
因此驱动中使用pipe_data表示一个UART逻辑通道，包含一个输出和一个输入端点，以及数据包URB的传输队列等。
上层IO接口的数据封装到skb结构中再添加到pipe_data的传输队列，同时逻辑通道接收到的数据提交到上层复用接口，
根据通道使用的具体复用协议解析再路由到具体的IO接口。
</p>

{% highlight c %}
	for (i = 0; i < iface_desc->desc.bNumEndpoints; ++i) {
		pipe_data = &usb_ld->devdata[i/2];
		endpoint = &iface_desc->endpoint[i].desc;
		if (usb_endpoint_is_bulk_in(endpoint))
			pipe_data->rx_pipe = usb_rcvbulkpipe(usbdev,
					endpoint->bEndpointAddress);

		if (usb_endpoint_is_bulk_out(endpoint))
			pipe_data->tx_pipe = usb_sndbulkpipe(usbdev,
					endpoint->bEndpointAddress);

		mif_info("endpoint %d maxp %d\n", i, usb_endpoint_maxp(endpoint));
	}

	for (i = 0; i < usb_ld->num_link_ch; ++i) {
		pipe_data = &usb_ld->devdata[i];

		pipe_data->idx = i;
		pipe_data->iod = &ld->iods[i];
		pipe_data->rx_buf_size = (16 * 1024);

		skb_queue_purge(&pipe_data->free_rx_q);
		ret = usb_init_rx_skb_pool(pipe_data);
		if (ret < 0) {
			mif_err("init rx skb pool failed\n");
			goto error_exit;
		}

		ret = hsic_rx_submit(usb_ld, pipe_data, GFP_ATOMIC);
		if (ret < 0) {
			mif_err("rx skb failed\n");
			goto error_exit;
		}
	}
{% endhighlight %}

<p class="paragraph">
上面在绑定时建立逻辑通道pipe_data，记录输入输出端点号，同时在输入端点上提交一次读请求到USB HOST，
后面只要modem在该端点上发送数据，都会回调到驱动的读请求回调。具体读请求使用下面代码实现。
</p>

{% highlight c %}
static int hsic_rx_submit(struct usb_link_device *usb_ld,
		struct if_usb_devdata *pipe_data, gfp_t gfp_flags)
{
	int ret = 0;
	struct urb *urb;
	struct sk_buff *skb;
	int delay = 0;

	skb = skb_dequeue(&pipe_data->free_rx_q);	/* free queue first */
	if (!skb) {
		skb = alloc_skb((pipe_data->rx_buf_size),
				GFP_ATOMIC); /* alloc new skb with GFP_ATOMIC */
		if (!skb) {
			mif_debug("alloc skb fail\n");
			pipe_data->defered_rx = true;
			delay = msecs_to_jiffies(20);
			goto defered_submit;
		}
	}

	skbpriv(skb)->context = pipe_data;
	urb = pipe_data->urb;
	urb->transfer_flags = 0; // NOTE::
	usb_fill_bulk_urb(urb, usb_ld->usbdev, pipe_data->rx_pipe,
		(void *)skb->data, pipe_data->rx_buf_size, hsic_rx_complete,
		(void *)skb);

	if (!usb_ld->if_usb_connected)
		return -ENOENT;

	usb_mark_last_busy(usb_ld->usbdev);
	ret = usb_submit_urb(urb, gfp_flags);
	if (ret)
		mif_err("submit urb fail with ret (%d)\n", ret);

defered_submit:
	/* Hold L0 until rx sumit complete */
	usb_mark_last_busy(usb_ld->usbdev);
	schedule_delayed_work(&pipe_data->rx_defered_work, delay);
	return ret;
}
{% endhighlight %}

<p class="paragraph">
驱动中使用一个链表管理空闲读请求内存，这样确保每次发送读请求时都有空闲内存可用。
提交到USB HOST的读请求URB在读到modem发来的数据或出错时都会回调hsic_rx_complete，
这里再把读到的数据发往上层，由上层根据具体协议解析处理。
</p>

<p class="paragraph">
驱动中读请求的数据发往上层、绑定到设备、唤醒等情形时都会再次提交读请求到USB HOST以便再次接收modem发送的数据。
这里同时提交多个读请求URB并不会提高传输效率，所有同向的URB会依次顺序处理，
当然如果驱动在处理数据延时较大时也可提交多个读URB使用流水线方式提高效率。
这里可以暂停读请求URB提交阻塞modem数据发送达到流控的效果。
</p>

<p class="paragraph">
上层接口的数据添加结构头等内容后发送到逻辑通道，驱动构建URB请求，
填充待发送数据，提交到USB HOST依次发送到modem。
</p>

{% highlight c %}
static int hsic_send(struct link_device *ld, int idx_link, struct sk_buff *skb)
{
	size_t tx_size;
	int ret, rpm_state;
	struct urb *urb;
	gfp_t mem_flags = in_interrupt() ? GFP_ATOMIC : GFP_KERNEL;
	struct usb_link_device *usb_ld = to_usb_link_device(ld);
	struct if_usb_devdata *pipe_data = &usb_ld->devdata[idx_link];

	/* store the tx size before run the tx_delayed_work*/
	tx_size = skb->len;

	/* drop packet, when link is not online */
	if (ld->com_state != COM_ONLINE && ld->com_state != COM_BOOT) {
		mif_err("drop packet, size=%d, com_state=%d\n",
				skb->len, ld->com_state);
		return -ENODEV;
	}

	/* get active async */
	rpm_state = link_pm_runtime_get_active_async(usb_ld->link_pm_data);
	if (!rpm_state)
		pm_runtime_get_noresume(&usb_ld->usbdev->dev);

	urb = usb_alloc_urb(0, mem_flags);
	if (!urb) {
		mif_err("alloc urb error\n");
		ret = -ENOMEM;
		goto done;
	}

	usb_fill_bulk_urb(urb, usb_ld->usbdev, pipe_data->tx_pipe, skb->data,
			skb->len, hsic_tx_complete, (void *)skb);

	skbpriv(skb)->context = pipe_data;
	skbpriv(skb)->ld = ld;
	skbpriv(skb)->urb = urb; /* kill urb when suspend if tx not complete*/

	if (usb_get_rpm_status(&usb_ld->usbdev->dev) != RPM_ACTIVE) {
		usb_anchor_urb(urb, &pipe_data->tx_deferd_urbs);
		ret = 0;
		goto done;
	}

	skb_queue_tail(&pipe_data->sk_tx_q, skb);

	ret = usb_submit_urb(urb, mem_flags);
	if (ret < 0) {
		mif_err("usb_submit_urb with ret(%d)\n", ret);
		skb_unlink(skb, &pipe_data->sk_tx_q);
		usb_anchor_urb(urb, &pipe_data->tx_deferd_urbs);
		goto done;
	}

done:
	if (!rpm_state) {
		usb_mark_last_busy(usb_ld->usbdev);
		pm_runtime_put(&usb_ld->usbdev->dev);
	}

	return tx_size;
}
{% endhighlight %}


<p class="paragraph">
发送流程看起来代码比较多，但关键部分很少，大部分都是一些状态判断，暂时不用关注USB设备的动态休眠功能，这个会单独更新。在设备ACTIVE时，直接提交填充好的URB请求到USB HOST，这里是发到输出端点，
如果设备在休眠还需要唤醒设备，为减少发送延时，这里使用异步唤醒，当不在ACTIVE状态时添加到tx_deferd_urbs在唤醒时再次提交发送。
</p>

{% highlight c %}
static int hsic_defered_tx_from_anchor(struct if_usb_devdata *pipe_data)
{
	struct urb *urb;
	struct sk_buff *skb;
	int cnt = 0;
	int ret = 0;

	while ((urb = usb_get_from_anchor(&pipe_data->tx_deferd_urbs))) {
		usb_put_urb(urb);
		skb = (struct sk_buff *)urb->context;

		/* drop package when disconnect */
		if (!pipe_data->usb_ld->if_usb_connected) {
			usb_free_urb(urb);
			dev_kfree_skb_any(skb);
			continue;
		}

		/* refrash the urb */
		usb_fill_bulk_urb(urb, pipe_data->usb_ld->usbdev, pipe_data->tx_pipe,
			skb->data, skb->len, hsic_tx_complete, (void *)skb);

		skb_queue_tail(&pipe_data->sk_tx_q, skb);

		ret = usb_submit_urb(urb, GFP_ATOMIC);
		if (ret < 0) {
			/* TODO: deferd TX again */
			mif_err("resume deferd TX fail(%d)\n", ret);
			skb_unlink(skb, &pipe_data->sk_tx_q);
			usb_anchor_urb_head(urb, &pipe_data->tx_deferd_urbs);
			goto exit;
		}
		cnt++;
	}
exit:
	if (cnt)
		mif_info("deferd tx urb=%d(CH%d)\n", cnt, pipe_data->idx);
	return ret;
}
{% endhighlight %}

<p class="paragraph">
发送时异步唤醒的方式可以节省usespace IO时间，发送与唤醒同时进行，在设备唤醒后再提交缓存的URB。接收时设备一定是处在ACTIVE状态，如果不在ACTIVE状态，modem会先主动唤醒AP再发送数据。关于AP与modem休眠唤醒流程会单独介绍。
</p>
<p class="paragraph">
上面CDC-ACM协议都是使用bulk-in/out类型端点发送数据，驱动中只需要注册USB接口驱动结构，实现bulk类型端点的数据传输，数据结构设计、数据传输流程等都可以参考内核中usb cdc驱动。
</p>

## 参考资料

- [USB High Speed Inter-Chip (HSIC) IP](http://blog.csdn.net/arnoldlu/article/details/19169371)
- [Transitioning from USB 2.0 HSIC to USB 3.0 SSIC](http://www.synopsys.com/Company/Publications/DWTB/Pages/dwtb-transitioning-usb-jan2013.aspx)
- [USB2.0 SPEC](http://www.usb.org/developers/docs/usb20_docs/)
