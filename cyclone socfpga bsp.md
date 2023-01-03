## <center>cyclone socfpga bsp</center>

[TOC]

**注意：**uboot和内核的交叉编译工具链是由`~/.profile`中的环境变量指定，而根文件系统的交叉编译工具链自己在`make menuconfig`中指定。
**源码原始地址：**

```c
/* preloader和uboot */
~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd/software/spl_bsp
/* 内核和设备树 */
~/V3.0/linux-socfpga
/* 根文件系统 */
~/yeshen/buildroot/buildroot
```

---

### 一、preloader和uboot配置编译

```c
/* 清洗 */
make distclean

/* 配置 */
make socfpga_cyclone5_defconfig
make menuconfig

/* 编译 */
make -j12

/* 公司版,可能要替换handoff和sopcinfo
 * 替换目录cd ~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd
 */
cd ~/altera/15.0/embedded 
./embedded_command_shell.sh
cd ~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd/software/spl_bsp
make clean	

//下面的bsp-editor如果替换handoff和sopcinfo才需要
//handoff和sopcinfo在~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd
cd ~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd/software/spl_bsp 
bsp-editor
//generate,exit

cd ~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd/software/spl_bsp
make clean	//和内核的make distclean一样
make		//preloader-mkpimage.bin
make uboot	//u-boot.img
```

&emsp;&emsp;最后编译得到的spl和uboot镜像分别在`spl_bsp/uboot-socfpga/spl`和`spl_bsp/uboot-socfpga`中。但板子要用的spl是**spl_bsp中的preloader-mkpimage.bin**，而不是`spl_bsp/uboot-socfpga/spl`中的u-boot-spl.bin，用的uboot是**spl_bsp/uboot-socfpga中的u-boot.img**。



### 二、内核和设备树和模块配置编译

```c
/* 清洗 */
make distclean 或者 make clean //make clean就不需要配置,.config还在

/* 配置 */
cp yzconfig .config
//make socfpga_defconfig	
make menuconfig

/* 编译内核 */
make zImage -j12
make LOADADDR=0x8000 uImage -j12 
/* 或者在uboot的tools目录下使用mkimage */
mkimage -A arm -O linux -T kernel -C none -a ${loadaddr} -e ${entrypoint} -d zImage uImage

/* 编译设备树 */
make socfpga_cyclone5_socdk.dtb -j12

/* 编译模块 */
make modules -j12
/* 安装模块，将包含模块的lib/modules/...安装到/home/hanglory/nfs_share/yz目录 */
make INSTALL_MOD_PATH=/home/hanglory/nfs_share/yz modules_install -j12
```

&emsp;&emsp;最后编译得到的内核和设备树镜像在`${kernelroot}/arch/arm/boot`和`${kernelroot}/arch/arm/boot/dts`中，`${kernelroot}`是内核源码根目录。



### 三、根文件系统配置编译

```c
cd ~/yezheng/buildroot-2020.08

/* 清洗 */
make clean/distclean

/* 配置 */
cp hanglory_defconfig .config
make menuconfig

/* 编译 */
make -j8

/* buildroot中的busybox */
make busybox-menuconfig
make busybox
```

&emsp;&emsp;编译后得到的根文件系统镜像在`${buildroot}/output/images`中，如果在buildroot配置了jffs则会额外生成rootfs.jffs2，可以将它烧写到qspi中。



### 四、烧写镜像到qspi

&emsp;&emsp;有两种方法烧写镜像到qspi。一种是通过uboot命令烧写，但前提是要有网可以用tftp命令从主机先下载镜像到板子内存，然后通过sf系列命令将内存中的镜像烧写到fpga。另一种是使用jtag和altera的软件烧写。

#### 4.1 uboot命令方法

##### 新布局

```c
/*
 * 不分区镜像烧写
 */ 

/* 不覆盖环境变量烧写preloader */
mw.b 0x8000 0xff 0x40000
tftp 0x8000 preloader-mkpimage.bin.primary.secondry
sf update 0x8000 0x00000 0x40000  
/* 覆盖环境变量烧写preloader */
mw.b 0x8000 0xff 0x50000
tftp 0x8000 preloader-mkpimage.bin.primary.secondry
sf update 0x8000 0x00000 0x50000   
    
/* 烧写dtb */
mw.b 0x8000 0xff 0x10000
tftp 0x8000 socfpga_cyclone5_socdk.dtb.primary.secondry
sf update 0x8000 0x50000 0x10000
    
/* 烧写uboot */
mw.b 0x8000 0xff 0x40000
tftp 0x8000 u-boot.img.primary.secondry
sf update 0x8000 0x60000 0x40000    
    
/* 烧写rootfs */
mw.b 0x8000 0xff 0x4000000
tftp 0x8000 rootfs.jffs2
sf update 0x8000 0x2000000 0x4000000
    

/*
 * primary镜像烧写
 */ 
    
/* 烧写uImage */
mw.b 0x8000 0xff 0x600000
tftp 0x8000 uImage.primary
sf update 0x8000 0xa0000 0x600000

/* 烧写rbf */
mw.b 0x8000 0xff 0x700000
tftp 0x8000 rbf.primary
sf update 0x8000 0x700000 0x700000

/*
 * secondry镜像烧写
 */ 
    
/* 烧写uImage */
mw.b 0x8000 0xff 0x600000
tftp 0x8000 uImage.secondry
sf update 0x8000 0x10a0000 0x600000

/* 烧写rbf */
mw.b 0x8000 0xff 0x700000
tftp 0x8000 rbf.secondry
sf update 0x8000 0x1700000 0x700000   
```

##### 老布局

```c
/* 在uboot命令行阶段设置环境变量和激活qspi flash */
setenv ipaddr 192.168.26.111	//设置板子的ip，到内核会被重新设置我们不管，反正uboot阶段就是我们设置的
setenv serverip 192.168.26.133 	//设置tftp服务器的ip，下载的是/home/hanglory/tftpboot中的文件   
  
/* 激活qspi */   
sf probe	
    
/* 擦除所以镜像 */
sf erase 0x0000 0x2000000

/* 烧写preloader到qspi 0x00000*/
擦除内存：mw.b 0x8000 0xff 0x50000
Tftp下载uboot：tftp 0x8000 preloader-mkpimage.bin
烧写uboot：sf update 0x8000 0x00000 0x50000

/* 烧写uboot到qspi 0x60000*/
擦除内存：mw.b 0x8000 0xff 0x40000
Tftp下载uboot：tftp 0x8000 u-boot.img
烧写uboot：sf update 0x8000 0x60000 0x40000
   
/* 烧写zImage到qspi 0xa0000 */
擦除内存：mw.b 0x8000 0xff 0x600000
Tftp下载zImage：tftp 0x8000 zImage
烧写zImage：sf update 0x8000 0xa0000 0x600000

/* 烧写dtb到qspi 0x50000*/
擦除内存：mw.b 0x8000 0xff 0x10000
Tftp下载dtb文件：tftp 0x8000 socfpga_cyclone5_socdk.dtb
烧写dtb：sf update 0x8000 0x50000 0x10000
    
/* 烧写rootfs到qspi 0x800000*/
擦除内存：mw.b 0x8000 0xff 0x800000
Tftp下载rootfs：tftp 0x8000 rootfs.jffs2
烧写rootfs：sf update 0x8000 0x800000 0x800000
    
/* 烧写rbf到qspi 0x1400000*/
擦除内存：mw.b 0x8000 0xff 0x800000
Tftp下载rbf：tftp 0x8000 printhead_v3.rbf
烧写rbf：sf update 0x8000 0x1400000 0x800000
```

#### 4.2 jtag方法

```c
首先双击打开E:\altera\15.0\embedded中的Embedded_Command_Shell.bat，然后进入cygwin。然后cd到镜像目录，在E:\altera\images中，将编译好的镜像放在这个目录，然后通过quartus_hps.exe -c 1 -o p -a <addr> <image>将该目录下的镜像<image>烧写到qspi地址<addr>中。

用jtag烧写preload：quartus_hps -c 1 -o p -a 0x000000 preloader-mkpimage.bin
用jtag烧写uboot：quartus_hps -c 1 -o p -a 0x60000 u-boot.img
用jtag烧写zImage：quartus_hps -c 1 -o p -a 0xa0000 zImage
用jtag烧写dtb：quartus_hps -c 1 -o p -a 0x50000 socfpga_cyclone5_socdk.dtb
用jtag烧写rootfs：quartus_hps -c 1 -o p -a 0x800000 rootfs.jffs2
用jtag烧写rbf：quartus_hps -c 1 -o p -a 0x1400000 soc_system.rbf
```



