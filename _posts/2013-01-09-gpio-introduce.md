---
layout: post
title: "EXYNOS平台GPIO驱动分析"
description: "introduce about gpio driver"
category: linux 
tags: [linux, kernel, driver]
---
{% include heljoy/setup %}

<p class="paragraph">
一般SOC出于节约芯片面积等考虑会复用引脚，即同一个引脚具有多种功能（输入、输出、I2C等），通过寄存器控制引脚的具体参数。给PCB设计带来了更多的灵活性，这些引脚都称为GPIO(General Purpose Input/Output)。本来将GPIO控制寄存器地址映射到内存区，需要访问GPIO引脚的驱动通过读写寄存器就能达到获取或设置GPIO参数，但这样显然不符合linux的原则，且每个人都要知道寄存器的哪一位控制什么功能，于是为了统一管理系统中所有的GPIO引脚，出现了GPIO驱动这样的基础组件。
</p>
<!-- more -->
<p class="paragraph">
GPIO是使用软件控制逻辑电信号，通常应用于嵌入式系统，每个GPIO代表某种BGA芯片的一个引脚，因此单个芯片一般会支持多组GPIO接口，每组又包含有多个引脚，跟具体的芯片型号相关。下面以EXYNOS4412为例分析实现细节。
</p>

## GPIO硬件规范

<p class="paragraph">
SPEC文档上有详细的介绍，4412平台包含304个多功能输入输出引脚，其中有37个通用接口组和2个内存接口组，其引脚可以多种功能复用，根据PCB安排系统启动时配置引脚的具体功能。4412平台包含4个GPIO控制器，或者说4个寄存器操作区间，每个区间包含数量不等的GPIO组，一个GPIO组使用一段连续的寄存器空间控制状态，同一组中不同的GPIO引脚使用寄存器的不同位控制。所有GPIO可分为两类，一类是alive的，一类是off alive的，alive的是指处理器休眠后仍然供电的引脚，可做为外部中断唤醒系统，off alive的休眠后处理器就不能保证状态了。
</p>

- Part 1：0x11400000，GPA0、GPA1、GPB、GPC0、GPC1、GPD0、GPD1、GPF0、GPF1、GPF2、GPF3、ETC1、GPJ0、GPJ0
- Part 2：0x11000000，K0、K1、K2、K3、L0、L1、L2、Y0、Y1、Y2、Y3、Y4、Y5、Y6、ETC0、ETC6、M0、M1、M2、M3、M4
- Part 3：0x03860000，Z
- Part 4：0x106E0000，V0、V1、ETC7、V2、V3、ETC8、V4

<p class="paragraph">
其中穿插有ETC这样的组，用于控制Xjxxx引脚，功能单一，基本不会用到。上面的这些GPIO组除GPX外都是标准的CON、DAT、PUD、DRV、CONPDN、PUDPDN这样6组寄存器位控制其中一个GPIO引脚，GPX少了休眠控制，关于这些寄存器的含义我们在下一小节专门介绍。
</p>

## GPIO控制寄存器

<p class="paragraph">
引脚复用就说明引脚有多种功能，但具体使用时只会固定在一种功能上，比如PCB上规定这两个引脚是用于I2C访问XXX芯片，软件上就只需要将引脚设置为I2C功能，如果是用于输出功能，就要设置为输出引脚，CON寄存器就干这事。<strong>由于UART、I2C接口需要多个引脚，设置时不能只配置其中部分引脚</strong>
</p>

#### GPXXCON－－－－－－－功能配置，输出、输入、组成某种通讯接口的一个引脚、中断功能

每个寄存器占4字节，GPXXCON寄存器统一使用4位配置一个引脚的功能，单组里最多可以配置8个引脚。例如要配置GPC0(3)引脚，对应的为GPC0CON\[3\]，表示GPC0CON寄存器的第4个引脚功能接口，为寄存器12到15位，datasheet中相关配置如下。

                           0x0 = Input 
                           0x1 = Output 
                           0x2 = I2S_1_SDI 
    GPC0CON[3]   15:12     0x3 = PCM_1_SIN 
                           0x4 = AC97SDI 
                           0x5 to 0xE = Reserved 
                           0xF = EXT_INT4[3]

