---
layout: post
title: "EXYNOS平台GPIO驱动分析"
description: "introduce about gpio driver"
category: linux 
tags: [linux, kernel, driver]
---
{% include heljoy/setup %}

一般SOC出于节约芯片面积等考虑会复用引脚，即同一个引脚具有多种功能（输入、输出、I2C等），通过寄存器控制引脚的具体参数。给PCB设计带来了更多的灵活性，这些引脚都称为GPIO(General Purpose Input/Output)。本来将GPIO控制寄存器地址映射到内存区，需要访问GPIO引脚的驱动通过读写寄存器就能达到获取或设置GPIO参数，但这样显然不符合linux的原则，且每个人都要知道寄存器的哪一位控制什么功能，于是为了统一管理系统中所有的GPIO引脚，出现了GPIO驱动这样的基础组件。

<!-- more start -->
GPIO是使用软件控制逻辑电信号，通常应用于嵌入式系统，每个GPIO代表某种BGA芯片的一个引脚，因此单个芯片一般会支持多组GPIO接口，每组又包含有多个引脚，跟具体的芯片型号相关(Documentation/gpio.txt)。下面以EXYNOS4412为例分析实现细节。

## GPIO硬件规范

SPEC文档上有详细的介绍，4412平台包含304个多功能输入输出引脚，其中有37个通用接口组和2个内存接口组，其引脚可以多种功能复用，根据PCB安排系统启动时配置引脚的具体功能。

4412平台包含4个GPIO控制器，或者说4个寄存器操作区间，每个区间包含数量不等的GPIO组，一个GPIO组使用一段连续的寄存器空间控制状态，同一组中不同的GPIO引脚使用寄存器的不同位控制。所有GPIO可分为两类，一类是alive的，一类是off alive的，alive的是指处理器休眠后仍然供电的引脚，可做为外部中断唤醒系统，off alive的休眠后处理器就不能保证状态了。

- Part 1：0x11400000，GPA0、GPA1、GPB、GPC0、GPC1、GPD0、GPD1、GPF0、GPF1、GPF2、GPF3、ETC1、GPJ0、GPJ0
- Part 2：0x11000000，K0、K1、K2、K3、L0、L1、L2、Y0、Y1、Y2、Y3、Y4、Y5、Y6、ETC0、ETC6、M0、M1、M2、M3、M4
- Part 3：0x03860000，Z
- Part 4：0x106E0000，V0、V1、ETC7、V2、V3、ETC8、V4

其中穿插有ETC这样的组，用于控制Xjxxx引脚，功能单一，基本不会用到。上面的这些GPIO组除GPX外都是标准的CON、DAT、PUD、DRV、CONPDN、PUDPDN这样6组寄存器位控制其中一个GPIO引脚，GPX少了休眠控制，关于这些寄存器的含义我们在下一小节专门介绍。

## GPIO控制寄存器

引脚复用就说明引脚有多种功能，但具体使用时只会固定在一种功能上，比如PCB上规定这两个引脚是用于I2C访问XXX芯片，软件上就只需要将引脚设置为I2C功能，如果是用于输出功能，就要设置为输出引脚，CON寄存器就干这事。**由于UART、I2C接口需要多个引脚，设置时不能只配置其中部分引脚**

- GPXXCON－－－－－－－功能配置，输出、输入、组成某种通讯接口的一个引脚、中断功能
- GPXXDAT－－－－－－－引脚电平状态，可读写
- GPXXPUD－－－－－－－引脚上下拉电阻设置，用于通讯接口时最好配置
- GPXXDRV－－－－－－－引脚驱动能力
- GPXXCONPND－－－－－－AP休眠时引脚功能配置，一般都配置为输出功能
- GPXXPUDPND－－－－－－AP休眠时引脚上下拉配置

每个寄存器占4字节，GPXXCON寄存器统一使用4位配置一个引脚的功能，单组里最多可以包含8个引脚。例如要配置GPC0(3)引脚，对应的为`GPC0CON[3]`，datasheet中相关配置如下：
{% highlight bash %}
                       0x0 = Input 
                       0x1 = Output 
                       0x2 = I2S_1_SDI 