### 五、驱动调试与分析

#### 5.1 watchdog驱动

#####  注册

```c
/* 设备树 */
watchdog0: watchdog@ffd02000 {		//使用了看门狗watchdog0
    compatible = "snps,dw-wdt";		//dw_wdt.c
    reg = <0xffd02000 0x1000>;
    interrupts = <0 171 4>;
    clocks = <&osc1>;
    resets = <&rst L4WD0_RESET>;
    status = "disabled";
};
&watchdog0 {		//使能了看门狗watchdog0
	status = "okay";
};

/* dw_wdt.c */
static const struct of_device_id dw_wdt_of_match[] = {
	{ .compatible = "snps,dw-wdt", },	//匹配设备树
	{ /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, dw_wdt_of_match);

static struct platform_driver dw_wdt_driver = {
	.probe		= dw_wdt_drv_probe,		//触发
	.remove		= dw_wdt_drv_remove,
	.driver		= {
		.name	= "dw_wdt",
		.of_match_table = of_match_ptr(dw_wdt_of_match),
		.pm	= &dw_wdt_pm_ops,
	},
};

module_platform_driver(dw_wdt_driver);
```

​		设备树中定义了两个看门狗节点，但只有watchdog0的status为okay，因此它会转化成platform_device注册到平台总线，所以会和驱动dw_wdt_driver匹配触发dw_wdt_drv_probe函数。

```c
static int dw_wdt_drv_probe(struct platform_device *pdev)
{
	wdd->ops = &dw_wdt_ops;		//看门狗操作函数
	ret = watchdog_register_device(wdd);	//注册看门狗
}

//open write ioctl close设备节点最后会调用到
static const struct watchdog_ops dw_wdt_ops = {
	.start		= dw_wdt_start,		//将超时时间写入看门狗计数器寄存器然后启动
	.stop		= dw_wdt_stop,		//停止看门狗
	.ping		= dw_wdt_ping,		//刷新看门狗计数器,也就是恢复成超时
	.set_timeout	= dw_wdt_set_timeout,	//设置超时
	.get_timeleft	= dw_wdt_get_timeleft,	//获得超时
	.restart	= dw_wdt_restart,			//刷新计数器或者使能看门狗
};
```

​		在dw_wdt_drv_probe函数中主要就是注册了看门狗，在注册之前设置它的操作函数，这些函数会在应用程序操作设备节点时最后调用到来实际处理看门狗。接下再继续深入分析注册看门狗。

```c
watchdog_register_device
    __watchdog_register_device
    	watchdog_dev_register
    		watchdog_cdev_register
```

​		注册的主要调用链如上，主要工作在watchdog_cdev_register。

```c
static int watchdog_cdev_register(struct watchdog_device *wdd)
{
	//如果id为0会额外注册watchdog_miscdev,并创建设备节点/dev/watchdog
	if (wdd->id == 0) {
		old_wd_data = wd_data;	//在watchdog_open中使用,使操作/dev/watchdog相当于/dev/watchdog0
		err = misc_register(&watchdog_miscdev);		//注册杂项设备和创建设备节点/dev/watchdog
	}

	dev_set_name(&wd_data->dev, "watchdog%d", wdd->id);	//设置字符设备对应的名字/dev/watchdogX
	cdev_init(&wd_data->cdev, &watchdog_fops);		//设置字符设备的操作函数
	cdev_device_add(&wd_data->cdev, &wd_data->dev);	//注册字符设备和创建设备节点/dev/watchdogX
}

//open write ioctl close设备节点首先会调用到
static const struct file_operations watchdog_fops = {
	.write		= watchdog_write,			
	.unlocked_ioctl	= watchdog_ioctl,		
	.open		= watchdog_open,			
	.release	= watchdog_release,
};

static struct miscdevice watchdog_miscdev = {
	.minor		= WATCHDOG_MINOR,
	.name		= "watchdog",		// /dev/watchdog
	.fops		= &watchdog_fops,	// 无论是/dev/watchdog还是/dev/watchdogX都是这个
};
```

​		对于每个看门狗会创建一个/dev/watchdogX和相应的字符设备，对于第一个还会创建一个额外的/dev/watchdog和杂项设备，操作/dev/watchdog和操作/dev/watchdog0效果一样。

#####  操作

​		看门狗有很多操作，主要常用几种操作，我在watchdogops.c中实现了以下几种常用操作的测试用例，具体原理看这个测试用例和内核驱动分析，这里列出了调用路径。

```c
//启动看门狗(默认超时30s在驱动注册时设置，可以自己改掉)
open("/dev/watchdog", O_RDWR)
    watchdog_open
        watchdog_start
            if (watchdog_running)
                dw_wdt_ping		//如果本来就运行的则刷新看门狗计数器
            else
                dw_wdt_start	//如果还没运行则将超时写入计数器然后启动

//喂狗
ioctl(fd, WDIOC_KEEPALIVE, NULL)  
	watchdog_ioctl
		watchdog_ping
			__watchdog_ping
				dw_wdt_ping		//刷新看门狗计数器

//设置超时
ioctl(fd, WDIOC_SETTIMEOUT, (unsigned long)&my_time)  
	watchdog_ioctl
		watchdog_set_timeout
			dw_wdt_set_timeout	//设置超时
		watchdog_ping
			__watchdog_ping
				dw_wdt_ping		//刷新看门狗计数器

//停止看门狗
arg = WDIOS_DISABLECARD;    
ioctl(fd, WDIOC_SETOPTIONS, (unsigned long)&arg)
	watchdog_ioctl
    	watchdog_stop
    		dw_wdt_stop		//停止看门狗
    
//启动看门狗另一种方法
arg = WDIOS_ENABLECARD;    
ioctl(fd, WDIOC_SETOPTIONS, (unsigned long)&arg)
	watchdog_ioctl
		watchdog_start
			if (watchdog_running)
				dw_wdt_ping		//如果本来就运行的则刷新看门狗计数器
			else
				dw_wdt_start	//如果还没运行则将超时写入计数器然后启动    
```



#### 5.2 tty uart驱动

#####  注册

```c
/* 设备树 */
ph0_uart@ff200800 {
    compatible = "altr,uart-1.0";	//altera-uart.c
    reg = <0xff200800 0x20>;
    interrupt-parent = <&intc>;
    interrupts = <0x0 0x4a IRQ_TYPE_LEVEL_HIGH>;	//fpga-irq34,发送和接收共用
    interrupt-names = "altera_uart0";	
    clock-frequency = <50000000>;
};	
ph1_uart@ff200820 {
    compatible = "altr,uart-1.0";
    ...
};	

/* 驱动altera-uart.c */
static const struct of_device_id altera_uart_match[] = {
	{ .compatible = "ALTR,uart-1.0", },
	{ .compatible = "altr,uart-1.0", },	//匹配设备树
	{},
};

static struct platform_driver altera_uart_platform_driver = {
	.probe	= altera_uart_probe,		//注册uart_port
	.remove	= altera_uart_remove,
	.driver	= {
		.name		= DRV_NAME,
		.of_match_table	= of_match_ptr(altera_uart_match),		//匹配设备树
	},
};

static int __init altera_uart_init(void)
{
	//注册tty_driver
	rc = uart_register_driver(&altera_uart_driver);

	//注册altera_uart_platform_driver然后触发altera_uart_probe
	//在altera_uart_probe中注册uart_port
	rc = platform_driver_register(&altera_uart_platform_driver);
}
```

​		设备树中定义了两个串口节点，这两个节点会被创建成两个platform_device然后注册到平台总线。在驱动加载后会触发altera_uart_init，然后注册uart_driver和uart_port。因此注册过程主要分为注册uart_driver和注册uart_port两个。

###### 注册uart_driver

