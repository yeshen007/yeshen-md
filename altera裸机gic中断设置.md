# <center>altera gic中断设置</center>

## 1.初始化

```c
ALT_STATUS_CODE init_global_int(void)
{
	ALT_STATUS_CODE status = ALT_E_SUCCESS;

	// Initialize the global and CPU interrupt items
	if (status == ALT_E_SUCCESS)
		{
			status = alt_int_global_init();		//分发器部分初始化设置
		}
	if (status == ALT_E_SUCCESS)
		{
			status = alt_int_cpu_init();		//CPU接口部分初始化设置
		}

//	alt_int_cpu_config_set(0,1,1);

	return status;

}
```

### 1.1 分发器部分初始化

```c
ALT_STATUS_CODE alt_int_global_init()
{
    int i;
    /* Cache the distributor and CPU base addresses
     / See: Cortex-A9 MPCore TRM, section 1.5. */
    {
        uint32_t periphbase = get_periphbase();		//Read Configuration Base Address Register
        alt_int_base_dist = periphbase + 0x1000;	//Interrupt Distributor
        alt_int_base_cpu  = periphbase + 0x0100;	//Interrupt controller cpu interfaces
    }

    /* Discover CPU and interrupt count
     / See GIC 1.0, section 4.3.2. */
    {
        uint32_t icdictr = alt_read_word(alt_int_base_dist + 0x4);		//Interrupt Controller Type Register (ICDICTR)
        alt_int_count_cpu = ((icdictr >> 5) & 0x7) + 1;		//the number of cpu interfaces 
        alt_int_count_int = ((icdictr & 0x1F) + 1) << 5;	//Indicates the maximum number of interrupts N,interrupts id 0 - N-1
    }

    /* Initialize the callback and context array
     / Initialize interrupt flags array */
    for (i = 0; i < ALT_INT_PROVISION_INT_COUNT; ++i)
    {
        alt_int_dispatch[i].callback = 0;
        alt_int_dispatch[i].context  = 0;

        alt_int_flag[i] = 0;		//初始化中断号支持变量， alt_int_flag[i]代表中断号i
    }

    /* Some preloaders enable interrupts but does not properly cleanup. This
     / will prevent interrupts from occuring when excution is handed off from
     / the preloader. */
    alt_int_global_disable();	/* The GIC ignores all peripheral interrupt signals, and does not forward pending interrupts to the CPU interfaces for non-secure */

    /* Discover all interrupts that are implemented in hardware.
     / See GIC 1.0, section 3.1.2. */
    for (i = 0; i < (ALT_INT_PROVISION_INT_COUNT / 32); ++i)
    {
        int j;
        uint32_t value;
        /* Set the whole bank to be enabled. */
        alt_write_word(alt_int_base_dist + 0x100 + i * sizeof(uint32_t), 0xffffffff); /* icdisern */  //使能中断

        /* Read it back to see which bits stick. */
        value = alt_read_word(alt_int_base_dist + 0x100 + i * sizeof(uint32_t)); /* icdisern */		//读取中断是否使能判断是否支持
        for (j = 0; j < 32; ++j)
        {
            if (((1 << j) & value) != 0)
            {
                alt_int_flag[i * 32 + j] |= INT_FLAG_IMPLEMENTED;
            }
        }

        /* Clear the whole bank to be disabled. */
        alt_write_word(alt_int_base_dist + 0x180 + i * sizeof(uint32_t), 0xffffffff); /* icdicern */	//关闭能关闭的中断
    }

    return ALT_E_SUCCESS;
}
```

### 1.2 CPU接口部分初始化

```c
ALT_STATUS_CODE alt_int_cpu_init()
{
    uint32_t cpu_num = get_current_cpu_num();		//获取当前cpu核号
    uint32_t stack_irq; 	
    ALT_STATUS_CODE status;

    if (cpu_num >= ALT_INT_PROVISION_CPU_COUNT)
    {
        return ALT_E_ERROR;
    }

    
    /* 设置IRQ栈 */
#if ALT_INT_PROVISION_STACK_SUPPORT

    /* The ARM stack lowers in address as it is being used. 16 is the alignment
     / of the block. */
    stack_irq = (uint32_t) &alt_int_stack_irq_block[cpu_num][sizeof(alt_int_stack_irq_block[0]) - 16];

    alt_int_fixup_irq_stack(stack_irq);

#endif /* #if ALT_INT_PROVISION_STACK_SUPPORT */

    /* Setup the Vector Interrupt Table */

#if ALT_INT_PROVISION_VECTOR_SUPPORT

    /* Set the vector interrupt table (VBAR) to use __cs3_interrupt_vector and
     / set SCTLR.V to always be 0. */

    /* For SCTLR.V information, See ARMv7, section B4.1.130.
     / For VBAR information, See ARMv7, section B4.1.156. */

    set_sctlr_vbit(false);		//设置异常向量表基地址为0x00000000，然后再映射到__cs3_interrupt_vector

    {
#if defined(__ARMCOMPILER_VERSION)
    extern char alt_interrupt_vector;
    uint32_t vector_table = (uint32_t)&alt_interrupt_vector;
    __asm("MCR p15, 0, %[vector_table], c12, c0, 0" : : [vector_table] "r" (vector_table));
#elif defined(__ARMCC_VERSION)
    extern char alt_interrupt_vector;
    uint32_t vector_table = (uint32_t)&alt_interrupt_vector;
    __asm("MCR p15, 0, vector_table, c12, c0, 0");
#else
    extern char __cs3_interrupt_vector;
    uint32_t vector_table = (uint32_t)&__cs3_interrupt_vector;
    __asm("MCR p15, 0, %0, c12, c0, 0" : : "r" (vector_table));		//设置__cs3_interrupt_vector为新的异常向量表基地址
#endif
    }
#endif /* #if ALT_INT_PROVISION_VECTOR_SUPPORT */
    
    /* Setup the priority mask and binary point defaults.
     / This will allow all interrupts to have sufficient priority to be
     / forwarded to the CPUs. */

    status = alt_int_cpu_priority_mask_set(0xff);	//设置权限掩码, 所有比0xff权限高的都能发送到该cpu接口
    if (status != ALT_E_SUCCESS)
    {
        return status;
    }
    status = alt_int_cpu_binary_point_set(3);   //设置binary point
    if (status != ALT_E_SUCCESS)
    {
        return status;
    }

    return ALT_E_SUCCESS;
}
```





## 2.使能

```c
ALT_STATUS_CODE enable_global_int(void)
{

	ALT_STATUS_CODE status = ALT_E_SUCCESS;

	// Enable the CPU and global interrupt
	if (status == ALT_E_SUCCESS)
	{
		// alt_setbits_word(alt_int_base_cpu + 0x0, 0x1);
		status = alt_int_cpu_enable();	//使能CPU接口部分
	}

	if (status == ALT_E_SUCCESS)
	{
		//alt_setbits_word(alt_int_base_dist + 0x0, 0x1);
		status = alt_int_global_enable();	//使能分发器部分
	}

	return status;

}
```

