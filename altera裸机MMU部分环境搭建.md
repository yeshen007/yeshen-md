# <center>双裸机通信的MMU部分环境搭建</center>

## 1. disable MMU

```c
ALT_STATUS_CODE alt_mmu_disable(void)
{
    uint32_t sctlr = sctlr_get_helper();	//获取系统控制寄存器sctlr的值
    if (sctlr & ALT_CPU_SCTLR_C_SET_MSK)
    {
        dprintf("WARN[MMU]: Data cache still active.\n");
    }
    sctlr &= ~ALT_CPU_SCTLR_M_SET_MSK;	//将第0位（mmu使能位）置0
    sctlr_set_helper(sctlr);	//写入系统控制寄存器sctlr的值

    return ALT_E_SUCCESS;
}
```



## 2. 建立页表



## 3. 设置TTBCR.N=0，使所有虚拟地址都是用TTBR0

```c
ttbcr_set_helper(0); 
```



##  4.设置页表基地址寄存器TTBR0指向页表

```c
/* 页表基地址addr */
ALT_STATUS_CODE alt_mmu_TTBR0_set(const void* addr)
{
    uint32_t ttbcr = ttbcr_get_helper();	//获取ttbcr寄存器的值
    uint32_t ttbcr_n = ALT_CPU_TTBCR_N_VALUE_GET(ttbcr); //获取ttbcr寄存器的N值
    uint32_t ttbr0; 

    if ((uintptr_t)addr & ~ALT_CPU_TTBR0_TTB0BASEADDR_SET_MSK(ttbcr_n))
    {
        /* 检测页表基地址addr是否满足2^(14 - TTBCR.N)对齐要求 */
        return ALT_E_BAD_ARG;
    }

    /* The Translation table must reside in Normal Memory, so pick the most
     / performant attributes. */
    ttbr0 =   ALT_CPU_TTBR0_RGN_WBA   /* Translation table is WBA for outer cacheability */
              | ALT_CPU_TTBR0_IRGN_WBA; /* Translation table is WBA for inner cacheability */
    ttbr0 &= ~ALT_CPU_TTBR0_TTB0BASEADDR_SET_MSK(ttbcr_n);	//属性值填入ttbr0低部分
    ttbr0 |= (uint32_t)addr;		//页表基地址填入ttbr0高部分

    ttbr0_set_helper(ttbr0);		//写入ttbr0寄存器

    return ALT_E_SUCCESS;
}
```



## 5. 设置DACR访问权限

```c
/* dacr设置为client access */
dacr_set_helper(dacr);
```



