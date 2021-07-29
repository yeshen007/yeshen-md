# AMP启动流程分析: 引导ROM、SPL、BM、U-BOOT四个阶段 

上电冷复位后的启动经历引导ROM、SPL、BM、U-BOOT四个阶段，以下是各阶段主要完成的工作：

## 引导ROM 
引导ROM 代码可以执行片上RAM 的热引导、器件的FPGA 部分的冷引导或闪存的冷引导。

**器件的FPGA 部分的冷引导**期间，引导ROM 代码等待直到FPGA 准备就绪，然后试图通过HPS- to- FPGA 桥接在地址0x0 直接执行。例如，引导软件可能会由器件的FPGA 部分中的地址0x0 的初始化的片上RAM 提供。

**闪存的冷引导**期间，引导ROM 代码试图将第一个预加载器镜像从闪存加载到片上RAM 并且将控制传递到预加载器。如果该镜像无效，那么引导ROM 代码试图从闪存加载接下来的3 个镜像。如果在接下来的加载中仍然没有有效镜像，那么引导ROM 代码检查器件的FPGA 部分是否存在一个备用镜像。

**片上RAM 的热引导**期间，引导ROM代码读取系统管理器中romcodegrp组的预加载器状态寄存器(initswstate)，以便决定片上RAM中是否具有有效预加载器镜像。如果片上RAM中具有有效预加载器镜像，那么引导ROM代码跳过从闪存加载预加载器镜像，而是将控制传递到片上RAM中的预加载器。如果片上RAM 中没有有效预加载器镜像，那么引导ROM 代码试图加载从闪存加载的最后一个有效预加载器镜像(由系统管理器romcodegrp组的初始软件最后镜像加载寄存器(initswlastld) 的index 域识别)。如果镜像无效，那么引导ROM 代码试图从闪存加载接下来的3 个镜像。如果片上RAM 或闪存中不存在有效预加载器镜像，那么引导ROM代码检查器件的FPGA 部分是否有备用镜像。

**复位后片上RAM（总共64KB）顶部4KB为引导ROM代码而保留，该区域包括共享内存，引导ROM代码将共享内存的位置传递到SPL的寄存器r0**

**引导ROM 执行以下操作来初始化HPS：**

- 使能指令缓存、分支预测器、浮点单元和NEON 矢量单元
- 设置level 4 (l4) 看门狗0 计时器
- 根据CSEL 值配置主PLL 和外设PLL
- 根据BSEL 值配置I/O 单元和管脚复用
- 以默认设置初始化闪存控制器


**当引导ROM 代码准备将控制传递到预加载器时，处理器(CPU0) 处于以下状态：**

- 指令缓存被使能
- 分支预测器被使能
- 数据缓存被禁用
- **MMU被禁用**
- 浮点单元被使能
- NEON矢量单元被使能
- 处理器处于ARM安全监督模式
- **r0包含共享内存模块的指针，可用于将信息从引导ROM代码传递到SPL,共享内存模块位于片上RAM的顶部4KB**
- r1包含共享内存的长度
- **引导ROM仍被映射到地址0x0**
- L4 看门狗0 计时器有效
## SPL 
- 初始化SDRAM接口
- **将片上RAM映射到地址0x0，以便SPL处理异常情况（片上RAM在地址0xFFFF0000也是可访问的。地址0x0是一个别名）**
- 通过扫描管理器配置HPS I/O（在引导ROM期间根据BSEL有一个预先配置）
- 通过系统管理器配置管脚复用（在引导ROM期间根据BSEL有一个预先配置）
- 通过时钟管理器配置HPS时钟（在引导ROM期间根据CSEL有一个主PLL和外设PLL预先配置）
- 初始化包含下一个阶段引导软件的闪存控制器(NAND、SD/MMC 或quad SPI)
- **将下一个阶段引导软件（bm）加载到SDRAM 并且将控制传递到下一个阶段引导软件**
## BM
 - 加载fpga配置镜像到sdram中
 - 通过fpga管理器和sdram中的fpga配置镜像配置fpga
 - 和跑linux的cpu0进行双核测试

## U-BOOT

