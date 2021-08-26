## <center>cyclone镜像配置编译</center>

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


/* 公司版 */
cd ~/yezheng/altera/15.0/embedded 
./embedded_command_shell.sh
cd ~/yezheng/altera/15.0/embedded/examples/hardware/cv_soc_devkit_ghrd/software/spl_bsp
make clean
make
make uboot
```

&emsp;&emsp;最后编译得到的spl和uboot镜像分别在`${ubootroot}/spl`和`${ubootroot}`中，`${ubootroot}`是指uboot源码根目录。



### 二、内核和设备树和模块配置编译

```c
/* 清洗 */
make distclean 或者 make clean //make clean就不需要配置,.config还在

/* 配置 */
make socfpga_defconfig
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
/* 安装模块 */
make INSTALL_MOD_PATH=/home/hanglory/nfs_share/yz modules_install -j12
```

&emsp;&emsp;最后编译得到的内核和设备树镜像在`${kernelroot}/arch/arm/boot`和`${kernelroot}/arch/arm/boot/dts`中，`${kernelroot}`是内核源码根目录。



### 三、根文件系统配置编译

```c
/* 大清洗 */
make distclean

/* 小清洗 */
make clean

/* 配置 */
cp yzconfig .config
make menuconfig

/* 编译 */
make -j12
```

&emsp;&emsp;编译后得到的根文件系统镜像在`${buildroot}/output/images`中，如果在buildroot配置了jffs则会额外生成rootfs.jffs2，可以将它烧写到qspi中。



### 四、烧写镜像到qspi

&emsp;&emsp;有两种方法烧写镜像到qspi。一种是通过uboot命令烧写，但前提是要有网可以用tftp命令从主机先下载镜像到板子内存，然后通过sf系列命令将内存中的镜像烧写到fpga。另一种是使用jtag和altera的软件烧写。

#### 4.1 uboot命令方法

```c
/* 在uboot命令行阶段设置环境变量和激活qspi flash */
setenv ipaddr 192.168.34.150	//设置板子的ip，到内核会被重新设置我们不管，反正uboot阶段就是我们设置的
setenv serverip 192.168.34.149 	//设置tftp服务器的ip，下载的是/home/hanglory/tftpboot中的文件    
sf probe	//激活qspi

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
Tftp下载rbf：tftp 0x8000 soc_system.rbf
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
```

