---
title: kendryte dvp
date: 2019-04-18 20:00:00
tags: [MCU, k210]
---

## 1.DVP

DVP 是摄像头接口模块，特性如下：
- 支持 DVP 接口的摄像头
- 支持 SCCB 协议配置摄像头寄存器
- 最大支持 640X480 及以下分辨率，每帧大小可配置
- 支持 YUV422 和 RGB565 格式的图像输入
- 支持图像同时输出到 KPU 和显示屏:
- 输出到 KPU 的格式可选 RGB888，或 YUV422 输入时的 Y 分量
- 输出到显示屏的格式为 RGB565
- 检测到一帧开始或一帧图像传输完成时可向 CPU 发送中断
- 板子上接的是 OV5640，框图和管脚定义如下：

{% asset_img "ov5604.png" 416 "ov5604" %}

管脚定义如下表所示

| 管脚名称 | 管脚类型	| 管脚描述                             |
|----------|:----------:|-------------------------------------:|
| SIO_C	   | 输入	    | SCCB 总线的时钟线，可类比 I2C 的 SCL |
| SIO_D	   | I/O		| SCCB 总线的数据线，可类比 I2C 的 SDA |
| RESET	   | 输入		| 系统复位管脚，低电平有效             |
| PWDN	   | 输入		| 掉电/省电模式，高电平有效            |
| HREF	   | 输出		| 行同步信号                           |
| VSYNC	   | 输出		| 帧同步信号                           |
| PCLK	   | 输出		| 像素同步时钟输出信号                 |
| XCLK	   | 输入		| 外部时钟输入端口，可接外部晶振       |
| Y2…Y9	   | 输出		| 像素数据输出端口                     |

DVP 外设的寄存器配置结构体如下

<pre class="themepre">
<span class="themespan">// 寄存器结构体定义</span>
typedef struct _dvp
{
    uint32_t dvp_cfg;
    uint32_t r_addr;     <span class="themespan">// AI 输出的地址</span>
    uint32_t g_addr;
    uint32_t b_addr;
    uint32_t cmos_cfg;
    uint32_t sccb_cfg;
    uint32_t sccb_ctl;
    uint32_t axi;
    uint32_t sts;
    uint32_t reverse;
    uint32_t rgb_addr;   <span class="themespan">// output 输出的地址设置</span>
} __attribute__((packed, aligned(4))) dvp_t;

<span class="themespan">// 操作寄存器的全局变量的定义</span>
#define DVP_BASE_ADDR       (0x50430000U)
volatile dvp_t* const dvp = (volatile dvp_t*)DVP_BASE_ADDR;
</pre>

相关的操作代码如下，根据 LCD 的需求来切换 dvp 数据放到哪个缓存，目的是保证 LCD 在输出的时候 dvp 不是正好输出到这个 buf。这里的显示的帧率不是按照输出的固定帧率来定的，而是根据 DVP 的输入帧率来决定 LCD 的输出帧率。当 DVP 接收一帧结束之后，LCD 显示 DVP 之前接收到的一帧数据。采用这种方式，是因为 LCD 输出模块是接的显示模组，而不需要内置 framebuffer 保持固定的刷新帧率（framebuffer 在显示模组的内部，帧率也是由显示模组自己控制），所以就只需要在能输出的时候输出。

<pre class="themepre">
volatile uint8_t g_dvp_finish_flag;
volatile uint8_t g_ram_mux;

<span class="themespan">// dvp 中断函数，原因有两个，一个是帧开始，一个是帧结束</span>
static int on_irq_dvp(void* ctx)
{
    <span class="themespan">// 获取一帧到目标地址之后的中断</span>
    if (dvp_get_interrupt(DVP_STS_FRAME_FINISH))
    {
        <span class="themespan">// 根据 g_ram_mux 的标志位来切换数据放到哪个 buf 中</span>
        /* switch gram */
        dvp_set_display_addr(g_ram_mux ? (uint32_t)g_lcd_gram0 : (uint32_t)g_lcd_gram1);

        dvp_clear_interrupt(DVP_STS_FRAME_FINISH);
        g_dvp_finish_flag = 1;
    }
    <span class="themespan">// 获取到一帧开始的中断，此时开始进行转换</span>
    else
    {
        dvp_start_convert();
        dvp_clear_interrupt(DVP_STS_FRAME_START);
    }

    return 0;
}

