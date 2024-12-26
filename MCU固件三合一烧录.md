## MCU固件三合一烧录



#### 一、合并文件

&emsp;&emsp;原来的mcu固件分为三个部分，通过keil烧写三次，现在通过一个bass上的mcu_three_to_one工具将这三分部合并为一个文件，再通过windows上的segger j-flash软件一次烧写即可。mcu_three_to_one使用方法如下：

```c
/opt/rb/bin/mcu_three_to_one upgrader.bin mainapp.bin backupapp.bin mcu.bin
```

以上命令将三个分散的mcu固件upgrader.bin，mainapp.bin，backupapp.bin合并成一个新的单独固件mcu.bin。

#### 二、烧录合并文件

&emsp;&emsp;在得到合并文件mcu.bin后接下来可以通过segger j-flash将它烧录到板子mcu里。首先将板子的swd通过jlink连接到电脑，然后电脑打开segger j-flash。如果安装了keil会附带安装好segger j-flash，如果没有安装好segger j-flash则从https://www.segger.com/downloads/jlink/下载安装。打开segger j-flash的界面如下，Data File中选择合并文件mcu.bin，Prog addr要设置成0x08000000，最后点击Program Device即可完成mcu烧录。

![](D:\bsp\docs\yeshen-md\pictures\17183474275247.png)

#### 注意

&emsp;&emsp;打开segger j-flash软件可能会出现找不到vcruntime140_1.dll,无法继续执行代码的问题，可以自行搜索解决，可以参考https://blog.csdn.net/qq_43082279/article/details/113787910。如果实在解决不了从我这里拷贝一个过去。