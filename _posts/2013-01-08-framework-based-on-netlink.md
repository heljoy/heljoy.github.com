---
layout: post
title: "基于netlink组件的检测程序一般框架"
description: "how-to about programming with kernel generic netlink"
category: linux
tags: [linux, kernel, how-to]
---
{% include heljoy/setup %}

netlink是linux内核提供的一种用户空间与内核通讯基础组件，基于网络实现，可实现多播、单播、组播复杂的通讯功能。内核驱动建立服务端，用户程序通过socket绑定服务端，可发送消息与接收消息，实现监听、系统调用等功能。其中generic netlink(genetlink)是基于netlink机制实现的一种通用协议，可直接使用到一般用户程序环境中。

<!-- more start -->
本文实现的是一种通知机制，用户程序监听内核netlink创建的端口，不能通过端口发消息到内核。这种机制适用于报告系统硬件状态，如电池电量、温度，SIM卡移除等

## 1. NETLINK内核编码相关

本框架是基于netlink的一种通用协议generic netlink(genetlink)，并没有单独占用新的协议号，内核服务端部分注册genetlink接口、相关操作函数和指定数据传输格式。

注册接口：用于通知内核有一个family添加，可提供服务
{% highlight c %}
ret = genl_register_family(&detect_family);     //注册family
if (ret != 0)
        return ret;

ret = genl_register_ops(&detect_family, &user_pid_ops); //family相关的操作函数
if (ret != 0)
        goto unreg_fam;
{% endhighlight %}

接口数据结构定义：family类型对应的操作接口，可定义多个，但cmd字段不同
{% highlight c %}
static struct genl_ops user_pid_ops = {
        .cmd = 0x01,
        .flags = 0,
        .policy = user_msg_policy,
        .doit = set_user_pid,
        .dumpit = NULL,
};
{% endhighlight %}

操作接口用于响应消息，用户给内核发送命令时需指定命令号cmd，发送的内容格式为policy指定，其他字段程序中没用到。同一family可注册多个命令，不同命令对应各自的处理函数doit。
{% highlight c %}
static struct genl_family detect_family = {
        .id = GENL_ID_GENERATE,
        .hdrsize = 0,
        .name = "DETECT_USB",
        .version = 0x01,
        .maxattr = DETECT_A_MAX,
};
{% endhighlight %}
协议结构，使用genl接口的id统一为GENL_ID_GENERATE，name字段用于标识特定的family，用户程序通过比较该字段连接到family。
此处用于响应用户消息的接口只接收用户进程的pid，之后内核会将消息发送到该pid进程

内核消息发送接口：将指定消息发送到用户进程，消息是一个32位整数，消息的定义内核与用户程序要一致
{% highlight c %}
skb = genlmsg_new(size, GFP_KERNEL);    //申请发送数据缓冲
if (skb == NULL)
        goto end;

//使用family协议发送数据，填充协议头
msg_head = genlmsg_put(skb, 0, 0/*seq*/, &detect_family,
                                           0x01/*cmd*/);    
if (msg_head == NULL) {
        nlmsg_free(skb);
        goto end;
}

if (nla_put_u32(skb, DETECT_A_UINT32, event->event) < 0) {
        nlmsg_free(skb);
        goto end;
}

if (genlmsg_end(skb, msg_head) < 0) {
        nlmsg_free(skb);
        goto end;
}

genlmsg_unicast(&init_net, skb, g_detect->user_pid);
{% endhighlight %}

在建立socket连接后内核可随时向用户空间发送消息，用户程序调用recv接收。框架暂时没有使用多播、组播功能。

## 2. NETLINK用户程序框架

基于NETLINK通讯的用户程序类似SOCKET程序，都是创建socket,绑定端口号，发送和接收数据等操作。框架中用户守护进程阻塞接收内核消息，再调用消息处理函数分发消息。

创建socket并绑定： 创建一个netlink类型的socket
{% highlight c %}
struct sockaddr_nl local;