```c
/*********************************************************************
 * 从uart_register_driver(&altera_uart_driver)开始分析注册uart_driver过程
 *********************************************************************/

static struct uart_driver altera_uart_driver = {
	.owner		= THIS_MODULE,
	.driver_name	= DRV_NAME,
	.dev_name	= "ttyAL",							//设备节点名字前缀
	.major		= SERIAL_ALTERA_MAJOR,
	.minor		= SERIAL_ALTERA_MINOR,
	.nr		= CONFIG_SERIAL_ALTERA_UART_MAXPORTS,	// 表示这个driver支持多少个port
	.cons		= ALTERA_UART_CONSOLE,
};

//uart_register_driver(&altera_uart_driver)
int uart_register_driver(struct uart_driver *drv)
{
	struct tty_driver *normal;
    
	//为驱动支持的每个串口分配uart_state
	drv->state = kcalloc(drv->nr, sizeof(struct uart_state), GFP_KERNEL);	
    /* 分配一个tty_driver,设置部分成员,比如
	 * 设置normal->num = drv->nr
	 * 为normal指向的tty_driver分别分配normal->num个
	 * tty_struct指针,ktermios指针,tty_port指针,cdev指针
	 */
	normal = alloc_tty_driver(drv->nr);	
	drv->tty_driver = normal;	

	/* 填充tty_driver */
	normal->driver_name	= drv->driver_name;
	normal->name		= drv->dev_name;		//tty_driver的名字设置为ttyAL
	normal->major		= drv->major;
	normal->minor_start	= drv->minor;
	normal->flags		= TTY_DRIVER_DYNAMIC_DEV;	
	tty_set_operations(normal, &uart_ops);		//tty_operations结构体

	/* 设置每个串口uart_state中的tty_port */
	for (i = 0; i < drv->nr; i++) {
		struct uart_state *state = drv->state + i;
		struct tty_port *port = &state->port;
		tty_port_init(port);
		port->ops = &uart_port_ops;		//tty_port_operations结构体
	}

	retval = tty_register_driver(normal);		//注册tty_driver
	return retval;
}

static const struct tty_operations uart_ops = {
    ...
	.open		= uart_open,				//open第二调用
    .write		= uart_write,				//wirte第二调用
	...
};

static const struct tty_port_operations uart_port_ops = {
	...
	.activate	= uart_port_activate,		//open第三调用
	.shutdown	= uart_tty_port_shutdown,
};
```

​		uart_register_driver(&altera_uart_driver)通过向内核注册一个uart_driver结构体altera_uart_driver，进入函数中根据uart_driver再分配设置注册tty_driver。需要注意tty_driver的name成员设置为传入的uart_driver的dev_name成员即ttyAL，这是open设备节点的前缀，后面会分析到。接着再继续分析注册tty_driver过程。

```c
/**************************************************
 * 继续分析注册uart_driver过程，最后一步注册tty_driver
 **************************************************/
//tty_register_driver(normal)
int tty_register_driver(struct tty_driver *driver)
{
	//申请设备号区域[dev, dev + driver->num)
    dev = MKDEV(driver->major, driver->minor_start);	
    error = register_chrdev_region(dev, driver->num, driver->name);	
    
    //为区域[dev, dev + driver->num)分配设置注册cdev
    error = tty_cdev_add(driver, dev, 0, driver->num);
}


//tty_cdev_add(driver, dev, 0, driver->num)
static int tty_cdev_add(struct tty_driver *driver, dev_t dev,
		unsigned int index, unsigned int count)
{
    //分配设置注册cdev
	driver->cdevs[index] = cdev_alloc();				
	driver->cdevs[index]->ops = &tty_fops;				//字符设备操作函数
	driver->cdevs[index]->owner = driver->owner;
	err = cdev_add(driver->cdevs[index], dev, count);		//注册tty字符设备
	return err;
}

static const struct file_operations tty_fops = {
	...
	.read		= tty_read,		//read时第一调用
	.write		= tty_write,	//wirte第一调用
	.open		= tty_open,		//open第一调用
    ...
};
```

​		进入tty_register_driver后首先申请能容纳所有port设备号区域，然后为区域[dev, dev + driver->num)分配设置注册cdev分配设置注册字符设备，字符设备的操作函数为&tty_fops，比如通过open(/dev/ttyALX)打开串口时就会调用到tty_open，设备节点/dev/ttyALX在后文的注册uart_port中创建。

###### 注册uart_port

```c
/********************************************
 * 从altera_uart_probe开始分析注册uart_port过程
 ********************************************/
static int altera_uart_probe(struct platform_device *pdev)
{
	struct uart_port *port;
	struct resource *res_mem;
	struct resource *res_irq;

	port = &altera_uart_ports[i].port;		//还没设置,再下面设置

	res_mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);	//reg属性
	port->mapbase = res_mem->start;
    
	res_irq = platform_get_resource(pdev, IORESOURCE_IRQ, 0);	//irq属性
	port->irq = res_irq->start;
    
	ret = of_property_read_u32(pdev->dev.of_node, "clock-frequency",
					   &port->uartclk);		//clock-frequency属性
    
	port->membase = ioremap(port->mapbase, ALTERA_UART_SIZE);	//映射reg属性

	/* 给port剩下的属性赋值 */
	port->line = i;						//id,第一个为0
	port->ops = &altera_uart_ops;		//串口操作函数 
	port->dev = &pdev->dev;

	platform_set_drvdata(pdev, port);	//pdev->dev->driver_data = port;

	uart_add_one_port(&altera_uart_driver, port);	//向uart_driver注册一个uart_port

	return 0;
}

static const struct uart_ops altera_uart_ops = {
	...
	.start_tx	= altera_uart_start_tx,		//wirte最后调用
	...
	.startup	= altera_uart_startup,		//open最后调用
	...
};
```

​		正如前文所说的，如果在设备树发现一个匹配的串口节点则调用altera_uart_probe，这是注册一个uart_port的起点。altera_uart_probe中根据port index来得到一个静态的uart_port，然后提取设备树的信息填入，另外设置一个uart_ops结构体&altera_uart_ops给uart_port，它就是串口的实际操作函数。最后通过uart_add_one_port注册一个uart_port。

```c
/**************************************************
 * 继续分析注册uart_port过程，分析uart_add_one_port
 **************************************************/
//uart_add_one_port(&altera_uart_driver, port)
int uart_add_one_port(struct uart_driver *drv, struct uart_port *uport)
{
	struct uart_state *state
	struct tty_port *port;
    
	//提取出uart_driver中的uart_state中的tty_port
	state = drv->state + uport->line;		
	port = &state->port;    

    //设置drv->tty_driver->ports[uport->line] = port
	tty_port_link_device(port, drv->tty_driver, uport->line);		

	//将uport->dev->driver_data设置为tty_port结构体变量port然后注册到驱动模型,
	//然后创建设备节点/dev/ttyALX
	tty_dev = tty_port_register_device_attr_serdev(port, drv->tty_driver,
			uport->line, uport->dev, port, uport->tty_groups);		

	return ret;
}
```

​		uart_add_one_port做的事不复杂，就像上面所述，最后创建了设备节点后就可以使用open打开然后进一步操作了。

##### 操作

​		详细分析可以参考韦东山视频和文档，这里主要列出调用流程和重点概括。

###### open

```c
/* uartfd = open("/dev/ttyAL0", ...) */
tty_open
	tty_open_by_driver(device, filp)	
		tty_lookup_driver(device, filp, &index)		//通过device的设备号找到tty_driver和index
		tty_driver_lookup_tty(driver, filp, index)	//通过tty_driver和index找到tty_struct指针
		tty_init_dev(driver, index)				//如果还没打开
            alloc_tty_struct(driver, idx)	//设置tty_struct的ops为tty_driver的ops
            tty_driver_install_tty(driver, tty)	//driver->ttys[tty->index] = tty
	tty->ops->open(tty, filp)	//调用uart_open
    	tty_port_open
            port->ops->activate		//uart_port_ops.activate(uart_port_activate)
            	uart_startup
            		uart_port_startup
            			uport->ops->startup	//altera_uart_ops.startup(altera_uart_startup)
```

​		以上就是open /dev/ttyALX设备节点时的调用路径，最后会调用到altera_uart_startup，最后再分析一下altera_uart_startup就结束open的分析。

