##                                                                                                            L2 cache设置

### 一.设置步骤

- 初始化L2 cache
  - 查询L2 cache的way数
  - 设置L2 cache的tag ram和data ram的读写延时
- 使能L2 cache的prefetch(预取指)
- 使能L2 cache的parity(校验)
- 使能L2 cache
  - 先失效所有的L2 cache
  - 使能L2 cache



### 二.设置代码

首先是使能整个l2 cache系统的整个初始化框架

```c
/* 使能整个l2 cache系统 */
GREE_STATUS_CODE gree_cache_system_enable(void)
{

    gree_cache_l2_init();	//初始化L2 cache
    gree_cache_l2_prefetch_enable();	//使能L2 cache的prefetch(预取指)
    gree_cache_l2_parity_enable();	//使能L2 cache的parity(校验)
    gree_cache_l2_enable();		//使能L2 cache

    return GREE_E_SUCCESS;
}
```



初始化 L2 cache

```c
GREE_STATUS_CODE gree_cache_l2_init(void)
{
    /* 查询L2 cache的way数存到gree_cache_l2_waymask中 */
    u32 auxctrl = gree_read_word(GREE_MPUL2_AUX_CONTROL_ADDR);

    if (auxctrl & GREE_MPUL2_AUX_CONTROL_ASSOCIATIVITY_SET_MSK)
    {
        gree_cache_l2_waymask = 0x0000ffff;		//16-way
    }
    else
    {
        gree_cache_l2_waymask = 0x000000ff;		//8-way
    }


	/* 设置L2 cache的tag ram和data ram的读写延时 */
    gree_write_word(GREE_MPUL2_TAG_RAM_CONTROL_ADDR,
                     GREE_MPUL2_TAG_RAM_CONTROL_WRITE_LATENCY_VALUE_SET(0)
                   | GREE_MPUL2_TAG_RAM_CONTROL_READ_LATENCY_VALUE_SET(0)
                   | GREE_MPUL2_TAG_RAM_CONTROL_SETUP_LATENCY_VALUE_SET(0));

    gree_write_word(GREE_MPUL2_DATA_RAM_CONTROL_ADDR,
                     GREE_MPUL2_DATA_RAM_CONTROL_WRITE_LATENCY_VALUE_SET(0)
                   | GREE_MPUL2_DATA_RAM_CONTROL_READ_LATENCY_VALUE_SET(1)
                   | GREE_MPUL2_DATA_RAM_CONTROL_SETUP_LATENCY_VALUE_SET(0));
    
    /* 清可能留存的中断 */
    gree_cache_l2_int_status_clear(GREE_CACHE_L2_INTERRUPT_ALL);

    return GREE_E_SUCCESS;
}
```



使能L2 cache的prefetch(预取指)

```c
GREE_STATUS_CODE gree_cache_l2_prefetch_enable(void)
{
    /* 写L2C-310的Prefetch Control Register来使能prefetch */
	gree_setbits_word(GREE_MPUL2_PREFETCH_CTRL_ADDR,
                          GREE_MPUL2_PREFETCH_CTRL_PF_EN_SET_MSK);

    return GREE_E_SUCCESS;
}
```



使能L2 cache的parity(校验)

```c
GREE_STATUS_CODE gree_cache_l2_parity_enable(void)
{
    
    bool l2_enabled = gree_cache_l2_is_enabled();
    
    /* 如果l2 cache已经使能则先关闭 */
    if (l2_enabled)
    {
        gree_cache_l2_disable();
    }

    /* 写Aux Control Register来使能parity */
    gree_setbits_word(GREE_MPUL2_AUX_CONTROL_ADDR,
					GREE_MPUL2_AUX_CONTROL_PARITY_EN_SET_MSK);

    /* 使能临时关闭的l2 cache */
    if (l2_enabled)
    {
        gree_cache_l2_enable();
    }

    return GREE_E_SUCCESS;
}
```



使能L2 cache

```c
GREE_STATUS_CODE gree_cache_l2_enable(void)
{
    if (!gree_cache_l2_is_enabled())
    {
        /* 先失效所有的L2 cache */
        gree_cache_l2_invalidate_all(); 
        /* 使能L2 cache */
        gree_write_word(GREE_MPUL2_CONTROL_ADDR, GREE_MPUL2_CONTROL_EN_SET_MSK);
    }

    return GREE_E_SUCCESS;
}
```

按way来失效L2 cache

```c
GREE_STATUS_CODE gree_cache_l2_invalidate_all(void)
{
    int i = 0;

    /* 根据gree_cache_l2_init()得到的ways数来失效L2 cache */
    gree_write_word(GREE_MPUL2_INV_WAY_ADDR, gree_cache_l2_waymask);

    /* 在指定时间内读取失效操作状态，若没有完成则返回超时错误 */
    while (gree_read_word(GREE_MPUL2_INV_WAY_ADDR))
    {
        if (i == GREE_CACHE_L2_INVALIDATE_ALL_TIMEOUT)
        {
            return GREE_E_TMO;
        }
        ++i;
    }

    return GREE_E_SUCCESS;
}
```





