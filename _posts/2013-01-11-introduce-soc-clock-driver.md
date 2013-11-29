---
layout: post
title: "SOC时钟系统驱动分析"
description: "introduce soc core clock driver"
category: linux
tags: [linux, kernel, clock, driver]
---
{% include heljoy/setup %}

本文主要介绍SOC片上时钟系统，对应到SPEC文档时钟部分，并不涉及系统的计时、定时等，这部分内容可参考[这篇文章](http://elinux.org/Kernel_Timer_Systems)。本文所提到的时钟是SOC片上设备工作的时钟，主要介绍时钟相关概念、硬件、核心时钟系统的结构与驱动、部分动态调频的原理。

<!-- more start -->
时钟是设备工作的脉搏，系统各部件正常的工作离不开合适的时钟。嵌入式平台上为降低系统功耗，部分模块的工作电压与时钟都是可以动态调整，并且可单独关闭某一模块的时钟以节省电源，为屏蔽时钟调整寄存器操作细节，方便时钟状态调整，统一访问接口，三星独立与LINUX时钟驱动，开发了平台相关的驱动程序，这里针对EXYNOS4412平台时钟系统，涉及到驱动框架和实现细节。


## 时钟相关硬件

时钟是一种有规律的电信号，具有特定的周期、频率，最常见的时钟信号的方波、正弦波等，这个是硬件晶振具有的特性。而系统中各模块工作于不同频率，为每个模块单独配置一个晶振显然不现实，并且有些模块可工作在多种频率模式，因此嵌入式时钟系统常具有动态调整频率功能，具体来说就是多路选择和分频，其时钟系统与树结构类似，拥有一个或几个时钟源，经过`PLL`电路，使用`MUX`、`DIV`器件为各模块提供多种工作频率。

- `PLL`：锁相环电路，[百度百科](http://baike.baidu.com/view/188802.htm)上有详细的解释，具体就是有反馈机制的电路，可调整部分参数达到输入与输出动态变化的目的，在SPEC文档上有调整PLL相关的寄存器说明，其中最重要的参数为PMS，确定输出频率与输入频率的关系，并有参数的推荐配置。

- `MUX`：多路选择电路，从上行多路时钟选择一路输出，必须有一路输出，不能不选，也是通过寄存器配置。另外`MUX`分为glitch-free和normal两种，只在时钟切换时有区别。

- `DIV`：分频电路，将上行时钟分频处理后输出，经过电路的时钟频率为离散值

- `GATE`：开关电路，截断上行时钟到下行设备的输出，一般针对终端设备，不要关闭下行有多个时钟或设备的时钟

## 片上设备时钟

一般SOC芯片都需要外部提供一个或几个稳定频率的时钟源，时钟源经过片上时钟管理系统最后提供给片上的设备，这部分的硬件连接图一般没有，但SPEC文档会介绍时钟管理系统对时钟源的处理，片上设备连接到哪个时钟域下(一般不会详细列出每个设备的时钟路线，只会给出哪些设备属于哪个时钟管理)，以下给出大概的时钟路线图。

{% highlight bash %}
     other clock ----------+                              +----> {GATE} DEVICE0
     other clock --------+ |                              |----> {GATE} DEVICE1
                         V V                              |----> {GATE} DEVICE2
clock_src ------>[PLL] ------->[MUX] ------->[DIV] ------------> {GATE} DEVICE3
{% endhighlight %}

时钟源`clock_src`之后到`DEVICE`之前都属于时钟管理系统，从时钟源到设备通常会经过多个`MUX`、`DIV`器件，具体要结合SPEC文档时钟管理器部分寄存器描述。弄清楚上述关系后就可以直接访问寄存器控制时钟系统，适用于没有操作系统的环境，但linux系统屏蔽了这些硬件细节，为上层驱动提供统一的访问接口，下面具体描述统一接口到寄存器操作的映射关系，这也是时钟驱动框架的核心部分。

## 系统中时钟的表示

时钟驱动中关键要表达以下几个概念：时钟、时钟源、设置/获取频率等操作。
{% highlight c %}
## arch/arm/plat-samsung/include/plat/clock.h
struct clk {
        struct list_head      list;    //用这个链接到系统全局时钟链表
        struct module        *owner;   //时钟也是系统中的设备
        struct clk           *parent;  //指向父时钟，用于时钟操作等
        const char           *name;
        const char           *devname;
        int                   id;
        int                   usage;   //使用计数
        unsigned long         rate;    //频率
        unsigned long         ctrlbit; //寄存器中第几位用于操作时钟

        struct clk_ops       *ops;     //操作接口，set/get等
        int                 (*enable)(struct clk *, int enable); //使能接口，配合usage使用
        struct clk_lookup     lookup;         //Linux驱动中用于时钟索引相关，暂时没有关注
#if defined(CONFIG_PM_DEBUG) && defined(CONFIG_DEBUG_FS)
        struct dentry           *dent;  /* For visible tree hierarchy */
#endif
};
{% endhighlight %}

看了这个结构心里还不会太紧张，成员不多，且大分部类型确定，起作用的也只有几个，唯一一个不明白的就是clk_ops了。
{% highlight c %}
## arch/arm/plat-samsung/include/plat/clock.h
struct clk_ops {
        int                 (*set_rate)(struct clk *c, unsigned long rate);  //设置频率
        unsigned long       (*get_rate)(struct clk *c);                      //获取频率
        unsigned long       (*round_rate)(struct clk *c, unsigned long rate);//获取与给定频率最接近的支持频率
        int                 (*set_parent)(struct clk *c, struct clk *parent);//设置父时钟
        struct clk *        (*get_parent)(struct clk *c);                    //获取父时钟
}; 
{% endhighlight %}

这个结构也没有我们想像的复杂，需要解释的是时钟是由`MUX`、`DIV`等出来的，并不是你设置一个频率，时钟就能以这个频率工作，通过设置`MUX`、`DIV`，时钟有一个离散工作频率范围。`MUX`是多选一的选择器，每个上行时钟都可能成为下行时钟的父时钟，下行时钟与父时钟的频率一致，对于`DIV`分频器，下行时钟只有一个父时钟，下行时钟与父时钟频率关系由分频器控制。

{% highlight c %}
## arch/arm/plat-samsung/include/plat/clock-clksrc.h
struct clksrc_clk {     
        struct clk              clk;          //也是一个时钟，这个成员还是要的
        struct clksrc_sources   *sources;     //时钟列表

        struct clksrc_reg       reg_src;      //寄存器
        struct clksrc_reg       reg_src_stat; //寄存器
        struct clksrc_reg       reg_div;      //寄存器
};

struct clksrc_sources {
        unsigned int    nr_sources;
        struct clk      **sources;
};

struct clksrc_reg {
        void __iomem            *reg;
        unsigned short          shift;
        unsigned short          size;
};
{% endhighlight %}
时钟源的表示，用来描述`MUX`、`DIV`器件这类连接有多个时钟，通过寄存器位控制上下行时钟关系。`MUX`的`sources`一般包含多个时钟，寄存器`reg_src`表示设置哪个为父时钟的寄存器位，`DIV`不用设置`sources`，使用`reg_div`设置分频参数。这里寄存器信息只给出了基地址，偏移量和位宽，位段内包含的信息由SPEC文档解释，一般所有`MUX`的格式统一，所有`DIV`的格式也统一，故可以使用上面的抽象。


## 时钟注册接口

这里分两类注册，一种时钟就是单个模块使用，有些可以关闭，且关闭了不会影响其它模块工作；另一类就是生成上面一类时钟的时钟，由选择器、分频器组成，可以说是生成第一种时钟的时钟，为时钟源，对应到上节中的两个结构。
{% highlight c %}
## arch/arm/plat-samsung/clock.c
int s3c24xx_register_clock(struct clk *clk)
{
        if (clk->enable == NULL)
                clk->enable = clk_null_enable;

        /* add to the list of available clocks */

        /* Quick check to see if this clock has already been registered. */
        BUG_ON(clk->list.prev != clk->list.next);

        spin_lock(&clocks_lock);
        list_add(&clk->list, &clocks);       //添加到全局时钟链表
        spin_unlock(&clocks_lock);

        /* fill up the clk_lookup structure and register it*/
        clk->lookup.dev_id = clk->devname;
        clk->lookup.con_id = clk->name;
        clk->lookup.clk = clk;
        clkdev_add(&clk->lookup);              //与时钟查找相关

        return 0;
}
{% endhighlight %}
注册时钟的接口比较简单，把时钟添加到全局时钟链表就可以了，与gpio驱动中gpio_desc数组有类似的作用，相比第二类时钟要复杂一些。

{% highlight c %}
void __init s3c_register_clksrc(struct clksrc_clk *clksrc, int size)
{
        int ret;        
        
        for (; size > 0; size--, clksrc++) {
                if (!clksrc->reg_div.reg && !clksrc->reg_src.reg)
                        printk(KERN_ERR "%s: clock %s has no registers set\n",
                               __func__, clksrc->clk.name);
        
                /* fill in the default functions */
        
                if (!clksrc->clk.ops) {                  //选择器、分频器操作寄存器接口
                        if (!clksrc->reg_div.reg)
                                clksrc->clk.ops = &clksrc_ops_nodiv;
                        else if (!clksrc->reg_src.reg)
                                clksrc->clk.ops = &clksrc_ops_nosrc;
                        else
                                clksrc->clk.ops = &clksrc_ops;
                }
        
                /* setup the clocksource, but do not announce it
                 * as it may be re-set by the setup routines
                 * called after the rest of the clocks have been
                 * registered
                 */
                s3c_set_clksrc(clksrc, false);                       //初始化时钟源
        
                ret = s3c24xx_register_clock(&clksrc->clk);        //将时钟源的时钟注册进系统

                if (ret < 0) {
                        printk(KERN_ERR "%s: failed to register %s (%d)\n",
                               __func__, clksrc->clk.name, ret);
                }
        }
}
{% endhighlight %}

注册时钟源要有选择或分频寄存器地址，也就是这个时钟源是可配置的，接口配置时钟源时主要是根据clk_reg设置寄存器，后面应用时再具体介绍。时钟源添加到系统时会进行初始化，这里涉及到寄存器操作。
{% highlight c %}
void __init_or_cpufreq s3c_set_clksrc(struct clksrc_clk *clk, bool announce)
{
        struct clksrc_sources *srcs = clk->sources;
        u32 mask = bit_mask(clk->reg_src.shift, clk->reg_src.size);
        u32 clksrc;

        if (!clk->reg_src.reg) {
                if (!clk->clk.parent)
                        printk(KERN_ERR "%s: no parent clock specified\n",
                                clk->clk.name);
                return;
        }

        clksrc = __raw_readl(clk->reg_src.reg);         //选择器开关状态
        clksrc &= mask;
        clksrc >>= clk->reg_src.shift;

        if (clksrc > srcs->nr_sources || !srcs->sources[clksrc]) {
                printk(KERN_ERR "%s: bad source %d\n",
                       clk->clk.name, clksrc);
                return;
        }

        clk->clk.parent = srcs->sources[clksrc];      //设置当前时钟的父结点

        if (announce)
                printk(KERN_INFO "%s: source is %s (%d), rate is %ld\n",
                       clk->clk.name, clk->clk.parent->name, clksrc,
                       clk_get_rate(&clk->clk));
}
{% endhighlight %}
初始化主要是设置时钟的父结点，利用选择器寄存器reg_src，从父结点列表中获取当前为选择器提供时钟的父结点，分频率只有一个父结点。

## 时钟驱动框架结构

EXYNOS4412平台所有的时钟都从以下几个时钟派生：

- XRTCXTI：实时钟输入，32.768KHZ，用于实时钟模块
- XXTI：不知道做什么用的，我们没有使用这个
- XUSBXTI：USB PHY模块使用，同时也用于系统PLL，24MHZ

主要是使用到了XUSBXTI作为时钟来源，为保证系统工作频率稳定，XUSBXTI分别输入到不同的PLL电路，经常见到的有APLL、EPLL、MPLL、VPLL，其主要针对SOC四大部分，XUSBXTI与PLL经过`MUX`产生SCLK_XPLL，其中APLL要分频输出SCLK_APLL，具体派生结构如下：
{% highlight bash %}
                              |\               +-----------+
xusbxit--+------------------->| | MUX(APLL)    |           |
         |   +--------+       | |------------->| DIV(APLL) |----------->SCLK_APLL
         +-->|  APLL  |------>| |              |           |
         |   +--------+       |/               +-----------+
         |   
         |
         |                    |\
         +------------------->| |  MUX(xPLL)            
         |   +------------+   | |-------------> SCLK_xPLL
         +-->| PLL(E,M,V) |-->| |              
             +------------+   | |
                              |/
         
{% endhighlight %}

上述5个时钟输入源，分别是xusbxit、apll_fout、epll_fout、mpll_fout、vpll_fout，根据上图建立驱动使用的模型：
{% highlight c %}
## arch/arm/plat-s5p/clock.c
/* Possible clock sources for APLL Mux */
static struct clk *clk_src_apll_list[] = {
        [0] = &clk_fin_apll,         //输入APLL前的时钟，也就是xusbxit
        [1] = &clk_fout_apll,        //从APLL输出的时钟
};
struct clksrc_sources clk_src_apll = {
        .sources        = clk_src_apll_list,
        .nr_sources     = ARRAY_SIZE(clk_src_apll_list),
};

## arch/arm/mach-exynos/clock-exynos4.c
static struct clksrc_clk exynos4_clk_mout_apll = {              //这个描述APLL后面的MUX
        .clk    = {
                .name           = "mout_apll",
        },
        .sources = &clk_src_apll,                                         //这个MUX的源时钟列表
        .reg_src = { .reg = EXYNOS4_CLKSRC_CPU, .shift = 0, .size = 1 },  //用于控制MUX寄存器位
};
static struct clksrc_clk exynos4_clk_sclk_apll = {             //这个描述MUX后面的DIV
        .clk    = {
                .name           = "sclk_apll",
                .parent         = &exynos4_clk_mout_apll.clk,   //这个的父结点就是MUX出来的时钟
        },
        .reg_div = { .reg = EXYNOS4_CLKDIV_CPU, .shift = 24, .size = 3 },    //用于设置分频器寄存器位           
};

## arch/arm/plat-s5p/clock.c
/* Possible clock sources for EPLL Mux */
static struct clk *clk_src_epll_list[] = {
        [0] = &clk_fin_epll,
        [1] = &clk_fout_epll,
};
struct clksrc_sources clk_src_epll = {
        .sources        = clk_src_epll_list,
        .nr_sources     = ARRAY_SIZE(clk_src_epll_list),
};

## arch/arm/mach-exynos/clock-exynos4.c
static struct clksrc_clk exynos4_clk_mout_epll = {
        .clk    = {
                .name           = "mout_epll",
        },
        .sources = &clk_src_epll,
        .reg_src = { .reg = EXYNOS4_CLKSRC_TOP0, .shift = 4, .size = 1 },
};

...
{% endhighlight %}

上面给出了建立sclk_apll和sclk_epll(mout_epll)时钟模型，其余PLL时钟模型相似。SOC上很多设备使用的时钟都是这几个PLL派生来，结合芯片datasheet，从代码中能清楚看到这种派生关系。

## 4. 时钟驱动接口实现

对时钟的操作首先都要获取时钟，由`get_clk`实现，由linux系统clk驱动完成，使用lookup成员，细节不作介绍。获取时钟后就可设置时钟频率或关闭时钟了。
{% highlight c %}
## drivers/clk/clk.c
unsigned long clk_get_rate(struct clk *clk)
{
        unsigned long rate;

        mutex_lock(&prepare_lock);
        rate = __clk_get_rate(clk);
        mutex_unlock(&prepare_lock);

        return rate;
}
unsigned long __clk_get_rate(struct clk *clk)
{
        unsigned long ret;

        if (!clk) {
                ret = -EINVAL;
                goto out;
        }

        ret = clk->rate;

        if (clk->flags & CLK_IS_ROOT)
                goto out;

        if (!clk->parent)
                ret = -ENODEV;

out:
        return ret;
}
{% endhighlight %}

获取时钟频率比较简单，在clk结构中有保存时钟频率。设置频率的接口要复杂得多，如果操作的为`DIV`，根据父时钟频率调整分频参数，找出一个与目标频率最接近的分频参数即可，如果操作的是`MUX`，可选的频率集合为父时钟频率集合，查找最接近即可。

{% highlight c %}
## arch/arm/plat-samsung/clock.c
int clk_set_rate(struct clk *clk, unsigned long rate)
{
        int ret;
        unsigned long flags;

        if (IS_ERR(clk))
                return -EINVAL;

        /* We do not default just do a clk->rate = rate as
         * the clock may have been made this way by choice.
         */

        WARN_ON(clk->ops == NULL);
        WARN_ON(clk->ops && clk->ops->set_rate == NULL);

        if (clk->ops == NULL || clk->ops->set_rate == NULL)
                return -EINVAL;

        spin_lock_irqsave(&clocks_lock, flags);
        trace_clock_set_rate(clk->name, rate, smp_processor_id());   //trace相关，不用关心
        ret = (clk->ops->set_rate)(clk, rate);                      //接口调用
        spin_unlock_irqrestore(&clocks_lock, flags);

        return ret;
}
{% endhighlight %}

这里就是调用clk设备的操作接口，第一节介绍注册时赋值。（`DIV`器件才有改变频率接口，`MUX`器件才有改变父结点接口，clk设备只有使能接口），时钟操作接口，结合时钟源的定义可关联到具体接口。
{% highlight c %}
static struct clk_ops clksrc_ops = {
        .set_parent     = s3c_setparent_clksrc,
        .get_parent     = s3c_getparent_clksrc,
        .get_rate       = s3c_getrate_clksrc,
        .set_rate       = s3c_setrate_clksrc,
        .round_rate     = s3c_roundrate_clksrc,
};
        
static struct clk_ops clksrc_ops_nodiv = {
        .set_parent     = s3c_setparent_clksrc,
        .get_parent     = s3c_getparent_clksrc,
};

static struct clk_ops clksrc_ops_nosrc = {
        .get_rate       = s3c_getrate_clksrc,
        .set_rate       = s3c_setrate_clksrc,
        .round_rate     = s3c_roundrate_clksrc,
};
{% endhighlight %}
改变时钟的频率clk_set_rate接口会调用到s3c_setrate_clksrc。

{% highlight c %}
static int s3c_setrate_clksrc(struct clk *clk, unsigned long rate)
{
        struct clksrc_clk *sclk = to_clksrc(clk);
        struct clk *parent;
        void __iomem *reg = sclk->reg_div.reg;
        unsigned int div;
        u32 mask = bit_mask(sclk->reg_div.shift, sclk->reg_div.size);
        u32 val;

        parent = __clk_get_parent(clk);     //获取父结点

        rate = clk_round_rate(clk, rate);
        div = clk_get_rate(parent) / rate;  //目标频率与当前父结点频率计算分频参数
        if (div > (1 << sclk->reg_div.size))
                return -EINVAL;

        val = __raw_readl(reg);
        val &= ~mask;
        val |= (div - 1) << sclk->reg_div.shift;
        __raw_writel(val, reg);   //修改reg_div[shift~shift+size]，定义时钟源时设置

        return 0;
}
{% endhighlight %}

其余接口不再介绍，利用上面的方法，分析datasheet文档，核对移植过程中是否存在问题。

## 总结

这样基础性质的驱动是系统能正常工作的保证，修改时需要认真核对SPEC文档，在移植前并不是所有的时钟都已经正确，最好是代码与文档对照来看，去掉没有的时钟，添加必须使用的时钟。有时候很多时钟的寄存器从来不会被改动，可以不用添加到系统，但为框架树形结构完整以及查找问题方便，还是建议统一都写上，从clock_tree能完整的掌握系统运行状态。

<!-- more end -->