<span class="themespan">// 使用 DVP 的例子代码</span>
    dvp_init(16);
    dvp_enable_burst();
    dvp_set_output_enable(0, 1);   <span class="themespan">// 使能 AI 的输出和 output 的输出，其中 index0 是 AI，index1 是 output</span>
    dvp_set_output_enable(1, 1);
    dvp_set_image_format(DVP_CFG_RGB_FORMAT);
    dvp_set_image_size(320, 240);
    <span class="themespan">// 这个函数中会通过 dvp 的 sccb 设置 sensor 的寄存器</span>
    ov5640_init();
    dvp_set_ai_addr((uint32_t)0x40600000, (uint32_t)0x40612C00, (uint32_t)0x40625800);
    dvp_set_display_addr((uint32_t)g_lcd_gram0);
    dvp_config_interrupt(DVP_CFG_START_INT_ENABLE | DVP_CFG_FINISH_INT_ENABLE, 0);
    dvp_disable_auto();

    /* DVP interrupt config */
    <span class="themespan">// 注册 dvp 的中断处理函数</span>
    printf("DVP interrupt config\n");
    plic_set_priority(IRQN_DVP_INTERRUPT, 1);
    plic_irq_register(IRQN_DVP_INTERRUPT, on_irq_dvp, NULL);
    plic_irq_enable(IRQN_DVP_INTERRUPT);

    /* enable global interrupt */
    sysctl_enable_irq();
    /* enable global interrupt */
    sysctl_enable_irq();

    /* system start */
    printf("system start\n");
    g_ram_mux = 0;
    g_dvp_finish_flag = 0;
    dvp_clear_interrupt(DVP_STS_FRAME_START | DVP_STS_FRAME_FINISH);
    <span class="themespan">// 当前的 dvp 中断有两个原因，一个是帧开始，一个是帧结束</span>
    dvp_config_interrupt(DVP_CFG_START_INT_ENABLE | DVP_CFG_FINISH_INT_ENABLE, 1);

    while (1)
    {
        /* ai cal finish*/
        while (g_dvp_finish_flag == 0)
            ;
        g_dvp_finish_flag = 0;
        /* display pic*/
        <span class="themespan">// 比如开始 g_ram_mux 是 0，dvp 是写入 buf1 的。这里将 ram_mux 改为 1，是为了让 dvp 后面写 buf0，以便可以使用 buf1</span>
        g_ram_mux ^= 0x01;
        lcd_draw_picture(0, 0, 320, 240, g_ram_mux ? g_lcd_gram0 : g_lcd_gram1);
    }
</pre>

## 2.LCD

板子上用的是并行接口（8080 方式的数据读写方式，地址空间中有数据和寄存器）方式的 TFT LCD 屏幕，NT35310。使用的 k210 的接口是 SPI 8 线模式。一些函数的实现如下，基本也就是对寄存器和地址空间的操作。

<pre class="themepre">
void lcd_clear(uint16_t color)
{
    uint32_t data = ((uint32_t)color << 16) | (uint32_t)color;

    <span class="themespan">// 设置写入数据的地址</span>
    lcd_set_area(0, 0, lcd_ctl.width, lcd_ctl.height);
    <span class="themespan">// 写入数据</span>
    tft_fill_data(&data, LCD_X_MAX * LCD_Y_MAX / 2);
}

void lcd_draw_rectangle(uint16_t x1, uint16_t y1, uint16_t x2, uint16_t y2, uint16_t width, uint16_t color)
{
    uint32_t data_buf[640] = {0};
    uint32_t *p = data_buf;
    uint32_t data = color;
    uint32_t index = 0;

    data = (data << 16) | data;
    for (index = 0; index < 160 * width; index++)
        *p++ = data;

    lcd_set_area(x1, y1, x2, y1 + width - 1);
    tft_write_word(data_buf, ((x2 - x1 + 1) * width + 1) / 2, 0);
    lcd_set_area(x1, y2 - width + 1, x2, y2);
    tft_write_word(data_buf, ((x2 - x1 + 1) * width + 1) / 2, 0);
    lcd_set_area(x1, y1, x1 + width - 1, y2);
    tft_write_word(data_buf, ((y2 - y1 + 1) * width + 1) / 2, 0);
    lcd_set_area(x2 - width + 1, y1, x2, y2);
    tft_write_word(data_buf, ((y2 - y1 + 1) * width + 1) / 2, 0);
}

void lcd_draw_picture(uint16_t x1, uint16_t y1, uint16_t width, uint16_t height, uint32_t *ptr)
{
    lcd_set_area(x1, y1, x1 + width - 1, y1 + height - 1);
    tft_write_word(ptr, width * height / 2, lcd_ctl.mode ? 2 : 0);
}
</pre>