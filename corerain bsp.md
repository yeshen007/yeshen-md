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
```



#### 2. 代码编译

```c
/* 配置环境变量 */
source  /opt/arm.env

/* 配置编译内核 */
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
```



#### 3. 仿真运行



#### 4. misc

```c
/* 查看文档 */
首先登陆共享文件系统
	进入系统
		hw_doc
			4_CAISA4(Bass)
				3_总体设计		//整体架构文档
				4_详细设计		//ip模块详细文档
				5_软件设计		//软件相关文档
					Bass_Address_Mapping	//物理地址映射
```