```c
static int altera_uart_startup(struct uart_port *port)
{
	if (!port->irq) {	//如果设备树没有irq则设置定时器模拟中断
		timer_setup(&pp->tmr, altera_uart_timer, 0);
		mod_timer(&pp->tmr, jiffies + uart_poll_timeout(port));
	} else {			//如果设备树指定irq则注册中断
		ret = request_irq(port->irq, altera_uart_interrupt, 0, DRV_NAME, port);
	}
}

//根据uart硬件状态决定是否收发数据
//然后继续修改定时超时时间来轮询
static void altera_uart_timer(struct timer_list *t)
{
	altera_uart_interrupt(0, port);
	mod_timer(&pp->tmr, jiffies + uart_poll_timeout(port));
}

//根据uart硬件状态决定是否收发数据
static irqreturn_t altera_uart_interrupt(int irq, void *data)
{
	//读取ALTERA_UART_STATUS_REG寄存器内容
	isr = altera_uart_readl(port, ALTERA_UART_STATUS_REG) & pp->imr;

	//如果有配置中断的话只要这两位任意一位置位都会产生中断
    //如果没有配置中断则通过则也可以通过检查这两标志位来决定是否要取数据或发数据
	if (isr & ALTERA_UART_STATUS_RRDY_MSK)
		altera_uart_rx_chars(pp);	//取数据
	if (isr & ALTERA_UART_STATUS_TRDY_MSK)	
		altera_uart_tx_chars(pp);	//发数据
}
```

​		正如前面所分析的，应用程序open(/dev/ttyALX)会最终调用到altera_uart_startup。它做的事看上面代码注释就清楚，简单来说就是注册一个中断或者定时器函数。最后再看看取数据和发数据的操作。

```c
static void altera_uart_rx_chars(struct altera_uart *pp)
{    
	//如果ALTERA_UART_STATUS_RRDY_MSK一直置位表示uart接收寄存器一直有数据到
	//然后就将这个寄存器的数据追加发送到tty buffer,一直轮询到uart接收寄存器
	//没有收到数据才退出轮询,最后通知行规层一次性取走tty buffer
	while ((status = altera_uart_readl(port, ALTERA_UART_STATUS_REG)) &
	       ALTERA_UART_STATUS_RRDY_MSK) {
		ch = altera_uart_readl(port, ALTERA_UART_RXDATA_REG);	//读取寄存器数据
		uart_insert_char(port, status, ALTERA_UART_STATUS_ROE_MSK, ch,flag);	//存入tty buffer
	}

	tty_flip_buffer_push(&port->state->port);		//通知行规层来取出tty buffer数据放入行规层
}
```

​		取数据过程如上所示，如果ALTERA_UART_STATUS_RRDY_MSK一直置位表示uart接收寄存器一直有数据到然后就将这个寄存器的数据追加发送到tty buffer,一直轮询到uart接收寄存器没有收到数据才退出轮询,最后通知行规层一次性取走tty buffer送给应用层或做相应的处理。

```c
static void altera_uart_tx_chars(struct altera_uart *pp)
{
	/* 应用层通过write是把数据存入uart_port的环形缓冲区而不是直接发送出去
	 * 如果串口发送寄存器空闲则从环形缓冲区取出数据写入发送寄存器
	 */
	while (altera_uart_readl(port, ALTERA_UART_STATUS_REG) &
	       ALTERA_UART_STATUS_TRDY_MSK) {
		if (xmit->head == xmit->tail)		//如果xmit写完了则退出循环
			break;
		altera_uart_writel(port, xmit->buf[xmit->tail],
		       ALTERA_UART_TXDATA_REG);		//将xmit中的一个数据写到发送寄存器
		xmit->tail = (xmit->tail + 1) & (UART_XMIT_SIZE - 1);
	}

	/* 如果环形缓冲区空了则将ALTERA_UART_CONTROL_TRDY_MSK位关闭就不会
     * 一直触发中断了,直到应用层通过write调用到altera_uart_start_tx
     * 才重新打开,此时就会再次触发发送中断
     * 这里还要判断xmit为空才会屏蔽ALTERA_UART_CONTROL_TRDY_MSK,
     * 因为上面的退出while循环如果不是因为xmit为空,而是因为ALTERA_UART_STATUS_TRDY_MSK
     * 暂时没有置位,那有可能是硬件还没将寄存器中的数据移送出去,当寄存器的数据被移送出去后
     * 会重新ALTERA_UART_STATUS_TRDY_MSK置位,会重新产生中断或者使用定时器轮询查看到,因此数据不会丢
	 */
	if (xmit->head == xmit->tail) {
		pp->imr &= ~ALTERA_UART_CONTROL_TRDY_MSK;
		altera_uart_update_ctrl_reg(pp);
	}
}
```

​		发数据过程如上所示。

###### read

​		read在在file_operations中有上层接口，在uart_ops没有底层接口，因为底层接口是通过中断接收函数，而通过file_operations上层read接口读取底层准备好的数据或者阻塞睡眠让出cpu等待底层数据准备好后唤醒再读取。下面看代码分析。

```c
/* read(uartfd, ...) */
tty_read
	ld->ops->read(tty, file, buf, count);	//n_tty_ops.read(n_tty_read)
```

​		应用程序通过read之后首先调用的是tty_read，在这函数中主要就是调用行规层n_tty_ops的read函数n_tty_read。

```c
static ssize_t n_tty_read(struct tty_struct *tty, struct file *file,
			 unsigned char __user *buf, size_t nr)
{
	DEFINE_WAIT_FUNC(wait, woken_wake_function);	//本进程的等待队列节点
	add_wait_queue(&tty->read_wait, &wait);			//如果要睡醒使用的等待队列
    
    /* 一直循环直到从行规层取完nr个数据 */
	while (nr) {
         /* 如果行规层没有数据 */
		if (!input_available_p(tty, 0)) {		
			tty_buffer_flush_work(tty->port);	//刷报给行规层数据的工作队列
			if (!input_available_p(tty, 0)) {	//如果行规层还没有数据
				timeout = wait_woken(&wait, TASK_INTERRUPTIBLE, 
                                     timeout);		 //没有数据则睡眠,被中断中的处理函数唤醒		
			}
		}
        
		/* 如果行规层有数据或者被唤醒了 */
		copy_from_read_buf(tty, &b, &nr);	//从行规层读取数据
	}
}
```

​		这里只分析阻塞模式，不考虑超时的处理。这个函数很复杂但主要流程就像上面被删减的代码这样，如果没有数据则调用read的进程休眠，直到中断或者超时，然后醒来读取数据；如果有数据直接读数据；这个循环直到读取完nr。

###### write

```c
/* write(uartfd, ...) */
tty_write
    n_tty_write
    	uart_write
    		memcpy(circ->buf + circ->head, buf, c)	//将数据写入环形缓冲区
    		altera_uart_start_tx	//设置串口发送硬件标志位(对于有中断的情况会直接触发中断)
```

​		应用层调用write进入内核的主要路径如上。所做的事情简单总结就是将数据写入环形缓冲区，然后触发中断。回顾一下前文在open中注册了中断处理函数，该中断函数根据是发送或者接收使用不同的处理，write是发送，因此会调用altera_uart_tx_chars将环形缓冲区的数据写到发送寄存器，最后关闭硬件发送中断位。

#### 5.3 网卡驱动

#####  注册

######  注册mac

