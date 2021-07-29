# <center>双裸机通信L2设置部分</center>

## 1.clean and validate L2

```c
ALT_STATUS_CODE alt_cache_l2_purge_all(void)
{
    int i = 0;

    /*Clean and invalidate by way, all ways.
    /  For more information, see L2C-310, section 3.3.10. */

    alt_write_word(ALT_MPUL2_CLEAN_INV_WAY_ADDR, alt_cache_l2_waymask);

    while (alt_read_word(ALT_MPUL2_CLEAN_INV_WAY_ADDR))
    {
        /* Background operation still in progress. */

        if (i == ALT_CACHE_L2_PURGE_ALL_TIMEOUT)
        {
            return ALT_E_TMO;
        }
        ++i;
    }

    dprintf("CACHE[L2]: Purge All time = %d.\n", i);

    return ALT_E_SUCCESS;
}
```



## 2.disable L2

```c
ALT_STATUS_CODE alt_cache_l2_disable(void)
{
    if (alt_cache_l2_is_enabled())
    {
       //alt_cache_l2_purge_all();  //第一步做了
        alt_write_word(ALT_MPUL2_CONTROL_ADDR, 0);	//offset 0x100
    }

    return ALT_E_SUCCESS;
}
```



## 3. 检测是有多少way,设置tag-ram和data-ram的访问延时,清中断

```c
ALT_STATUS_CODE alt_cache_l2_init(void)
{


    /* Query the cache characteristics */

    uint32_t auxctrl = alt_read_word(ALT_MPUL2_AUX_CONTROL_ADDR);

    if (auxctrl & ALT_MPUL2_AUX_CONTROL_ASSOCIATIVITY_SET_MSK)
    {
        alt_cache_l2_waymask = 0x0000ffff;		//16-way
    }
    else
    {
        alt_cache_l2_waymask = 0x000000ff;		//8-way
    }

    /* Initialize the L2CC using instructions from L2C-310, section 3.1.1. */

    /* Write global configuration:
     *  - Associativity, way size
     *  - Latencies for RAM accesses
     *  - Allocation policy
     *  - Prefetch and power capability
     * (Not needed as the associated lines are by default set to the instantiation parameters) */

    /* Invalidate by way all cache entries
     * (Not needed as it will be invalidated when L2 is enabled) */
	//设置tag-ram的访问延时
    alt_write_word(ALT_MPUL2_TAG_RAM_CONTROL_ADDR,
                     ALT_MPUL2_TAG_RAM_CONTROL_WRITE_LATENCY_VALUE_SET(0)
                   | ALT_MPUL2_TAG_RAM_CONTROL_READ_LATENCY_VALUE_SET(0)
                   | ALT_MPUL2_TAG_RAM_CONTROL_SETUP_LATENCY_VALUE_SET(0));
	//设置data-ram的访问延时
    alt_write_word(ALT_MPUL2_DATA_RAM_CONTROL_ADDR,
                     ALT_MPUL2_DATA_RAM_CONTROL_WRITE_LATENCY_VALUE_SET(0)
                   | ALT_MPUL2_DATA_RAM_CONTROL_READ_LATENCY_VALUE_SET(1)
                   | ALT_MPUL2_DATA_RAM_CONTROL_SETUP_LATENCY_VALUE_SET(0));
    
    /* Clear interrupts just in case. */
    alt_cache_l2_int_status_clear(ALT_CACHE_L2_INTERRUPT_ALL);

    return ALT_E_SUCCESS;
}
```

