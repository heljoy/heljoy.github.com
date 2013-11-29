---
layout: post
title: "Android Recovery添加触摸支持"
description: "touch input support to recovery, and classes it like key input"
category: android
tags: [android, recovery]
---
{% include heljoy/setup %}

Android系统的Recovery模式是独立于正常使用系统的另一种操作环境，其使用精简的内核与工具集，不提供控制台SHELL环境，仅运行recovery可执行程序完成升级、恢复出厂设置等任务。为提高用户体验，我们在Recovery模式下增加了触摸功能，配合单选框、按钮图标的显示，实现触摸事件驱动的Recovery图形界面环境，取消按键菜单项模式。关于Recovery模式的详细介绍可以参考[这里](http://blog.csdn.net/tronteng/article/details/7590326)转载的文件。

<!-- more start -->

原生Android系统Recovery模式只支持按键消息，显示界面显示当前可供选择的操作项，使用音量加减键移动选取项，电源键确认选取。我们将升级操作全部使用图形界面完成，单选框选取任务，按钮确认或取消操作，进度条显示操作进度，并提供错误处理界面，界面的各元素都使用图标文件显示，比如单选框选取与未选取状态加载不同的图标文件。本文并不涉及到很多图形显示的细节问题，只介绍在Recovery模式下添加触摸消息支持，包括触摸点到单选框的选取，触摸滑动时按钮的效果以及触摸消息到按钮消息等内容，适合基于触摸驱动的升级模式两次开发。


## 添加内核触摸事件接收支持

如果用于recovery模式的内核不提供触摸驱动支持，下面是没办法继续进行的。原生系统在bootable/recovery/minui目录的events.c文件中实现按键消息获取分发操作，使用poll方式等待所有按键消息，有按键消息时回调通知处理程序，具体的代码不做展示。添加触摸事件只需要在初始化时额外添加触摸输入设备到poll等待的设备池，这样有触摸消息时与按键一样回调通知处理，[这篇文章](http://blog.csdn.net/fe421504975/article/details/8272126)介绍得比较详细，但其并不适应我们的需求。

{% highlight c %}
int ev_init(ev_callback input_cb, void *data)
{
...
+    ioctl(fd, EVIOCGBIT(EV_ABS, sizeof(abs_ev_bits)), abs_ev_bits);
+    ioctl(fd, EVIOCGBIT(0, sizeof(key_ev_bits)), key_ev_bits);

+    if (test_bit(ABS_MT_POSITION_X, abs_ev_bits)
+                    && test_bit(ABS_MT_POSITION_Y, abs_ev_bits)) {
+          //TODO: get touch x.y and something
+    }
...
}
{% endhighlight %}
上述将触摸设备添加到监控数组中，同时还需要获取触摸的边界值，可辅助触摸坐标到LCD显示坐标转换。

## 添加触摸消息处理虚函数

谷歌在recovery中使用一个抽象基类实现按键消息的接收，其单独创建一个消息接收线程，使用minui中的辅助函数实现消息队列功能，应用只需要调用类成员waitkey即可阻塞等待按键消息，屏蔽了底层设备消息读取与队列管理等。添加触摸输入设备支持后ev_get_input会读取到触摸消息，在基类的消息回调函数中添加`EV_SYN`和`EV_ABS`支持，由于从触摸事件中只能获取坐标点等信息，要转换为按钮消息需要图形界面信息，并且在按钮上滑动需要更换图标文件以显示不同的按钮状态，因此使用虚函数的方式，在其子类中有足够的坐标、绘图功能时再实现触摸消息的处理。

{% highlight c %}
int RecoveryUI::input_callback(int fd, short revents, void* data)
{
...
    switch (ev.type) {
    case EV_ABS:
    case EV_SYN:
         ret = self->handle_touch_input(&ev);
         if (ret < PRESS_MAX)
             self->process_key(ret, 0);
         break;
    ...
    }
...
}
{% endhighlight %}
有触摸消息时调用触摸处理函数，其结合界面元素与触摸动作，如果是正确的按钮点击则返回按钮消息，与按键流程相同进入队列返回给等待消息的应用。提供单选框是否选取状态接口用于开始操作前获取任务列表。

{% highlight c %}
class RecoveryUI {
...
    virtual int handle_touch_input(struct input_event *ev);
    virtual bool getButtonStatus(int btn);
...
}
{% endhighlight %}


## 触摸消息处理

内核关于多点触摸支持两种[协议](http://www.kernel.org/doc/Documentation/input/multi-touch-protocol.txt)，一次触摸事件会上报多次消息，包括X坐标、Y坐标、多点触摸ID、触摸面宽度、压力等，每次报告完一组数据会追加一个SYN同步消息。这里需要反复调试，而且A、B两种协议上报消息方式不一样，处理也有差异，下面以伪代码的方式给出主要处理思路。

{% highlight c %}
int ScreenRecoveryUI::handle_touch_input(struct input_event *ev)
{
    // 过滤正在处理等界面触摸消息
    判断当前显示的用户界面是否支持触摸消息，不支持则返回

    判断消息类型，如果是EV_ABS消息：

       读取消息中包含的坐标信息，
       使用静态变量标记当前是否为释放消息 

    如果是EV_SYN消息：

       检查当前收集到的坐标点属于哪个单选框或按钮

       如果是坐标点在某个图标上并且是这次触摸的第一点：

            图标状态标记取反，并绘图，
            设置触摸第一点标记假

       如果当前为释放消息：
            
            消除坐标记录，设置触摸第一点标记真
            如果当前按钮为按下状态，设置按钮未按下并绘图，返回按钮消息

       如果以上都不是（应该是滑动状态）：

            滑动点滑出按钮区域内时加载按钮未被选取图标，重绘按钮
}
{% endhighlight %}
<br />
**PS:在`screen_ui`类参照进度条实现绘制背景、按钮等操作接口** 

<!-- more end -->