```c
/* socfpga.dtsi */
gmac0: ethernet@ff700000 {
    compatible = "altr,socfpga-stmmac", "snps,dwmac-3.70a", "snps,dwmac";	//dwmac-socfpga.c
    altr,sysmgr-syscon = <&sysmgr 0x60 0>;		//
    reg = <0xff700000 0x2000>;
    interrupts = <0 115 4>;		//mac对cpu的中断号
    interrupt-names = "macirq";		//
    mac-address = [00 00 00 00 00 00];/* Filled in by U-Boot */
    clocks = <&emac_0_clk>;
    clock-names = "stmmaceth";
    snps,multicast-filter-bins = <256>;
    snps,perfect-filter-entries = <128>;
    resets = <&rst EMAC0_RESET>;	//保留和删除都没事
    reset-names = "stmmaceth";		//保留和删除都没事
    //max-frame-size = <3800>;		//保留和删除都没事,而且好像对ifconfig mtu没影响,都显示1500
    tx-fifo-depth = <4096>;
    rx-fifo-depth = <4096>;
    status = "disabled";
};

/* socfpga_cyclone5_socdk.dts */
//如何读取phy addr? 答:使用uboot的mii命令 mii read 0xa 2
//如何读取phy id? 答:通过uboot命令得到id 0x2000a0f1
&gmac0 {		//socfpga-dwmac.txt  snps,dwmac.yaml dwmac-socfpga.c
    status = "okay";
    phy-mode = "rgmii";		
    snps,phy-addr = <10>;	//为什么一定要这个属性?答:因为stmmac_open中需要
    mdio0 {
        compatible = "snps,dwmac-mdio";		//snps,dwmac.yaml中的示例有,源代码也有
        #address-cells = <1>;
        #size-cells = <0>;
        /* dp83869hw */
        phy0: ethernet-phy@0 {		//ti,dp83869.yaml dp83869.c
            compatible = "ethernet-phy-id2000.a0f1";	//先uboot读取phy id再和ethernet-phy-id拼接
            reg = <10>;		//可加可不加,如果加了一定要等于snps,phy-addr,但有个疑问为什么可以不加?
            ti,op-mode = <DP83869_RGMII_1000_BASE>;
            rx-fifo-depth = <DP83869_PHYCR_FIFO_DEPTH_4_B_NIB>;
            tx-fifo-depth = <DP83869_PHYCR_FIFO_DEPTH_4_B_NIB>;
            ti,clk-output-sel = <DP83869_CLK_O_SEL_REF_CLK>;
        };
    };
};

/* dwmac-socfpga.c */
static const struct socfpga_dwmac_ops socfpga_gen5_ops = {
	.set_phy_mode = socfpga_gen5_set_phy_mode,
};

static const struct of_device_id socfpga_dwmac_match[] = {
	{ .compatible = "altr,socfpga-stmmac", .data = &socfpga_gen5_ops },	//匹配设备树 
	{ .compatible = "altr,socfpga-stmmac-a10-s10", .data = &socfpga_gen10_ops },
	{ }
};
MODULE_DEVICE_TABLE(of, socfpga_dwmac_match);

static struct platform_driver socfpga_dwmac_driver = {
	.probe  = socfpga_dwmac_probe,	//注册
	.remove = stmmac_pltfr_remove,
	.driver = {
		.name           = "socfpga-dwmac",
		.pm		= &socfpga_dwmac_pm_ops,
		.of_match_table = socfpga_dwmac_match,	//匹配设备树 
	},
};
module_platform_driver(socfpga_dwmac_driver);
```

​		平台驱动和节点匹配后调用socfpga_dwmac_probe，主要流程如下。

```c
static int socfpga_dwmac_probe(struct platform_device *pdev)
{
	int ret;
	struct plat_stmmacenet_data *plat_dat;	//保存设备树节点或者设备的除了mac地址,irq,reg之外的大部分资源
	struct stmmac_resources stmmac_res;		//保存设备树节点或者设备的mac地址,irq,reg资源
	struct device *dev = &pdev->dev;
	struct socfpga_dwmac *dwmac;				//代表dwmac的struct device的封装结构体
	struct net_device *ndev;
	struct stmmac_priv *stpriv;					//net_device的私有数据
	const struct socfpga_dwmac_ops *ops;

	ops = device_get_match_data(&pdev->dev);
	//从设备树获取macirq对应的虚拟中断号和reg对应的虚拟地址
	ret = stmmac_get_platform_resources(pdev, &stmmac_res);	
	//从设备树获取mac地址填充到stmmac_res.mac和其他设备树信息填充到plat_dat
    //通过设备树属性snps,phy-addr得到plat_dat->phy_addr = 10
	plat_dat = stmmac_probe_config_dt(pdev, &stmmac_res.mac);	
    
	dwmac = devm_kzalloc(dev, sizeof(*dwmac), GFP_KERNEL);
	ret = socfpga_dwmac_parse_data(dwmac, dev);	//获取mac寄存器信息
	dwmac->ops = ops;						//&socfpga_gen5_ops
	plat_dat->bsp_priv = dwmac;	
	plat_dat->fix_mac_speed = socfpga_dwmac_fix_mac_speed;		

	/* 重点 */
     //根据plat_dat和stmmac_res构造net_device和私有数据stmmac_priv,然后注册到内核
	ret = stmmac_dvr_probe(&pdev->dev, plat_dat, &stmmac_res);	

	ndev = platform_get_drvdata(pdev);	//ndev = pdev->dev->driver_data
	stpriv = netdev_priv(ndev);			//stpriv =  (char *)ndev + sizeof(struct net_device)
	ret = ops->set_phy_mode(dwmac);		//通过socfpga_gen5_set_phy_mode真正的硬件设置phy mode

	return 0;
}
```

​		socfpga_dwmac_probe中根据设备树节点提取信息到plat_dat和stmmac_res中，然后根据plat_dat和stmmac_res通过stmmac_dvr_probe构造net_device和私有数据stmmac_priv，注册到内核，最后调用socfpga_gen5_set_phy_mode设置硬件phy mode。下面继续分析stmmac_dvr_probe。

```c
int stmmac_dvr_probe(struct device *device,
		     struct plat_stmmacenet_data *plat_dat,
		     struct stmmac_resources *res)
{
	struct net_device *ndev = NULL;
	struct stmmac_priv *priv;

	//分配net_device和私有数据struct stmmac_priv的空间
    //ndev->name = eth%d,还没完整
    //ndev->ethtool_ops = &default_ethtool_ops
	ndev = devm_alloc_etherdev_mqs(device, sizeof(struct stmmac_priv),
				       MTL_MAX_TX_QUEUES, MTL_MAX_RX_QUEUES);		
	
	stmmac_set_ethtool_ops(ndev);		//重新设置ndev->ethtool_ops = &stmmac_ethtool_ops
    
	if (!IS_ERR_OR_NULL(res->mac))		//mac地址写入priv->dev->dev_addr
		memcpy(priv->dev->dev_addr, res->mac, ETH_ALEN);	
    
	if (priv->plat->stmmac_rst) {		//复位网口
		ret = reset_control_assert(priv->plat->stmmac_rst);
		reset_control_deassert(priv->plat->stmmac_rst);
	}

	// mac硬件设置和mac地址检验 
	ret = stmmac_hw_init(priv);
	stmmac_check_ether_addr(priv);
    
	netif_set_real_num_rx_queues(ndev, priv->plat->rx_queues_to_use);	//配置接收队列
	netif_set_real_num_tx_queues(ndev, priv->plat->tx_queues_to_use);	//配置发送队列  
    
    /* 重点 */
	ndev->netdev_ops = &stmmac_netdev_ops;		//操作函数

	// 设置napi函数到napi_struct 
	maxq = max(priv->plat->rx_queues_to_use, priv->plat->tx_queues_to_use);
	for (queue = 0; queue < maxq; queue++) {
		struct stmmac_channel *ch = &priv->channel[queue];
		if (queue < priv->plat->rx_queues_to_use) 
			netif_napi_add(ndev, &ch->rx_napi, stmmac_napi_poll_rx,
				       NAPI_POLL_WEIGHT);		//设置接收的napi poll函数
		if (queue < priv->plat->tx_queues_to_use) 
			netif_tx_napi_add(ndev, &ch->tx_napi, stmmac_napi_poll_tx,
					  NAPI_POLL_WEIGHT);	//设置发送的napi poll函数
	}

    /* 重点 */
	ret = stmmac_mdio_register(ndev);	//注册mdio结构体和扫描注册phy结构体
    
	ret = stmmac_phy_setup(priv);	//phy硬件设置
    
    /* 重点 */
	ret = register_netdev(ndev);	//注册net_device

	return ret;
}
```

​		进入stmmac_mdio_register分析。

```c
int stmmac_mdio_register(struct net_device *ndev)
{
    int err = 0;
    struct mii_bus *new_bus;
    struct stmmac_priv *priv = netdev_priv(ndev);
    struct device_node *mdio_node = priv->plat->mdio_node;	
    struct device *dev = ndev->dev.parent;
    int addr, found, max_addr;

    new_bus = mdiobus_alloc();	//分配mdio bus结构体
    
    //设置mdio bus结构体
    new_bus->name = "stmmac";
    new_bus->read = &stmmac_mdio_read;		//读phy芯片的函数
    new_bus->write = &stmmac_mdio_write;	//写phy芯片的函数
    max_addr = PHY_MAX_ADDR;			   //mdio支持的最大phy数目32,因此最大的phy addr是3
    new_bus->priv = ndev;
    new_bus->phy_mask = mdio_bus_data->phy_mask;
    new_bus->parent = priv->device;

    /* 重点 */
    err = of_mdiobus_register(new_bus, mdio_node);	//注册mdio bus

    priv->mii = new_bus;
    return 0;
}
```

​		进入of_mdiobus_register看看。

