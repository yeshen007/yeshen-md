# MCU需求方案实现UG



## MCU功能需求概述

​		mcu在n4x产品中主要做一些辅助bass的需求，主要有mcu升级，mcu烧录，bass上下电控制，和一些小功能如烧写读取序列号、读mcu版本号、控制led灯等。



## MCU启动流程

​		在讲mcu的各个需求和方案实现前有必要先讲一下mcu的启动流程，因为板子上电最先跑的是mcu，了解mcu的启动流程是对后续的一些需求实现的必要条件，比如mcu的升级和烧录。
​		n430 mcu是gd32f303cbt6，n460 mcu是gde103cbt6，它们都是0x08000000-0x0801ffff的128kb flash, 0x20000000-0x20007fff的32k sram。板子电源上电后，mcu会从0x0000 0000获取栈顶值设置栈顶，从0x0000 0004获取起始代码地址，跳转到起始代码地址执行。n430和n460产品的mcu引导源是flash，0x0800 0000的flash起始地址会被映射到0x0000 0000，因此mcu上电或者复位时是从flash最开始的8个字节中取出栈顶和起始代码地址运行。
​		为了实现mcu在线升级和backup启动，mcu固件分成了upgrader、mainapp、backapp三部分。这几个部分在flash和sram中的布局如下：

```
      flash 						
++++++++++++++++++++ 0x0802 0000		//460: 0x0801 0000
|                  |
|                  |
|      newapp      |
|                  |
++++++++++++++++++++ 0x0801 8000		//460: 0x0800 c000
|                  |
|                  |
|     backapp      |
|                  |
++++++++++++++++++++ 0x0801 0000		//460: 0x0800 8000
|                  |
|                  |
|     mainapp      |
|                  |
++++++++++++++++++++ 0x0800 8000		//460: 0x0800 4000
|      flags       |
++++++++++++++++++++ 0x0800 6000		//460: 0x0800 3000
|                  |
|      upgrader    |
|                  |
++++++++++++++++++++ 0x0800 0000		//460: 0x0800 0000
```

```
       sram
++++++++++++++++++++ 0x2000 8000
|     reserved     |
++++++++++++++++++++ 0x2000 6000
|     backapp      |
++++++++++++++++++++ 0x2000 4000
|     mainapp      |
++++++++++++++++++++ 0x2000 2000
|     upgrader     |
++++++++++++++++++++ 0x2000 0000
```

​		其中固件运行的时候代码段在flash，数据段和栈在sram。固件每个部分都是以bin文件的格式存放在flash中，并且每个bin文件的前8个字节都是该部分的栈顶和起始代码地址。为了满足在线升级的需求，每次mcu上电或重启先要跑upgrader，因此把它放在0x0800 0000的flash起始地址。upgrader会根据flags中的值做相应的事情，具体如下：

1）首先读取flash中0x0800 6000处的flag，根据flag的值做相应的启动流程。
2）如果flag为0表示正常启动，upgrader会从flash mainapp区域的起始地址取出栈地址和起始代码地址，然后跳到起始代码地址运行程序。
3）如果flag为1表示需要升级app，upgrader会先设置flag为2，然后从flash newapp区域读取镜像到mainapp区覆盖原来的内容，再从flash mainapp区域的起始地址取出栈地址和起始代码地址，最后跳到起始代码地址运行程序。之所以先设置flag为2是保证新的mainapp启动不了重新上电也能启动backapp。
4）如果flag为2表示启动备份程序，upgrader会从flash backapp区域的起始地址取出栈地址和起始代码地址，然后跳到起始代码地址运行程序。

​		mainapp是mcu固件的**主控模块**，负责主要的业务逻辑。backup是mcu固件的**备份主控模块**。upgrader是mcu固件的**升级模块**，负责启动mainapp或者backupapp以及在接收升级固件后更新flash。mianapp和backupapp的流程基本一样，除了mianapp启动时将flash 0x0800 6000的升级flag设置为0，backupapp启动时将升级flag设置为2，其他流程一样。mainapp或者backupapp的主要流程如下：

1）初始化时钟，uart，i2c，watchdog，gpio，led等设备。
2）轮询检测power按键、reset按键和监听bass i2c命令。
3）检测到power按键则给bass下电或上电，检测到reset按键则复位bass。
4）监听到bass发送的i2c命令则做对应的事，比如在线升级，返回mcu版本号，喂狗，控制led等。

​		基于固件的flash和sram布局，需要对upgrader，mainapp，backupapp的keil工程做如下设置：

