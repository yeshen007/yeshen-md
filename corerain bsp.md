## <center>corerain bsp</center>

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
addr: 192.168.11.247
account: zye
passwd: letmein
登录方式：通过mobaxterm或类似工具ssh连接

/* gitlab */
addr: http://192.168.11.225/
account: zye
passwd: letmein
登录方式：直接将addr输入浏览器

/* 共享文件系统 */
addr: http://192.168.11.36:8100/DocSystem/web/index.html
account: zye
passwd: letmein 
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
```

#### 2. 代码编译

```c
/* 配置环境变量 */
source  /opt/arm.env

/* 配置编译内核和设备树,得到arch/arm64/boot/Image和arch/arm64/boot/dts/corerain/bass-haps.dtb */
cd /home/zye/kernel
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- bass_linux_mini_defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- menuconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- -j16
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- distclean

/* 编译spacc驱动模块和测试应用 */
首先编译内核
cd /home/zye/spacc
make -j16
make clean

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
config_haps /home/syshaps/workspace/storage/yqwu/haps_projects/confpro_ws_20230325_4boot_20230412/designs

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
```

