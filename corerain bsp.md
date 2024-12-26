<center>corerain bsp</center>

[TOC]

### 一. 开发环境

#### 1. 账号登录

```c
/* haps服务器 */
addr: 192.168.11.216
account: syshaps
passwd: admin
登录方式：通过mobaxterm或类似工具ssh连接

/* haps串口 */
addr: 192.168.11.215
account: corerain
passwd: 123456
登录方式：通过mobaxterm或类似工具ssh连接

/* 开发服务器 */
addr: 192.168.1.227
account: zye
passwd: letmein
登录方式：通过mobaxterm或类似工具ssh连接

/* gitlab */
addr: http://192.168.1.200/
account: zye
passwd: 8702642ye
登录方式：直接将addr输入浏览器

/* 共享文件系统 */
addr: http://192.168.11.36:8100/DocSystem/web/index.html
account: zye
passwd: ky123456
登录方式：直接将addr输入浏览器

/* 远程办公 */
addr: 183.56.145.96 4443
account: zye
passwd: Kyun@yz
登录方式：参考it文档

/* 云桌面 */
addr: 192.168.11.146
account: zye
passwd: Yezheng@123
登录方式：通过办公室pc或者家里电脑vpn远程控制办公室pc打开远程桌面连接来连接

/* 本机ip账号密码 */
192.168.11.84
255.255.255.0
192.168.11.1
Administrator
yes!00
    
/* atrust */
zye
8702642yes!00
    
/* 阿里企业邮箱 */
zheng.ye@corerain.com
8702642yE@

192.168.11.218
corerain
corerain

//192.168.11.149
root
Test@2024

//n网
569242715@qq.com
8702642yE@yes

//appleid
569242715@qq.com
8702642yS

//phub
569242715@qq.com
8702642yE@

//instagram
569242715@qq.com
8702642yE@

//youtube
569242715@qq.com
15895833516ye

//谷歌浏览器
569242715@qq.com
15895833516ye

//x
569242715@qq.com
15895833516ye

//github
yeshen007或者569242715@qq.com
8702642ye
```

#### 2. 代码编译

```c
/* 配置环境变量 */
source  /opt/arm.env

/* 配置编译内核和设备树,得到arch/arm64/boot/Image和arch/arm64/boot/dts/corerain/bass-haps.dtb */
cd /home/zye/kernel
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- bass_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- menuconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j16
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- distclean

/* 编译spacc驱动模块和测试应用 */
首先编译内核
cd /home/zye/spacc
make -j16
make clean 

/* 编译根文件系统 */
cd /local/zye/make_image/buildroot
make bass_minimal_glibc_defconfig
make menuconfig
make

/* spacc驱动模块和测试用例拷贝到cpio文件系统的以下目录 */
path_to_your_cpiofs/usr/local/spacc
```

#### 3. 仿真运行

```c
/* 先登录215链接串口,216上跑工程在215上打印 */
sudo minicom

/* 切换到216 haps服务器 */
cd /home/syshaps/workspace/zye/haps/bin

/* scp拷贝247服务器编译好的镜像到该目录(如果重新编译内核和设备树的话) */
scp zye@192.168.11.247:/home/zye/kernel/arch/arm64/boot/dts/corerain/bass-haps.dtb XX
scp zye@192.168.11.247:/home/zye/kernel/arch/arm64/boot/Image XX

/* scp拷贝247服务器编译好的驱动 */
scp zye@192.168.11.247:/home/zye/spacc/bin/spacc-crypto.ko .

/* 配置模拟硬件(./zye_config) */
//config_haps /home/syshaps/workspace/storage/yqwu/haps_projects/confpro_ws_20230325_4boot_20230412/designs
config_haps /home/syshaps/workspace/storage/yqwu/haps_projects/confpro_ws_20230430_4boot_media/designs

/* 将镜像写入ddr并启动(./run.sh) */
./ddr_writing bl31.bin 0x400000000
./ddr_writing bass-haps.dtb 0x400100000
./ddr_writing Image 0x400200000
./ddr_writing xxx.cpio 0x440000000
./write_csr 0x403000000 0x20221109
    
/* 最后查看215串口打印和操作加载驱动模块或者测试用例 */
等好一会
```

