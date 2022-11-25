# AMP双核方式加载启动bm过程

```c
/* common\spl\spl.c */
(1)amp_share_param_init();	//0x1E100000,u-boot, bare metal, linux的共享内存区域地址 

(2)load_bm_start();		//获取加载bm.bin的开始时间

(3)spl_mmc_probe();		//初始化sd/mmc控制器和设备

(4)spl_mmc_load_bm_image_mbr();	//从sd/mmc设备中加载bm到sdram中

(5)load_bm_end();		//获取加载bm.bin的结束时间 

(6)spl_mmc_save_func();		//保存spl_mmc_probe等函数到共享内存给bm使用

(7)boot_bm_on_cpu1();		//从cpu1启动bm

(8)cpu0_wait_cpu1_load_rbf();	//等待cpu1中运行的bm加载rbf完成

(9)spl_mmc_load_image();	//将u-boot镜像加载到sdram中
```

### (1)amp\_share\_param\_init()

```c
void amp_share_param_init(void)
{
	memset(asp, 0x0, sizeof(struct amp_share_param));
}
```

将共享内存地址asp(0x1E100000)清零,该地址是u-boot,bm,linux的一个共享参数内存区域。

### (2)load\_bm\_start()

```c
void load_bm_start(void)
{
	reset_timer();
	asp->load_bm_start = get_timer(0);	//记录从mmc加载bm到ddr的开始时间
}
```

使用定时器测量加载镜像时间, 获取当前开始时间点

### (3)spl\_mmc\_probe()
**初始化sd/mmc控制器和设备**

```c
struct mmc * spl_mmc_probe(void)
{
struct mmc *mmc;
int err;

mmc_initialize(gd->bd);	//(3.1)
/* We register only one device. So, the dev id is always 0 */
mmc = find_mmc_device(0);	//(3.2)
if (!mmc) {
	puts("spl: mmc device not found!!\n");
	hang();
}

err = mmc_init(mmc);	//(3.3)
if (err) {
	printf("spl: mmc init failed: err - %d\n", err);
	hang();
}

dw_mmc = mmc;	//
offset = spl_mmc_get_preloader_offset_in_mbr(dw_mmc);		//preloader在sd/mmc中的起始扇区

return mmc;
}
```

#### (3.1)mmc_initialize(gd->bd)
**注册设备**

初始化一个mmc设备链表&mmc_devices,初始化mmc设备号为0

通过cpu\_mmc\_init(bis)跳转到altera\_dwmmc\_init(CONFIG\_SDMMC\_BASE,CONFIG_DWMMC\_BUS\_WIDTH, 0)

其中CONFIG\_SDMMC\_BASE=0xff704000，CONFIG\_DWMMC\_BUS\_WIDTH=4

```c
int mmc_initialize(bd_t *bis)
{
	INIT_LIST_HEAD (&mmc_devices);
	cur_dev_num = 0;

	if (board_mmc_init(bis) < 0)
		cpu_mmc_init(bis);

	print_mmc_devices(',');

	return 0;
}
```


```c
int altera_dwmmc_init(u32 regbase, int bus_width, int index)
{
	struct dwmci_host *host = NULL;
	host = malloc(sizeof(struct dwmci_host));
	if (!host) {
		printf("dwmci_host malloc fail!\n");
		return 1;
	}
	host->caps  = 0;						//位宽和电压初始值，既还没有设置
	host->name = ALTERA_NAME;				//"ALTERA DWMMC"
	host->ioaddr = (void *)regbase;			//sd/mmc控制器寄存器基地址0xff704000
	host->buswidth = bus_width;				//总线宽度4位
	host->clksel = altera_dwmci_clksel;		//配置sdmmc_drv_clk相移135°和sdmmc_sample_clk相移0°
	host->dev_index = index;				//0
	host->bus_hz = CONFIG_DWMMC_BUS_HZ;		//sdmmc_clk/4，其中sdmmc_clk=200mhz
	host->fifoth_val = MSIZE(0x2) |
		RX_WMARK(CONFIG_DWMMC_FIFO_DEPTH / 2 - 1) |
		TX_WMARK(CONFIG_DWMMC_FIFO_DEPTH / 2);

	add_dwmci(host, host->bus_hz, 400000);	//注册一个mmc设备

	return 0;
}
```


