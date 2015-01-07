---
layout: post
title: "PXA1802固件下载过程解析"
description: "pxa1802 modem frimware download detail"
category: modem
tags: [modem, pxa1802, linux]
---
{% include heljoy/setup %}

<p class="paragraph">
MX4PRO项目上使用Marvell公司的PXA1802基带芯片，该芯片支持GSM/TD-SCDMS/WCDMA/FDD/TDD五种制式，为MX4PRO提供良好的移动网络体验。使用在MX4PRO上的PXA1802没有内置Flash，所有固件文件放在AP侧的Flash里，在芯片上电时再下载到芯片中，最后跳转到下载的程序运行。PXA1802使用两套固件支持五模，分为LWG和LTG，LTG支持GSM/TD-SCDMA/TDD/FDD四种，LWG支持GSM/WCDMA/FDD/TDD四种，分别对应到中国移动与中国联通的运营要求。
</p>

<p class="paragraph">
PXA1802基带芯片用于定制机时通常只需要一套固件，通用版本时允许两套固件切换，具体使用的固件与SIM卡、网络信号环境等因素有关。另外，芯片分两步下载固件，先下载基础服务程序，再使用服务程序下载运行固件，最后跳转到运行固件建立网络等具体电话业务，芯片与AP使用HSIC接口通讯，每次跳转时都需要重新枚举。
</p>

<!-- more -->

## PXA1802基带芯片固件文件

<p class="paragraph">
LTG与LWG两套固件分别应对移动与联通的网络环境，每套固件都由5个文件和一个RD分区组成，其中DKBI为基础服务程序，主要用于下载运行时的固件文件，SKL,Skylark为RF芯片固件，OBMI为初始化程序，HSIC为主要运行程序，RD分区保存一些参数，如功率校准参数、IMEI号等，5个文件一个分区分两次下载到基带芯片。
</p>

{% highlight bash %}
root@mx4pro:/ # ls /etc/firmware/                                              
NZ_LTG_DKBI.bin
NZ_LTG_DL_M09_Y0_AI_SKL_Flash.bin
NZ_LTG_DL_MEIZU_HSIC.bin
NZ_LTG_MDB.zip
NZ_LTG_OBMI.bin
NZ_LWG_DKBI.bin
NZ_LWG_DL_M09_Y0_AI_SKL_Flash.bin
NZ_LWG_DL_MEIZU_HSIC.bin
NZ_LWG_MDB.zip
NZ_LWG_OBMI.bin
Skylark_LTG.bin
Skylark_LWG.bin
{% endhighlight %}

#### MBD文件为数据库文件，用于解析modem生产的日志文件，不属于运行固件。

## 固件下载流程

<p class="paragraph">
LTG与LWG固件下载的流程相同，只是对应的固件文件不一样，下载时的通讯协议完全一样，下面仅以LTG固件为例说明。LTG与LWG切换相当重新下载一次固件，只是不同固件支持的协议不一样。
</p>

- 给基带芯片上电，然后初始化AP侧EHCI控制器，等待HSIC PORT枚举（这里枚举不上一般是硬件问题）
- 枚举到设备后下载基础服务程序DKBI，下载协议后面再给出，下载时每个包都需要反馈状态
- 下载DKBI完成后AP与CP同时会关掉HSIC，AP先初始化完成HSIC后CP再跳转上电HSIC，再次枚举
- 枚举到设备后开始下载运行固件，依次下载OBMI、HSIC、SKL、Skylark、RD分区。下载协议比较简单，后面给出
- 运行固件下载完成后还有一次跳转，同样AP与CP关掉HSIC，等CP切换后AP初始化HSIC，CP再上电HSIC，第三次枚举
- 第三次枚举完成后还需要在通道0上发送AT+CMUX=0切换到MUX通道，这时RIL/at_client等各种应用才能与modem通讯

<p class="paragraph">
整个下载过程涉及到三次枚举，6个文件传输，持续7~9s。枚举由硬件信号发起，软件检测到设备加载驱动；固件下载由校验与确认包保证正确性，各文件的传输协议基本相同。HSIC PORT枚举过程中软件完成的动作会在USB驱动一篇再介绍，本文主要围绕PXA1802讲述下载协议，为改造meizu_modem模块提供支持。
</p>

