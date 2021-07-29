# <center>双裸机通信的L1 cache disable部分</center>

## 1.disable L1 parity

```c
ALT_STATUS_CODE alt_cache_l1_parity_disable(void)
{
    uint32_t actlr = actlr_read_helper();
    actlr &= ~ALT_CPU_ACTLR_PARITYON_SET_MSK;	//校验禁止(1 << 9)A9,4.3.10.
    actlr_write_helper(actlr);

    return ALT_E_SUCCESS;
}
```



## 2.disable L1 Instruction cache

```c
ALT_STATUS_CODE alt_cache_l1_instruction_disable(void)
{
    /* Update SCTLR.I bit (bit 12)
     / See Cortex-A9 TRM, section 4.3.9 */

    uint32_t sctlr = sctlr_read_helper();
    sctlr &= ~ALT_CPU_SCTLR_I_SET_MSK;	//(1 << 12) A9  4.3.9 
    sctlr_write_helper(sctlr);

    return ALT_E_SUCCESS;
}
```



## 3.clean L1 data cache,disable L1 data cache

```c

ALT_STATUS_CODE alt_cache_l1_data_disable(void)
{
    /* Update SCTLR.C bit (bit 2)
     / See Cortex-A9 TRM, section 4.3.9 */

    uint32_t sctlr = sctlr_read_helper();
    if ((sctlr & ALT_CPU_SCTLR_C_SET_MSK) != 0)  //如果数据caching enabled
    {
        alt_cache_l1_data_clean_all();		//先clean数据缓存

        sctlr &= ~ALT_CPU_SCTLR_C_SET_MSK;		

        sctlr_write_helper(sctlr);	//再disable数据缓存
    }

    return ALT_E_SUCCESS;
}
```

```c
ALT_STATUS_CODE alt_cache_l1_data_clean_all(void)
{
    /* Gather parameters for DCCSW (Data Cache Clean by Set / Way) data format
     / See ARMv7-A,R, section B4.2.1 */

    /* Query the log2(line size in bytes), log2(associativity), log2(set count) for the data cache. */

    uint32_t L = 0;
    uint32_t A = 0;
    uint32_t S = 0;
    int way;

    /* Ensure that it is turned on before we start trying to clear it */
    if ( ! alt_cache_l1_data_is_enabled())
    {
        return ALT_E_BAD_OPERATION;
    }
    cssidr_decode_helper(false, &L, &A, &S);

    /* Repeatedly call DCCSW and loop for every cache way and set. */

    for (way = 0; way < (1 << A); ++way)	//先按way来clean
    {
        int set;
        for (set = 0; set < (1 << S); ++set)	//在一个way中循环clean掉一个个set，其中这种情况一个set就是一个line
        {
            uint32_t way_set_info = 0;
            way_set_info |= way << (32 - A);
            way_set_info |= set << (L);
            /* Level is 0 because we're invalidating the L1. */

#if   defined(__ARMCOMPILER_VERSION)
            __asm("MCR p15, 0, %[way_set_info], c7, c10, 2" : : [way_set_info] "r" (way_set_info));
#elif defined(__ARMCC_VERSION)
            __asm("MCR p15, 0, way_set_info,    c7, c10, 2");
#else
            __asm("MCR p15, 0, %0,              c7, c10, 2" : : "r" (way_set_info));
#endif
        }
    }

    /* Ensure all cache maintenance operations complete before returning. */
    __asm("dsb");

    return ALT_E_SUCCESS;
}
```

```c
static void cssidr_decode_helper(bool query_i_cache, uint32_t * log2_L, uint32_t * log2_A, uint32_t * log2_S)
{
    /* First query the D cache information using CSSELR to select the data cache
     / See ARM Cortex-A9 TRM, section 4.3.8. */

    /* Wait for the CSSELR to flush. */

    /* Then use CSSIDR to get the cache charateristics
     / See ARMv7-A,R, section B4.1.19 and Cortex-A9 TRM, section 4.3.5. */

    uint32_t csselr;
    uint32_t cssidr;

    if (query_i_cache)
    {
        csselr = ALT_CPU_ACTLR_IND_SET_MSK;		//选择instruction cache
    }
    else
    {
        csselr = 0;				//选择data cache
    }

#if   defined(__ARMCOMPILER_VERSION)
    __asm("MCR p15, 2, %[csselr], c0, c0, 0" : : [csselr] "r" (csselr));
    __asm("ISB");
    __asm("MRC p15, 1, %[csselr], c0, c0, 0" : [csselr] "=r" (cssidr));
#elif defined(__ARMCC_VERSION)
    __asm("MCR p15, 2, csselr, c0, c0, 0");
    __asm("ISB");
    __asm("MRC p15, 1, cssidr, c0, c0, 0");
#else
    __asm("MCR p15, 2, %0, c0, c0, 0" : : "r" (csselr));	//因为query_i_cache 为fasle所以选择data cache
    __asm("ISB");
    __asm("MRC p15, 1, %0, c0, c0, 0" : "=r" (cssidr));		//然后获取data cache的line，way，set
#endif
    {
    /* Extract the associativity, line length, and number of sets. */
    int linesize      = ALT_CPU_CCSIDR_LINESIZE_VALUE_GET(cssidr) + 2 + 2; 	//提取出log2(line length in bytes)
        /* {log2(line length in words) - 2} + 2 + 2 => (... in bytes) */
    int associativity = ALT_CPU_CCSIDR_ASSOCIATIVITY_VALUE_GET(cssidr) + 1;	//提取出way
    int numsets       = ALT_CPU_CCSIDR_NUMSETS_VALUE_GET(cssidr) + 1;		//提取出set

    /* Determine the log2 of the associativity and numsets, rounded up */
    int L = linesize; /* log2(line length in bytes) */
    int A = 0;        /* log2(associativity) rounded up */
    int S = 0;        /* log2(number of sets) rounded up */

    while ((1 << A) < associativity)
    {
        ++A;		//计算出log2(ways)
    }

    while ((1 << S) < numsets)
    {
        ++S;		//计算出log2(sets)
    }

    /* Output the parameters */
    *log2_L = L;
    *log2_A = A;
    *log2_S = S;
    }
}
```



## 4 .disable L1 branch

```c
ALT_STATUS_CODE alt_cache_l1_branch_disable(void)
{
    uint32_t sctlr = sctlr_read_helper();
    sctlr &= ~ALT_CPU_SCTLR_Z_SET_MSK;	//(1 << 11)Cortex-A9 TRM, section 4.3.9
    sctlr_write_helper(sctlr);

    return ALT_E_SUCCESS;
}
```



## 5.disable L1 prefetch

```c
ALT_STATUS_CODE alt_cache_l1_prefetch_disable(void)
{
    uint32_t actlr = actlr_read_helper();
    actlr &= ~ALT_CPU_ACTLR_L1PREFETCHEN_SET_MSK;	//A9 4.3.10 (1<<2)
    actlr_write_helper(actlr);

    return ALT_E_SUCCESS;
}
```