注册过程add\_dwmci(host, 50000000, 400000)如下：

先做一些初始赋值

```c
int add_dwmci(struct dwmci_host *host, u32 max_clk, u32 min_clk)
{
	struct mmc *mmc;
	int err = 0;

	mmc = malloc(sizeof(struct mmc));
	if (!mmc) {
		printf("mmc malloc fail!\n");
		return -1;
	}

	mmc->priv = host;		//mmc设备绑定主机host
	host->mmc = mmc;		//主机绑定mmc设备,既相互绑定

	sprintf(mmc->name, "%s", host->name);	//mmc->name="ALTERA DWMMC"
	mmc->send_cmd = dwmci_send_cmd;		//sd/mmc设备绑定的主机的发送命令
	mmc->set_ios = dwmci_set_ios;		//设置sd/mmc时钟控制模块的clkdiv,clksrc,clkena,数据格式位宽
	mmc->init = dwmci_init;		//使能sd/mmc卡上电，清除和屏蔽所有sd/mmc中断；DMA的fifo，中断，内部寄存器的复位；
									关闭sd/mmc卡时钟sdmmc_cclk_out
	mmc->f_min = min_clk;
	mmc->f_max = max_clk;

	mmc->voltages = MMC_VDD_32_33 | MMC_VDD_33_34 | MMC_VDD_165_195;

	mmc->host_caps = host->caps;

	if (host->buswidth == 8) {
		mmc->host_caps |= MMC_MODE_8BIT;
		mmc->host_caps &= ~MMC_MODE_4BIT;
	} else {
		mmc->host_caps |= MMC_MODE_4BIT;
		mmc->host_caps &= ~MMC_MODE_8BIT;
	}
	mmc->host_caps |= MMC_MODE_HS | MMC_MODE_HS_52MHz | MMC_MODE_HC;		//主机4位宽，1.8v,3.3v

	err = mmc_register(mmc);		//注册sd/mmc设备

	return err;
}
```

然后注册mmc

```c
int mmc_register(struct mmc *mmc)
{
	/* Setup the universal parts of the block interface just once */
	mmc->block_dev.if_type = IF_TYPE_MMC;	//接口类型为mmc
	mmc->block_dev.dev = cur_dev_num++;		//设备号0
	mmc->block_dev.removable = 1;
	mmc->block_dev.block_read = mmc_bread;	//初始化块读操作
	mmc->block_dev.block_write = mmc_bwrite;//初始化块写操作
	mmc->block_dev.block_erase = mmc_berase;//初始化块擦除操作
	if (!mmc->b_max)
		mmc->b_max = CONFIG_SYS_MMC_MAX_BLK_COUNT;

	INIT_LIST_HEAD (&mmc->link);

	list_add_tail (&mmc->link, &mmc_devices);		//将mmc添加到设备链表&mmc_devices中

	return 0;
}
```

#### (3.2)find\_mmc\_device(0)
**从设备链表&mmc_devices中查找设备号为0的sd/mmc设备**

#### (3.3)mmc_init(mmc)
**sd/mmc控制器和设备的真正初始化**