#### GPXXDAT－－－－－－－引脚电平状态，可读写

GPXXDAT寄存器使用一位表示一个引脚电平信息，因此其32位中只使用了低8位，高位不可访问。

#### GPXXPUD－－－－－－－引脚上下拉电阻设置，用于通讯接口时最好配置

GPXXPUD寄存器使用两位配置一个引脚，设置内部上下拉电阻，阻值一般在10K左右，对应关系如下。

                          0x0 = Disables Pull-up/Pull-down
                          0x1 = Enables Pull-down
    GPC0PUD[3]    7:6     0x2 = Reserved
                          0x3 = Enables Pull-up
                      
#### GPXXDRV－－－－－－－引脚驱动能力

GPXXDRV寄存器同样使用两位配置一个引脚，表示引脚的驱动能力，值越大输出电流的能力越强，当然也更耗电。一般作为通讯接口时才设置，最好用示波器查看一下通讯时的波形来确定，对应关系如下。

                          0x0 = 1x
                          0x1 = 3x
    GPC0DRV[3]    7:6     0x2 = 2x
                          0x3 = 4x

#### GPXXCONPND－－－－－－AP休眠时引脚功能配置，一般都配置为输出功能

GPXXCONPND寄存器也使用两位配置一个引脚，表示休眠后引脚的功能状态。这里的设置与系统正常使用是没有关系的，主要用于休眠时通知外设和降低系统功耗，对应关系如下。里头的0x3状态比较有意思，保留休眠前引脚状态。

                          0x0 = Output 0
                          0x1 = Output 1
    GPC0CONPND[3]   7:6   0x2 = Input
                          0x3 = Previous state
                      
#### GPXXPUDPND－－－－－－AP休眠时引脚上下拉配置

GPXXPUDPND与上面的GPXXPUD配置一样，用两位设置一个引脚，对应关系也一样，只在休眠时生效。

<p class="paragraph">
ETC引脚的寄存器只有上下拉与驱动能力配置寄存器，PCB板上并没有使用，但其寄存器分散GPIO地址范围，驱动编码地址应注意。对于没有GPIO驱动层的系统看到这里就可以了，直接读写寄存器位控制引脚，但是linux没这么做，于是有本文下面的部分。
</p>
<p class="paragraph">
Linux内核提供统一的GPIO设置接口，包括输入输出、上下拉、驱动能力等等，驱动程序开发人员只需要使用系统预定义的GPIO编号为参数，就可以配置该引脚的所有功能，而不需要了解寄存器分布与对应关系等硬件细节，大大方便了开发与维护。下面详细介绍这种抽象功能的具体实现，这也是芯片级系统移植的基础。
</p>

## GPIO分组与编号

<p class="paragraph">
硬件上SOC包含的GPIO控制器由四个地址区间控制，每个区间包含有几个GPIO组，系统中对所有端口统一编号，使用组名和序号索引。注意系统中还可能其他芯片也提供GPIO功能引脚，这里没有列出来，但同样需要统一编号，且没有重复编号。
</p>