GPC0CON[3]   15:12     0x3 = PCM_1_SIN 
                       0x4 = AC97SDI 
                       0x5 to 0xE = Reserved 
                       0xF = EXT_INT4[3] 
{% endhighlight %}
GPXXDAT寄存器使用一位表示一个引脚电平信息。GPXXPUD寄存器使用两位配置一个引脚，对应关系如下
{% highlight bash %}
                      0x0 = Disables Pull-up/Pull-down
                      0x1 = Enables Pull-down
GPC0PUD[3]    7:6     0x2 = Reserved
                      0x3 = Enables Pull-up
{% endhighlight %}
GPXXDRV寄存器同样使用两位配置一个引脚，表示引脚的驱动能力，值越大能力越强。对应关系如下
{% highlight bash %}
                      0x0 = 1x
                      0x1 = 3x
GPC0DRV[3]    7:6     0x2 = 2x
                      0x3 = 4x
{% endhighlight %}
GPXXCONPND寄存器也使用两位配置一个引脚，表示休眠后引脚的功能，对应关系如下
{% highlight bash %}
                      0x0 = Output 0
                      0x1 = Output 1
GPC0CONPND[3]   7:6   0x2 = Input
                      0x3 = Previous state
{% endhighlight %}
GPXXPUDPND与GPXXPUD的配置相同，只不过是表示休眠后的上下拉，不再重复。ETC引脚的寄存器只有上下拉与驱动能力配置寄存器，PCB板上并没有使用，但其寄存器分散GPIO地址范围，驱动编码地址应注意。对于没有GPIO驱动层的系统看到这里就可以了，直接读写寄存器位控制引脚，但是linux没这么做，于是有本文下面的部分。


## GPIO分组与编号

硬件上SOC包含的GPIO控制器由四个地址区间控制，每个区间包含有几个GPIO组，系统中对所有端口统一编号，使用组名和序号索引。注意系统中还可能其他芯片也提供GPIO功能引脚，这里没有列出来，但同样需要统一编号。
{% highlight c %}
/* 组内GPIO端口数 */
#define EXYNOS4_GPIO_A0_NR      (8)
#define EXYNOS4_GPIO_A1_NR      (6)
#define EXYNOS4_GPIO_B_NR       (8)
#define EXYNOS4_GPIO_C0_NR      (5)
#define EXYNOS4_GPIO_C1_NR      (5)
#define EXYNOS4_GPIO_D0_NR      (4)
...