```c
int mmc_init(struct mmc *mmc)
{
	int err;

	if (mmc_getcd(mmc) == 0) {
		mmc->has_init = 0;
		printf("MMC: no card present\n");
		return NO_CARD_ERR;
	}

	if (mmc->has_init)
		return 0;

	err = mmc->init(mmc);		//使能sd/mmc卡上电，清除和屏蔽所有sd/mmc中断；
								  DMA的fifo，中断，内部寄存器的复位；
								  关闭sd/mmc卡时钟sdmmc_cclk_out
	if (err)
		return err;

	mmc_set_bus_width(mmc, 1);	//设置sd/mmc设备为1位模式
	mmc_set_clock(mmc, 1);	//设置host->clock=mmc->clock=400000,

	/* Reset the Card */
	err = mmc_go_idle(mmc);		//发送cmd0给设备，复位设备到idle状态

	if (err)
		return err;

	/* The internal partition reset to user partition(0) at every CMD0*/
	mmc->part_num = 0;

	/* Test for SD version 2 */
	err = mmc_send_if_cond(mmc);	//发送cmd8给设备，如果收到回复则是version2版本以后的sd卡

	/* Now try to get the SD card's operating condition */
	err = sd_send_op_cond(mmc);		//先发送cmd55给设备，再发送acmd41给设备,从回复看到卡使用电压3.3V，支持sdhc和sdxc的									高速模式，rca=0x0000

	/* If the command timed out, we check for an MMC card */
	if (err == TIMEOUT) {
		err = mmc_send_op_cond(mmc);

		if (err) {
			printf("Card did not respond to voltage select!\n");
			return UNUSABLE_ERR;
		}
	}

	err = mmc_startup(mmc);		//ALL_SEND_CID (CMD2)，CMD9和SEND_RELATIVE_ADDR (CMD3)，设置设备的读写块长度为512字								节，将设备位宽从1设为4，频率从40khz设为50mhz
	if (err)
		mmc->has_init = 0;
	else
		mmc->has_init = 1;		//标志sd/mmc设备已经初始化
	return err;
}
```

### (4)spl\_mmc\_load\_bm\_image\_mbr()

```c
void spl_mmc_load_bm_image_mbr(void)
{
	s32 err;
	u32 image_size_sectors;

	//将在sd/mmc设备中的bm.bin的大小0x20000字节转化为512字节为单位的扇区对齐的扇区长度
	image_size_sectors = (CONFIG_PRELOADER_SDMMC_BARE_METAL_IMAGE_SIZE + 
						dw_mmc->read_bl_len - 1) /dw_mmc->read_bl_len;
	
	//将bm从sd卡读到sdram的0x1E000000地址处,因为bm比较小到不了0x1E100000所以上面的共享内存地址ok
	err = dw_mmc->block_dev.block_read(0,
			(offset + CONFIG_SYS_MMCSD_RAW_MODE_BARE_METAL_SECTOR), 
			image_size_sectors, (void *)AMPPHY_START);		
	if (err <= 0) {
		printf("spl: mmc blk read err - %d\n", err);
		hang();
	}
}
```

### (5)load_bm_end()

```c
void load_bm_end(void)
{
	asp->load_bm_end  = get_timer(0);	//记录从mmc加载bm到ddr的结束时间
}
```

### (6)spl_mmc_save_func()

```c
/* 
 * we reuse the preloader tf card driver in the bare metal, and we save
 * the tf card probe and read functions in the share parameter memory region, and reuse
 * it in the bare metal for loading the fpga image.
 */
void spl_mmc_save_func(void)
{
	writel(spl_mmc_probe, &(asp->sd_bi.preloader_sd_probe_fn));
	writel(spl_mmc_get_preloader_offset_in_mbr, &(asp->sd_bi.preloader_sd_get_offset_fn));
	
	if(dw_mmc)
	{
		writel(dw_mmc->block_dev.block_read, &(asp->sd_bi.preloader_sd_read_fn));
		asp->sd_bi.sd_read_bl_len = dw_mmc->read_bl_len;
	}
}
```

### (7)boot\_bm\_on\_cpu1()