```
/* upgrader工程设置 */
1.打开upgrader工程的GD32303C_START.uvprojx
2.打开Options for Target 'GD32303C_START'
3.选择Target
4.设置IROM1:0x8000000 0x6000; IRAM1:0x20000000 0x2000 
5.选择User
6.Run#1打勾并填上：fromelf --bin --output "$L@L.bin" "#L"
    
/* mainapp工程设置 */ 
1.打开mainapp工程的GD32303C_START.uvprojx
2.打开Options for Target 'GD32303C_START'
3.选择Target
4.设置IROM1:0x8008000 0x8000; IRAM1:0x20002000 0x2000 
5.选择User
6.Run#1打勾并填上：fromelf --bin --output "$L@L.bin" "#L"
    
/* backupapp工程设置 */ 
1.打开backupapp工程的GD32303C_START.uvprojx
2.打开Options for Target 'GD32303C_START'
3.选择Target
4.设置IROM1:0x8010000 0x8000; IRAM1:0x20004000 0x2000 
5.选择User
6.Run#1打勾并填上：fromelf --bin --output "$L@L.bin" "#L"
```



## MCU-BASS通信协议

​		mcu的需求方案实现依赖mcu和bass之间的通信协议，bass作为master通过i2c通信协议告知mcu需要做什么，通过uart协议可以传输相对较大的数据量。比如在线升级bass先通过i2c告知mcu准备需要升级，然后通过uart发送升级固件给mcu。

### I2C协议

​		mcu和bass的i2c协议是mcu作为slave，bass作为master，一次i2c传输数据包是16字节，但不一定16个字节都是有效数据，有效数据是多少字节根据不同的命令而不同。具体格式如下，其中bass发mcu收的i2c数据包第一个字节代表命令，第二个字节根据cmd决定是否使用，而后续的14字节data根据cmd和subcmd决定是否使用；mcu发bass收的i2c数据包是16字节的数据。详细的命令在后文的需求方案实现中列出。

bass发mcu收i2c协议数据包格式：

```
|    1byte    |    1byte    |                  14bytes                     |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
|     cmd     |  subcmd(op) |                  data(op)                    |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++    
```

mcu发bass收i2c协议数据包格式：

```
|                                 16bytes                                  |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
|                                   data                                   |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++    
```

### UART协议

​		mcu和bass的uart协议用来传输大数据量数据，目前只用来在在线升级中传输mcu固件。uart协议数据包格式如下。每个包固定80字节，但是最大负载数据长度是64字节，实际负载长度由payloadlen指定，如果payloadlen小于64，那么payload剩下的64-payloadlen是什么都无所谓。flag如果为0表示这是bass发送的最后一包，否则表示除了这包还有其他数据没发完。mcu每接收到一包bass发过来的uart数据，都会通过uart返回"$READY"，这个是直接的裸数据不用mcu-bass uart协议的数据包格式。

mcu-bass uart协议数据包格式：

