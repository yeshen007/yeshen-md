##SPL memory

	/amp_config.h/
	//0x1E000000~0x1E0FFFFF:	1MB		BM context region
	//0x1E100000~0x1E1FFFFF:	1MB		share memory ioctl region
	//0x1E200000~0x1FDFFFFF:	12MB	share buffer region
	//0x1ED00000~0x1FFFFFFF:	2MB		DMA map share buffer

	/* spl/u-boot-spl.lds */
	 MEMORY { .sram : ORIGIN = 0xFFFF0000, LENGTH = (64 * 1024) }	(spl\u-boot-spl.lds)
	_TEXT_BASE:0x01000040	(spl刚加载u-boot镜像除去头部后在sdram中的地址)
	_start:0xffff0000


	__ecc_padding_start：0xffffa700
	__image_copy_end：0xffffa700
	_end：0xffffa700
	__bss_start：0xffffa700
	__bss_end__：0xffffa79c
	__malloc_start：0xffffa79c
	__malloc_end：0xffffbb9c
	__stack_start：0xffffcba0
	__ecc_padding_end：0xffffcba8

	_bss_start_ofs: __bss_start - _start = 0xa700
	_image_copy_end_ofs: __image_copy_end - _start = 0xa700
	_bss_end_ofs: __bss_end__ - _start = 0xa79c
	_end_ofs: _end - _start = 0xa700

##U-BOOT memory
	/* socfpga_clcone5.h */
	#define CONFIG_SYS_INIT_RAM_ADDR	0xFFFF0000
	/* Reserving 0x100 space at back of scratch RAM for debug info */
	#define CONFIG_SYS_INIT_RAM_SIZE	(0x10000 - 0x100)
	/* Stack pointer prior relocation, must situated at on-chip RAM */
	#define CONFIG_SYS_INIT_SP_ADDR		(CONFIG_SYS_INIT_RAM_ADDR + CONFIG_SYS_INIT_RAM_SIZE - GENERATED_GBL_DATA_SIZE)
										//0xFFFF0000 + 0x10000 - 0x100 - GENERATED_GBL_DATA_SIZE
										//GENERATED_GBL_DATA_SIZE是gd指向的内容的大小

	#define CONFIG_SYS_SDRAM_BASE		0x00000000

	/* u-boot.lds */

	_start:0x01000040
	_TEXT_BASE:0x01000040

	__ecc_padding_start：
	__image_copy_end：
	_end：
	__bss_start：
	__bss_end__：
	__malloc_start：
	__malloc_end：
	__stack_start：
	__ecc_padding_end：

	_bss_start_ofs: __bss_start - _start = 
	_image_copy_end_ofs: __image_copy_end - _start = 
	_bss_end_ofs: __bss_end__ - _start = 
	_end_ofs: _end - _start = 
	_dynsym_start_ofs:	__dynsym_start - _start				  
		
###U-BOOT init\_sequence

	init_fnc_t *init_sequence[] = {

		arch_cpu_init,		//什么都没做，返回0
		board_early_init_f,	//关闭l4wd0看门狗
		timer_init,			//加载新计数，使能osc1timer1
		env_init,			//gd->env_addr = (ulong)&default_environment[0]; gd->env_valid = 1;(common\env_mmc.c)
		init_baudrate,		//gd->baudrate = getenv_ulong("baudrate", 10, CONFIG_BAUDRATE) = 57600
		serial_init,		//根据gd的波特率设置串口0的控制器(drivers\serial\serial.c)
		console_init_f,		//gd->have_console = 1;可以用printf了
		display_banner,		//打印点东西
		print_cpuinfo,		//打印CPU型号信息
		checkboard,			//打印板型号信息
		dram_init,			//gd->ram_size = get_ram_size(0x0, 0x40000000);
		NULL,

	};	