```c
void boot_bm_on_cpu1(void)
{
	unsigned long boot_entry;
	int trampoline_size;
	unsigned int val;
	unsigned int cpu=1;

	/* save the bm start time stamp */
	memcpy(&(asp->boot_start_stamp), &save_timer_value, sizeof(unsigned int));

	writel(0, 0xffd0501c);	//将三个桥从复位释放
    
	/* code loading the real executable address, and it locates at arch/arm/cpu/armv7/start.S */
    //将secondary_trampoline处的代码写入地址0,因为arm核复位后会从0开始执行
	trampoline_size = &secondary_trampoline_end - &secondary_trampoline;
	memcpy(0, &secondary_trampoline, trampoline_size);	
    
	/* write bare metal start address into cpu1startaddress of the sysmgr module */
    //将0x1E000000作为cpu1复位释放后开始运行的地址
	writel(AMPPHY_START, (sys_manager_base_addr+(cpu1start_addr & 0x000000ff)));	

	__asm__ __volatile__("dsb\n");

	/* release the CPU1 reset */
	writel(0x0, rst_manager_base_addr + 0x10);		//将cpu1从复位状态释放，something happen
}
```

### (8)cpu0\_wait\_cpu1\_load\_rbf()

```c
int cpu0_wait_cpu1_load_rbf(void)
{
	unsigned int timeout  = 5000; /*  unit ms*/
	unsigned int start;
	
	reset_timer();
	start = get_timer(0);
	while(1)	
         //一直循环直到cpu1中运行的bm在共享内存中写入如下约定的数据
         //因为bm负责加载fpga
		if (readl(&(asp->preloader_wait_bm_load_rbf)) == (('R' << 16)|('B' << 8)|('F' << 0)))
			return 0;
	
	return 1;
}
```

### (9)spl\_mmc\_load\_image()

```c
void spl_mmc_load_image(void)
{
	int err;
	u32 boot_mode;

	boot_mode = spl_boot_mode();	//MMCSD_MODE_MBR
	if (boot_mode == MMCSD_MODE_RAW) {
		debug("boot mode - RAW\n");
		mmc_load_image_raw(dw_mmc);	
	} else if (boot_mode == MMCSD_MODE_MBR) {
		debug("boot mode - MBR\n");
		mmc_load_image_mbr(dw_mmc);		//从sd卡加载u-boot镜像到sdram中，并分析头部
	} else {
		puts("spl: wrong MMC boot mode\n");
		hang();
	}
}
```

从sd卡加载u-boot镜像到sdram中，并分析头部

```c
static void mmc_load_image_mbr(struct mmc *mmc)
{
	s32 err;
	u32 image_size_sectors;
	const struct image_header *header;
	
	header = (struct image_header *)(CONFIG_SYS_TEXT_BASE -
						sizeof(struct image_header));	//0x0100 0040 - 0x40 = 0x0100 0000

	/* read image header to find the image size & load address */
	err = mmc->block_dev.block_read(0,
			(offset + CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR), 1,
			(void *)header);		//将sd卡中的u-boot镜像第一个扇区读到0x0100 0000地址中去

	if (err <= 0)
		goto end;

	spl_parse_image_header(header);		//解析u-boot镜像头部，主要获取加载地址，入口地址和镜像长度写入spl_image中
										加载地址：头部的起始地址0x1000000
										入口地址：payload的起始地址0x1000040（从头部后的第一个字节）
										镜像长度：头部加payload的长度0x3b124字节
	/* convert size to sectors - round up */
	image_size_sectors = (spl_image.size + mmc->read_bl_len - 1) /
				mmc->read_bl_len;	//将镜像长度从字节单位转换为扇区（512字节）单位0x1d9

	/* Read the header too to avoid extra memcpy */
	err = mmc->block_dev.block_read(0,
			(offset + CONFIG_SYS_MMCSD_RAW_MODE_U_BOOT_SECTOR),
			image_size_sectors, (void *)spl_image.load_addr);		//将u-boot镜像从sd卡读到sdram加载地址上

end:
	if (err <= 0) {
		printf("spl: mmc blk read err - %d\n", err);
		hang();
	}
}
```



# 疑点

- dwmci\_writel(host, DWMCI\_TMOUT, 0xFFFFFFFF);	//dw\_mmc.c
- mmc->block\_dev.if\_type = IF\_TYPE\_MMC;		//mmc.c
- mmc->bus\_width等于1，而主机为4（已解决，在mmc\_startup(mmc)中设置为4）