/* 每组的启始编号，GPIO_SPACE为每组间的间隔，在内核config时设置 */
#define EXYNOS_GPIO_NEXT(__gpio) \
        ((__gpio##_START) + (__gpio##_NR) + CONFIG_S3C_GPIO_SPACE + 1)

enum exynos4_gpio_number {
        EXYNOS4_GPIO_A0_START           = 0,
        EXYNOS4_GPIO_A1_START           = EXYNOS_GPIO_NEXT(EXYNOS4_GPIO_A0),
        EXYNOS4_GPIO_B_START            = EXYNOS_GPIO_NEXT(EXYNOS4_GPIO_A1),
        EXYNOS4_GPIO_C0_START           = EXYNOS_GPIO_NEXT(EXYNOS4_GPIO_B),
...
}

/* 得到每组的起始编号后就可通过组名及组内序号确定单个端口的编号 */
#define EXYNOS4_GPA0(_nr)       (EXYNOS4_GPIO_A0_START + (_nr))
#define EXYNOS4_GPA1(_nr)       (EXYNOS4_GPIO_A1_START + (_nr))
#define EXYNOS4_GPB(_nr)        (EXYNOS4_GPIO_B_START + (_nr))
#define EXYNOS4_GPC0(_nr)       (EXYNOS4_GPIO_C0_START + (_nr))
#define EXYNOS4_GPC1(_nr)       (EXYNOS4_GPIO_C1_START + (_nr))
...
{% endhighlight %}
上面就是我们在驱动中经常使用GPIO编号了，留给GPIO驱动的问题就是如何将对每个GPIO端口的操作映射到相应的寄存器设置。

## 5. GPIO端口结构与操作接口

提到GPIO驱动，不得不先说其中最关键的几个结构。

芯片结构：其实只用于表示单个GPIO端口
{% highlight c %}
struct gpio_chip {
        const char              *label;
        struct device           *dev;
        struct module           *owner;

        int                     (*request)(struct gpio_chip *chip,
                                                unsigned offset);
        void                    (*free)(struct gpio_chip *chip,
                                                unsigned offset);

        int                     (*direction_input)(struct gpio_chip *chip,
                                                unsigned offset);
        int                     (*get)(struct gpio_chip *chip,
                                                unsigned offset);
        int                     (*direction_output)(struct gpio_chip *chip,
                                                unsigned offset, int value);
        int                     (*set_debounce)(struct gpio_chip *chip,
                                                unsigned offset, unsigned debounce);

        void                    (*set)(struct gpio_chip *chip,
                                                unsigned offset, int value);

        int                     (*to_irq)(struct gpio_chip *chip,
                                                unsigned offset);

        void                    (*dbg_show)(struct seq_file *s,
                                                struct gpio_chip *chip);
        int                     base;
        u16                     ngpio;
        const char              *const *names;
        unsigned                can_sleep:1;
        unsigned                exported:1;
};
{% endhighlight %}

GPIO驱动向系统引出了很多通用接口，其中的大部分会调用到上述结构的回调接口，具体到单个调用会先检查端口状态等。
{% highlight c %}
gpio_request---------------------->   chip->request
gpio_free------------------------->   chip->free
gpio_direction_input-------------->   chip->direction_input
gpio_direction_output------------->   chip->direction_output
gpio_get_value-------------------->   chip->get
gpio_set_value-------------------->   chip->set
gpio_to_irq----------------------->   chip->to_irq
{% endhighlight %}

系统全局GPIO端口数组，所有系统中可供配置的GPIO端口都必须添加到这个数组
{% highlight c %}
struct gpio_desc {
        struct gpio_chip        *chip;
        unsigned long           flags; /* 用于标识当前端口的状态：是否占用、输入输出、触发方式等 */
};
static struct gpio_desc gpio_desc[ARCH_NR_GPIOS];   /* 这个就是定义GPIO编号时指定的最大编号 */
{% endhighlight %}

## GPIO编号到物理地址映射

前面说到系统中GPIO编号是连续的，使用组名及组内序号索引，硬件上同组端口使用一组寄存器控制，所有GPIO控制寄存器区间分为四部分，每部分包含不同组数的端口。

定义各部分区间内GPIO组
{% highlight c %}
static struct samsung_gpio_chip exynos4212_gpios_1[] = {
        {
                .chip   = {
                        .base   = EXYNOS4_GPA0(0),
                        .ngpio  = EXYNOS4_GPIO_A0_NR,
                        .label  = "GPA0",
                },
        }, {
                .chip   = {
                        .base   = EXYNOS4_GPA1(0),
                        .ngpio  = EXYNOS4_GPIO_A1_NR,
                        .label  = "GPA1",
                },
        }, {
                .chip   = {
                        .base   = EXYNOS4_GPB(0),
                        .ngpio  = EXYNOS4_GPIO_B_NR,
                        .label  = "GPB",
                },
        }, {
...
}
{% endhighlight %}

结构`samsung_gpio_chip`是对gpio_chip的封装，包含组寄存器基址、组内引脚数、组名等信息。`exynos4212_gpios_1`数组包含映射到地址区间一的所有gpio组，因此定义GPIO编号时最好按硬件安排顺序设置。

在GPIO驱动初始化时`samsung_gpiolib_init`会添加chip到`gpio_desc`数组中。
{% highlight c %}
gpio_base1 = ioremap(EXYNOS4_PA_GPIO1, SZ_4K);     //寄存器区间一基址
if (gpio_base1 == NULL) {
        pr_err("unable to ioremap for gpio_base1\n");
        goto err_ioremap1;
}

chip = exynos4212_gpios_1;                  //区间一GPIO组
nr_chips = ARRAY_SIZE(exynos4212_gpios_1);

for (i = 0; i < nr_chips; i++, chip++) {    //设置组号与配置接口，后面使用的时候再回头看
        if (!chip->config) {
                chip->config = &exynos_gpio_cfg;
                chip->group = group++;
        }
}
/*添加区间一GPIO组到系统，4bit表示所有GPIO使用4位的GPXXCON设置端口功能*/
samsung_gpiolib_add_4bit_chips(exynos4212_gpios_1, nr_chips, gpio_base1);
{% endhighlight %}

前面讲到`samsung_gpio_chip`结构中每个chip表示一组，因此组号顺序递增很好理解，其是依照硬件安排编组，`exynos_gpio_cfg`接口在使用时再介绍，现在只需要了解每组`chip->config`都是调用这一接口，除非单独指定。

重点在最后那个函数，主要意思就是将区间一内所有GPIO组添加到系统`gpio_desc`结构，配置这些GPIO的控制寄存器基址为`gpio_base1`，再看看具体实现。
{% highlight c %}
static void __init samsung_gpiolib_add_4bit_chips(struct samsung_gpio_chip *chip,
                                                  int nr_chips, void __iomem *base)
{
        int i;

        for (i = 0 ; i < nr_chips; i++, chip++) {
		/* 设置每组的input/output接口，注意这里是chip->chip */
                chip->chip.direction_input = samsung_gpiolib_4bit_input;
		/* 驱动引出的gpio_request_XXX接口调用对应到这里 */
                chip->chip.direction_output = samsung_gpiolib_4bit_output;

                if (!chip->config)
                        chip->config = &samsung_gpio_cfgs[2];   //这个上面设置过，不用管
                if (!chip->pm)
                        chip->pm = __gpio_pm(&samsung_gpio_pm_4bit); //关于电源管理部分的
                if ((base != NULL) && (chip->base == NULL))
                        chip->base = base + ((i) * 0x20);      //GPIO组对应到控制寄存器地址

                samsung_gpiolib_add(chip);                  //还是三星自家的函数，需要确认
        }
}
{% endhighlight %}

上述base表示区间一基址，`chip->base`表示组起始寄存器地址，每组寄存器占用0x20长度空间，可以去datasheet确认一下。

区间一中`A0、A1、B、C0、C1、D0、D1`是顺序排列，偏移量从`0x00`到`0xC0`依次递增，每组占用`0x20`长度。`F0、F1、F2、F3`偏移量从`0x180`开始，每组占用`0x20`长度，`J0、J1`偏移量从`0x240`开始，每组占用`0x20`长度，其中还包含有`ETC1`，由于系统中没有使用，上面没有实现。

根据datasheet说明，需要对区间一中部分组的基址调整，因为有些组跳跃式增长，与前一组间隔大于0x20。
{% highlight c %}
                        pr_err("unable to ioremap for gpio_base1\n");
                        goto err_ioremap1;
                }

+               exynos4212_gpios_1[7].base = gpio_base1 + 0x180; /* GPF0 */
+               exynos4212_gpios_1[8].base = gpio_base1 + 0x1A0; /* GPF1 */
+               exynos4212_gpios_1[9].base = gpio_base1 + 0x1C0; /* GPF2 */
+               exynos4212_gpios_1[10].base = gpio_base1 + 0x1E0; /* GPF3 */
+               exynos4212_gpios_1[11].base = gpio_base1 + 0x240; /* GPJ0 */
+               exynos4212_gpios_1[12].base = gpio_base1 + 0x260; /* GPJ1 */

                chip = exynos4212_gpios_1;
                nr_chips = ARRAY_SIZE(exynos4212_gpios_1);
{% endhighlight %}

这样每组的`chip->base`就都为其控制寄存器基址了，`chip->chip.base`就是组编号，用同样的方法调整其余几组。接着前面的`samsung_gpiolib_add`函数。
{% highlight c %}
static void __init samsung_gpiolib_add(struct samsung_gpio_chip *chip)
{
        struct gpio_chip *gc = &chip->chip;
        int ret;

        BUG_ON(!chip->base);
        BUG_ON(!gc->label);
        BUG_ON(!gc->ngpio);

        spin_lock_init(&chip->lock);

        if (!gc->direction_input)
                gc->direction_input = samsung_gpiolib_2bit_input; //这两个在调用前已经赋值了
        if (!gc->direction_output)
                gc->direction_output = samsung_gpiolib_2bit_output;
        if (!gc->set)
                gc->set = samsung_gpiolib_set;    //set、get接口，对应gpio_set/get_value
        if (!gc->get)
                gc->get = samsung_gpiolib_get;

#ifdef CONFIG_PM                                  //电源管理相关，暂时还是不看
        if (chip->pm != NULL) {
                if (!chip->pm->save || !chip->pm->resume)
                        printk(KERN_ERR "gpio: %s has missing PM functions\n",
                               gc->label);
        } else
                printk(KERN_ERR "gpio: %s has no PM function\n", gc->label);
#endif

        /* gpiochip_add() prints own failure message on error. */
	//系统gpiolib中的函数，将chip添加到gpio_desc数组中，gc->ngpio为新加的数组项数
        ret = gpiochip_add(gc);      
        if (ret >= 0)
                s3c_gpiolib_track(chip);        //track相关，与驱动功能没关系
}
{% endhighlight %}

由于`gpio_desc`数组包含系统中所有可用GPIO端口，上述将gc安排在数组`gpio_desc[gc->base...gc->base+gc->ngpio]`，而`gc->base`就是组编号，因此直接通过GPXX(X)作为索引就能找到我们使用的是哪个组哪个端口，对应到哪个控制寄存器基址。

## GPIO操作接口到寄存器配置流程示例

映射关系正确，驱动的移植就完成大半了，为理清GPIO驱动流程下面给出一个简单的示例，以常用的简单接口`gpio_set_value`为例。
{% highlight c %}
#define gpio_set_value  __gpio_set_value         //直接使用底层接口

void __gpio_set_value(unsigned gpio, int value)
{
        struct gpio_chip        *chip;

        chip = gpio_to_chip(gpio);            //找到对应的chip结构，直接是gpio_desc[gpio]
        /* Should be using gpio_set_value_cansleep() */
        WARN_ON(chip->can_sleep);             //睡眠相关，新特性
        trace_gpio_value(gpio, 0, value);     //trace相关，不理
        if (test_bit(FLAG_OPEN_DRAIN,  &gpio_desc[gpio].flags))
                _gpio_set_open_drain_value(gpio, chip, value);
        else if (test_bit(FLAG_OPEN_SOURCE,  &gpio_desc[gpio].flags))
                _gpio_set_open_source_value(gpio, chip, value);
        else
                chip->set(chip, gpio - chip->base, value);    //上面两个很少能使用到，跳过
}
{% endhighlight %}

经过上面的分析，`chip->set`调用就为`samsung_gpiolib_set`，现在可以看看这个了。
{% highlight c %}
static void samsung_gpiolib_set(struct gpio_chip *chip,
                                unsigned offset, int value)
{
        struct samsung_gpio_chip *ourchip = to_samsung_gpio(chip);
        void __iomem *base = ourchip->base;                 //组控制寄存器基址，关键部分
        unsigned long flags;
        unsigned long dat;

        samsung_gpio_lock(ourchip, flags);

        dat = __raw_readl(base + 0x04);    //0x04为GPXXDAT寄存器相对基址的偏移
        dat &= ~(1 << offset);             //offset为端口在组内的序号，GPXXDAT中一位表示一个端口
        if (value)
                dat |= 1 << offset;        //根据value值设置需要写入的值
        __raw_writel(dat, base + 0x04);

        samsung_gpio_unlock(ourchip, flags);
}
{% endhighlight %}
看看注释应该就能理清流程，其他接口不再分析。

<br />
PS: GPIO还有配置中断的相关寄存器，属于SOC中断部分，这部分后面会有单独介绍。

<!-- more end -->