## SPL流程详细分析：
### SPL入口
**从连接脚本spl/u-boot-spl.lds的开始片段和编译输出信息中的-Ttext 0xFFFF0000中可以得到SPL入口在_start,运行时应该在地址0xFFFF0000（片上RAM的起始地址总会映射到这里，还有可能有另外映射地址）处。**


	连接脚本
	MEMORY { .sram : ORIGIN = 0xFFFF0000, LENGTH = (64 * 1024) }
	OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
	OUTPUT_ARCH(arm)
	ENTRY(_start)
	SECTIONS
	{
	 .text :
	 {
	  arch/arm/cpu/armv7/start.o (.text)
	  *(.text*)
	 } >.sram
	
	编译打印信息
	cd /home/hn/TEST/u-boot-altera-2012.10/spl/ && arm-linux-gnueabihf-ld.bfd  -T /home/hn/TEST/u-boot-altera-2012.10/spl/u-boot-spl.lds --gc-sections -Bstatic -Ttext 0xFFFF0000 arch/arm/cpu/armv7/start.o...

### SPL总体流程
**\_start --> reset --> save\_boot\_params(什么都没做) --> 设置管理模式，禁止IRQ和FIQ中断 --> cpu\_init\_cp15 --> cpu\_init\_crits --> lowlevel\_init --> s\_init --> call\_board\_init\_f --> board\_init\_f --> board\_init\_r**
	
### SPL流程各阶段详细分析

**\_start --> reset --> save\_boot\_params(什么都没做) --> 设置管理模式，禁止IRQ和FIQ中断 --> cpu\_init\_cp15 --> cpu\_init\_crit**(arch\arm\cpu\armv7\start.S)


	.globl _start
	_start: b	reset
	
	......................................................
	
	#ifdef CONFIG_SPL_BUILD
	.globl  save_timer_value
	save_timer_value:
		.word  0xabcdabcd
	#endif
	
	reset:
	/* 使用OSC1计时器1测试SPL到U-BOOT的时间  */
	#ifdef CONFIG_SPL_BUILD
		bl osc_timer_reset_release	@ 将OSC1定时器1从复位释放
		bl osc_timer_config			@ 将0xffffffff写入计数器中，屏蔽OSC1定时器1中断，自由模式，使能OSC1定时器1（复位后是disabled状态）		
		bl osc_timer_get			@ 获取osc1timer1当前值读入r0
		adr r1, save_timer_value	@ 将save_timer_value标号的地址读到r1
		str r0, [r1]				@ 将得到的osc1timer1当前值写到save_timer_value处
	#endif


	bl	save_boot_params
	/* 设置为svc管理模式，禁止IRQ和FIQ中断 */
	mrs	r0, cpsr		@ 将cpsr寄存器读到r0中
	bic	r0, r0, #0x1f	@ 将读到r0中的cpsr副本前5位清零
	orr	r0, r0, #0xd3	@ 打开r0中的cpsr副本第0，1，4，6，7位
	msr	cpsr,r0			@ 将处理好后的值写入cpsr寄存器中
	
	......................................................
	
	/* SPL会执行，U-BOOT阶段不执行 */
	#ifndef CONFIG_SKIP_LOWLEVEL_INIT	
		bl	cpu_init_cp15	
		bl	cpu_init_crit
	#endif
	
	......................................................
	
	ENTRY(cpu_init_cp15)
		/*
		 * Invalidate L1 I/D
		 */
		mov	r0, #0			@ 设置MCR协处理器
		mcr	p15, 0, r0, c8, c7, 0	@ 失效 TLBs
		mcr	p15, 0, r0, c7, c5, 0	@ 失效 icache
		mcr	p15, 0, r0, c7, c5, 6	@ 失效 分支预测
		mcr p15, 0, r0, c7, c10, 4	@ DSB，数据同步屏障，内存和缓存，tlb等维护操作完成才进行后面指令的操作
		mcr p15, 0, r0, c7, c5, 4	@ ISB，指令同步屏障，清空流水线
	
		/*
		 * disable MMU stuff and caches
		 */
		mrc	p15, 0, r0, c1, c0, 0	@ 将协处理器p15的系统控制寄存器读到r0中
		bic	r0, r0, #0x00002000	@ clear bits 13 (--V-)	@ 异常向量表映射到0x00000000
		bic	r0, r0, #0x00000007	@ clear bits 2:0 (-CAM)	@ 关闭mmu，关闭对齐检查，关闭数据和统一缓存
		orr	r0, r0, #0x00000002	@ set bit 1 (--A-) Align		@ 打开对齐检查
		orr	r0, r0, #0x00000800	@ set bit 11 (Z---) BTB		@ 打开分支预测
	
		orr	r0, r0, #0x00001000	@ set bit 12 (I) I-cache		@ 使能指令缓存
	
		mcr	p15, 0, r0, c1, c0, 0	@写入协处理器p15的系统控制寄存器中
		mov	pc, lr			@ back to my caller
	ENDPROC(cpu_init_cp15)
	
	ENTRY(cpu_init_crit)
		/*
		 * Jump to board specific initialization...
		 * The Mask ROM will have already initialized
		 * basic memory. Go here to bump up clock rate and handle
		 * wake up conditions.
		 */
		b	lowlevel_init		@ 设置pll,mux,sdram
	ENDPROC(cpu_init_crit)


**lowlevel\_init**(arch\arm\cpu\armv7\lowlevel_init.S)

	ENTRY(lowlevel_init)
		/*
		 * 设置一个临时栈
		 */
		ldr	sp, =CONFIG_SYS_INIT_SP_ADDR	@ 0xFFFF0000 + 0x10000 - 0x100 - GENERATED_GBL_DATA_SIZE(一定要在片上ram中)
		bic	sp, sp, #7  	@ 将栈指针8字节对齐
		push	{ip, lr}	@ 将老的lr和当前lr保存在临时栈中，当前lr指向call_board_init_f，是bl cpu_init_crit时设置的
		bl	s_init			@ 去设置pl,mux,sdram
		pop	{ip, pc}		@ 返回到cpu_init_crit下一条指令执行（call_board_init_f）
	ENDPROC(lowlevel_init)


**s\_init**(arch\arm\cpu\armv7\socfpga\s_init.c)

	/* 第一个c函数 */
	void s_init(void)
	{
		unsigned long reg;
		/*
		 * First C code to run. Clear fake OCRAM ECC first as SBE
		 * and DBE might triggered during power on
		 */
		reg = readl(CONFIG_SYSMGR_ECC_OCRAM);	//On-chip RAM ECC Enable Register
		if (reg & SYSMGR_ECC_OCRAM_SERR)	//如果不为0则代表有单个位的On-chip RAM ECC错误中断，可纠正，由硬件设置，但需要写1清除中断标志
			writel(SYSMGR_ECC_OCRAM_SERR | SYSMGR_ECC_OCRAM_EN,
				CONFIG_SYSMGR_ECC_OCRAM);	//使能On-chip RAM ECC功能，并清除single bit ecc错误中断
		if (reg & SYSMGR_ECC_OCRAM_DERR)	//如果不为0则代表有双位的On-chip RAM ECC错误中断，不可纠正，由硬件设置，需要写1清除中断标志
			writel(SYSMGR_ECC_OCRAM_DERR  | SYSMGR_ECC_OCRAM_EN,
				CONFIG_SYSMGR_ECC_OCRAM);	//使能On-chip RAM ECC功能，并清除double bit ecc错误中断
	
		/*
		 * 配置remap(L3 NIC-301 GPV)然后on-chip RAM会映射到0x0，取消ROM映射0x0
		 * 同时使能 HPS2FPGA and LWHPS2FPGA
		 */
		writel(0x19,SOCFPGA_L3REGS_ADDRESS);	
																								
		/* re-setup watchdog */
		DEBUG_MEMORY
		if (!(is_wdt_in_reset())) {		//如果不在复位状态则执行
			/*
			 * only disabled if wdt not in reset state
			 * disable the watchdog prior PLL reconfiguration
			 */
			DEBUG_MEMORY
			watchdog_disable();			//先关闭osc1wd0（l4wd0）,reset_manager.c
		}
	
	#ifdef CONFIG_HW_WATCHDOG
		/* release osc1 watchdog timer 0 from reset */
		DEBUG_MEMORY
		reset_deassert_osc1wd0();	//将osc1wd0（l4wd0）从复位状态释放
	
		/* reconfigure and enable the watchdog */
		DEBUG_MEMORY
		hw_watchdog_init();		//在drivers\watchdog\designware_wdt.c，使能l4wd0，刷新l4wd0的计数器重新计数
		
		WATCHDOG_RESET();		//再次刷新l4wd0的计数器从新计数,
	#endif /* CONFIG_HW_WATCHDOG */
	
		DEBUG_MEMORY
		/* 在连接脚本中定义，用来数据对齐设置 */
		if (&__ecc_padding_start < &__ecc_padding_end) {
			memset(&__ecc_padding_start, 0,
				&__ecc_padding_end - &__ecc_padding_start);
		}
	}

**call\_board\_init\_f**(arch\arm\cpu\armv7\start.S)


**从新设置临时栈，设置参数为0x00000000，然后跳转到board\_init\_f，从board\_init\_f返回后会到下一步relocate\_code，既真正的u-boot**

	call_board_init_f:
		ldr	sp, =(CONFIG_SYS_INIT_SP_ADDR)
		bic	sp, sp, #7 /* 8-byte alignment for ABI compliance */
		ldr	r0,=0x00000000
		bl	board_init_f	


**board\_init\_f**(arch\arm\lib\spl.c)

	void __weak board_init_f(ulong dummy)
	{
		/* Set the stack pointer. */
		asm volatile("mov sp, %0\n" : : "r"(CONFIG_SPL_STACK));		//在连接脚本中的__stack_start
	
		/* Clear the BSS. */
		memset(__bss_start, 0, __bss_end__ - __bss_start);	//清bss段
	
		/* Set global data pointer. */
		gd = &gdata;		//gd是存在r8寄存器当中的，通过宏DECLARE_GLOBAL_DATA_PTR
	
		saved_cpu0_global_data_ptr = (u32)gd;	//将gd写在start.S中的saved_cpu0_global_data_ptr地址处
		asm volatile("dsb");
		
		board_init_r(NULL, 0);	
	}

**board\_init\_r**(common\spl\spl.c)

**board\_init\_r的流程如下三个主要阶段：**

**1.spl\_board\_init()**的板级初始化阶段，有初始化时钟，引脚复用，串口，sdram，中断向量

**2.amp\_share\_param\_init()到spl\_mmc\_load\_image()**的初始化sd/mmc和加载启动bm阶段

**3.jump\_to\_image\_no\_args()**的spl最后一个阶段，跳转到u-boot启动运行

	void board_init_r(gd_t *dummy1, ulong dummy2)
	{
		u32 boot_device;
		debug(">>spl:board_init_r()\n");
	
		mem_malloc_init(CONFIG_SYS_SPL_MALLOC_START,
				CONFIG_SYS_SPL_MALLOC_SIZE);	//将__malloc_start和__malloc_end之间的内存清零
	
		timer_init();	//加载0xffffffff新计数，用户计数模式，使能osc1timer1
	
		spl_board_init();	//初始化时钟，引脚复用，串口，sdram，中断向量
	
		boot_device = spl_boot_device();	//获取boot device类型(BOOT_DEVICE_MMC1)
		debug("boot device - %d\n", boot_device);
	
		switch (boot_device) {	//加载启动bm
	
		case BOOT_DEVICE_MMC1:
		case BOOT_DEVICE_MMC2:
		case BOOT_DEVICE_MMC2_2:
			amp_share_param_init();			//0x1E100000,u-boot, bare metal, linux的共享内存区域地址 
	
			load_bm_start();				//获取加载bm.bin的开始时间 
	
			spl_mmc_probe();				//初始化sd/mmc控制器和设备
	
			spl_mmc_load_bm_image_mbr();	//将bm.bin从sd/mmc加载到sdram中
	
			load_bm_end();					//获取加载bm.bin的结束时间 
	
			spl_mmc_save_func();			//保存spl_mmc_probe等函数到共享内存给bm使用
	
			boot_bm_on_cpu1();				//从cpu1启动bm
			
			cpu0_wait_cpu1_load_rbf();		//等待cpu1中运行的bm加载rbf完成
	
			spl_mmc_load_image();			//将u-boot镜像加载到sdram中
	
			break;
	
		default:
			debug("SPL: Un-supported Boot Device\n");
			hang();
		}
	
		switch (spl_image.os) {		//spl_image在spl_mmc_load_image()里填充
		case IH_OS_U_BOOT:
			debug("Jumping to U-Boot\n");
			break;
	
		default:
			debug("Unsupported OS image.. Jumping nevertheless..\n");
		}
	
		jump_to_image_no_args();		//跳转到u-boot启动运行
	}


**1.spl\_board\_init()：**（arch\arm\cpu\armv7\socfpga\spl.c）

	void spl_board_init(void)
	{
		cm_config_t cm_default_cfg = {		//main,Peripheral和sdram pll时钟组设置数据
			/* main group */
			MAIN_VCO_BASE,							//M=63+1,N=0+1，main_vco_clk = 64eosc1_clk
			CLKMGR_MAINPLLGRP_MPUCLK_CNT_SET(		
				CONFIG_HPS_MAINPLLGRP_MPUCLK_CNT),	//mpu_clk = main_vco_clk/2 = 32eosc1_clk
			CLKMGR_MAINPLLGRP_MAINCLK_CNT_SET(		
				CONFIG_HPS_MAINPLLGRP_MAINCLK_CNT),	//main_clk = l3_main_clk = l4_main_clk = main_vco_clk/4 = 16eosc1_clk
			CLKMGR_MAINPLLGRP_DBGATCLK_CNT_SET(
				CONFIG_HPS_MAINPLLGRP_DBGATCLK_CNT),	//dbg_base_clk = 16eosc1_clk
			CLKMGR_MAINPLLGRP_MAINQSPICLK_CNT_SET(
				CONFIG_HPS_MAINPLLGRP_MAINQSPICLK_CNT),	//main_qspi_clk = main_vco_clk/4 = 16eosc1_clk
			CLKMGR_PERPLLGRP_PERNANDSDMMCCLK_CNT_SET(
				CONFIG_HPS_MAINPLLGRP_MAINNANDSDMMCCLK_CNT),	//main_nand_sdmmc_clk = main_vco_clk/4 = 16eosc1_clk
			CLKMGR_MAINPLLGRP_CFGS2FUSER0CLK_CNT_SET(
				CONFIG_HPS_MAINPLLGRP_CFGS2FUSER0CLK_CNT),	//cfg_s2f_user0_clk = main_vco_clk/16 = 4eosc1_clk
			CLKMGR_MAINPLLGRP_MAINDIV_L3MPCLK_SET(
				CONFIG_HPS_MAINPLLGRP_MAINDIV_L3MPCLK) |	//l3_mp_clk = l3_main_clk/(value+1) = l3_main_clk/2 = 8eosc1_clk
			CLKMGR_MAINPLLGRP_MAINDIV_L3SPCLK_SET(
				CONFIG_HPS_MAINPLLGRP_MAINDIV_L3SPCLK) |	//l3_sp_clk = l3_mp_clk/(value+1) = l3_mp_clk/2 = 4eosc1_clk
			CLKMGR_MAINPLLGRP_MAINDIV_L4MPCLK_SET(
				CONFIG_HPS_MAINPLLGRP_MAINDIV_L4MPCLK) |	//l4_mp_clk = periph_base_clk/2
			CLKMGR_MAINPLLGRP_MAINDIV_L4SPCLK_SET(
				CONFIG_HPS_MAINPLLGRP_MAINDIV_L4SPCLK),		//l4_sp_clk = periph_base_clk/2
			CLKMGR_MAINPLLGRP_DBGDIV_DBGATCLK_SET(
				CONFIG_HPS_MAINPLLGRP_DBGDIV_DBGATCLK) |	//dbg_at_clk = dbg_base_clk
			CLKMGR_MAINPLLGRP_DBGDIV_DBGCLK_SET(
				CONFIG_HPS_MAINPLLGRP_DBGDIV_DBGCLK),		//dbg_clk = dbg_at_clk/2 = dbg_base_clk/2
			CLKMGR_MAINPLLGRP_TRACEDIV_TRACECLK_SET(
				CONFIG_HPS_MAINPLLGRP_TRACEDIV_TRACECLK),	//dbg_trace_clk = dbg_base_clk
			CLKMGR_MAINPLLGRP_L4SRC_L4MP_SET(
				CONFIG_HPS_MAINPLLGRP_L4SRC_L4MP) |		//设置l4_mp_clk的时钟源为periph_base_clk
			CLKMGR_MAINPLLGRP_L4SRC_L4SP_SET(
				CONFIG_HPS_MAINPLLGRP_L4SRC_L4SP),		//设置l4_sp_clk的时钟源为periph_base_clk
	
			/* peripheral group */
			PERI_VCO_BASE,			//设置peripheral PLL组的时钟源为eosc1_clk，peri_vco_clk=(79+1)eosc1_clk/(1+1) = 40eosc1_clk
			CLKMGR_PERPLLGRP_EMAC0CLK_CNT_SET(		
				CONFIG_HPS_PERPLLGRP_EMAC0CLK_CNT),	//emac0_clk=emac0_base_clk=peri_vco_clk/(3+1)=peri_vco_clk/4 = 10eosc1_clk
			CLKMGR_PERPLLGRP_EMAC1CLK_CNT_SET(
				CONFIG_HPS_PERPLLGRP_EMAC1CLK_CNT),//emac1_clk=emac0_base_clk=peri_vco_clk/(3+1)=peri_vco_clk/4 = 10eosc1_clk
			CLKMGR_PERPLLGRP_PERQSPICLK_CNT_SET(
				CONFIG_HPS_PERPLLGRP_PERQSPICLK_CNT),//periph_qspi_base_clk=peri_vco_clk/2 = 20eosc1_clk
			CLKMGR_PERPLLGRP_PERNANDSDMMCCLK_CNT_SET(
				CONFIG_HPS_PERPLLGRP_PERNANDSDMMCCLK_CNT),//periph_nand_sdmmc_clk=peri_vco_clk/5 = 8eosc1_clk
			CLKMGR_PERPLLGRP_PERBASECLK_CNT_SET(
				CONFIG_HPS_PERPLLGRP_PERBASECLK_CNT),//periph_base_clk=peri_vco_clk/5 = 8eosc1_clk
			CLKMGR_PERPLLGRP_S2FUSER1CLK_CNT_SET(
				CONFIG_HPS_PERPLLGRP_S2FUSER1CLK_CNT),//h2f_user1_base_clk=peri_vco_clk/10 = 4eosc1_clk
			CLKMGR_PERPLLGRP_DIV_USBCLK_SET(
				CONFIG_HPS_PERPLLGRP_DIV_USBCLK) |//usb_mp_clk=periph_base_clk=peri_vco_clk/5 = 8eosc1_clk
			CLKMGR_PERPLLGRP_DIV_SPIMCLK_SET(
				CONFIG_HPS_PERPLLGRP_DIV_SPIMCLK) |//spi_m_clk=periph_base_clk=peri_vco_clk/5 = 8eosc1_clk
			CLKMGR_PERPLLGRP_DIV_CAN0CLK_SET(
				CONFIG_HPS_PERPLLGRP_DIV_CAN0CLK) |//can0_clk=periph_base_clk/2=peri_vco_clk/10 = 4eosc1_clk
			CLKMGR_PERPLLGRP_DIV_CAN1CLK_SET(
				CONFIG_HPS_PERPLLGRP_DIV_CAN1CLK),//can1_clk=periph_base_clk/2=peri_vco_clk/10 = 4eosc1_clk
			CLKMGR_PERPLLGRP_GPIODIV_GPIODBCLK_SET(
				CONFIG_HPS_PERPLLGRP_GPIODIV_GPIODBCLK),//gpio_db_clk=periph_base_clk/6250=peri_vco_clk/31250 = 4eosc1_clk/3125
			CLKMGR_PERPLLGRP_SRC_QSPI_SET(
				CONFIG_HPS_PERPLLGRP_SRC_QSPI) |//设置qspi_clk的时钟源为main_qspi_clk
			CLKMGR_PERPLLGRP_SRC_NAND_SET(
				CONFIG_HPS_PERPLLGRP_SRC_NAND) |//设置nand_clk的时钟源为periph_nand_sdmmc_clk
			CLKMGR_PERPLLGRP_SRC_SDMMC_SET(
				CONFIG_HPS_PERPLLGRP_SRC_SDMMC),//设置sdmmc_clk的时钟源为periph_nand_sdmmc_clk
	
			/* sdram pll group */
			SDR_VCO_BASE,	//设置sdram PLL组的时钟源为eosc1_clk，sdr_vco_clk=(63+1)eosc1_clk/(1+1)=32eosc1_clk
			CLKMGR_SDRPLLGRP_DDRDQSCLK_PHASE_SET(
				CONFIG_HPS_SDRPLLGRP_DDRDQSCLK_PHASE) | //设置ddr_dqs_base_clk和sdr_voc_clk同一相位
			CLKMGR_SDRPLLGRP_DDRDQSCLK_CNT_SET(
				CONFIG_HPS_SDRPLLGRP_DDRDQSCLK_CNT),	//ddr_dqs_clk=sdr_vco_clk/2 = 16eosc1_clk
			CLKMGR_SDRPLLGRP_DDR2XDQSCLK_PHASE_SET(
				CONFIG_HPS_SDRPLLGRP_DDR2XDQSCLK_PHASE) |	//设置ddr_2x_dqs_clk和sdr_vco_clk同一相位
			CLKMGR_SDRPLLGRP_DDR2XDQSCLK_CNT_SET(
				CONFIG_HPS_SDRPLLGRP_DDR2XDQSCLK_CNT),	//ddr_2x_dqs_clk = sdr_vco_clk/2 = 16eosc1_clk
			CLKMGR_SDRPLLGRP_DDRDQCLK_PHASE_SET(
				CONFIG_HPS_SDRPLLGRP_DDRDQCLK_PHASE) |	//设置ddr_dq_clk的相位比sdr_voc_clk增加4*45°
			CLKMGR_SDRPLLGRP_DDRDQCLK_CNT_SET(
				CONFIG_HPS_SDRPLLGRP_DDRDQCLK_CNT),		//ddr_dq_clk = sdr_vco_clk/2 = 16eosc1_clk
			CLKMGR_SDRPLLGRP_S2FUSER2CLK_PHASE_SET(
				CONFIG_HPS_SDRPLLGRP_S2FUSER2CLK_PHASE) |	//设置s2f_user2_clk和sdr_vco_clk同一相位
			CLKMGR_SDRPLLGRP_S2FUSER2CLK_CNT_SET(
				CONFIG_HPS_SDRPLLGRP_S2FUSER2CLK_CNT),		////s2f_user2_clk = sdr_vco_clk/2 = 16eosc1_clk
		};
	
		WATCHDOG_RESET();		//使能l4wd0，刷新计数
	
		debug("Freezing all I/O banks\n");
		/* freeze all IO banks */		
		sys_mgr_frzctrl_freeze_req(FREEZE_CHANNEL_0,
			FREEZE_CONTROLLER_FSM_SW);	//冻结通道0的bank7D和bank7E
		sys_mgr_frzctrl_freeze_req(FREEZE_CHANNEL_1,
			FREEZE_CONTROLLER_FSM_SW);	//冻结通道1的bank7B和bank7C
		sys_mgr_frzctrl_freeze_req(FREEZE_CHANNEL_2,
			FREEZE_CONTROLLER_FSM_SW);	//冻结通道2的bank7A
		sys_mgr_frzctrl_freeze_req(FREEZE_CHANNEL_3,
			FREEZE_CONTROLLER_FSM_SW);	//冻结通道3的bank6
	
		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		debug("Asserting reset to all except L4WD and SDRAM\n");
		reset_assert_all_peripherals_except_l4wd0_osc1timer1();	//除l4wd0和osc1timer1外的外设复位置1，使pll重配置期间没有跳变
		reset_assert_all_bridges();		//三个桥复位置1



		debug("Deassert reset for OSC1 Timer\n");
		/*
		 * deassert reset for osc1timer0. We need this for delay
		 * function that required during PLL re-configuration
		 */
		reset_deassert_osc1timer0();	//osc1timer0复位置0
	
		debug("Init timer\n");
		/* init timer for enabling delay function */
		timer_init();	//加载0xffffffff新计数，用户计数模式，使能osc1timer1


	#ifndef CONFIG_SOCFPGA_VIRTUAL_TARGET
	
		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		debug("Reconfigure Clock Manager\n");
		cm_basic_init(&cm_default_cfg);		//利用上文设置好的数据配置main pll,Peripheral PLL,和SDRAM PLL三个时钟组的时钟
	
		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		/* Skip configuration is warm reset happen and WARMRSTCFGIO set */
	
		if (((readl(CONFIG_SYSMGR_ROMCODEGRP_CTRL) &
			SYSMGR_ROMCODEGRP_CTRL_WARMRSTCFGIO) == 0) ||
			((rst_mgr_status & RSTMGR_WARMRST_MASK) == 0)) {	//如果warmrstcfgio置1则跳过配置IO
	
			/* Enable handshake bit with BootROM */
			setbits_le32(CONFIG_SYSMGR_ROMCODEGRP_CTRL,
				SYSMGR_ROMCODEGRP_CTRL_WARMRSTCFGIO);	//warmrstcfgio置1,引导rom将会在热复位后配置IO
	
			debug("Configure IOCSR\n");
	
			/* 启动启动扫描管理器对HPS I/O配置位移寄存器进行配置 */
			scan_mgr_io_scan_chain_prg(
				IO_SCAN_CHAIN_0,
				CONFIG_HPS_IOCSR_SCANCHAIN0_LENGTH,
				iocsr_scan_chain0_table);
			scan_mgr_io_scan_chain_prg(
				IO_SCAN_CHAIN_1,
				CONFIG_HPS_IOCSR_SCANCHAIN1_LENGTH,
				iocsr_scan_chain1_table);
			scan_mgr_io_scan_chain_prg(
				IO_SCAN_CHAIN_2,
				CONFIG_HPS_IOCSR_SCANCHAIN2_LENGTH,
				iocsr_scan_chain2_table);
			scan_mgr_io_scan_chain_prg(
				IO_SCAN_CHAIN_3,
				CONFIG_HPS_IOCSR_SCANCHAIN3_LENGTH,
				iocsr_scan_chain3_table);
	
			/* Clear handshake bit with BootROM */
			DEBUG_MEMORY
			clrbits_le32(CONFIG_SYSMGR_ROMCODEGRP_CTRL,
				SYSMGR_ROMCODEGRP_CTRL_WARMRSTCFGIO);	//warmrstcfgio置0,引导rom不会在热复位后配置IO
	
		}
	
		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		/* Skip configuration is warm reset happen and WARMRSTCFGPINMUX set */
		if (((readl(CONFIG_SYSMGR_ROMCODEGRP_CTRL) &
			SYSMGR_ROMCODEGRP_CTRL_WARMRSTCFGPINMUX) == 0) ||
			((rst_mgr_status & RSTMGR_WARMRST_MASK) == 0)) {	//如果warmrstcfgpinmux置1则跳过配置pinmux


			/* Enable handshake bit with BootROM */
			setbits_le32(CONFIG_SYSMGR_ROMCODEGRP_CTRL,
				SYSMGR_ROMCODEGRP_CTRL_WARMRSTCFGPINMUX);		//warmrstcfgpinmux置1,引导rom将会在热复位后配置引脚复用
	
			debug("Configure PINMUX\n");
			sysmgr_pinmux_init();		//通过系统管理器配置管脚复用		
	
			/* Clear handshake bit with BootROM */
			DEBUG_MEMORY
			clrbits_le32(CONFIG_SYSMGR_ROMCODEGRP_CTRL,
				SYSMGR_ROMCODEGRP_CTRL_WARMRSTCFGPINMUX);	//warmrstcfgpinmux置0,引导rom不会在热复位后配置引脚复用
	
		}


		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		debug("Deasserting resets\n");
		/* de-assert reset for peripherals and bridges based on handoff */
	
		reset_deassert_peripherals_handoff();	//peripherals复位置1？不是置0码？（在boot_bm_on_cpu1中释放）暖复位握手fpga,etr,sdram
	
		reset_deassert_bridges_handoff();		//bridges复位置1？不是置0码？（在boot_bm_on_cpu1中释放）


		WATCHDOG_RESET();		//使能l4wd0，刷新计数
	
		debug("Unfreezing/Thaw all I/O banks\n");
		/* unfreeze / thaw all IO banks */
		sys_mgr_frzctrl_thaw_req(FREEZE_CHANNEL_0,
			FREEZE_CONTROLLER_FSM_SW);	//解冻通道0的bank7D和bank7E
		sys_mgr_frzctrl_thaw_req(FREEZE_CHANNEL_1,
			FREEZE_CONTROLLER_FSM_SW);	//解冻通道1的bank7B和bank7C
		sys_mgr_frzctrl_thaw_req(FREEZE_CHANNEL_2,
			FREEZE_CONTROLLER_FSM_SW);	//解冻通道2的bank7A
		sys_mgr_frzctrl_thaw_req(FREEZE_CHANNEL_3,
			FREEZE_CONTROLLER_FSM_SW);	//解冻通道3的bank6


		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		/* enable console uart printing */
		preloader_console_init();	//common\spl\spl.c,初始化串口控制器，波特率57600，然后可以用串口在spl阶段调试


		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		puts("SDRAM : Initializing MMR registers\n");
		/* SDRAM MMR initialization */
		if (sdram_mmr_init_full() != 0)		//初始化SDRAM控制器
			hang();


		WATCHDOG_RESET();	//使能l4wd0，刷新计数


		puts("SDRAM : Calibrationg PHY\n");
		/* SDRAM calibration */
		if (sdram_calibration_full() == 0)	//校准sdram phy
			hang();
	
		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		/* configure the interconnect NIC-301 security */
		nic301_slave_ns();	//arch\arm\cpu\armv7\socfpga\nic301.c
							  设置只有非安全主机才能访问LWHPS2FPGA从机，HPS2FPGA从机，MPU ACP从机，ROM从机,ON CHIP RAM从机，SDRAM DATA从机
	
		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		init_boot_params();		//boot_params_ptr = 0,片上ram的开始地址
	}

2.**amp\_share\_param\_init()到spl\_mmc\_load\_image()**的初始化sd/mmc和加载启动bm阶段详细分析见**bm\_load\_boot.pdf**

- **cpu0初始化sd/mmc控制器和设备**
- **cpu0将bm镜像加载到sdram的0x1E000000地址处**
- **cpu1从复位释放开始从0x1E000000运行bm**
- **cpu0等待cpu1中的bm加载配置完rbf**
- **cpu0从sd卡加载u-boot镜像到sdram的0x1000000地址处启动**


**3.jump\_to\_image\_no\_args()**(common\spl\spl.c):spl最后一个阶段，跳转到u-boot启动运行

	static void __noreturn jump_to_image_no_args(void)
	{
		typedef void __noreturn (*image_entry_noargs_t)(u32 *);
		image_entry_noargs_t image_entry =
				(image_entry_noargs_t) spl_image.entry_point;	//获取u-boot入口点地址（u-boot第一条指令的地址）
	
		WATCHDOG_RESET();		//使能l4wd0，刷新计数
	
		u32 calculated_crc;
		if (spl_image.crc_size != 0) {
			debug("Verifying Checksum ... ");
			calculated_crc = crc32_wd(0,
				(unsigned char *)spl_image.entry_point,
				spl_image.crc_size, CHUNKSZ_CRC32);
			if (calculated_crc != spl_image.crc) {
				puts("Bad image with mismatched CRC\n");
				debug("CRC calculate from 0x%08x "
					"with length 0x%08x\n",
					spl_image.entry_point, spl_image.size);
				debug("CRC Result : Expected 0x%08x "
					"Calculated 0x%08x\n",
					spl_image.crc, calculated_crc);
				hang();
			} else
				debug("OK\n");
		}
	
		debug("image entry point: 0x%X\n", spl_image.entry_point);
	
		/* to indicate a successful run */
		writel(CONFIG_PRELOADER_STATE_VALID, CONFIG_PRELOADER_STATE_REG);	//将魔数写入initswstate寄存器中，代表preloder完成加载下一阶段镜像到sdram的任务
	
		u32 boot_params_ptr_addr = (u32)&boot_params_ptr;	//boot_params_ptr=0
	
		WATCHDOG_RESET();	//使能l4wd0，刷新计数
	
		image_entry((u32 *)boot_params_ptr_addr);	//跳转到u-boot入口点地址启动u-boot，spl结束
	 
	}

## BM流程详细分析：
### BM入口
**从连接脚本bm.lds中可以得到bm入口在start,运行时应该在地址0x1E000000，而且makefile没有另外的设置，所以连接地址就是0x1E000000。**

	OUTPUT_ARCH(arm)
	ENTRY(start)
	
	SECTIONS
	{
		. = 0x1E000000;
		.text	:
		{
		        AMP_BM_REF_DESIGN/start.o (.text)
			*(.text)
		}
		.rodata : { *(.rodata) }
		.data :   { *(.data) }
		.bss : { 
			bss_start = .;
			*(.bss) 
			. = ALIGN(4);
			bss_end = .;
		}
	}

### BM总体流程
**start-->reset-->jump\_to\_virt-->clbss\_l-->main**

### BM流程各阶段详细分析
**start-->reset**(start.s)

	start:	
	        b	reset
	.....................................
	reset:
	
	@@ 设置为svc管理模式，禁止IRQ和FIQ中断
	mrs	r0, cpsr			
	bic	r0, r0, #0x1f
	orr	r0, r0, #0xd3
	msr	cpsr_cxsf,r0	



	@@ 设置cache 
	mov     r0, #0					@ r0 = 0x00000000
 	mcr     p15, 0, r0, c7, c5, 0   @ 使失效I-cache
    mcr     p15, 2, r0, c0, c0, 0	@ 将0x00000000写入CSSELR(Cache Size Selection Register)
    								@ 选择数据或者统一的L1 cache	
    								
    mrc     p15, 1, r0, c0, c0, 0	@ 将CCSIDR(Cache Size ID Registers)读到r0
    ldr     r1, =0x7fff				@ r1 = 0x7fff
    and     r2, r1, r0, lsr #13		@ r2 = r1 & (r0 >> 13) 
    								@ 将CCSIDR第13位和以后的位的往右移13，再屏蔽后面四个位，只保留NumSets
    ldr     r1, =0x3ff				@ r1 = 0x3fff
    and     r3, r1, r0, lsr #3      @ r3 = r1 & (r0 >> 3)   (Associativity of cache(NumWays) - 1)
    								@ 将CCSIDR第3位和以后的位的往右移13，再屏蔽后面19个位，只保留Associativity
    add     r2, r2, #1              @ NumSets
       
    and     r0, r0, #0x7	@ r0 = r0 & 7,只保留CCSIDR前三位，(Log2(Number of words in cache line)) -2
    add     r0, r0, #4      @ SetShift
    
    clz     r1, r3          @ WayShift   
    add     r4, r3, #1      @ NumWays
      
    sub     r2, r2, #1      @ NumSets--
    mov     r3, r4          @ Temp = NumWays
     
    subs    r3, r3, #1      @ Temp--
    mov     r5, r3, lsl r1
    mov     r6, r2, lsl r0
    orr     r5, r5, r6      @ Reg = (Temp<<WayShift)|(NumSets<<SetShift)
    mcr     p15, 0, r5, c7, c6, 2
    bgt     2b
    cmp     r2, #0
    bgt     1b
    dsb
    isb


	@@ 将异常向量表映射到0x00000000 
	mrc     p15, 0, r0, c1, c0, 0   @ SCTLR读进r0
	bic     r0, #(0x1<<13)    		@ V = 0
	mcr     p15, 0, r0, c1, c0, 0   @ Write CP15 SCTRL Register


    @@ 将Monitor mode和Hyp mode之外的异常向量地址重新映射到start处 
    ldr     r0, =start  
    mcr     p15, 0, r0, c12, c0, 0  @ Set VBAR
    
    @@ 浮点相关设置
    mrc     p15, 0, r0, c1, c0, 2	@ CPACR读进r0	
    orr     r0, r0, #(0x3  <<20)	@ 打开r0的第20和21位(所有pl可以访问cp10)
    orr     r0, r0, #(0x3  <<22)	@ 打开r0的第22和23位(所有pl可以访问cp11)	
    mcr     p15, 0, r0, c1, c0, 2	@ 写入CPACR
    isb								@ 指令同步
    mov     r0, #(0x1 << 30)  		
    @vmsr	fpexc, r0 <- not correctly supported in binutils 2.22
    mcr     p10, 7, r0, cr8, cr0, 0 @ alternate mnemonic for vmsr fpexec, r0
     
    @@ 设置栈
    ldr sp, =usr_stack_end


	@@ 建立设置页表
	bl  build_PGTL
	
	@@ 设置页表的基地址
	orr    r0, r0, #TTB_FLAGS_SMP		@ 打开TTBR0的第1，3，5和6位
	mcr    p15, 0, r0, c2, c0, 0		
	
	isb
	dsb
	
	@@ 设置内存访问不在检查页表的权限位 
	ldr    r0, =0xFFFFFFFF             @ Set full access for all domains
	mcr    p15, 0, r0, c3, c0, 0		
	
	@@ 使能smp,不在armv7中定义，在a9中定义
	mov r0, #0x41
	mcr	p15, 0, r0, c1, c0, 1
	isb
	dsb
	
	mov	r0, #0
	mcr	p15, 0, r0, c7, c5, 0		@ 失效icache
	dsb
	mcr	p15, 0, r0, c8, c7, 0       @ 失效TLBs


	dsb
	isb

**jump\_to\_virt**(start.S)

	ldr sp, =usr_stack_end
	@@ set the stack pinters for svc and irq mode. We don@t do anything
	@@ that needs a stack in the other modes so do not bother 
	@@ setting them up
	msr     CPSR_c, #Mode_IRQ | I_Bit | F_Bit		@ 切换为IRQ模式并开irq中断和fiq中断
	ldr	sp,=irq_stack_end							@ 设置IRQ模式的栈指针
	
	msr     CPSR_c, #Mode_SVC | I_Bit | F_Bit		@ 切换回SVC模式并开irq中断和fiq中断
	
	@@ 清bss 
	ldr	r0,=bss_start		@ find start of bss segment
	ldr	r1,=bss_end			@ stop here
	mov	r2, #0x00000000		@ clear value


**clbss\_l**(start.S)
	



## U-BOOT流程详细分析：
### U-BOOT入口
**从连接脚本spl/u-boot.lds的开始片段中可以得到U-BOOT入口在_start,运行时应该在地址0x00000000，但是经过makefile的设置，从编译输出打印信息可以看到真正的连接地址是0x01000040（在SPL阶段将片上RAM映射到0x00000000，另外片上ram还会映射到0xffff0000）处。其实u-boot payload加载的地址是0x01000040，由于位置无关技术，刚开始的部分可以正常运行，直到重定位代码。**
	
	连接脚本
	OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
	OUTPUT_ARCH(arm)
	ENTRY(_start)
	SECTIONS
	{
	 . = 0x00000000;
	 . = ALIGN(4);
	 .text :
	 {
	  __image_copy_start = .;
	  arch/arm/cpu/armv7/start.o (.text)
	  *(.text)
	 }
	
	编译打印信息
	cd /home/hn/TEST/u-boot-altera-2012.10 && arm-linux-gnueabihf-ld.bfd  -pie -T u-boot.lds -Bstatic -Ttext 0x01000040 $UNDEF_SYM arch/arm/cpu/armv7/start.o ......

### U-BOOT总体流程
**\_start-->reset-->save\_boot\_params(什么都没做)-->设置管理模式，禁止IRQ和FIQ中断-->board\_init\_f-->relocate_code -->board\_init\_r-->main\_loop()**
	

### U-BOOT流程各阶段详细分析

**\_start --> reset --> save\_boot\_params(什么都没做) --> 设置管理模式，禁止IRQ和FIQ中断**(arch\arm\cpu\armv7\start.S)

	.globl _start
	_start: b	reset
	
	......................................................
	
	reset:
		bl	save_boot_params
		/* 设置为svc管理模式，禁止IRQ和FIQ中断 */
		mrs	r0, cpsr		@ 将cpsr寄存器读到r0中
		bic	r0, r0, #0x1f	@ 将读到r0中的cpsr副本前5位清零
		orr	r0, r0, #0xd3	@ 打开r0中的cpsr副本第0，1，4，6，7位
		msr	cpsr,r0			@ 将处理好后的值写入cpsr寄存器中


**board\_init\_f(0)**(arch\arm\lib\board.c)


	void board_init_f(ulong bootflag)
	{
		bd_t *bd;
		init_fnc_t **init_fnc_ptr;
		gd_t *id;
		ulong addr, addr_sp;



		bootstage_mark_name(BOOTSTAGE_ID_START_UBOOT_F, "board_init_f");
	
		/* Pointer is writable since we allocated a register for it */
		gd = (gd_t *) ((CONFIG_SYS_INIT_SP_ADDR) & ~0x07);	//gd指向的区域起始地址，0xffff0000到该地址前用作临时栈
		/* compiler optimization barrier needed for GCC >= 3.4 */
		__asm__ __volatile__("": : :"memory");
	
		memset((void *)gd, 0, sizeof(gd_t));		//初始化gd区域
	
		gd->mon_len = _bss_end_ofs;		//__bss_end__ - _start
	
		/* Allow the early environment to override the fdt address */
		gd->fdt_blob = (void *)getenv_ulong("fdtcontroladdr", 16,
							(uintptr_t)gd->fdt_blob);		//从环境变量中得到设备树地址	
	
		for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {		//大部分初始化工作在这里
			if ((*init_fnc_ptr)() != 0) {
				hang ();
			}
		}
		
		/*
		init_fnc_t *init_sequence[] = {
			arch_cpu_init,		//什么都没做，返回0
			board_early_init_f,	//关闭l4wd0看门狗
			timer_init,			//加载新计数，使能osc1timer1
			env_init,			//gd->env_addr = (ulong)&default_environment[0]; gd->env_valid = 1;(common\env_mmc.c)
			init_baudrate,		//gd->baudrate = getenv_ulong("baudrate", 10, CONFIG_BAUDRATE) = 57600
			serial_init,		//根据gd的波特率设置串口0的控制器(drivers\serial\serial.c)
			console_init_f,		//gd->have_console = 1;可以用printf,puts等打印函数了
			display_banner,		//打印点东西
			print_cpuinfo,		//打印CPU型号信息
			checkboard,			//打印板型号信息
			dram_init,			//gd->ram_size = get_ram_size(0x0, 0x40000000); 1G
			NULL,
		};	
		*/
	
		debug("monitor len: %08lX\n", gd->mon_len);
		/*
		 * Ram is setup, size stored in gd !!
		 */
		debug("ramsize: %08lX\n", gd->ram_size);
	
		addr = CONFIG_SYS_SDRAM_BASE + gd->ram_size;		//0x0 + gd->ram_size
	
		/* reserve TLB table */
		addr -= (4096 * 4);
	
		/* round down to next 64 kB limit */
		addr &= ~(0x10000 - 1);
	
		gd->tlb_addr = addr;		//保留给tlb用
		debug("TLB table at: %08lx\n", addr);
	
		/* round down to next 4 kB limit */
		addr &= ~(4096 - 1);
		debug("Top of RAM usable for U-Boot at: %08lx\n", addr);
	
		/*
		 * reserve memory for U-Boot code, data & bss
		 * round down to next 4 kB limit
		 */
		addr -= gd->mon_len;		//保留给u-boot的代码，数据，bss段
		addr &= ~(4096 - 1);
	
		debug("Reserving %ldk for U-Boot at: %08lx\n", gd->mon_len >> 10, addr);
	
		/*
		 * reserve memory for malloc() arena
		 */
		addr_sp = addr - TOTAL_MALLOC_LEN;
		debug("Reserving %dk for malloc() at: %08lx\n",
				TOTAL_MALLOC_LEN >> 10, addr_sp);
		/*
		 * (permanently) allocate a Board Info struct
		 * and a permanent copy of the "global" data
		 */
		addr_sp -= sizeof (bd_t);
		bd = (bd_t *) addr_sp;		//保留给bd结构
		gd->bd = bd;				
		debug("Reserving %zu Bytes for Board Info at: %08lx\n",
				sizeof (bd_t), addr_sp);



		addr_sp -= sizeof (gd_t);		
		id = (gd_t *) addr_sp;		//保留给gd结构
		debug("Reserving %zu Bytes for Global Data at: %08lx\n",
				sizeof (gd_t), addr_sp);
	
		/* setup stackpointer for exeptions */
		gd->irq_sp = addr_sp;		//gd->irq_sp = id
	
		/* leave 3 words for abort-stack    */
		addr_sp -= 12;
	
		/* 8-byte alignment for ABI compliance */
		addr_sp &= ~0x07;



		debug("New Stack Pointer is: %08lx\n", addr_sp);



		gd->bd->bi_baudrate = gd->baudrate;		//bd波特率与gd同步
		/* Ram ist board specific, so move it to board code ... */
		dram_init_banksize();    //gd->bd->bi_dram[0].start = 0x0; gd->bd->bi_dram[0].size = gd->ram_size
		display_dram_config();	/* and display it */
	
		gd->relocaddr = addr;			//重地位地址
		gd->start_addr_sp = addr_sp;		//新的栈指针
		gd->reloc_off = addr - _TEXT_BASE;	//重定位地址到_TEXT_BASE的重定位偏移
		debug("relocation Offset is: %08lx\n", gd->reloc_off);
		memcpy(id, (void *)gd, sizeof(gd_t));	//将保存到r8寄存器的gd指针指向的数据写入sdram对应地址中传送给linux
	
		relocate_code(addr_sp, id, addr);		//重地位u-boot代码
	
		/* NOTREACHED - relocate_code() does not return */
	}

**relocate\_code(addr\_sp, gd, addr)**(arch\arm\cpu\armv7\start.S)

**不返回**

	ENTRY(relocate_code)
		mov	r4, r0	/* save addr_sp */					@ r4=addr_sp
		mov	r5, r1	/* save addr of gd */				@ r5=gd	 (gd地址)
		mov	r6, r2	/* save addr of destination */		@ r6=addr(重定位地址)
	
		/* Set up the stack */
	stack_setup:
		mov	sp, r4
	
		adr	r0, _start														@将_start标号处地址给r0(0x01000040)
		cmp	r0, r6
		moveq	r9, #0		/* no relocation. relocation offset(r9) = 0 */	@如果r0==r6则r9=0
		beq	clear_bss		/* skip relocation */							@如果r0==r6则不需要重定位，直接跳到后面
		mov	r1, r6			/* r1 <- scratch for copy_loop */				@如果r0!=r6则将重定位目标地址给r1
		ldr	r3, _image_copy_end_ofs											@r3=将.rel.dyn之前的镜像长度
		add	r2, r0, r3		/* r2 <- source end address	    */				@r2=r3+r0
	
	copy_loop:
		ldmia	r0!, {r9-r10}		/* copy from source address [r0]    */	@r9=[r0],r10=[r0+4],r0=r0+4+4
		stmia	r1!, {r9-r10}		/* copy to   target address [r1]    */	@[r1]=r9,[r1+4]=r10,r1=r1+4+4
		cmp	r0, r2			/* until source end address [r2]    */			@r0-r2
		blo	copy_loop														@如果r0-r2<0继续copy_loop
																			@@将_image_copy_start和_image_copy_end之间的数据复制到gd->relocaddr地址处
	
		/*
		 * fix .rel.dyn relocations
		 */
		ldr	r0, _TEXT_BASE		/* r0 <- Text base */						@r0=0x01000040
		sub	r9, r6, r0		/* r9 <- relocation offset */					@r9=r6-r0=gd->reloc_off
		ldr	r10, _dynsym_start_ofs	/* r10 <- sym table ofs */				@r10 = __dynsym_start - _start
		add	r10, r10, r0		/* r10 <- sym table in FLASH */				@r10 = .dynsym段开头的加载地址
		ldr	r2, _rel_dyn_start_ofs	/* r2 <- rel dyn start ofs */			@r2 = __rel_dyn_start - _start
		add	r2, r2, r0		/* r2 <- rel dyn start in FLASH */				@r2 = .rel.dyn段开头的加载地址
		ldr	r3, _rel_dyn_end_ofs	/* r3 <- rel dyn end ofs */				@r3 = __rel_dyn_end - _start
		add	r3, r3, r0		/* r3 <- rel dyn end in FLASH */					@r3 = .rel.dyn段末尾的加载地址
	fixloop:
		ldr	r0, [r2]		/* r0 <- location to fix up, IN FLASH! */		
		add	r0, r0, r9		/* r0 <- location to fix up in RAM */
		ldr	r1, [r2, #4]
		and	r7, r1, #0xff
		cmp	r7, #23			/* relative fixup? */
		beq	fixrel
		cmp	r7, #2			/* absolute fixup? */
		beq	fixabs
		/* ignore unknown type of fixup */
		b	fixnext												
	fixabs:
		/* absolute fix: set location to (offset) symbol value */
		mov	r1, r1, LSR #4		/* r1 <- symbol index in .dynsym */
		add	r1, r10, r1		/* r1 <- address of symbol in table */
		ldr	r1, [r1, #4]		/* r1 <- symbol value */
		add	r1, r1, r9		/* r1 <- relocated sym addr */
		b	fixnext
	fixrel:
		/* relative fix: increase location by offset */
		ldr	r1, [r0]
		add	r1, r1, r9
	fixnext:
		str	r1, [r0]
		add	r2, r2, #8		/* each rel.dyn entry is 8 bytes */
		cmp	r2, r3
		blo	fixloop
		b	clear_bss
	_rel_dyn_start_ofs:
		.word __rel_dyn_start - _start
	_rel_dyn_end_ofs:
		.word __rel_dyn_end - _start
	_dynsym_start_ofs:
		.word __dynsym_start - _start
	
	clear_bss:
		ldr	r0, _bss_start_ofs						@r0 = __bss_start - _start
		ldr	r1, _bss_end_ofs						@r1 = __bss_end - _start
		mov	r4, r6			/* reloc addr */		@r4 = addr	
		add	r0, r0, r4								@r0 = __bss_start - _start + addr(新的__bss_start)
		add	r1, r1, r4								@r1 = __bss_end - _start + addr(新的__bss_end)
		mov	r2, #0x00000000		/* clear */			@r2 = #0x00000000
	
	clbss_l:
		cmp	r0, r1			/* clear loop... */					@r0-r1
		bhs	clbss_e			/* if reached end of bss, exit */	@r0大于或等于r1则跳转到clbss_e
		str	r2, [r0]											@清零r0指向的一个4字节的内存
		add	r0, r0, #4											@r0+=4
		b	clbss_l												@循环继续清0
	clbss_e:
	
	/*
	 * We are done. Do not return, instead branch to second part of board
	 * initialization, now running from RAM.
	 */
	jump_2_ram:
	/*
	 * If I-cache is enabled invalidate it
	 */
	#ifndef CONFIG_SYS_ICACHE_OFF
		mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache		@ 开icache
		mcr     p15, 0, r0, c7, c10, 4	@ DSB				@ DSB，数据同步屏障，内存操作完成才进行后面指令的操作
		mcr     p15, 0, r0, c7, c5, 4	@ ISB				@ ISB，指令同步屏障，清空流水线
	#endif
	/*
	 * Move vector table
	 */
	#if !defined(CONFIG_TEGRA20)
		/* Set vector address in CP15 VBAR register */
		ldr     r0, =_start							@0x01000040
		add     r0, r0, r9							@重定位后的_start地址
		mcr     p15, 0, r0, c12, c0, 0  @Set VBAR	@将异常向量地址设定为重定位后的_start地址处
	#endif /* !Tegra20 */
	
		ldr	r0, _board_init_r_ofs			@ r0 = board_init_r - _start		
		adr	r1, _start						@ r1 = _start	
		add	lr, r0, r1						@ lr = board_init_r
		add	lr, lr, r9						@ 重定位后的board_init_r地址lr = board_init_r + gd->reloc_off
		/* setup parameters for board_init_r */
		mov	r0, r5		/* gd_t */			@ 设置board_init_r第一个参数id
		mov	r1, r6		/* dest_addr */		@ 设置board_init_r第二个参数addr
		/* jump to it ... */
		mov	pc, lr							@ 跳转到重定位后的board_init_r地址去运行，拜拜，不回来了
	
	_board_init_r_ofs:
		.word board_init_r - _start
	ENDPROC(relocate_code)


**board\_init\_r(id, addr)**(arch\arm\lib\board.c)

	void board_init_r(gd_t *id, ulong dest_addr)
	{
		ulong malloc_start;
	
		gd = id;	
	
		gd->flags |= GD_FLG_RELOC;	/* tell others: relocation done */
		bootstage_mark_name(BOOTSTAGE_ID_START_UBOOT_R, "board_init_r");
	
		monitor_flash_len = _end_ofs;		//__image_copy_end - _start
	
		/* Enable caches */
		enable_caches();					//使能D-cache. I-cache已经在start.S使能了
	
		debug("monitor flash len: %08lX\n", monitor_flash_len);
		board_init();	/* Setup chipselects */	//gd->bd->bi_boot_params = 0x00000100; tag或者设备树地址
		/*
		 * TODO: printing of the clock inforamtion of the board is now
		 * implemented as part of bdinfo command. Currently only support for
		 * davinci SOC's is added. Remove this check once all the board
		 * implement this.
		 */
	
		debug("Now running in RAM - U-Boot at: %08lx\n", dest_addr);


		/* The Malloc area is immediately below the monitor copy in DRAM */
		malloc_start = dest_addr - TOTAL_MALLOC_LEN;			//malloc堆的起始地址
		mem_malloc_init (malloc_start, TOTAL_MALLOC_LEN);	//初始化malloc堆
	
	    puts("MMC:   ");
	    mmc_initialize(gd->bd);								//初始化&mmc_devices链表，初始化一个mmc结构体，注册mmc设备（将mmc添加到&mmc_devices链表中）
	
		/* initialize environment */
		env_relocate();		//通过sd卡初始化环境变量,如果没有或者其他问题则使用默认环境变量，存在env_htab中(env_mmc.c)
		/* get the devices list going. */
		stdio_init();		//初始化devs.list链表，注册一个stdio_dev设备到该链表中，设备名为“serial”(stdio.c)
	
		jumptable_init();	//分配内存给跳转表并赋值，模拟系统调用
	
		console_init_r();	//gd->flags |= GD_FLG_DEVINIT;设置stdio_devices[]三个设备stdio,stdout,stderr都为stdio_init()中注册的名为“serial”的设备
	
		/* miscellaneous platform dependent initialisations */
		misc_init_r();
	
		 /* set up exceptions */
		interrupt_init();			//没有使用
		/* enable exceptions */
		enable_interrupts();		//没有使用
	
		/* Initialize from environment */
		load_addr = getenv_ulong("loadaddr", 16, load_addr);		//获取环境变量loadaddr
	
		board_late_init();
	
		puts("Net:   ");
		eth_initialize(gd->bd);
	
		/* main_loop() can return to retry autoboot, if so just run it again. */
		for (;;) {
			main_loop();			//进入最后加载启动linux阶段
		}
	
		/* NOTREACHED - no way out of command loop except booting */
	}

**main_loop()(common\main.c)**





###SPL疑点
- writel(TIMER\_LOAD\_VAL, &timer\_base->curr\_val);	//不是只读的吗？   arch\arm\cpu\armv7\socfpga\timer.c
- I/O如何配置硬件参数 (Quartus® II得到？)
- BOOTSEL(BSEL)和CLKSEL(CSEL)引脚怎么在引导rom之前设置

###SPL回头分析
- 中断详细分析
- 定时器详细分析

###BM疑点
- .map文件
- @@ enable smp

		mov r0, #0x41
		mcr	p15, 0, r0, c1, c0, 1

- @ r0 = r1 & 0x1f r1是什么？通过r0传进来的异常id有什么用？

		AND     R0, R1, #OS_CPU_ARM_MODE_MASK			

- TTBCR怎么控制使用ttbr0或者ttbr1

###BM回头分析
- 双核测试
- /* 开mmu,虚拟地址开始作用 */处的内敛汇编


###U-BOOT疑点
- 环境变量bootargs和baudrate，CONFIG_BAUDRATE的波特率参数不一样，不知道有什么影响
- gd->bd->bi\_boot\_params = 0x00000100(0x00000100是协议好的tag或者设备树地址吗)

###U-BOOT回头分析
- init\_sequence中的serial\_init和dram\_init(arch\arm\lib\board.c\board\_init\_f())
- .rel.dyn和.dynsym重定位(重定位.zip和深入理解bootloader)
- main_loop()
- spl和u-boot串口打印函数详细分析









​			