## DKBI下载过程解析

<p class="paragraph">
DKBI文件包含TIM与DKB两部分，TIM占前面512字节，DKB从1024字节到文件结尾。两部分的下载流程相似，使用开发过程中的部分代码标识流程。
</p>

{% highlight c %}
	// send preamble first
	hsic_wait_for_req(nz_modem, preamble_buf, PREAMBLE_LEN);
	// get CP version
	hsic_wait_for_req(nz_modem, getversion_buf, GETVERSION_LEN);

send_bin_data:
	// send left data length
	header_buf[8] = send_len & 0xFF;
	header_buf[9] = (send_len >> 8) & 0xFF;
	header_buf[10] = (send_len >> 16) & 0xFF;
	header_buf[11] = (send_len >> 24) & 0xFF;
	ret = hsic_wait_for_req(nz_modem, header_buf, 12);


	// get request send length above, send the request data
	sendbin_buf[4] = nz_modem->req_dkbi_len & 0xFF;
	sendbin_buf[5] = (nz_modem->req_dkbi_len >> 8) & 0xFF;
	sendbin_buf[6] = (nz_modem->req_dkbi_len >> 16) & 0xFF;
	sendbin_buf[7] = (nz_modem->req_dkbi_len >> 24) & 0xFF;
	memcpy(tmp_buf, sendbin_buf, 8);
	memcpy(tmp_buf + 8, buf_pos, nz_modem->req_dkbi_len);
	ret = hsic_wait_for_req(nz_modem, tmp_buf, pkt_len);

	if (data_left)
		goto send_bin_data;

	// send done tag
	hsic_wait_for_req(nz_modem, senddone_buf, 8);

	// send disconnect at the end of DKBI
	if (DKB)
		hsic_wait_for_req(nz_modem, disconnect_buf, 8);
{% endhighlight %}

<p class="paragraph">
各类型数据包的包头信息固定，CP可以很容易从包头中区分传输的内容，整个传输过程就是先告诉需要传输的长度，CP返回允许传输的长度，再实际传输指定长度的内容，直到所有内容传输完成。在DKB的最后需要发disconnect信号，并且所有的传输都需要检查返回值，只有正确才能传输下一包。本次所有内容传输完成后会断开HSIC，重新枚举。再次枚举连接上时CP已经运行在DKBI环境中，主要下载运行固件。
</p>

## OBMI下载过程解析

<p class="paragraph">
OBMI是第二次枚举后下载的第一个文件，下载过程也以部分代码标识。
</p>

{% highlight c %}
// OBMI handshake request
ret = send_img_cmd(nz_modem, 0x55545311, 0);
// OBMI start request
ret = send_img_cmd(nz_modem, 0x55545312, 0);
// OBMI data send
ret = send_img_data(nz_modem, obmi_data);
// OBMI end request
ret = send_img_cmd(nz_modem, 0x55545314, 0);
// OBMI check total byte request
ret = send_img_cmd(nz_modem, 0x55545315, fw->size);
{% endhighlight %}

<p class="paragraph">
OBMI下载前后也都需要发送确认信息帧，信息帧只有8个字节，前4个字节表示帧类型，后4个字节内容含义由前面确定，上面标识出了OBMI下载时发送的几种帧。OBMI文件内容的发送操作与后面的固件文件都在这一次枚举内完成，发送的流程也类似，最后再分析。
</p>

## ARBEL\MSA\MSA2\RELIABLE文件下载过程解析

<p class="paragraph">
除OBMI外，后面的几个文件的下载流程类似，除数据内容不一样，其余信息帧也基本相同，放在一起讨论。
</p>

{% highlight c %}