```c
int of_mdiobus_register(struct mii_bus *mdio, struct device_node *np)
{
	struct device_node *child;
	bool scanphys = false;
	int addr, rc;

	mdio->phy_mask = ~0;	//屏蔽mdiobus_register中扫描生成phy device
	mdio->dev.of_node = np;
	mdio->dev.fwnode = of_fwnode_handle(np);

	/* 重点 */
	//注册mdio bus,扫描创建注册phy device.注意上面mdio->phy_mask对每个phy addr
	//都屏蔽了扫描,因此这里不会扫描创建注册任何一个phy device.
	rc = mdiobus_register(mdio);		

    //检查设备树mdio节点,对于每个子节点如果有reg属性则创建注册phy device
    //如果有任何一个子节点没有reg那么scanphys = true,并且不会改回false
    //对每个有reg的子节点则从它的reg提取phy addr,从compatile提取phy id(如果提取
    //不出phy id则创建注册mdio_device代表mdio上通用的设备,这种情况忽略),然后根据
    //phy addr和phy id创建注册phy device.
    for_each_available_child_of_node(np, child) {
        addr = of_mdio_parse_addr(&mdio->dev, child);	//提取reg属性作为phy addr
        if (addr < 0) {		//如果有任何一个子节点没有reg那么scanphys = true,并且不会改回false
            scanphys = true;	
            continue;
        }

        if (of_mdiobus_child_is_phy(child))	//查看compatile属性是否为ethernet-phy-idX.Y格式
            //从compatile提取phy id,和已经提取的phy addr一起创建注册phy device
            rc = of_mdiobus_register_phy(mdio, child, addr);
        else
            rc = of_mdiobus_register_device(mdio, child, addr);
    }
    
	//如果scanphys不为true说明上面的mdio设备树节点的子节点都满足条件
	//并被创建和注册所以就直接返回
    if (!scanphys)
        return 0;

    //如果到了这里说明前面mdio设备树节点存在子节点没有reg属性因此不能直接创建注册phy device,
    //因此这里扫描这些没有reg属性的子节点,通过从0轮询addr来获取一个没有被注册的phy addr,
    //然后从compatile中提取phy id,根据这个phy addr和phy id创建注册phy device,
    //需要注意该phy addr只是软件上和注册的phy device对应,硬件上不一定对应,但是也能正常工作,
    //因为内核已经记录就会认为phy addr就是phy device
    for_each_available_child_of_node(np, child) {
        if (of_find_property(child, "reg", NULL))	//跳过有reg的节点
            continue;

        //对于没有reg的节点,从0开始找到一个没有被注册的phy addr
        //然后注册phy device
        for (addr = 0; addr < PHY_MAX_ADDR; addr++) {
            if (mdiobus_is_registered_device(mdio, addr))	//跳过已经占用的phy addr
                continue;

            if (of_mdiobus_child_is_phy(child)) 
                rc = of_mdiobus_register_phy(mdio, child, addr);
        }
    }

    return 0;
}
```

​		回到stmmac_dvr_probe函数，看完了stmmac_mdio_register后最后一个重点要看的是register_netdev。

```c
int register_netdevice(struct net_device *dev)
{
	int ret;
	struct net *net = dev_net(dev);

	//eth%d变成eth0或者eth1等等
	ret = dev_get_valid_name(net, dev, dev->name);

	//注册
	ret = netdev_register_kobject(dev);	
}
```

​		register_netdevice主要就是设置名字，然后注册。

###### 注册phy

```c
static int __init phy_init(void)
{
	int rc;

	/* 注册mdio总线 */
	rc = mdio_bus_init();	

	/* 将genphy_driver注册到mdio总线,probe在里面设置为phy_probe */
	rc = phy_driver_register(&genphy_driver, THIS_MODULE);
    
	return rc;
}
```

​		首先内核phy_init先注册mdio总线和一个通用的phy驱动genphy_driver，如果没有自己可以匹配的驱动就会用这个来匹配。

```c
#define DP83869_PHY_ID		0x2000a0f1		//phy id
#define DP83869_DEVADDR		0x0A			//phy addr

static struct phy_driver dp83869_driver[] = {
	{
		PHY_ID_MATCH_MODEL(DP83869_PHY_ID),
		.probe          = dp83869_probe,
		.config_init	= dp83869_config_init,
	},
};
        
module_phy_driver(dp83869_driver);

static struct mdio_device_id __maybe_unused dp83869_tbl[] = {
	{ PHY_ID_MATCH_MODEL(DP83869_PHY_ID) },
	{ }
};
MODULE_DEVICE_TABLE(mdio, dp83869_tbl);
```

​		我们项目的phy有另外的驱动注册，不详细分析了，都是些硬件设置的东西。

#####  操作

###### 启动网卡

```c
/* ifconfig ethX up */
...		//调用路径上网查
	stmmac_open	
		stmmac_init_phy(dev);			//创建phylink
		alloc_dma_desc_resources		//申请发送和接收的dma资源
		init_dma_desc_rings				//dma资源生成ring buffer
		request_irq(dev->irq, stmmac_interrupt, ...)	//硬件中断处理函数,发送接收共用
		//启动这些接收和发送队列的napi,没挂入每cpu队列轮询,需要在接收到中断才挂入每cpu队列
		stmmac_enable_all_queues(priv);	
		stmmac_start_all_queues(priv);	//允许上层调用hard_start_xmit
		
```

​		上面是启动网卡做的主要工作，这些步骤意思都很明确，或者看看代码就清楚了，唯一需要再细看的是中断处理函数。

```c
static irqreturn_t stmmac_interrupt(int irq, void *dev_id)		
{
	/* 重点 */
	//读取硬件寄存器判断是接收中断还是发送中断
	//以及对应的哪些队列状态是来数据的
	//然后对每个有数据的队列调用napi_schedule来将本队列
	//的napi_struct挂入每cpu队列并启动软中断
	stmmac_dma_interrupt(priv);		

	return IRQ_HANDLED;
}
```

​		重点只有stmmac_dma_interrupt，再继续进去。

```c
static void stmmac_dma_interrupt(struct stmmac_priv *priv)
{
	int status[max_t(u32, MTL_MAX_TX_QUEUES, MTL_MAX_RX_QUEUES)];
    
	/* 重点 */
	//每个status数组元素既可以表示接收也可以表示发送还可以表示两者都有
	for (chan = 0; chan < channels_to_check; chan++)
		status[chan] = stmmac_napi_check(priv, chan);	
}
```

​		再进去看看stmmac_napi_check。

```c
static int stmmac_napi_check(struct stmmac_priv *priv, u32 chan)
{
	//读寄存器查看发送或者接收或者两者都有
	int status = stmmac_dma_interrupt_status(priv, priv->ioaddr,
						 &priv->xstats, chan);
	struct stmmac_channel *ch = &priv->channel[chan];	//每个channel包含发送和接收队列

	/* 硬件发现有接收中断状态
	 * 首先通过napi_schedule_prep查看当前接收队列napi是否正在poll.如果有直接跳过.
	 * 如果没有则通过stmmac_disable_dma_irq关闭当前接收队列的中断,
	 * 但没有关闭其他队列的中断,这些中断通过一个共享中断来注册,就是注册中断函数这个共享中断.
	 * 最后通过__napi_schedule_irqoff将当前接收队列的napi挂入每cpu队列并开启软中断
	 */
	if ((status & handle_rx) && (chan < priv->plat->rx_queues_to_use)) {
		if (napi_schedule_prep(&ch->rx_napi)) {		//看当前队列的napi是否在运行
			stmmac_disable_dma_irq(priv, priv->ioaddr, chan);	//关闭当前队列的中断
			__napi_schedule_irqoff(&ch->rx_napi);		//将napi挂入每cpu队列并开启软中断
			status |= handle_tx;		//我认为这是为父级函数处理异常情况的
		}
	}

	/* 硬件发现有发送中断状态
	 * 先通过napi_schedule_prep查看当前发送队列napi是否正在poll.如果有直接跳过.
	 * 如果没有则通过__napi_schedule_irqoff将当前发送队列的napi挂入每cpu队列并开启软中断
	 * 注意这里是napi_schedule_irqoff而上面是__napi_schedule_irqoff,
	 * 还有无论是接收还是发送硬件中断都是触发NET_RX_SOFTIRQ软中断
	 */
	if ((status & handle_tx) && (chan < priv->plat->tx_queues_to_use))
		napi_schedule_irqoff(&ch->tx_napi);		//先napi_schedule_prep再__napi_schedule_irqoff

	return status;
}
```

###### 关闭网卡

```c
/* ifconfig ethX down */
...		//调用路径上网查
    stmmac_release
        ...		//和stammc_open相反的操作
```

###### 接收数据

