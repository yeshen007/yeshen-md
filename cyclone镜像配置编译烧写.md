## <center>cyclone镜像配置编译烧写</center>

[TOC]

**注意：**uboot和内核的交叉编译工具链是由`~/.profile`中的环境变量指定，而根文件系统的交叉编译工具链自己在`make menuconfig`中指定。
**源码原始地址：**

```c
/* preloader和uboot */
~/yezheng/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd/software/spl_bsp
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
 * 替换目录~/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd
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

&emsp;&emsp;最后编译得到的spl和uboot镜像分别在`spl_bsp/uboot-socfpga/spl`和`spl_bsp/uboot-socfpga`中。但板子要用的spl是spl_bsp中的preloader-mkpimage.bin，而不是`spl_bsp/uboot-socfpga/spl`中的u-boot-spl.bin，用的uboot是`spl_bsp/uboot-socfpga`中的u-boot.img。



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

```c
/* 在uboot命令行阶段设置环境变量和激活qspi flash */
setenv ipaddr 192.168.34.150	//设置板子的ip，到内核会被重新设置我们不管，反正uboot阶段就是我们设置的
setenv serverip 192.168.34.149 	//设置tftp服务器的ip，下载的是/home/hanglory/tftpboot中的文件   
  
/* 激活qspi */   
sf probe	
    
/* 擦除所以镜像 */
sf erase 0x0000 0x2000000

/* 烧写preloader到qspi 0x00000*/
擦除内存：mw.b 0x8000 0xff 0x50000
Tftp下载uboot：tftp 0x8000 preloader-mkpimage.bin
烧写uboot：sf update 0x8000 0x00000 0x50000

/* 烧写uboot到qspi 0x60000*/
擦除内存：mw.b 0x8000 0xff 0x50000
Tftp下载uboot：tftp 0x8000 u-boot.img
烧写uboot：sf update 0x8000 0x60000 0x50000
    
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
Tftp下载rbf：tftp 0x8000 printerhead.rbf	//tftp 0x8000 printhead_v3.rbf
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

### 五、正式板uboot启动环境变量

#### 5.1 不配置fpga的bootcmd

```c
/* 倒计时结束后自动执行或者boot手动执行 */
bootcmd=mw.b 0x100 0xff 0x700000;tftp 0x8000 zImage;tftp 0x100 socfpga_cyclone5_socdk.dtb;run qspiboot

/* 启动内核 */
qspiboot=setenv bootargs console=ttyS0,115200 root=${qspiroot} rw rootfstype=${qspirootfstype};bootz ${loadaddr} - ${fdtaddr}
```

#### 5.2 配置fpga的bootcmd

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



### 六、misc

#### 6.1 挂载nfs

```c
mount -t nfs -o nolock 192.168.34.119:/home/hanglory/nfs_share /root/nfs
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

#### 6.3 eds制作设备树

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

#### 6.4 评估板3.10

#### 6.4.1 buildroot

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

#### 6.4.2 linux kernel

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

#### 6.4.3 preloader和uboot

```c
/******************************************************************************
 * 环境 -- ubuntu14.04
 * 版本 -- 2013.01
 * 编译工具链 -- 交叉4.9
 * 注意 -- 只是编译，无法直接在sd卡中替换
 ******************************************************************************/

export CROSS_COMPILE=/home/yeshen/yezheng/socfpga/toolchain/gcc-linaro-arm-linux-gnueabihf-4.9-2014.09_linux/bin/arm-linux-gnueabihf-
export ARCH=arm

make mrproper
make socfpga_cyclone5_config
make 
```


#### 6.4.4 实时内核

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



