## <center>cyclone镜像配置编译</center>

[TOC]

**注意：**uboot和内核的交叉编译工具链是由`~/.profile`中的环境变量指定，而根文件系统的交叉编译工具链自己在`make menuconfig`中指定。

---

### 一、uboot配置编译

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



### 二、内核配置编译

```c
/* 清洗 */
make distclean 或者 make clean

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

&emsp;&emsp;编译后得到的根文件系统镜像在`${buildroot}/output/images`中。