fd = socket(AF_NETLINK, SOCK_RAW, NETLINK_GENERIC);
if (fd < 0)
        return -1;

memset(&local, 0, sizeof(local));
local.nl_family = AF_NETLINK;
local.nl_groups = 0;
if (bind(fd, (struct sockaddr *)&local, sizeof(local)) < 0)
        goto error;

return fd;
{% endhighlight %}
创建NETLINK_GENERIC类型socket，绑定端口。

查找DETECT_USB服务端，这部分属于genetlink公用部分。
{% highlight c %}
/* Get family name */
family_req.n.nlmsg_type = GENL_ID_CTRL;
family_req.n.nlmsg_flags = NLM_F_REQUEST;
family_req.n.nlmsg_seq = 0;
family_req.n.nlmsg_pid = getpid();
family_req.n.nlmsg_len = NLMSG_LENGTH(GENL_HDRLEN);
family_req.g.cmd = CTRL_CMD_GETFAMILY;
family_req.g.version = 0x1;

na = (struct nlattr *)GENLMSG_DATA(&family_req);
na->nla_type = CTRL_ATTR_FAMILY_NAME;
na->nla_len = strlen("DETECT_USB") + 1 + NLA_HDRLEN;
strcpy(NLA_DATA(na), "DETECT_USB");

family_req.n.nlmsg_len += NLMSG_ALIGN(na->nla_len);

if (sendto_fd(sd, (char *)&family_req, family_req.n.nlmsg_len) < 0)
        return -1;
        
rep_len = recv(sd, &ans, sizeof(ans), 0);
if (rep_len < 0)
        return -1;

na = (struct nlattr *)GENLMSG_DATA(&ans);
/* step to next nlattr for family id */
na = (struct nlattr *)((char *)na + NLA_ALIGN(na->nla_len));
if (na->nla_type == CTRL_ATTR_FAMILY_ID)
        id = *(__u16 *) NLA_DATA(na);
{% endhighlight %}
这里查找使用的字符串必须与内核中注册接口结构中定义的字符串相同，用于绑定到我们注册的接口。


发送消息相关程序：用户程序初始化时运行一次，用于将自己的pid通知到内核
{% highlight c %}
/* Send command needed */
req.n.nlmsg_len = NLMSG_LENGTH(GENL_HDRLEN);
req.n.nlmsg_type = id;
req.n.nlmsg_flags = NLM_F_REQUEST;
req.n.nlmsg_seq = 0;
req.n.nlmsg_pid = getpid();
req.g.cmd = 1;

na = (struct nlattr *)GENLMSG_DATA(&req);
na->nla_type = 1;                        //DETECT_A_MSG，消息格式类型
snprintf(message, 63, "usb detect deamon setup with pid %d", getpid());
na->nla_len = 64 + NLA_HDRLEN;
memcpy(NLA_DATA(na), message, 64);
req.n.nlmsg_len += NLMSG_ALIGN(na->nla_len);

memset(&nladdr, 0, sizeof(nladdr));
nladdr.nl_family = AF_NETLINK;

sendto(sd, (char *)&req, req.n.nlmsg_len, 0, (struct sockaddr *)&nladdr, sizeof(nladdr));
{% endhighlight %}

接收消息相关接口：这里放在一个循环里来做，也可以用poll实现
{% highlight c %}
rep_len = recv(sd, &ans, sizeof(ans), 0);  //阻塞接收内核消息
if (ans.n.nlmsg_type == NLMSG_ERROR)
        return -1;

if (rep_len < 0)
        return -1;

if (!NLMSG_OK((&ans.n), rep_len))
        return -1;

na = (struct nlattr *)GENLMSG_DATA(&ans);   //验证正确后做消息解析。
{% endhighlight %}

## 总结

上面实现了基于genetlink内核组件的内核消息单播框架，并不是一个完成的应用程序，这也是我在做netlink程序时认为最主要的内容，其实将内核与用户进程互发消息调试通过之后的事情就不与netlink相关了，知道这个框架能提供些什么服务，应用程序也好做扩展。

<!-- more end -->
