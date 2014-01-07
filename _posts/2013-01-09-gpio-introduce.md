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
