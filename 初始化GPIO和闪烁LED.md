# </center>初始化GPIO和闪烁LED<center>

## 一.流程

1.通过硬件原理图看接到使用的LED是哪个GPIO（在soc开发板使用GPIO53）

2.配置GPIO53为输出

3.进入闪烁循环，每次通过寄存器切换GPIO53电平后延时一段时间。



## 二.代码

```c
/* GPIO1寄存器组基地址 */
#define HPS_GPIO1_BASE_ADDR		0xff709000
/* GPIO1寄存器组基配置寄存器 */
#define HPS_GPIO_DIR_OFFSET		0x4
/* GPIO1寄存器组基数据寄存器 */
#define HPS_GPIO_DAT_OFFSET		0x0

/* 延时 */
static void delay_yeshen(void)
{
	volatile u32 i = 0x3FFFFF;
	while(i>0) i--;
}

/* 初始化GPIO */
static void gpio_init(void)
{
	u32 reg;
	reg =  readl(HPS_GPIO1_BASE_ADDR + HPS_GPIO_DIR_OFFSET);		
	reg |= 0x1<<24;
	writel(reg, HPS_GPIO1_BASE_ADDR + HPS_GPIO_DIR_OFFSET);		//gpio53配置为输出

	reg = readl(HPS_GPIO1_BASE_ADDR + HPS_GPIO_DAT_OFFSET);
	reg |= 0x1<<24;	
	writel(reg, HPS_GPIO1_BASE_ADDR + HPS_GPIO_DAT_OFFSET);		//gpio53初始化输出高电平
}

/* 闪烁led */
static void led_blink(void)
{
	u32 reg;
	reg = readl(HPS_GPIO1_BASE_ADDR + HPS_GPIO_DAT_OFFSET);
	reg ^= (0x1<<24);
	writel(reg, HPS_GPIO1_BASE_ADDR + HPS_GPIO_DAT_OFFSET);	//gpio53切换高低电平，以便闪烁
}

/* 测试led闪烁 */
static void gpio_led_test(void)
{
    gpio_init();	//初始化gpio53为输出，默认输出高电平，因此开始led亮
    for (;;) {
        delay_yeshen();		//延时
        led_blink();		//切换gpio电平，led由灭变亮或者由亮变灭
    }
}
```