```c
/* step 1 */
网卡收到数据，收满一个dma长度的数据后自动启动dma搬到内存
    
/* step 2 */
dma搬到内存后触发中断(stmmac_interrupt)
    
/* step 3 */
查看哪个通道发生了中断，并且查看发生中断的通道是因为接收还是发送触发的中断
    
/* step 4 */
因为是接收触发的，所以会stmmac_disable_dma_irq关闭当前通道的中断，然后通过
__napi_schedule_irqoff(&ch->rx_napi)将当前接收队列的napi挂入每cpu队列并
开启软中断net_rx_action
    
/* step 5 */
net_rx_action轮询每cpu队列最终会调用到ch->rx_napi的stmmac_napi_poll_rx
    
/* step 6 */
static int stmmac_napi_poll_rx(struct napi_struct *napi, int budget)
{
	work_done = stmmac_rx(priv, budget, chan);	//轮询接收了work_done个dma包

	//如果轮询接收到的dma包小于budget则说明网卡接收缓冲空了,
	//并且通过napi_complete_done通知内核成功从每cpu队列取下napi_struct
	//后调用stmmac_enable_dma_irq重新开启中断
	if (work_done < budget && napi_complete_done(napi, work_done))
		stmmac_enable_dma_irq(priv, priv->ioaddr, chan);
	return work_done;
} 
```

###### 发送数据

```c
/* step 1 */
将数据从应用层传到协议栈最后到驱动层
    
/* step 2 */
dma搬到网卡缓冲区然后硬件自动发送出去，发送出去后触发硬件中断(stmmac_interrupt 注意：发送中断和接收中断是同一个)
    
/* step 3 */
查看哪个通道发生了中断，并且查看发生中断的通道是因为接收还是发送触发的中断
    
/* step 4 */  
因为是发送触发的，所以会通过napi_schedule_irqoff(&ch->tx_napi)将当前接
收队列的napi挂入每cpu队列并开启软中断net_rx_action(注意，和接收一样也是net_rx_action)    

/* step 5 */   
net_rx_action轮询每cpu队列最终会调用到ch->tx_napi的stmmac_napi_poll_tx  
    
/* step 6 */  
stmmac_napi_poll_tx负责清除skb和dma ringbuf内存    
```


#### 5.4 pcie驱动

#####  注册

```c
msi0: msi@0xFF200000 {
	compatible = "altr,msi-1.0";		//pcie-altera-msi.c
	reg = < 0xFF200000 0x00000010
		0xFF200010 0x00000080 >;
	reg-names = "csr", "vector_slave";
	interrupt-parent = < &hps_0_arm_gic_0 >;
	interrupts = < 0 42 4 >;
	msi-controller = < 1 >;
	num-vectors = < 32 >;
};

pcie_0: pcie@0x0 {
	compatible = "altr,pcie-root-port-1.0";		//pcie-altera.c
	reg = < 0xc0000000 0x20000000
		0xFF240000 0x00004000  >;
	reg-names = “Txs”, “Cra”;	// Tx slave port, Control Register Access port
	interrupt-parent = < &hps_0_arm_gic_0 >;
	interrupts = < 0 43 4 >;
	interrupt-controller;
	#interrupt-cells = < 1 >;
	altr,msi = <&msi0>;
	device_type = "pci";
};
```



#####  总结

```c
/* root complex如何扫描pcie设备 */

```



#### 5.5 mtd驱动

#####  注册

#####  调试


### 六、misc

#### 6.1 挂载nfs

```c
//启动后手动挂载到一个指定目录
mount -t nfs -o nolock 192.168.26.133:/home/hanglory/nfs_share /root/nfs
mount -t debugfs none /sys/kernel/debug	//挂载debugfs 
    
//bootloader阶段挂载为根文件系统
bootcmd.nfs=run qspifpga; run bridge_enable_handoff; run qspiload; run nfsboot
nfsroot=192.168.26.133:/home/hanglory/nfs_share/rootfs-best
nfsboot=setenv bootargs console=ttyS0,115200 root=/dev/nfs rw ip=dhcp nfsroot=${nfsroot};bootz ${loadaddr} - ${fdtaddr}

//制作nfs根文件系统
正常制作得到tar然后解压到/home/hanglory/nfs_share/rootfs-best
```

#### 6.2 ssh

```c
/* 服务器配置文件 */
/etc/ssh/sshd_config
PermitRootLogin yes		//允许root用户登录
Port 22	//登录端口

/* 客户端配置文件 */
/etc/ssh/ssh_config

/* 客户端操作 */

//登录
ssh -l sshusr sshdip
ssh -l sshusr -p changedport sshdip

// 退出连接 
exit
```

#### 6.3 buildroot

```c
/* 源码下载 */
https://buildroot.org/download.html
	Older versions can be downloaded from the release archive.	//其他版本
        
/* 配置，主要是工具链 */
Toolchain
	-> Toolchain type 	= External toolchain
	-> Toolchain 		= Custom toolchain			//用户自己的交叉编译器
	-> Toolchain origin	= Pre-installed toolchain		//预装的编译器
	-> Toolchain path	= /home/hanglory/yezheng/gcc-arm-7.5-2019.12-x86_64-arm-linux-gnueabihf 
	-> Toolchain prefix		= $(ARCH)-linux-gnueabihf 	//前缀
	-> External toolchain gcc version  		= 7.x
	-> External toolchain kernel headers series 	= 4.10.x		//交叉编译器的linux版本号
	-> External toolchain C library 			= glibc/eglibc 
	-> [*] Toolchain has SSP support? (NEW) 	//选中
	-> [*] Toolchain has RPC support? (NEW) 	//选中
	-> [*] Toolchain has C++ support? 		//选中
	-> [*] Enable MMU support (NEW)  		//选中   
    
注意：交叉编译器的内核版本号需要和真正的内核的版本号一样或者旧一点。交叉编译器的内核版本号在交叉编译器目录的version.h中#define LINUX_VERSION_CODE 264707以及KERNEL_VERSION(a,b,c) (((a)<<16) + ((b)<<8) + (c)),
其中264707是十进制换成16进制是40a03,所以a是4,b是0a,c是03,所以版本号是4.10.03因此选4.10.x,而我们的内核是5.5,因此该编译器符合要求,如果编译器计算出来的版本号大于5.5那么就要换旧的编译器或者换新的内核。
    
/* 修改Makefile */
.PHONY: target-post-image
target-post-image: $(TARGETS_ROOTFS) target-finalize staging-finalize
     @rm -f $(ROOTFS_COMMON_TAR)
     $(Q)mkdir -p $(BINARIES_DIR)
     #@$(foreach s, $(call qstrip,$(BR2_ROOTFS_POST_IMAGE_SCRIPT)), \
         $(call MESSAGE,"Executing post-image script $(s)"); \
         $(EXTRA_ENV) $(s) $(BINARIES_DIR) $(call qstrip,$(BR2_ROOTFS_POST_SCRIPT_ARGS))$(sep))
```

#### 6.4 altera提供的sdk

##### 6.4.1 buildroot

```c
/******************************************************************************
 * 环境 -- ubuntu14.04
 * buildroot版本 -- 2014.08
 * 编译buildroot工具链 -- 交叉4.9
 ******************************************************************************/

/* 首先切换到buildroot目录 */
cd ~/yezheng/socfpga/buildroot

/* 清洗 */
make clean

/* 配置 */
cp yzconfig .config 
make menuconfig

/* 编译 */
make 
make	//也许你很疑惑，我他妈也是啊，第一次会报错，第二次就好了

```

##### 6.4.2 linux kernel

```c
/******************************************************************************
 * 环境 -- ubuntu14.04
 * kernel版本 -- 3.10
 * 编译kernel工具链 -- 交叉4.9
 ******************************************************************************/

/* 首先切换到kernel目录 */
cd ~/yezheng/socfpga/linux-socfpga-socfpga-3.10-ltsi

/* 导出环境变量 */
export CROSS_COMPILE=/home/yeshen/yezheng/socfpga/toolchain/gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux/bin/arm-linux-gnueabihf-
export ARCH=arm
export PATH=/home/yeshen/yezheng/socfpga/u-boot-socfpga/tools:$PATH

/* 大清洗 */
make distclean

/* 配置 */
make socfpga_defconfig

/* 编译内核 */
make zImage -j8
make uImage LOADADDR=0x8000 -j8

/* 编译设备树 */
make socfpga_cyclone5.dtb -j8

/* 编译模块 */
make modules -j8

/* 安装模块，将包含模块的lib/modules/...安装到/home/yeshen/yeshen-disk目录 */
make INSTALL_MOD_PATH=/home/yeshen/yeshen-disk modules_install -j8
```

