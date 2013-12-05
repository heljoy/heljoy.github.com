---
layout: post
title: "跨平台共享鼠标键盘工具"
description: "geek tools to share your mouse and keyboard between different platforms"
category: tools
tags: [geek, tools, synergy]
---
{% include heljoy/setup %}

<p class="paragraph">
最近在网上发现了个比较好玩的工具synergy，感觉自己out了。由于我使用两台PC，公司的主力机和自己的本本，经常在两个机器前面转来转去有点麻烦，synergy这个工具正好能解决我的问题，在两个PC上无缝地切换键鼠设备，这样只需要把两PC的显示器摆在一起就可以同时操作了。Cool!
</p>

<!-- more -->

<p class="paragraph">
由于我的都是使用ubuntu系统，正好在系统的软件中心都有quicksynergy软件包，这是一个图形方式配置synergy的工具，下面也只记录下简单的配置过程。
</p>

## 服务端配置

<p class="paragraph">
quicksynergy软件同时支持客户端与服务端，服务端只需要配置Share选项页，设置Above/Left/Below/Right中的一个或几个，我只需要控制另一PC，且放在右边，故在右框中填上待控制PC的hostname。
</p>

{% highlight bash %}
# hostname
{% endhighlight %}

<p class="paragraph">
在客户端机器上执行以上命令，填到服务端Right框中，同时Execute并最小化窗口。
</p>

## 客户端配置

<p class="paragraph">
客户端在USE选项页正确填上服务端的IP地址。
</p>

{% highlight bash %}
ifconfig | grep 'inet addr' | cut -d ':' -f2 | awk '{print $1}'
{% endhighlight %}

<p class="paragraph">
Fill the right IP address and execute. OK, then you will share your mouse and keyboard between dispalys.
</p>

<p class="paragraph">
<span class="label label-important">skill:</span> this can also share your clipboard, Good luck!
</p>
