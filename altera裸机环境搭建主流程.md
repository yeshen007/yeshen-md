# <center>双裸机通信环境搭建内存主流程</center>

## 1. 设置MMU

### 1.1 disable MMU（开机默认是关的）

### 1.2 建立页表（只用一个section的一级页表）

### 1.3 设置TTBCR.N=0，使所有虚拟地址都是用TTBR0 

### 1.4 设置页表基地址寄存器TTBR0指向页表

### 1.5 设置DACR访问权限



## 2.禁L1 cache

### 2.1 disable parity

### 2.2 disable L1 Instruction cache

### 2.3 clean L1 data cache,disable L1 data cache

### 2.4 disable L1 branch

### 2.5 disable L1 prefetch



## 3. 设置L2 cache

### 3.1 clean and validate L2(L2是两个核公用，因此需要clean，防止第一个启动的核没有将脏数据更新到主存，或者考虑延时两个核)

### 3.2 disable L2 

### 3.3 检测是有多少way,设置tag-ram和data-ram的访问延时,清中断




## 4. 使能MMU，L1 cache, L2 cache

### 4.1 使能smp

```c
init_smp:
	@@ 使得辅助控制寄存器的SMP位在非安全状态下可写，但写其他位没作用：a9 4.3.13
	MRC    p15, 0, r1, c1, c1, 2
	LDR    r2, =0x40000  @ 打开NSACR的位18
	ORR    r1, r1, r2	
	MCR    p15, 0, r1, c1, c1, 2
        
	@@ 使该核参与scu一致性维护， a9 4.3.10
	MRC    p15, 0, r1, c1, c0, 1
	ORR    r1, r1, #0x40  @ 打开ACTLR的smp位
	MCR    p15, 0, r1, c1, c0, 1


```

### invalidate TLB,L1 Instruction cache,L1 data cache,L1 branch

```c
ALT_STATUS_CODE alt_mmu_tlb_invalidate(void)
{
    /* Issue TLBIALL (TLB Invalidate All)
     / See ARMv7-A,R, section B4.1.135. */

    uint32_t dummy = 0;

#if   defined(__ARMCOMPILER_VERSION)
    __asm("MCR p15, 0, %[dummy], c8, c3, 0" : : [dummy] "r" (dummy));
#elif defined(__ARMCC_VERSION)
    __asm("MCR p15, 0, dummy,    c8, c3, 0");
#else
    __asm("MCR p15, 0, %0,       c8, c3, 0" : : "r" (dummy));
#endif

    /* Ensure all TLB maintenance operations complete before returning. */
    __asm("dsb");

    return ALT_E_SUCCESS;
}
```

```c
ALT_STATUS_CODE alt_cache_l1_instruction_invalidate(void)
{
    /* Issue the ICIALLUIS (Instruction Cache Invalidate ALL to point of
     / Unification Inner Shareable) cache maintenance operation
     / See ARMv7-A,R, section B4.2.1. */

    uint32_t dummy = 0;

#if   defined(__ARMCOMPILER_VERSION)
    __asm("MCR p15, 0, %[dummy], c7, c1, 0" : : [dummy] "r" (dummy));
#elif defined(__ARMCC_VERSION)
    __asm("MCR p15, 0, dummy,    c7, c1, 0");
#else
    __asm("MCR p15, 0, %0,       c7, c1, 0" : : "r" (dummy));
#endif

    return ALT_E_SUCCESS;
}
```

```c
/* 和alt_cache_l1_data_clean_all差不多 */
ALT_STATUS_CODE alt_cache_l1_data_invalidate_all(void)
{
    int way; 
    /* Gather parameters for DCISW (Data Cache Invalidate by Set / Way) data format.
     / See ARMv7-A,R, section B4.2.1 */

    /* Query the log2(line size in bytes), log2(associativity), log2(set count) for the data cache. */

    uint32_t L = 0;
    uint32_t A = 0;
    uint32_t S = 0;

    cssidr_decode_helper(false, &L, &A, &S);

    /* Repeatedly call DCISW and loop for every cache way and set. */

    for (way = 0; way < (1 << A); ++way)
    {
        int set;
        for (set = 0; set < (1 << S); ++set)
        {
            uint32_t way_set_info = 0;
            way_set_info |= way << (32 - A);
            way_set_info |= set << (L);
            /* Level is 0 because we're invalidating the L1. */

#if   defined(__ARMCOMPILER_VERSION)
            __asm("MCR p15, 0, %[way_set_info], c7, c6, 2" : : [way_set_info] "r" (way_set_info));
#elif defined(__ARMCC_VERSION)
            __asm("MCR p15, 0, way_set_info,    c7, c6, 2");
#else
            __asm("MCR p15, 0, %0,              c7, c6, 2" : : "r" (way_set_info));
#endif
        }
    }

    /* Ensure all cache maintenance operations complete before returning. */
    __asm("dsb");

    return ALT_E_SUCCESS;
}
```

```c
ALT_STATUS_CODE alt_cache_l1_branch_invalidate(void)
{
    /* Issue BPIALLIS (Branch Predictor Invalidate ALL, Inner Shareable). */

    uint32_t dummy = 0;

#if   defined(__ARMCOMPILER_VERSION)
    __asm("MCR p15, 0, %[dummy], c7, c1, 6" : : [dummy] "r" (dummy));
#elif defined(__ARMCC_VERSION)
    __asm("MCR p15, 0, dummy,    c7, c1, 6");
#else
    __asm("MCR p15, 0, %0,       c7, c1, 6" : : "r" (dummy));
#endif

    /* Ensure all branch predictor maintenance operations complete before returning. */
    __asm("dsb");

    return ALT_E_SUCCESS;
}
```