##### 6.4.3 preloader和uboot

```c
/******************************************************************************
 * 环境 -- ubuntu14.04
 * 版本 -- 2013.01
 * 编译工具链 -- 交叉4.9
 * 注意 -- 只是编译，无法直接在sd卡中替换
 ******************************************************************************/

export CROSS_COMPILE=/home/yeshen/yezheng/socfpga/toolchain/gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux/bin/arm-linux-gnueabihf-
export ARCH=arm

cd ~/yezheng/socfpga/u-boot-socfpga

make mrproper
make socfpga_cyclone5_config
make 
```


##### 6.4.4 实时内核

> 注：altera论坛提供的已经打好preempt rt补丁的3.10内核

```c
/******************************************************************************
 * 环境 -- ubuntu14.04
 * kernel版本 -- 3.10
 * 编译kernel工具链 -- 交叉4.9
 ******************************************************************************/

/* 首先切换到kernel目录 */
cd ~/yezheng/socfpga/linux-socfpga-socfpga-3.10-ltsi-rt

/* 导出环境变量 */
export CROSS_COMPILE=/home/yeshen/yezheng/socfpga/toolchain/gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux/bin/arm-linux-gnueabihf-
export ARCH=arm
export PATH=/home/yeshen/yezheng/socfpga/u-boot-socfpga/tools:$PATH

/* 大清洗 */
make distclean

/* 配置 */
make socfpga_defconfig

/* 编译内核 */
make zImage -j8
//make -j8 zImage 2>&1 | tee ../linux-rt-log.txt
make uImage LOADADDR=0x8000 -j8

/* 编译设备树 */
make socfpga_cyclone5.dtb -j8

/* 编译模块 */
make modules -j8

/* 安装模块，将包含模块的lib/modules/...安装到/home/yeshen/yeshen-disk目录 */
make INSTALL_MOD_PATH=/home/yeshen/yeshen-disk modules_install -j8
    
```

##### 6.4.5 eds制作设备树

```c
/* 导出eds路径到环境变量$PATH */
$ cd ~/altera/15.0/embedded/
$ ./embedded_command_shell.sh

/* 制作dts */
$ cd ~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd
$ sopc2dts --input soc_system.sopcinfo \
 --output socfpga.dts --type dts \
 --board soc_system_board_info.xml \
 --board hps_common_board_info.xml \
 --bridge-removal all \
 --clocks
 
/* 编译dtb，不是用socfpga.dts，源和制作dts一样都是soc_system.sopcinfo */ 
$ cd ~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd
$ sopc2dts --input soc_system.sopcinfo \
 --output socfpga.dtb --type dtb \
 --board soc_system_board_info.xml \
 --board hps_common_board_info.xml \
 --bridge-removal all \
 --clocks
```

#### 6.5 preempt rt

```c
/******************************************************************************
 * 环境 -- 公司服务器
 * kernel版本 -- 5.4.161
 * 编译kernel工具链 -- 公司服务器工具链7.5
 ******************************************************************************/    
    
/* 下载源码和补丁 */
https://wiki.linuxfoundation.org/realtime/preempt_rt_versions	//补丁网站
https://www.kernel.org/pub/		//内核网站

/* 打补丁 */
将补丁和源码文件夹放在同一级目录下
cd到源码文件夹
patch -p1 < ../xx.patch
    
/* 配置编译 */
make distclean
make socfpga_defconfig
配置CONFIG_PREEMPT_RT
配置CONFIG_HIGH_RES_TIMERS
配置CONFIG_NO_HZ		//可能不用配这个
make zImage -j8
...
```

#### 6.6 xenomai

##### 6.6.1 打xenomai补丁

```c
/******************************************************************************
 * 环境 -- 公司服务器
 * kernel版本 -- 4.19.82
 * 编译kernel工具链 -- 公司服务器工具链7.5和自己下载的11.3.1都可以
 ******************************************************************************/    
    
/* 下载源码和补丁 */
https://www.kernel.org/pub/		//linux内核网站
https://xenomai.org/downloads/xenomai/stable/latest/	//xenomai源码网站(cobalt补丁)
https://xenomai.org/downloads/ipipe/	//ipipe补丁

/* 打补丁 */
将linux源码和xenomai源码(cobalt补丁)和ipipe补丁放在同一级目录下
cd到xenomai源码文件夹
./scripts/prepare-kernel.sh --linux=../linux-4.19.82 \
	--ipipe=../ipipe-core-4.19.82-arm-6.patch --arch=arm
    
/* 配置编译已经打补丁的内核 */
cd到linux内核目录
make distclean
make socfpga_defconfig
配置Preemtiple Kernel			//这个配置应该是非xenomai状态下的抢占模式			
配置CONFIG_HIGH_RES_TIMERS	//在socfpga_defconfig中已经配置了
去掉配置CONFIG_MIGRATION	//通过警告下面的内存管理去掉内存规整
make zImage -j8
...
     
```

##### 6.6.2 xenomai用户空间

```c
//使用buildroot
Target packages  
    -> Real-Time
    	-> Xenomai Userspace
```

##### 6.6.3 xenomai驱动

```c

```

##### 6.6.4 linux、preempt rt和xenomai的性能测试对比

```c

```

#### 6.7 toolchain

```c
http://releases.linaro.org/components/toolchain/binaries/
https://www.linaro.org/downloads/			//较新
	 Arm Developer website
	 GNU Toolchain Integration Builds	//这个更通用感觉
```

#### 6.8 公司uboot

##### 6.8.1 遗留环境变量

###### 不配置fpga的bootcmd

```c
/* 倒计时结束后自动执行或者boot手动执行 */
bootcmd=mw.b 0x100 0xff 0x700000;tftp 0x8000 zImage;tftp 0x100 socfpga_cyclone5_socdk.dtb;run qspiboot

/* 启动内核 */
qspiboot=setenv bootargs console=ttyS0,115200 root=${qspiroot} rw rootfstype=${qspirootfstype};bootz ${loadaddr} - ${fdtaddr}
```

###### 配置fpga的bootcmd

```c
/* 倒计时结束后自动执行或者boot手动执行 */
bootcmd=run qspifpga; run bridge_enable_handoff; run qspiload; run qspiboot

/* 从qspi加载rbf到内存，然后再加载入fpga */
qspifpga=sf probe ${qspiloadcs}; sf read ${fpgadata} ${qspifpgaaddr} ${fpgadatasize};fpga load ${qspiloadcs} ${fpgadata} ${fpgadatasize}                               

/* 配置fpga */
bridge_enable_handoff=mw $fpgaintf ${fpgaintf_handoff}; go $fpga2sdram_apply; mw $fpga2sdram ${fpga2sdram_handoff}; mw $axibridge ${axibridge_handoff}; mw $l3remap ${l3remap_handoff}

/* 从qspi读取内核和设备树到内存 */
qspiload=sf probe ${qspiloadcs};sf read ${loadaddr} ${qspibootimageaddr} ${bootimagesize};sf read ${fdtaddr} ${qspifdtaddr} ${fdtimagesize};
 
/* 启动内核 */                                 
qspiboot=setenv bootargs console=ttyS0,115200 root=${qspiroot} rw rootfstype=${qspirootfstype};bootz ${loadaddr} - ${fdtaddr}       

/* 待整理  */
setenv qspifpga  sf probe ${qspiloadcs}; sf read ${fpgadata} ${qspifpgaaddr} ${fpgadatasize}; fpga load ${qspiloadcs} ${fpgadata} ${fpgadatasize};
setenv resetfpga  run h2f_reset_handle

setenv bootcmd run qspifpga; run bridge_enable_handoff; run h2f_reset_handle; run qspiload; run qspiboot

fatload mmc 01 $fpgadata soc_system.rbf;fpga load 0 $fpgadata $filesize;run bridge_enable_handoff; run h2f_reset_handle;run mmcload;run mmcboot;
```

##### 6.8.2 修改文件

```c
main.c
socfpga_common.h
socfpga_common.c   	//setenv_ethaddr_eeprom    
hanglory_config.h
watchdog.h
designware_wdt.c
time.c
start.S
cmd_hangloryboot.c	//legacy
hanglory_spl_spi.c	//legacy
spl.c
```

