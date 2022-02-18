## 一. Cortex-A9 MPCore双核通信方案总体流程

![](pictures\a9double.png)

<center>图1 双核通信方案流程</center>

​		Cortex-A9 MPCore双核通信方案如上图所示。上电时cpu0运行，而cpu1处于休眠状态。cpu0从复位异常向量地址处0x0开始运行，即厂家的固件，固件代码将preloader加载到on-chip ram上运行。preloader负责初始化ddr，然后将0核裸机程序cpu0.bin和1核裸机程序cpu1.bin分别从qspi加载到对应的链接地址，然后运行0核裸机程序，0核裸机程序负责唤醒1核然后进行双核通信。各模块主要工作：

### **1. preloader**

- 初始化ddr和qspi
- 加载0核裸机程序和1核裸机程序到ddr对应的连接地址中
- 启动0核裸机程序

  

### **2. 0核裸机程序**

- 设置异常向量表和异常处理函数

- 初始化L1cache

- 设置栈

- 设置页表

- 设置和使能中断控制器的分发器和CPU接口，注册中断处理函数

- 唤醒1核

- 使能SCU，SMP，MMU和L1 cache

- 等待1核的交互通信

  

### **3. 1核裸机程序**

- 设置异常向量表和异常处理函数
- 初始化L1cache
- 设置栈
- 设置页表
- 设置和使能中断控制器的CPU接口，注册中断处理函数
- 使能SMP，MMU和L1 cache
- 设置和使能L2 cache
- 发起和0核的交互通信



## 二. 内存分配

### **1. 物理内存划分**

![](pictures\a9phy.png)

<center>图2 物理内存分布</center>

​		上图是双裸机通信方案的物理内存分布。具体分配如下：

- 片上静态内存SRAM：0xFFFF0000~0xFFFFFFFF，大小64KB

- SOC外设寄存器空间：0xFC000000~0xFFFEFFFF，大小63MB+960KB

- 共享内存区域：0x20000000~0x20FFFFFF，大小16MB

- 1核裸机代码区域：0x1F000000~0x1FFFFFFF，大小16MB

- 0核裸机代码区域：0x1E000000~0x1EFFFFFF，大小16MB

  

### **2. 0核裸机虚拟地址映射**

![](pictures\cpu0vir.png)

<center>图3 0核裸机虚拟地址映射</center>

### **3. 1核裸机虚拟地址映射**

![](pictures\core1vir.png)

<center>图4 1核裸机虚拟地址映射</center>