// CPIMG handshake request
ret = send_img_cmd(nz_modem, 0x58862345, 0x58873401);
// arbel start request
ret = send_img_cmd(nz_modem, 0x58862346, 0);
// arbel data send
ret = send_img_data(nz_modem, arbel_data);
// arbel end request
ret = send_img_cmd(nz_modem, 0x58862348, 0);
// msa start request
ret = send_img_cmd(nz_modem, 0x58862349, 0);
// msa data send
ret = send_img_data(nz_modem, msa_data);
// msa end request
ret = send_img_cmd(nz_modem, 0x58862350, 0);
// msa2 start request
ret = send_img_cmd(nz_modem, 0x58862355, 0);
// msa2 data send
ret = send_img_data(nz_modem, msa2_data);
// msa2 end request
ret = send_img_cmd(nz_modem, 0x58862356, 0);
// reliable start request
ret = send_img_cmd(nz_modem, 0x58862353, 0);
// reliable data send
ret = send_img_data(nz_modem, reliable_data);
// reliable end request
ret = send_img_cmd(nz_modem, 0x58862354, 0);
// send total byte request
ret = send_img_cmd(nz_modem, 0x58862351, ARBEL+MSA+MSA2+RELIABLE);

{% endhighlight %}

<p class="paragraph">
基本可以看出第二次枚举下载过程可分为OBMI与CPIMG两个阶段，每个阶段包含一个handshake和total_check，其余文件下载时都需要一个开始与结束的信息帧。以下再具体介绍数据内容发送过程。
</p>

## 第二次枚举数据发送流程

{% highlight c %}

	if (OBMI_IMAGE) {
		tag = 0x55545313;
		pkt_len = OBMI_DATA_PACKET_LEN;
	} else if (CPIMG_IMAGE) {
		tag = 0x58862347;
		pkt_len = IMG_DATA_PACKET_LEN;
	}


	offset = idx = 0;
	len = IMG_PAYLOAD(pkt_len);
	while(len) {
		data_ptr[0] = tag;
		data_ptr[1] = len;
		data_ptr[2] = idx;
		memcpy(data_ptr+3, fw->data+offset, len);
		data_ptr[IMG_CHECKSUM_POS(pkt_len)] = 
			calc_packet_checksum(data_ptr, IMG_CHECKSUM_POS(pkt_len));
		ret = hsic_wait_for_req(nz, (char *)data_ptr, pkt_len);
		if (ret < 0) {
			pr_err("%s: write obmi data error(%d)\n", __func__, ret);
			goto out;
		}

		// retry the frame again if response error idx
		if (nz->req_idx != idx + 1)
			continue;

		// prepare next frame
		idx++;
		offset += len;
		len = fw->size - offset;
		if (len > IMG_PAYLOAD(pkt_len))
			len = IMG_PAYLOAD(pkt_len);
	}

{% endhighlight %}

<p class="paragraph">
数据内容帧包含帧头6字节，数据段，校验2字节，每次发送完成后CP都会递增idx返回，出错需要重新发送当前帧。OBMI数据帧长度为2044，CPIMG帧长度为16000，都包含帧头和校验开销。
</p>

## 第三次枚举及MUX通道

<p class="paragraph">
CPIMG下载成功后正常就会跳转运行，启动CP到正常可服务状态，通过中断触发第三次枚举，枚举完成后AP就能与CP传输AT命令了，CP也会去搜网等操作。由于AT与NVM等操作是在MUX通道上进行，在枚举完成后HSIC通道0会切换到MUX协议状态，多达可支持23个独立MUX通道，RIL、audio、NVM等都是在mux通道上运行，网络数据PS使用HSIC通道3，调试服务使用HSIC通道1，其余通道暂时保留不用。
</p>

<p class="paragraph">
MUX通道使用3GPP GSM-07.10协议，AP与CP侧在HSIC通道0上同时都会启动MUX协议，MUX是多路复用协议，在单一通道上可以虚拟多个独立的逻辑通道，各通道通讯不相干扰，关于MUX协议的细节后面还有章节讨论。
</p>


<p class="paragraph">
<span class="label label-important">PS:</span> CPIMG启动后的一次枚举，HSIC共有7个bulk in，7个bulk out端点，组成7对in/out，实现7个HSIC虚拟通道，MUX就是建立在虚拟通道0上。
</p>