```
|   4bytes   |   4bytes   |       8bytes       |                   64bytes               |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
|    flag    | payloadlen |      reserved      |                  payload                |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

### 460 UART协议

​		460和bass的uart协议在兼容430的协议的基础上进行了扩展，因为460取消了bass和mcu的i2c通信，只能移植到uart中来。当flag为2时表示之前bass通过i2c传过来的命令。然后flag后面跟着的16字节是之前的16字节i2c包，如下。而数据包格式和上面一样。

```
|   4bytes   |              16bytes            |                  60bytes                |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
|     0x2    |            bass cmd             |                 reserved                |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```



## MCU控制BASS上下电

​		mcu可以有四种方式控制bass上下电，通过按下power按键控制bass上下电，按下reset按键控制bass上下电，以及bass通过输入poweroff和reboot而发送给mcu的i2c命令控制bass上下电。其中power按键和reset按键的短按和长按效果也有不同。

### power按键控制bass上下电

​		如果bass处于上电状态，短按power按键会控制bass下电；如果bass处于上电状态，短按power按键会控制bass上电。无论bass处于上电状态还是下电状态，长按power按键都会使bass下电。

短按power按键下电流程：

1）bass处于上电状态。
2）mcu检测到power按键按下超过100ms小于3s。
3）通过gpio告知bass即将下电，并通过gpio检测等待bass回应。
4）如果bass回应及时，那mcu继续等待bass的poweroff i2c命令，收到命令后por down，下电。
5）如果bass回应超时，那mcu先控制por down，然后下电。

短按power按键上电流程：

1）bass处于下电状态。
2）mcu检测到power按键按下超过100ms小于3s。
3）mcu先控制上电，然后等待电源稳定，再延时1s，最后por up。

长按power按键强制下电流程：

1）bass处于上电或下电状态。
2）mcu检测到power按键按下超过3s。
3）mcu控制por down，然后控制下电。

### reset按键控制bass上下电

​		如果bass处于上电状态，短按reset按键会控制bass先下电后上电；如果bass处于下电状态，短按reset按键无效。无论bass处于上电状态还是下电状态，长按reset按键都会使bass先下电后上电。

短按reset按键下电上电流程：

1）bass处于上电状态。
2）mcu检测到reset按键按下超过100ms小于3s。
3）mcu控制por down，下电，等待500ms，然后上电，等待电源稳定，等待500ms，最后por up。

长按reset按键强制下电上电流程：

1）bass处于上电或下电状态。
2）mcu检测到reset按键按下超过3s。
3）mcu控制por down，下电，等待500ms，然后上电，等待电源稳定，等待500ms，最后por up。

### poweroff命令控制bass上下电

​		bass用户输入poweroff会最终进入bl31，在bl31中发送i2c命令给mcu，然后mcu才控制por down和下电。这个poweroff i2c命令如下：

```
|     cmd     |    subcmd   |                noused 14 bytes               |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
|     0xaa    |     0x0f    |                      XXX                     |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```

​		其实bass只要发送0xaa0x0f两个字节的i2c数据给mcu即可，但是为了规范还是要遵守前文的i2c协议，最后再填充14字节的任意数据（为了更规范最好全零）发送一个完整的16字节数据包给mcu。

### reboot命令控制bass上下电

​		和poweroff命令一样，bass用户输入reboot也会进入bl31，从bl31中发送i2c命令给mcu，然后mcu控制por down，下电，等待500ms，然后上电，等待电源稳定，等待500ms，最后por up。reboot i2c命令如下：

```
|     cmd     |    subcmd   |                noused 14 bytes               |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
|     0xaa    |     0xeb    |                      XXX                     |
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
```



## MCU在线升级

​		mcu在线升级是mcu除了控制bass上下电外的另一个主要需求，在mcu和bass都在运行的时候升级mcu固件主控模块。如前文mcu启动流程中所描述，mcu固件分为**升级模块**，**主控模块**和**备份主控模块**三部分，对应的flash和sram中的布局分别是前文中的flahs和sram的upgrader，mainapp，backupapp区域。具体的升级流程如下：

1）mcu接收到bass发送的i2c命令cmd 0xcc，subcmd 0x17进入在线升级流程。
2）mcu等待bass串口发送0xa0a1表示准备开始。
3）mcu收到串口数据0xa0a1，然后擦除flash newapp区，为了将后面收到的固件主控模块烧写进去。
4）mcu通过串口发送"$READY"给bass表示已经准备好接收下一次uart数据。
5）mcu通过串口等待uart协议数据包。
6）mcu收到bass发送来的一个80字节的uart协议数据包，将这包数据写入flash newapp对应的地址，然后通过串口发送"$READY"给bass表示已经处理完该uart协议包数据。
7）如果已处理完最后一个uart协议数据包，则将flash 0x0800 6000地址的升级标志写1，将flash 0x0800 6400地址的辅助标志写0xac；如果还有mcu固件数据接收则跳到步骤 5）继续处理下一个uart协议数据包。
8）等待bass输入reboot命令。
9）mcu收到bass输入的reboot命令间接发送过来的i2c命令cmd 0xaa，subcmd 0xeb。然后读取flash 0x0800 6400地址的辅助标志，发现为0xac，然后复位mcu从升级模块开始运行，接下来的流程参考前文mcu启动流程。



## MCU其他辅助需求

### 1. 接通电源的led状态
​		盒子接通电源后，power、sys、err灯闪烁一下后，power灯常亮，sys、err熄灭。

### 2. sys led控制
​		mcu收到i2c命令cmd 0xff，subcmd 0x11，data[0] 0x1后点亮sys led；收到cmd 0xff，subcmd 0x11，data[0] 0x0后熄灭sys led；除了这两条命令外还有cmd 0xbb，subcmd 0x17和cmd 0xbb，subcmd 0x0f也分别会点亮和熄灭sys led，但是这两条命令还会影响看门狗，后面会讲。

### 3. err led控制
​		mcu收到i2c命令cmd 0xff，subcmd 0x22并且data[0]为1就点亮err led；收到cmd 0xff，subcmd 0x22并且data[0]为0就熄灭err led。

### 4. 版本号

​		mcu收到i2c命令cmd 0xdd后返回版本号给bass。

### 5. 温度监控

​		mcu收到i2c命令cmd 0xbb，subcmd 0xea，附带的数据data[2]开始的四字节表示板级tsensor读取的温度，data[6]开始的四字节表示芯片tsensor读取的温度。

### 6. 序列号烧录读取

​		mcu收到i2c命令cmd 0xff，subcmd 0x33进入序列号烧录或读取逻辑。如果data[0]为0表示读序列号，为1表示写序列号，其他为非法值。data[1]为0表示核心板序列号，为1表示接口板序列号，为2表示产品序列号。因此各子命令如下：

**读核心板序列号：**cmd 0xff，subcmd 0x33，data[0] 0，data[1] 0；从flash 0x08007000处读取12字节。
**读接口板序列号：**cmd 0xff，subcmd 0x33，data[0] 0，data[1] 1；从flash 0x0800700c处读取12字节。
**读产品序列号：**cmd 0xff，subcmd 0x33，data[0] 0，data[1] 2；从flash 0x08007018处读取16字节。
**读board id：**cmd 0xff，subcmd 0x33，data[0] 0，data[1] 3；从flash 0x08007028处读取16字节。
**烧写核心板序列号：**cmd 0xff，subcmd 0x33，data[0] 1，data[1] 0；将12字节序核心板列号烧入flash 0x08007000。
**烧写接口板序列号：**cmd 0xff，subcmd 0x33，data[0] 1，data[1] 1；将12字节接口板序列号烧入flash 0x0800700c。
**烧写产品序列号：**cmd 0xff，subcmd 0x33，data[0] 1，data[1] 2；将16字节产品序列号烧入flash 0x08007018。
**烧写board id：**cmd 0xff，subcmd 0x33，data[0] 1，data[1] 3；将4字节board id烧入flash 0x08007028。

### 7. 看门狗控制

​		mcu的看门狗控制按照mcu固件正常运行流程的时间顺序如下：

**1）mcu启动默认自动喂狗：**盒子上电时mcu启动时bass还没上电，此时mcu自己喂狗。
**2）mcu关闭自动喂狗：**bass上电启动后台进程会发送i2c命令cmd 0xbb，subcmd 0x17给mcu，mcu然后关闭自动喂狗。
**3）mcu定期收到i2c喂狗命令然后喂狗：**bass定期发送i2c命令cmd 0xbb，subcmd 0xfd给mcu，mcu收到后喂一次狗。
**4）mcu切换回自动喂狗：**通过按键、poweroff、reboot命令对bass关机或重启时会使mcu切换回自动喂狗。

### 8. mcu复位后自动上电bass的情况

​		有两种情况会使得mcu复位后自动上电bass。

**1）升级后reboot**
如前文所描述，在mcu接收完bass的mcu升级固件后会将flash 0x08006400地址的辅助标志写0xac，然后等待bass输入reboot，mcu收到reboot命令后读取flash 0x08006400发现是0xac，然后复位mcu，mcu复位后进入主控模块后进入监控bass命令主流程前会读取flash 0x08006400的辅助标志，发现是0xac然后就对bass上电，然后将flash 0x08006400的辅助标志设置为0xff。
**2）bass后台进程意外终止**
如前文所述，mcu是默认自动喂狗的，在bass后台进程起来后发送i2c命令cmd 0xbb，subcmd 0x17给mcu后才会关闭自动喂狗，然后bass定期发送i2c命令cmd 0xbb，subcmd 0xfd给mcu喂狗。如果没有其他措施bass后台进程意外终止后mcu看门狗肯定会超时复位mcu，此时bass不会自动上电。解决办法是在bass后台进程起来后先发送一条cmd 0xbb，subcmd 0xe7给mcu，mcu收到命令后会将flash 0x08006404的一个辅助标志设置为0xac(注意：和0x08006400不是一个标志)，如果之后走正常的bass关机或重启流程那么mcu会将该辅助标志清掉(写成0xff)，如果是bass后台意外终止导致mcu看门狗超时复位，那么在mcu进入主控模块后也会像上文一样在进入监控bass命令主流程前会读取flash 0x08006404的辅助标志，发现是0xac然后就对bass上电，然后将flash 0x08006404的辅助标志设置为0xff。



