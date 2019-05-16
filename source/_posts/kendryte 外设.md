---
title: kendryte 外设
date: 2019-04-18 22:30:20
tags: [MCU, k210]
---

## 1.FPIOA

Field Programmable GPIO Array (FPIOA)，The FPIOA peripheral supports the following features:
- 8 IO with 256 functions
- Schmitt trigger
- Invert input and output
- Pull up and pull down
- Driving selector
- Static input and output
- Any IO of FPIOA have 256 functions, it is a IO-function matrix. All IO have default reset function, after reset, re-configure IO function is required.

48 个 IO 初始化的时候都有一个默认配置，每一个都可以被重新配置成 256 个功能中的一个。

寄存器的配置结构体如下，包含两个配置。一个是 48 个 IO，每个 IO 的功能配置的数组，其中可以配置这个 IO 用作什么功能，还有相应的供电和负载的方式。另外一个是 256 个功能，每个功能所使用的 IO 是否强制输入为特定值（这个有点难理解，目前是只有 SPI 是输入强制为 1 的）。程序中直接通过全局变量来操作寄存器。

<pre class="themepre">
typedef struct _fpioa_io_config
{
    uint32_t ch_sel : 8;
    /*!< Channel select from 256 input. */
    uint32_t ds : 4;
    /*!< Driving selector. */
    uint32_t oe_en : 1;
    /*!< Static output enable, will AND with OE_INV. */
    uint32_t oe_inv : 1;
    /*!< Invert output enable. */
    uint32_t do_sel : 1;
    /*!< Data output select: 0 for DO, 1 for OE. */
    uint32_t do_inv : 1;
    /*!< Invert the result of data output select (DO_SEL). */
    uint32_t pu : 1;
    /*!< Pull up enable. 0 for nothing, 1 for pull up. */
    uint32_t pd : 1;
    /*!< Pull down enable. 0 for nothing, 1 for pull down. */
    uint32_t resv0 : 1;
    /*!< Reserved bits. */
    uint32_t sl : 1;
    /*!< Slew rate control enable. */
    uint32_t ie_en : 1;
    /*!< Static input enable, will AND with IE_INV. */
    uint32_t ie_inv : 1;
    /*!< Invert input enable. */
    uint32_t di_inv : 1;
    /*!< Invert Data input. */
    uint32_t st : 1;
    /*!< Schmitt trigger. */
    uint32_t resv1 : 7;
    /*!< Reserved bits. */
    uint32_t pad_di : 1;
    /*!< Read current IO's data input. */
} __attribute__((packed, aligned(4))) fpioa_io_config_t;

typedef struct _fpioa_tie
{
    uint32_t en[FUNC_MAX / 32];       <span class="themespan">// 相应的功能是否强制输入为特定值</span>
    /*!< FPIOA GPIO multiplexer tie enable array */
    uint32_t val[FUNC_MAX / 32];
    /*!< FPIOA GPIO multiplexer tie value array */
} __attribute__((packed, aligned(4))) fpioa_tie_t;

typedef struct _fpioa
{
    fpioa_io_config_t io[FPIOA_NUM_IO];     <span class="themespan">// io pad 的配置</span>
    /*!< FPIOA GPIO multiplexer io array */
    fpioa_tie_t tie;                        <span class="themespan">// 功能对应的 io 是否强制输入</span>
    /*!< FPIOA GPIO multiplexer tie */
} __attribute__((packed, aligned(4))) fpioa_t;

<span class="themespan">// 程序中直接通过全局变量来操作寄存器</span>
volatile fpioa_t *const fpioa = (volatile fpioa_t *)FPIOA_BASE_ADDR;
</pre>

芯片初始化的时候是有默认的 IO 对应的功能配置的，驱动中在初始化的时候只是设置了每个功能的 tie，并没有重新去配置 IO 的功能。初始化代码如下，是将 function_config[i] 的 tie_en 和 tie_val 设置给相应的寄存器。虽然没有去配置 IO 的功能，但是功能是能在相应的寄存器中读到的。