###U-BOOT默认环境变量

	const uchar default_environment[] = {
		"bootargs=console=ttyS0,115200n8\0"
		"bootcmd=run mmcload; run mmcboot\0"
		"bootdelay=5\0"
		"baudrate=57600\0"

		"verify=n\0" \
		"loadaddr=0x7fc0\0" \
		"fdtaddr=0x00000100\0" \
		"bootimage=uImage\0" \
		"bootimagesize=0x600000\0" \
		"fdtimage=socfpga.dtb\0" \
		"fdtimagesize=0x2000\0" \
		"mmcloadcmd=fatload\0" \
		"mmcloadpart=1\0" \
		"mmcroot=/dev/mmcblk0p2\0" \
		"qspiloadcs=0\0" \
		"qspibootimageaddr=0xa0000\0" \
		"qspifdtaddr=0x50000\0" \
		"qspiroot=/dev/mtdblock1\0" \
		"qspirootfstype=jffs2\0" \
		"ramboot=setenv bootargs console=ttyS0,115200n8;" \
			"bootm ${loadaddr} - ${fdtaddr}\0" \
		"mmcload=mmc rescan;" \
			"${mmcloadcmd} mmc 0:${mmcloadpart} ${loadaddr} ${bootimage};" \
			"${mmcloadcmd} mmc 0:${mmcloadpart} ${fdtaddr} ${fdtimage}\0" \
		"mmcboot=setenv bootargs console=ttyS0,115200n8 root=${mmcroot} rw rootwait;" \
			"bootm ${loadaddr} - ${fdtaddr}\0" \
		"netboot=dhcp ${bootimage} ; " \
			"tftp ${fdtaddr} ${fdtimage} ; run ramboot\0" \
		"qspiload=sf probe ${qspiloadcs};" \
			"sf read ${loadaddr} ${qspibootimageaddr} ${bootimagesize};" \
			"sf read ${fdtaddr} ${qspifdtaddr} ${fdtimagesize};\0" \
		"qspiboot=setenv bootargs console=ttyS0,115200n8 \
			" root=${qspiroot} rw rootfstype=${qspirootfstype};"\
			"bootm ${loadaddr} - ${fdtaddr}\0" \
		"fpga=0\0" \
		"fpgadata=0x2000000\0" \
		"fpgadatasize=0x700000\0"

		"\0"
	};			 

	
