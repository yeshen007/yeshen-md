# </center>定时器<center>

## 一.说明

使用osc1timer0定时器，时钟源是osc1_clk，来自HPS_CLK1引脚，不可用软件配置，由硬件决定，开发板默认值25mhz，以下代码就以该值计算。

初始化流程：

1.禁止定时器

2.选择用户自定义计数模式

3.屏蔽计时器中断

4.加载计数值

5.使能定时器



## 二.代码

```c
/* OSC1TIMER0的寄存器基地址 */
#define YESHEN_TIMER_ADDRESS 0xFFD00000

/* 用户自定义计数模式时加载的计数值 */
#define YESHEN_TIMER_LOAD_VAL 0xFFFFFFFF

/* 定时器的时钟频率 */
#define YESHEN_TIMER_CLOCK_KHZ 25000

/* 定时器寄存器组结构体 */
struct yeshen_timer {
	u32	load_val;		//加载计数值
	u32	curr_val;		//读取当前计数值
	u32	ctrl;			//控制寄存器，使能禁止计时器，选择用户模式或自由模式，中断屏蔽
	u32	eoi;			//清除中断
	u32	int_stat;		//中断状态
};

/* OSC1TIMER0 */
static const struct yeshen_timer *yeshen_timer_base = (void *)YESHEN_TIMER_ADDRESS;	


/* 初始化计时器OSC1TIMER0 */
static void yeshen_timer_init(void)
{
    u32 reg;
    
    /* 禁止计时器 */
    reg = readl(&yeshen_timer_base->ctrl);
    reg &= ~(0x1<<0);
    writel(reg, &yeshen_timer_base->ctrl);
    
    /* 选择用户自定义计数模式 */
    reg = readl(&yeshen_timer_base->ctrl);
    reg |= 0x1<<1;
    writel(reg, &yeshen_timer_base->ctrl);
        
    /* 屏蔽计时器中断 */
    reg = readl(&yeshen_timer_base->ctrl);
    reg |= 0x1<<2;
    writel(reg, &yeshen_timer_base->ctrl);
    
    /* 加载计数值 */
	writel(YESHEN_TIMER_LOAD_VAL, &yeshen_timer_base->load_val);	
    
    /* 使能计时器 */
    reg = readl(&yeshen_timer_base->ctrl);
    reg |= 0x1<<0;
    writel(reg, &yeshen_timer_base->ctrl);    

}


/* 读取计时器的当前计数值 */
static u32 yeshen_read_timer(void)
{
	return readl(&yeshen_timer_base->curr_val);
}


/* 延时msec毫秒 */
static void yeshen_mdelay(unsigned int msec)
{
	unsigned int now, last;
    int rest;

	rest = msec * YESHEN_TIMER_CLOCK_KHZ;	//msec毫秒对应的定时器时钟周期计数
	last = yeshen_read_timer();				//读取当前周期计数值
    
    /* tmo减少到小于或等于0说明延时到了 */
	while (rest > 0) {
		now = yeshen_read_timer();	//读取当前周期计数值
		if (last >= now)	// 一般情况，计数寄存器没有益出
			rest -= last - now;
		else				// 计数寄存器益出 
			rest -= YESHEN_TIMER_LOAD_VAL - now + last;
		last = now;
	}
}


/* 测试样例 */
static void timer_test(void)
{
	yeshen_timer_init();
	for (;;) {
		yeshen_mdelay(1000);
		putc('a');		//每隔一秒打印一个a
	}
}
```