```
//how many GPIO pins in one GPIO array
#define EXYNOS4_GPIO_A0_NR      (8)
#define EXYNOS4_GPIO_A1_NR      (6)
#define EXYNOS4_GPIO_B_NR       (8)
#define EXYNOS4_GPIO_C0_NR      (5)
#define EXYNOS4_GPIO_C1_NR      (5)
#define EXYNOS4_GPIO_D0_NR      (4)

//we set CONFIG_S3C_GPIO_SPACE in kernel .config, that means hole between two array
#define EXYNOS_GPIO_NEXT(__gpio) \
        ((__gpio##_START) + (__gpio##_NR) + CONFIG_S3C_GPIO_SPACE + 1)

// Number of GPIO array, and NO. of the first GPIO pin in this array
enum exynos4_gpio_number {
        EXYNOS4_GPIO_A0_START           = 0,
        EXYNOS4_GPIO_A1_START           = EXYNOS_GPIO_NEXT(EXYNOS4_GPIO_A0),
        EXYNOS4_GPIO_B_START            = EXYNOS_GPIO_NEXT(EXYNOS4_GPIO_A1),
        EXYNOS4_GPIO_C0_START           = EXYNOS_GPIO_NEXT(EXYNOS4_GPIO_B),
...
}

// GPIO NO., it's unique in system for drivers
#define EXYNOS4_GPA0(_nr)       (EXYNOS4_GPIO_A0_START + (_nr))
#define EXYNOS4_GPA1(_nr)       (EXYNOS4_GPIO_A1_START + (_nr))
#define EXYNOS4_GPB(_nr)        (EXYNOS4_GPIO_B_START + (_nr))
#define EXYNOS4_GPC0(_nr)       (EXYNOS4_GPIO_C0_START + (_nr))
#define EXYNOS4_GPC1(_nr)       (EXYNOS4_GPIO_C1_START + (_nr))
```

<p class="paragraph">
上面就是我们在驱动中经常使用GPIO编号了，留给GPIO驱动的问题就是如何将对每个GPIO端口的操作映射到相应的寄存器设置。当然有时候项目组长为方便管理还会将GPx(n)定义成NFC_WAKE_GPIO等带明显含义的字符再供驱动人员使用，稍稍留意就行，原理与上面是一样的。
</p>

## GPIO端口结构与操作接口

<p class="paragraph">
底层GPIO驱动都是围绕几个GPIO结构操作，首先需要了解几个基本结构。
</p>

```
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
```

<p class="paragraph">
上述结构实例表示一个GPIO pin，系统中每个GPIO pin都有一个实例。驱动中使用的GPIO编号首先转换成GPIO组号与组内序号，每组GPIO操作有关的寄存器位数一致，一般都是上下拉两位，输入输出两位，组内单个pin使用组起始编号加偏移就可以得到具体操作地址。系统中可供使用的GPIO API最终都会映射到结构的回调接口，具体可以查看内核代码driver/gpio/gpiolib.c。
</p>

    gpio_request---------------------->   chip->request
    gpio_free------------------------->   chip->free
    gpio_direction_input-------------->   chip->direction_input
    gpio_direction_output------------->   chip->direction_output
    gpio_get_value-------------------->   chip->get
    gpio_set_value-------------------->   chip->set
    gpio_to_irq----------------------->   chip->to_irq
    
<p class="paragraph">
内核有专门的变量表示系统中所有可用的GPIO资源，数组的序号与我们之前定义的GPIO编号一致，GPIO驱动在系统启动加载时注册所有GPIO到该数组，注册时gpio_chip结构已经填充，包括组起始编号base，组内偏移以及回调接口。
</p>

```
struct gpio_desc {
        struct gpio_chip        *chip;
        unsigned long           flags; /* 用于标识当前端口的状态：是否占用、输入输出、触发方式等 */
};
static struct gpio_desc gpio_desc[ARCH_NR_GPIOS];   /* 这个就是定义GPIO编号时指定的最大编号 */
```

## GPIO编号到物理地址映射

<p class="paragraph">
前面说到系统中GPIO编号是连续的，驱动在使用时通常以GPx(n)方式使用组名及组内序号索引，而硬件上不同组端口使用不同的寄存器区间控制，每组的端口数也可能都不相同。因此当驱动使用GPIO编号设置GPIO功能时，首先需要查找到GPIO所在组，同组内使用同一区间地址，再利用偏移与位宽就可以计算出该功能对应到的物理地址了。
</p>

```
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
```

<p class="paragraph">
结构<strong>samsung_gpio_chip</strong>是对gpio_chip的封装，包含组起始编号、组内端口数、组名等信息。<strong>exynos4212_gpios_1</strong>数组包含映射到地址区间一的所有gpio组，因此定义GPIO编号时最好按硬件安排顺序设置。在GPIO驱动初始化时samsung_gpiolib_init会添加chip到gpio_desc数组中。
</p>