<pre class="themepre">
int fpioa_init(void)
{
    int i = 0;

    /* Enable fpioa clock in system controller */
    sysctl_clock_enable(SYSCTL_CLOCK_FPIOA);

    /* Initialize tie */
    fpioa_tie_t tie = { 0 };

    /* Set tie enable and tie value */
    for (i = 0; i < FUNC_MAX; i++)
    {
        tie.en[i / 32] |= (function_config[i].tie_en << (i % 32));
        tie.val[i / 32] |= (function_config[i].tie_val << (i % 32));
    }

    /* Atomic write every 32bit register to fpioa function */
    for (i = 0; i < FUNC_MAX / 32; i++)
    {
        /* Set value before enable */
        fpioa->tie.val[i] = tie.val[i];
        fpioa->tie.en[i] = tie.en[i];
    }

    return 0;
}
</pre>

设置 IO 口的功能比较简单，就是通过查找一个 function 的表，这个表中有这个 function 应该有的 IO 口配置，然后把这些配置全部设置给 IO 寄存器就行了。注意的是，代码里面还检查了是否已经有 IO 口配置了这个功能，如果有的话，会先把以前的那个 IO 的功能设置为 reserved，然后再设置当前 IO，避免冲突。

<pre class="themepre">
<span class="themespan">// 功能列表的全局变量部分内容</span>
/* Function list */
static const fpioa_assign_t function_config[FUNC_MAX] =
{
    {
        .ch_sel  = FUNC_JTAG_TCLK,
        .ds      = 0x0,
        .oe_en   = 0,
        .oe_inv  = 0,
        .do_sel  = 0,
        .do_inv  = 0,
        .pu      = 0,
        .pd      = 0,
        .resv1   = 0,
        .sl      = 0,
        .ie_en   = 1,
        .ie_inv  = 0,
        .di_inv  = 0,
        .st      = 1,
        .tie_en  = 0,
        .tie_val = 0,
        .resv0   = 0,
        .pad_di  = 0
    },
    {
        .ch_sel  = FUNC_JTAG_TDI,
        .ds      = 0x0,
        .oe_en   = 0,
        .oe_inv  = 0,
        .do_sel  = 0,
        .do_inv  = 0,
        .pu      = 0,
        .pd      = 0,
        .resv1   = 0,
        .sl      = 0,
        .ie_en   = 1,
        .ie_inv  = 0,
        .di_inv  = 0,
        .st      = 1,
        .tie_en  = 0,
        .tie_val = 0,
        .resv0   = 0,
        .pad_di  = 0
    },

<span class="themespan">// 设置 IO 口功能的函数，其实就是直接把 function_config[] 中的内容赋值过来</span>
int fpioa_set_function_raw(int number, fpioa_function_t function)
{
    /* Check parameters */
    if (number < 0 || number >= FPIOA_NUM_IO || function < 0 || function >= FUNC_MAX)
        return -1;
    /* Atomic write register */
    fpioa->io[number] =(const fpioa_io_config_t)
    {
        .ch_sel = function_config[function].ch_sel,
        .ds     = function_config[function].ds,
        .oe_en  = function_config[function].oe_en,
        .oe_inv = function_config[function].oe_inv,
        .do_sel = function_config[function].do_sel,
        .do_inv = function_config[function].do_inv,
        .pu     = function_config[function].pu,
        .pd     = function_config[function].pd,
        .sl     = function_config[function].sl,
        .ie_en  = function_config[function].ie_en,
        .ie_inv = function_config[function].ie_inv,
        .di_inv = function_config[function].di_inv,
        .st     = function_config[function].st,
        /* resv and pad_di do not need initialization */
    };
    return 0;
}

int fpioa_set_function(int number, fpioa_function_t function)
{
    uint8_t index = 0;
    /* Check parameters */
    if (number < 0 || number >= FPIOA_NUM_IO || function < 0 || function >= FUNC_MAX)
        return -1;
    if (function == FUNC_RESV0)
    {
        fpioa_set_function_raw(number, FUNC_RESV0);
        return 0;
    }
    /* Compare all IO */
    <span class="themespan">// 查找所有的 IO，看是否已经有 IO 配置了这个功能，如果有的话，先把那些 IO 配置为 reserved 功能状态</span>
    <span class="themespan">// 这里上电已经默认有了功能的情况，这里还是去读取，是能够读到默认的 IO 功能配置的</span>
    for (index = 0; index < FPIOA_NUM_IO; index++)
    {
        if ((fpioa->io[index].ch_sel == function) && (index != number))
            fpioa_set_function_raw(index, FUNC_RESV0);
    }
    fpioa_set_function_raw(number, function);
    return 0;
}
</pre>