##结构体

	typedef struct {
		/* main group */
		uint32_t main_vco_base;
		uint32_t mpuclk;
		uint32_t mainclk;
		uint32_t dbgatclk;
		uint32_t mainqspiclk;
		uint32_t mainnandsdmmcclk;
		uint32_t cfg2fuser0clk;
		uint32_t maindiv;
		uint32_t dbgdiv;
		uint32_t tracediv;
		uint32_t l4src;
	
		/* peripheral group */
		uint32_t peri_vco_base;
		uint32_t emac0clk;
		uint32_t emac1clk;
		uint32_t perqspiclk;
		uint32_t pernandsdmmcclk;
		uint32_t perbaseclk;
		uint32_t s2fuser1clk;
		uint32_t perdiv;
		uint32_t gpiodiv;
		uint32_t persrc;
	
		/* sdram pll group */
		uint32_t sdram_vco_base;
		uint32_t ddrdqsclk;
		uint32_t ddr2xdqsclk;
		uint32_t ddrdqclk;
		uint32_t s2fuser2clk;
	} cm_config_t;


	struct spl_image_info {
		const char *name;
		u8 os;
		u32 load_addr;
		u32 entry_point;
		u32 size;
		u32 flags;
		u32 crc;
		u32 crc_size;
	};

	//
	typedef struct image_header {	//u-boot镜像的前64字节头部
		uint32_t	ih_magic;	/* Image Header Magic Number	*/
		uint32_t	ih_hcrc;	/* Image Header CRC Checksum	*/
		uint32_t	ih_time;	/* Image Creation Timestamp	*/
		uint32_t	ih_size;	/* Image Data Size		*/
		uint32_t	ih_load;	/* Data	 Load  Address		*/
		uint32_t	ih_ep;		/* Entry Point Address		*/
		uint32_t	ih_dcrc;	/* Image Data CRC Checksum	*/
		uint8_t		ih_os;		/* Operating System		*/
		uint8_t		ih_arch;	/* CPU architecture		*/
		uint8_t		ih_type;	/* Image Type			*/
		uint8_t		ih_comp;	/* Compression Type		*/
		uint8_t		ih_name[IH_NMLEN];	/* Image Name		*/
	} image_header_t;


	/* arch/arm/include/asm/arch-socfpga/amp_config.h */
	struct amp_share_param {	//u-boot,bm,linux的一个共享参数内存区域结构
		struct sh_ioctl_region sra[16]; //for 16 sgi
		struct sh_text_addr sta;
		struct raw_spinlock rslocks[RAW_SPIN_LOCK_OBJECT_NR];  //spinlock object
		struct semaphore semobj[16];
		u32 logindex;
		u32 logbuf_overlap;
		char logbuf[DLOG_LEN];              //actual 4KB logbuf, 4kB for overlap handle use
		u32 boot_start_stamp;
		u32 boot_end_stamp;
		u32 preloader_wait_bm_load_rbf;
		u32 load_bm_start;
		u32 load_bm_end;
		struct amp_share_qspi_boot qspi_bi; /* qspi boot info */
		struct amp_share_sd_boot   sd_bi;	 /* sd boot info */
		struct raw_spinlock	*gic_dist_lock;
	};

	/* include/mmc.h */
	struct mmc {	//代表sd/mmc设备
		struct list_head link;
		char name[32];
		void *priv;
		uint voltages;
		uint version;
		uint has_init;
		uint f_min;
		uint f_max;
		int high_capacity;
		uint bus_width;
		uint clock;
		uint card_caps;
		uint host_caps;
		uint ocr;
		uint scr[2];
		uint csd[4];
		uint cid[4];
		ushort rca;
		char part_config;
		char part_num;
		uint tran_speed;
		uint read_bl_len;
		uint write_bl_len;
		uint erase_grp_size;
		u64 capacity;
		block_dev_desc_t block_dev;
		int (*send_cmd)(struct mmc *mmc,
				struct mmc_cmd *cmd, struct mmc_data *data);
		void (*set_ios)(struct mmc *mmc);
		int (*init)(struct mmc *mmc);
		int (*getcd)(struct mmc *mmc);
		uint b_max;
	};

	/* include/dwmmc.h */
	struct dwmci_host {		//代表sd/mmc主机控制器
		char *name;
		void *ioaddr;
		unsigned int quirks;
		unsigned int caps;
		unsigned int version;
		unsigned int clock;
		unsigned int bus_hz;
		int dev_index;
		int buswidth;
		u32 fifoth_val;
		struct mmc *mmc;
	
		void (*clksel)(struct dwmci_host *host);
		unsigned int (*mmc_clk)(int dev_index);
	};

	/* include/part.h */
	typedef struct block_dev_desc {		//sd/mmc块设备描述符结构体，主要包含接口类型，设备号，块读写，擦除操作
		int		if_type;	/* type of the interface */
		int		dev;		/* device number */
		unsigned char	part_type;	/* partition type */
		unsigned char	target;		/* target SCSI ID */
		unsigned char	lun;		/* target LUN */
		unsigned char	type;		/* device type */
		unsigned char	removable;	/* removable device */

		lbaint_t	lba;		/* number of blocks */
		unsigned long	blksz;		/* block size */
		char		vendor [40+1];	/* IDE model, SCSI Vendor */
		char		product[20+1];	/* IDE Serial no, SCSI product */
		char		revision[8+1];	/* firmware revision */
		unsigned long	(*block_read)(int dev,
					      unsigned long start,
					      lbaint_t blkcnt,
					      void *buffer);
		unsigned long	(*block_write)(int dev,
					       unsigned long start,
					       lbaint_t blkcnt,
					       const void *buffer);
		unsigned long   (*block_erase)(int dev,
					       unsigned long start,
					       lbaint_t blkcnt);
		void		*priv;		/* driver private struct pointer */
	}block_dev_desc_t;

	/* inlcude/mmc.h */
	struct mmc_cmd {	//
		ushort cmdidx;
		uint resp_type;
		uint cmdarg;
		uint response[4];
	};

	typedef	struct	global_data {
		bd_t		*bd;
		unsigned long	flags;
		unsigned long	baudrate;
		unsigned long	have_console;	/* serial_init() was called */

		unsigned long	env_addr;	/* Address  of Environment struct */
		unsigned long	env_valid;	/* Checksum of Environment valid? */
		unsigned long	fb_base;	/* base address of frame buffer */

		unsigned long	relocaddr;	/* Start address of U-Boot in RAM */
		phys_size_t	ram_size;	/* RAM size */
		unsigned long	mon_len;	/* monitor len */
		unsigned long	irq_sp;		/* irq stack pointer */
		unsigned long	start_addr_sp;	/* start_addr_stackpointer */
		unsigned long	reloc_off;

		unsigned long	tlb_addr;

		const void	*fdt_blob;	/* Our device tree, NULL if none */
		void		**jt;		/* jump table */
		char		env_buf[32];	/* buffer for getenv() before reloc. */

	} gd_t;


	typedef struct bd_info {
	    int			bi_baudrate;	/* serial console baudrate */
	    ulong	        bi_arch_number;	/* unique id for this board */
	    ulong	        bi_boot_params;	/* where this board expects params */
		unsigned long	bi_arm_freq; /* arm frequency */
		unsigned long	bi_dsp_freq; /* dsp core frequency */
		unsigned long	bi_ddr_freq; /* ddr frequency */
	    struct				/* RAM configuration */
	    {
		ulong start;
		ulong size;
	    }			bi_dram[CONFIG_NR_DRAM_BANKS];
	} bd_t;


	typedef struct environment_s {
		uint32_t	crc;		/* CRC32 over data bytes	*/
		unsigned char	data[ENV_SIZE]; /* Environment data		*/
	} env_t;