```
gpio_base1 = ioremap(EXYNOS4_PA_GPIO1, SZ_4K);     //寄存器区间一基址

chip = exynos4212_gpios_1;                  //区间一GPIO组
nr_chips = ARRAY_SIZE(exynos4212_gpios_1);  //区间一内的组数

for (i = 0; i < nr_chips; i++, chip++) {    //设置组号与配置接口，后面使用的时候再回头看
        if (!chip->config) {
                chip->config = &exynos_gpio_cfg;
                chip->group = group++;
        }
}
/*添加区间一GPIO组到系统，4bit表示所有GPIO使用4位的GPXXCON设置端口功能*/
samsung_gpiolib_add_4bit_chips(exynos4212_gpios_1, nr_chips, gpio_base1);
```

<p class="paragraph">
前面讲到<strong>samsung_gpio_chip</strong>结构中每个chip表示一组，因此组号顺序递增很好理解，其是依照硬件安排编组，<strong>exynos_gpio_cfg</strong>接口在使用时再介绍，现在只需要了解每组<strong>chip->config</strong>都是调用这一接口，除非单独指定。重点在最后那个函数，主要意思就是将区间一内所有GPIO组添加到系统<strong>gpio_desc</strong>结构，配置这些GPIO的控制寄存器基址为gpio_base1，同一区间内一般每组GPIO占用的地址空间大小相同，再看看具体实现。
</p>

```
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
                        chip->base = base + ((i) * 0x20);      //GPIO组对应到控制寄存器地址，组间隔为0x20

                samsung_gpiolib_add(chip);                  //还是三星自家的函数，需要确认
        }
}
```

<p class="paragraph">
上述base表示区间一基址，<strong>chip->base</strong>表示组起始寄存器地址，每组寄存器占用0x20长度空间，这个需要结合硬件SPEC文档认真核对，如果组间隔大小有差异这里还需要手动调整。
</p>

<p class="paragraph">
区间一中'A0、A1、B、C0、C1、D0、D1'是顺序排列，偏移量从'0x00'到'0xC0'依次递增，每组占用'0x20'长度。'F0、F1、F2、F3'偏移量从'0x180'开始，每组占用'0x20'长度，'J0、J1'偏移量从'0x240'开始，每组占用`0x20`长度，其中还包含有'ETC1'，由于系统中没有使用，上面没有实现。组起始地址核对SPEC文档确认，下面手动调整了部分地址。
</p>

```
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
```

<p class="paragraph">
这样每组的chip->base就都为其控制寄存器基址了，chip->chip.base就是组编号，用同样的方法调整其余几组，使组起始地址与SPEC文档中描述的地址一致，这里不给出具体代码。再接着前面的samsung_gpiolib_add函数，这个函数太重要了，以至于我贴出了全部代码。
</p>
```
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
```
<p class="paragraph">
由于<strong>gpio_desc</strong>数组包含系统中所有可用GPIO端口，上述将gc安排在数组<strong>gpio_desc[gc->base...gc->base+gc->ngpio]</strong>，而<strong>gc->base</strong>就是组编号，因此直接通过GPXX(X)作为索引就能找到对应的数组项，数组项里保存了组寄存器起始地址以及组内序号，就很容易计算出具体pin的操作地址了。
</p>

## GPIO操作接口到寄存器配置流程示例

<p class="paragraph">
映射关系正确，底层GPIO驱动的移植基本完成了，为理清GPIO驱动流程下面给出一个简单的示例，以常用的简单接口gpio_set_value为例说明各接口以及回调接口的关系。
</p>

```
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
```
<p class="paragraph">
有了上面的基础，可以很容易看出chip->set调用了samsung_gpiolib_set。
</p>

```
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
```
<p class="paragraph">
这样就将对GPIO编号的操作对应到具体寄存器的设置了，用户并不需要关心完成这个设置都要操作哪个或哪几个寄存器，Linux的中驱动的分层抽象就是这么厉害，很多部分都是类似框架。
</p>

<p class="paragraph">
<span class="label label-important">PS:</span> GPIO还有配置中断的相关寄存器，属于SOC中断部分，这部分后面有机会再单独介绍。
</p>