### 4.2 使能MMU

```c
ALT_STATUS_CODE alt_mmu_enable(void)
{
    uint32_t sctlr;
    alt_mmu_tlb_invalidate();		//使能之前先失效TLB

    sctlr = sctlr_get_helper();
    sctlr |= ALT_CPU_SCTLR_M_SET_MSK;	//(1<<0)
    sctlr_set_helper(sctlr);

    return ALT_E_SUCCESS;
}
```

### 4.3 使能L1cache

- enable parity
```c
ALT_STATUS_CODE alt_cache_l1_parity_enable(void)
{
    uint32_t actlr = actlr_read_helper();
    if ((actlr & ALT_CPU_ACTLR_PARITYON_SET_MSK) == 0)
    {
        /* Determine which caches which will be affected by parity being enabled
         / are currently enabled. */
        bool dcache_en = alt_cache_l1_data_is_enabled();
        bool icache_en = alt_cache_l1_instruction_is_enabled();
        bool branch_en = alt_cache_l1_branch_is_enabled();

        /* For those caches, disable them temporarily */
        if (icache_en == true)
        {
            alt_cache_l1_instruction_disable();
        }
        if (dcache_en == true)
        {
            alt_cache_l1_data_disable();
        }
        if (branch_en == true)
        {
            alt_cache_l1_branch_disable();
        }

        /* Turn on parity in the L1. */
        actlr |= ALT_CPU_ACTLR_PARITYON_SET_MSK;
        actlr_write_helper(actlr);

        /* Now enable them again. */
        if (icache_en == true)
        {
            alt_cache_l1_instruction_enable();
        }
        if (dcache_en == true)
        {
            alt_cache_l1_data_enable();
        }
        if (branch_en == true)
        {
            alt_cache_l1_branch_enable();
        }
    }

    return ALT_E_SUCCESS;
}
```
- enable L1 Instruction cache

```c
ALT_STATUS_CODE alt_cache_l1_instruction_enable(void)
{
    /* Update SCTLR.I bit (bit 12)
     / See Cortex-A9 TRM, section 4.3.9 */

    uint32_t sctlr = sctlr_read_helper();
    if ((sctlr & ALT_CPU_SCTLR_I_SET_MSK) == 0)
    {
        alt_cache_l1_instruction_invalidate();	//开启l1 instruction cache前先invalidate

        sctlr |= ALT_CPU_SCTLR_I_SET_MSK;
        sctlr_write_helper(sctlr);
    }

    return ALT_E_SUCCESS;
}
```

 - enable L1 data cache
```c
ALT_STATUS_CODE alt_cache_l1_data_enable(void)
{
    /* Update SCTLR.C bit (bit 2)
     / See Cortex-A9 TRM, section 4.3.9 */

    uint32_t sctlr = sctlr_read_helper();
    if ((sctlr & ALT_CPU_SCTLR_C_SET_MSK) == 0)
    {
        alt_cache_l1_data_invalidate_all();		//开启l1 data cache前先invalidate

        sctlr |= ALT_CPU_SCTLR_C_SET_MSK;
        sctlr_write_helper(sctlr);
    }

    return ALT_E_SUCCESS;
}
```
 - enable L1 branch
```c
ALT_STATUS_CODE alt_cache_l1_branch_enable(void)
{
    uint32_t sctlr; 
    alt_cache_l1_branch_invalidate();		//开启l1 data branch前先invalidate
    sctlr = sctlr_read_helper();
    sctlr |= ALT_CPU_SCTLR_Z_SET_MSK;
    sctlr_write_helper(sctlr);

    return ALT_E_SUCCESS;
}
```
 - enable L1 prefetch
```c
ALT_STATUS_CODE alt_cache_l1_prefetch_enable(void)
{
    uint32_t actlr = actlr_read_helper();
    actlr |= ALT_CPU_ACTLR_L1PREFETCHEN_SET_MSK;
    actlr_write_helper(actlr);

    return ALT_E_SUCCESS;
}
```

### 4.4 使能L2 cache

- enable L2 prefetch
```c
ALT_STATUS_CODE alt_cache_l2_prefetch_enable(void)
{
    /* Use the Prefetch Control Register instead of Aux Control. This is
     / because the Prefetch Control can be changed while the L2 is enabled.
     / For more information, see L2C-310, section 3.3.14. */

    alt_setbits_word(ALT_MPUL2_PREFETCH_CTRL_ADDR, ALT_MPUL2_PREFETCH_CTRL_PF_EN_SET_MSK);

    return ALT_E_SUCCESS;
}
```
- enable L2 parity
```c
ALT_STATUS_CODE alt_cache_l2_parity_enable(void)
{
    bool l2_enabled = alt_cache_l2_is_enabled();

    if (l2_enabled)	//没有使能，所以不执行
    {
        alt_cache_l2_disable();
    }

    alt_setbits_word(ALT_MPUL2_AUX_CONTROL_ADDR, ALT_MPUL2_AUX_CONTROL_PARITY_EN_SET_MSK);	//0x104,(1<<21)

    if (l2_enabled)	//没有使能，所以不执行
    {
        alt_cache_l2_enable();
    }

    return ALT_E_SUCCESS;
}
```
- enable  L2 cache
```c
ALT_STATUS_CODE alt_cache_l2_enable(void)
{
    if (!alt_cache_l2_is_enabled())
    {
        alt_cache_l2_invalidate_all();	//其实可以不用，因为之前有invalidate过，但是这样规范
        alt_write_word(ALT_MPUL2_CONTROL_ADDR, ALT_MPUL2_CONTROL_EN_SET_MSK);
    }

    return ALT_E_SUCCESS;
}
```