#### 4. misc

```c
/* 查看文档 */
首先登陆共享文件系统
	进入系统
		hw_doc
			CAISA4(Bass)
				3_总体设计		//整体架构文档
				4_详细设计		//ip模块详细文档
				5_软件设计		//软件相关文档
					Bass_Address_Mapping	//物理地址映射
    
/* 拷贝文件到hpas文件系统 */    
cd /home/syshaps/workspace/zye/haps/cpio_mnt
cpio -iv < ../bin/rootfs_haps_crypto.cpio		//解压到目录
...		//增删改
注释掉/etc/inittab	#ttyS1::respacd wn:/sbin/getty...	//目前不知道有啥用
find . | cpio --quiet -o -H newc > ../bin/rootfs_haps_new.cpio	//重新打包
参考HAPS_UG.pdf更改bass-haps.dts重新编译内核和设备树		//如果文件系统大小变化了

/* 查看交叉编译工具aarch64-none-linux-gnu-gcc的头文件库文件默认搜素路径 */
aarch64-none-linux-gnu-gcc -print-sysroot

/* arm和216传送文件 */
//arm收216发
./soc_trans_receiver USB
./usb_trans_sender 0x80000000 tarm

//216收arm发
./usb_trans_receiver 0x80000000
./soc_trans_sender USB dir/file

/* 启动haps */
 source ~/.bashrc
 sudo su
 cd /root
 ./rc.local
 exit
 
 /* trace32 */
 ./trace32_config
 bin\windows64\t32marm.exe -c config_eth.t32 -s scripts\bass_haps.cmm
 
 /* 复位工程 */
 reset_haps .

/* git log 图形化查看 */
git log --all --pretty=oneline --abbrev-commit --graph

/**/
git diff <old-commit> <new-commit> --name-only
git diff <old-commit> <new-commit> file

git checkout <some-commit> -- file


/* 查看修改git远程仓库 */  
git remote -v
git remote set-url <remote-name> <new-url>		//<remote-name>是远程仓库的名称（通常为"origin"）

/* git解决冲突 */
git merge <target_branch>
...	//fix conflit
git add <conflit file>
git merge --continue
    
    
/* */
vim plat/corerain/bass/include/platform_def.h 


/* 编译bl2 */
make distclean
make PLAT=bass DEBUG=0 LOG_LEVEL=40 CROSS_COMPILE=aarch64-none-elf- HAPS=0 HAPS_SKIP_BL2=0 BASS_CHIP_TEST=0 BASS_DDR0_ONLY=1 BASS_UBOOT=1 BASS_BL2=1 bl2	//ddr0

/* 编译bl3 */
make distclean
make PLAT=bass DEBUG=0 LOG_LEVEL=40 CROSS_COMPILE=aarch64-none-elf- HAPS=0 HAPS_SKIP_BL2=0 BASS_CHIP_TEST=0 BASS_DDR0_ONLY=1 BASS_UBOOT=1 BASS_BL2=0 bl31	//ddr0

/* 编译uboot */
make distclean
export CROSS_COMPILE=aarch64-none-linux-gnu- 
make bass_ddr0_mini_defconfig	//ddr0      ddr2:make bass_mini_defconfig
make


/* */
build/bass/debug/bl2.bin
build/bass/debug/bl31.bin

/* bl2.bin转化位bl2.img */
cd /local/zye/make_image/rsa/build
cp /local/zye/arm-trusted-firmware/build/bass/debug/bl2.bin .
./rsa_sign bl2.bin
./pkgtool bl2.bin rsa_pub_N.bin 0x10001 bl2.bin.sig.bin
cp fip_all.img bl2.img
cp bl2.img /local/zye/arm-trusted-firmware/build/bass/debug/

scp bl2.img admin@192.168.11.218:/home/admin/workspace/zye/
sudo dd if=bl2.img of=/dev/sdb1;sync

sudo dd if=/dev/zero of=/dev/sdb3;sync

/**/
PC测试网络配置：
sudo ifconfig ceth0 up
sudo ifconfig ceth0 192.168.222.2
bass侧网络配置
ifconfig meth0 up
ifconfig meth0 192.168.222.1
    
/**/
第一步先登录到服务器
ssh admin@192.168.11.20
第二步从服务器登录到板卡
ssh root@192.168.222.1
第三步 从服务器拷贝数据到板卡
scp -r xxx root@192.168.222.1:~


/* i2c */
//初始化
dw_i2c_plat_probe
	i2c_dw_configure
	i2c_dw_probe
    	i2c_dw_probe_master
    		dev->init(dev)		//i2c_dw_init_master
    		i2c_dw_disable_int
    		devm_request_irq	

//启动读写
ioctl(file,I2C_SMBUS,&args)	//应用层
i2cdev_ioctl	
	copy_from_user
	i2cdev_ioctl_smbus
		i2c_smbus_xfer
			__i2c_smbus_xfer
				i2c_smbus_xfer_emulated
					__i2c_transfer
						adap->algo->master_xfer(adap, msgs, num)	//i2c_dw_xfer
						                                                            
i2c_dw_xfer                                                                          
	i2c_dw_wait_bus_not_busy
	i2c_dw_xfer_init
	wait_for_completion_timeout
	__i2c_dw_disable_nowait
    

//处理中断
i2c_dw_isr
	i2c_dw_irq_handler_master
		stat = i2c_dw_read_clear_intrbits
		if (stat & DW_IC_INTR_RX_FULL)
			i2c_dw_read(dev);	
		if (stat & DW_IC_INTR_TX_EMPTY)
			i2c_dw_xfer_msg(dev);
		complete(&dev->cmd_complete);	


/* 
 * max20796 sysfs操作 
 */

//在驱动pmbus.c中确认名字max20796，0x11是max20796的i2c地址，连接在i2c0总线上
echo max20796 0x11 > /sys/bus/i2c/devices/i2c-4/new_device 	

//查看新生成的max20796 sysfs目录，比如是/sys/class/hwmom/hwmonX
ls -l /sys/class/hwmom	

//查看max20796 sysfs文件 
cd /sys/class/hwmom/hwmonX
ls

//查看标签，比如in1_label是vin，代表in1_开头的文件都是输入电压相关的数值
cat XX_label

//查看输入或输出电压
cat inx_input

//查看输入或输出电流
cat currx_input


/* 
 * ina226 sysfs操作 
 */
 
 //查看对应生成的sysfs目录
 cd /sys/class/hwmon
 ls -l
 
 //进入其中一个
 cd /sys/class/hwmon/hwmonX
 
 //读电压
 cat in1_input
 
 //读电流
 curr1_input
 
 //读功率
 cat power1_input
 
 
/*
 * uart操作
 */
cat /proc/tty/driver/serial
 
//232
stty -F /dev/ttyS9 ispeed 115200 ospeed 115200 cs8 -parenb -cstopb -echo raw
echo xxx > /dev/ttyS9
cat /dev/ttyS9 

//485
stty -F /dev/ttyS9 ispeed 115200 ospeed 115200 cs8 -parenb -cstopb -echo raw
devmem 0x1060004 32 0xc0020000
devmem 0x1060000 32 0xc0020000
echo xxx > /dev/ttyS9
devmem 0x1060004 32 0xc0020000
devmem 0x1060000 32 0xc0000000
cat /dev/ttyS9


/* 
 * reboot流程
 */
1. 用户输入reboot
2. 进入系统调用reboot		//SYSCALL_DEFINE4(reboot
3. kernel_restart
4. machine_restart
5. arm_pm_restart	//psci_sys_reset
6. invoke_psci_fn	//__invoke_psci_fn_smc
7. arm_smccc_smc	//触发el3异常进入arm_trusted_firmware
8. bl31_entrypoint -> ... -> smc_handler64
9. std_svc_smc_handler
10. psci_smc_handler
11. psci_system_reset
12. bass_system_reset

/* 
 * poweroff流程
 */
1. 用户输入poweroff
2. 进入系统调用reboot		//SYSCALL_DEFINE4(reboot
3. kernel_power_off
4. machine_power_off
5. pm_power_off		//psci_sys_poweroff
6. invoke_psci_fn	//__invoke_psci_fn_smc
7. arm_smccc_smc	//触发el3异常进入arm_trusted_firmware
8. bl31_entrypoint -> ... -> smc_handler64
9. std_svc_smc_handler
10. psci_smc_handler
11. psci_system_off
12. bass_system_off

/*
 * ds1337 rtc（0x68 i2c4）
 */ 
 date	//查看系统时钟
 date -s '2024-04-24 11:25:00' '+%s'	//设置系统时钟
 cd /sys/class/rtc/rtc0	//看看有没有这个目录
 hwclock --systohc	//系统时钟同步到rtc
 cat /sys/class/rtc/rtc0/date	//查看rtc日期
 cat /sys/class/rtc/rtc0/time	//查看rtc时分秒
 
 /*
  * 板级tsensor tmp75b（0x48 i2c4）
  */
 cd /sys/class/hwmom/hwmonX
 cat temp1_input	//38062(38.062度)
 
 
 /*
  * mcu升级
  */
IAP烧录方式的烧录逻辑是：在内部Flash的最前面烧录第1个程序（称为Bootloader程序），其次在相对08000000具有偏移的位置存储第2个程序（称为APP程序）。在Bootloader程序中，完成APP程序的更新，以及跳转至APP程序执行。

fromelf --bin --output "$L@L.bin" "#L"
    
__set_PRIMASK(0); //开启所有中断   
__set_PRIMASK(1);//关闭所有中断
__disable_irq();//关闭所有中断
__set_FAULTMASK(1);//关闭所有中断

./pack_swupdate.sh mcu
scp to board
swupdate -i xx.swu

/*
 * dpkg
 */
cd rbcrypto
./pack.sh rbcrypto 1.0.0 caisa430	//./pack.sh rbcrypto 1.0.0 caisa430
dpkg -i rbcrypto_v1.0.0_ubuntu_caisa430.deb
dpkg -l | grep rbcrypto
dpkg -r rbcrypto

/*
 *
 */
./pack_rpm.sh rbcrypto 1.2.0
./pack_rpm.sh rbperipherals 1.1.6
   
 rpm -i   
 rpm -qa
 rpm -e

/*
 * 去除^M
 */
 sed -i 's/\r$//' pack.sh


source-directory /etc/network/interfaces.d
auto eth0
iface eth0 inet static
address 192.168.11.7
netmask 255.255.255.0
gateway 192.168.11.1

serial8250_init    
    serial8250_isa_init_ports    
        serial8250_init_port
            serial8250_pops	//
                serial8250_set_termios
                	serial8250_do_set_termios
    	
dw8250_set_termios
	serial8250_do_set_termios
		serial8250_get_baud_rate
		serial8250_get_divisor
		serial8250_set_divisor
			serial8250_do_set_divisor
				serial_dl_write(up, quot)
/*
 * n460h mcu
 */
1. mcu上电时硬件也会拉高12v和拉高bass1的por,然后由bass1的软件先后拉高bass2和bass3。
2. mcu起来后通过uart监控bass1命令或者主动发命令给bass1。
3. 在bass1起来后通过uart协议让mcu监控几个电源状态并通过uart发送给bass1，如果有异常bass1再打印出来。

// n460交叉编译 192.168.1.227
cd build
cmake3 .. -DCMAKE_TOOLCHAIN_FILE=/opt/rb/cc/cc.cmake  -DCMAKE_INSTALL_PREFIX=/opt/rb/
make

cmake3 .. -DCMAKE_TOOLCHAIN_FILE=/local/zye/opt/rb/cc/cc.cmake  -DCMAKE_INSTALL_PREFIX=/opt/rb/cc/sysroot/
make
make install DESTDIR=./
    
cmake3 .. -DCMAKE_TOOLCHAIN_FILE=/local/zye/opt/rb/cc/cc.cmake  -DCMAKE_INSTALL_PREFIX=/opt/rb/
make
# 安装到./localrb/opt/rb
make install DESTDIR=./localrb
# 安装到/opt/rb/cc/sysroot_bsp/opt/rb
sudo make install DESTDIR=/opt/rb/cc/sysroot_bsp/
# 安装到/opt/rb/cc/rootfs/opt/rb
sudo make install DESTDIR=/opt/rb/cc/rootfs/
# 安装到/opt/rb
sudo make install


//rbcv
yuv2rgb
rgb2yuv

pp_dbuf
pbuf	//地址生成器

./yuv2rgba /dev/ve0 ../yuv2rgba/480_272/ahb_reg ../tests/golden/272_480/272_480_u8_nv12.yuv 272 480 testo.rgb


//mcu问题
1. i2c发送可以修改
		/* send a data byte */
		i2c_data_transmit(I2C0, *i2c_txbuffer++);
		i2c_tx_nbytes++;
2.
   
//ipmb    
智能平台管理总线或 IPMB 是一种 I2C 总线，为机箱内不同板卡之间提供标准化互连。

//问题1
ipmb response第一行最开始是否有rqSA
答：没有，从linux驱动代码中可以看到

//问题2
checksum怎么计算
答：加起来等于0

//问题3
smc如何response,自己作主还是等待bmc读
答：自己作主


NetFn: 6 bit
LUN: 2 bit
Command: 8 bit
Completion Code: 8 bit

Note that the completion code of a response is the first item in “data”
 
ipmb-dev-int - This is the driver needed on a Satellite MC to receive IPMB messages from a BMC and send a response back. This driver works with the I2C driver and a userspace program such as OpenIPMI:
1. It is an I2C slave backend driver. So, it defines a callback function to set the Satellite MC as an I2C slave. This callback function handles the received IPMI requests.
2. It defines the read and write functions to enable a user space program (such as OpenIPMI) to communicate with the kernel.
    
static unsigned char
ipmb_checksum(unsigned char *data, int size)
{
	unsigned char csum = 0;

	for (; size > 0; size--, data++)
		csum += *data;

	return -csum;
}  


ipmi_ioctl(IPMICTL_SEND_COMMAND)
    handle_send_req
    	ipmi_request_settime
        ...	
			format_ipmb_msg

Reset_Handler
	SystemInit
    	system_clock_config
    		system_clock_120m_hxtal
    			#ifdef HXTAL_VALUE_8M
    			#elif defined (HXTAL_VALUE_25M)		//GD32E103V_EVAL
    			#endif
	main
		systick_config
			SysTick_Config
		usart_baudrate_set

stty -F /dev/ttyS3 ispeed 115200 ospeed 115200 cs8 -parenb -cstopb -echo raw
cat /dev/ttyS3
echo -ne "\xa0\xa1" > /dev/ttyS3

//从pcie服务器启动bass
sudo /opt/rb/bin/pcie_start_first_level  99 /opt/rb/image/

//192.168.1.227连接192.168.99.102
先连接192.168.11.214		//corerain corerain
再连接192.168.99.102		//root admin(可能需要配置服务器ip)
    
//查看是否建链
lspci | grep 1f4a
    
//最新一台
/opt/rb/bin/n460x_bringup_os 192.168.99.101 /opt/rb/image/

//460拷文件到bass
192.168.1.227拷到192.168.11.5
192.168.11.5拷到连接板子的host
host拷贝到板子

i2cdetect -l
i2cdetect -y -r 4
    
    
//读
i2cget -y 4 0x13		
i2cget -y 4 0x13 0x1 
i2cget -y 4 0x13 0x1 w	 
i2ctransfer -y -f 4 w1@0x13 0x01 r4
    
//写    
i2cset -y 4 0x13 0x1 
i2cset -y 4 0x13 0x1 0x2    
i2cset -y 4 0x13 0x1 0x2030 w   

//举例
i2cset -y 4 0x13 0x7 0x6688 w	//设置device id
i2cget -y 4 0x13 0x6 w			//读取device id

echo 0 > /sys/class/pwm/pwmchip0/export
echo 200000 > /sys/class/pwm/pwmchip0/pwm0/period
echo 100000 > /sys/class/pwm/pwmchip0/pwm0/duty_cycle
echo 1 > /sys/class/pwm/pwmchip0/pwm0/enable

echo 0 > /sys/class/pwm/pwmchip0/pwm0/enable
echo 0 > /sys/class/pwm/pwmchip0/unexport

//堡垒机
192.168.1.100  zye letmein  2222
    
    
//


板级t-sensor的数据
板级电源芯片的数据
处理的话建议写到文件里面，然后还要发送到bass2 bass3，最终在每个bass上面我都可以读取文件里面的信息获取到温度和电流 电压。

状态0：50%
状态0->状态1：当温度大于75C，风扇从50%调到65%；状态1->状态0：在状态1下，温度降低到65C时，风扇调回50%
状态1->状态2：当温度大于80C，风扇从65%调到80%；状态2->状态1：如果在状态2下，温度降低到70C时，风扇调回65%
状态2->状态3：当温度大于85C，风扇从80%调到100%；状态3->状态2：如果在状态3下，温度降低到75C时，风扇调回80%

    
cd /root/e2e_rt2.0_yolov3_416x416/
nohup ./example_e2e cfg/430P/eng0.ini &> eng0.log &
nohup ./example_e2e cfg/430P/eng1.ini &> eng1.log &
    
cat /sys/kernel/debug/pinctrl/soc\:pinctrl-bass-pinctrl/gpio-ranges

//cmake命令行传参
cmake -DMY_MACRO_DEFINED=ON
if(MY_MACRO_DEFINED)
    add_definitions(-DMY_MACRO)
    add_definitions(-DMY_MACRO=0x1000)
    add_compile_definitions(MY_MACRO)		//新版本有的命令
    add_compile_definitions(MY_MACRO=0x1000)//新版本有的命令
endif()

    
    
温度监控
N460: 板温超过85度，或者结温超过110度，强制断电
N430: 板温超过85度，或者结温超过110度，强制断电，并且红色LED(ERR)闪烁


comaker-DCMAKE_TOOLCHAIN_FILE=/local/zye/opt/rb/cc/cc.cmake  -DCMAKE_INSTALL_PREFIX=/rbinband/
make
make install DESTDIR=./
    
//460   
1. 460 uart包要修复，发送非协议包也需要加4个字节不含a0a1的头部，防止mcu这边处理出错    
2. 以mcu send终止

//430第二代
1. 和460一样mcu通过i2c读取sensor传给bass写到文件，文件锁

状态0: 50%
状态0->状态1：当温度大于47C，风扇从50%调到65%；状态1->状态0：在状态1下，温度降低到40C时，风扇调回50%
状态1->状态2：当温度大于52C，风扇从65%调到80%；状态2->状态1：如果在状态2下，温度降低到45C时，风扇调回65%
状态2->状态3：当温度大于60C，风扇从80%调到100%；状态3->状态2：如果在状态3下，温度降低到53C时，风扇调回80%

```

#### 5. end

```c

比亚迪,中兴,腾讯,维谛技术,tcl,汇川

```

